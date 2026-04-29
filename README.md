# Work-Report-Helper
https://www.bilibili.com/read/cv48339625
<!DOCTYPE html>
<html lang="zh-CN">
<head>
    <meta charset="UTF-8">
    <title>Invoice Auto-Aggregator v7</title>
    <style>
        :root { --urgent: #ff4d4f; --primary: #1a73e8; --success: #52c41a; --warning: #faad14; }
        body { font-family: "Segoe UI", system-ui; background: #f0f2f5; padding: 20px; color: #333; }
        .container { max-width: 1100px; margin: auto; }
        
        /* 预警通知 */
        .alert-item { background: #fff2f0; border: 1px solid #ffccc7; color: #a8071a; padding: 12px 20px; border-radius: 8px; margin-bottom: 8px; display: flex; justify-content: space-between; align-items: center; border-left: 5px solid #ff4d4f; }
        .alert-item.is-muted { background: #f5f5f5; border-color: #d9d9d9; color: #8c8c8c; opacity: 0.6; border-left-color: #bfbfbf; }

        /* 输入面板 */
        .panel { background: white; padding: 20px; border-radius: 12px; box-shadow: 0 4px 12px rgba(0,0,0,0.08); margin-bottom: 20px; }
        .input-grid { display: grid; grid-template-columns: repeat(3, 1fr); gap: 10px; }
        h3 { margin-top: 0; color: var(--primary); font-size: 16px; grid-column: span 3; border-bottom: 1px solid #eee; padding-bottom: 8px; }
        input, select { padding: 8px; border: 1px solid #d9d9d9; border-radius: 4px; font-size: 13px; }
        .btn { padding: 10px; border: none; border-radius: 4px; cursor: pointer; font-weight: bold; }

        /* 内部文件卡片 */
        .doc-group { background: white; border-radius: 10px; margin-bottom: 20px; box-shadow: 0 2px 8px rgba(0,0,0,0.06); border-top: 4px solid #999; overflow: hidden; }
        .doc-header { padding: 15px 20px; background: #fafafa; display: grid; grid-template-columns: 1fr auto; align-items: center; border-bottom: 1px solid #eee; }
        .doc-info { display: flex; flex-direction: column; gap: 5px; }
        .mismatch-tip { color: var(--warning); font-weight: bold; cursor: help; margin-left: 10px; padding: 2px 6px; background: #fffbe6; border-radius: 4px; border: 1px solid #ffe58f; }
        
        /* 发票条目 */
        .inv-item { padding: 12px 20px; display: grid; grid-template-columns: 1fr 180px 150px 40px; gap: 15px; border-bottom: 1px solid #f0f0f0; align-items: center; font-size: 13px; }
        .status-col { display: flex; flex-direction: column; gap: 4px; font-size: 11px; }
        .amt-val { font-family: 'Consolas', monospace; font-weight: bold; text-align: right; }
        .comment-box { width: 100%; margin-top: 5px; height: 32px; font-size: 11px; background: #fffbe6; border: 1px solid #eee; padding: 4px; border-radius: 4px; }
        .completed { opacity: 0.5; filter: grayscale(1); }
    </style>
</head>
<body>

<div class="container">
    <div id="alert-zone"></div>

    <div class="panel">
        <div class="input-grid">
            <h3>📂 创建空内部文件 (New Internal Batch)</h3>
            <input type="text" id="docId" placeholder="内部文件编号 (如: BATCH-0429-01)">
            <select id="docType">
                <option value="付款">预期类型: 付款 (Payment)</option>
                <option value="收款">预期类型: 收款 (Receipt)</option>
            </select>
            <input type="date" id="docDueDate">
            <button class="btn" style="grid-column: span 3; background: var(--primary); color: white;" onclick="createDoc()">创建容器</button>
        </div>
    </div>

    <div id="list"></div>
</div>

<script>
    let documents = JSON.parse(localStorage.getItem('vibe_docs_v7')) || [];
    let threshold = localStorage.getItem('vibe_threshold') || 10000;
    let mutedAlerts = JSON.parse(localStorage.getItem('vibe_muted_alerts')) || [];

    function createDoc() {
        const id = document.getElementById('docId').value;
        const date = document.getElementById('docDueDate').value;
        if(!id || !date) return alert("请完整填写编号和截止日期");

        documents.push({
            id: Date.now(),
            docId: id,
            type: document.getElementById('docType').value,
            dueDate: date,
            invoices: []
        });
        saveAndRender();
    }

    function addInvoice(docId) {
        const invA = prompt("发票前缀 (A):");
        const invB = prompt("发票后缀 (B):");
        const amt = parseFloat(prompt("金额 (支出填正, 冲销/收款填负):") || 0);
        const vendor = prompt("供应商 (Vendor):");
        if(!invA) return;

        const idx = documents.findIndex(d => d.id === docId);
        documents[idx].invoices.push({
            id: Date.now(),
            invNo: invA + "-" + invB,
            vendor: vendor,
            amount: amt,
            comment: '',
            step1: false, step2: false, step3: false
        });
        saveAndRender();
    }

    function toggleMute(date) {
        mutedAlerts.includes(date) ? mutedAlerts = mutedAlerts.filter(d=>d!==date) : mutedAlerts.push(date);
        localStorage.setItem('vibe_muted_alerts', JSON.stringify(mutedAlerts));
        saveAndRender();
    }

    function saveAndRender() {
        documents.sort((a,b) => {
            const aD = a.invoices.length > 0 && a.invoices.every(i => i.step1 && i.step2 && i.step3);
            const bD = b.invoices.length > 0 && b.invoices.every(i => i.step1 && i.step2 && i.step3);
            if(aD !== bD) return aD ? 1 : -1;
            return new Date(a.dueDate) - new Date(b.dueDate);
        });
        localStorage.setItem('vibe_docs_v7', JSON.stringify(documents));
        
        // 预警逻辑：基于实时累加的付款批次总额
        const alertZone = document.getElementById('alert-zone');
        alertZone.innerHTML = '';
        const dailyPay = {};
        documents.forEach(d => {
            const isDone = d.invoices.length > 0 && d.invoices.every(i => i.step1 && i.step2 && i.step3);
            if(!isDone && d.type === '付款') {
                const currentSum = d.invoices.reduce((s, i) => s + i.amount, 0);
                dailyPay[d.dueDate] = (dailyPay[d.dueDate] || 0) + currentSum;
            }
        });
        Object.keys(dailyPay).filter(date => dailyPay[date] > threshold).forEach(date => {
            const muted = mutedAlerts.includes(date);
            alertZone.innerHTML += `<div class="alert-item ${muted?'is-muted':''}">
                <span><input type="checkbox" ${muted?'checked':''} onclick="toggleMute('${date}')"> <b>${date}</b> 付款总额超标: <b>$${dailyPay[date].toLocaleString()}</b></span>
            </div>`;
        });

        const list = document.getElementById('list');
        list.innerHTML = '';
        documents.forEach(doc => {
            const currentSum = doc.invoices.reduce((s, i) => s + i.amount, 0);
            const isDone = doc.invoices.length > 0 && doc.invoices.every(i => i.step1 && i.step2 && i.step3);
            
            // 校验逻辑：累加值与预期类型是否冲突
            // 付款预期下，累加值不应为负；收款预期下，累加值不应为正
            const typeMismatch = (doc.type === '付款' && currentSum < 0) || (doc.type === '收款' && currentSum > 0);

            list.innerHTML += `
                <div class="doc-group ${isDone?'completed':''}">
                    <div class="doc-header">
                        <div class="doc-info">
                            <span class="doc-title">${doc.docId} <small>(${doc.type})</small></span>
                            <span style="font-size:12px; color:#666">
                                实时合计 (Net Sum): <b style="color:${currentSum >= 0 ? '#cf1322' : '#389e0d'}">$${currentSum.toLocaleString()}</b>
                                ${typeMismatch ? `<span class="mismatch-tip" title="警告：净额方向与设定类型不符">❓ Type Mismatch</span>` : ''}
                            </span>
                        </div>
                        <div>
                            <button class="btn" style="background:#f0f0f0" onclick="addInvoice(${doc.id})">+ Add Invoice</button>
                            <button class="btn" style="color:red" onclick="if(confirm('Delete Batch?')){documents=documents.filter(d=>d.id!==${doc.id});saveAndRender()}">✕</button>
                        </div>
                    </div>
                    ${doc.invoices.map(inv => `
                        <div class="inv-item">
                            <div>
                                <b>${inv.invNo}</b> <span style="font-size:11px; color:var(--primary)">[${inv.vendor || 'N/A'}]</span>
                                <textarea class="comment-box" onchange="const dIdx=documents.findIndex(d=>d.id===${doc.id}); const iIdx=documents[dIdx].invoices.findIndex(i=>i.id===${inv.id}); documents[dIdx].invoices[iIdx].comment=this.value; localStorage.setItem('vibe_docs_v7', JSON.stringify(documents));" placeholder="Comment...">${inv.comment}</textarea>
                            </div>
                            <div class="status-col">
                                <label><input type="checkbox" ${inv.step1?'checked':''} onclick="const dIdx=documents.findIndex(d=>d.id===${doc.id}); const iIdx=documents[dIdx].invoices.findIndex(i=>i.id===${inv.id}); documents[dIdx].invoices[iIdx].step1=!documents[dIdx].invoices[iIdx].step1; saveAndRender();"> Created</label>
                                <label><input type="checkbox" ${inv.step2?'checked':''} onclick="const dIdx=documents.findIndex(d=>d.id===${doc.id}); const iIdx=documents[dIdx].invoices.findIndex(i=>i.id===${inv.id}); documents[dIdx].invoices[iIdx].step2=!documents[dIdx].invoices[iIdx].step2; saveAndRender();"> Sent</label>
                                <label><input type="checkbox" ${inv.step3?'checked':''} onclick="const dIdx=documents.findIndex(d=>d.id===${doc.id}); const iIdx=documents[dIdx].invoices.findIndex(i=>i.id===${inv.id}); documents[dIdx].invoices[iIdx].step3=!documents[dIdx].invoices[iIdx].step3; saveAndRender();"> Approved</label>
                            </div>
                            <div class="amt-val" style="color:${inv.amount >= 0 ? '#cf1322' : '#389e0d'}">$${inv.amount.toLocaleString()}</div>
                            <button class="btn" style="background:none; color:#ccc" onclick="const dIdx=documents.findIndex(d=>d.id===${doc.id}); documents[dIdx].invoices=documents[dIdx].invoices.filter(i=>i.id!==${inv.id}); saveAndRender();">✕</button>
                        </div>
                    `).join('')}
                </div>
            `;
        });
    }
    saveAndRender();
</script>
</body>
</html>
