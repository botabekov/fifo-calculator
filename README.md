<!DOCTYPE html>
<html lang="ru">
<head>
  <meta charset="UTF-8" />
  <meta name="viewport" content="width=device-width, initial-scale=1.0" />
  <title>Калькулятор FIFO — стабильная и быстрая версия</title>
  <style>
    :root{--muted:#667085;--line:#e6eaf0;--bg:#f4f6f8;--ink:#222;--blue:#0066ff;--blue2:#0050d1;--danger:#ff5a5a;--danger2:#e60000}
    html,body{height:100%}
    body{font-family:'Times New Roman',serif;background:var(--bg);color:var(--ink);margin:0}
    .container{max-width:760px;margin:30px auto;background:#fff;border-radius:14px;box-shadow:0 4px 12px rgba(0,0,0,.08);padding:20px}
    h2{text-align:center;font-size:18px;margin:0 0 14px}
    .instruction{background:#fef9e7;border:1px solid #f1d76e;border-radius:10px;padding:10px;font-size:13px;margin-bottom:12px;color:#665c00}
    table{width:100%;border-collapse:collapse;font-size:13px;margin-bottom:10px;table-layout:fixed}
    th,td{border-bottom:1px solid var(--line);padding:8px;text-align:center;word-wrap:break-word}
    th:first-child,td:first-child{width:40px}
    th{background:#f9fafb;color:#333;font-weight:600}
    .purchase{background:#e9f2ff;color:#0043ce;font-weight:600}
    .sale{background:#eaffea;color:#006b00;font-weight:600}
    .transfer{background:#fff6e6;color:#9c6400;font-weight:600}
    .btn{background:var(--blue);color:#fff;border:none;padding:10px;border-radius:8px;cursor:pointer;font-size:14px;transition:background .2s ease}
    .btn:hover{background:var(--blue2)}
    .btn-del{background:var(--danger);color:#fff;border:none;border-radius:6px;padding:4px 8px;font-size:12px;cursor:pointer;transition:background .2s ease}
    .btn-del:hover{background:var(--danger2)}
    input,select{width:100%;padding:8px;border-radius:8px;border:1px solid #ccd3db;margin-bottom:8px;font-size:14px}
    .actions{display:flex;gap:8px;margin-top:12px}
    .actions .btn{flex:1}
    .muted{color:var(--muted);font-size:12px}
    .info-line{font-size:13px;color:#333;text-align:left;margin:4px 0;padding-left:8px;font-style:italic}
    #resultsBox{display:none;background:#f9fafb;border:1px solid #ccd3db;border-radius:10px;padding:12px;margin-top:12px;font-size:13px;white-space:pre-line;line-height:1.6}
  </style>
</head>
<body>
  <div class="container" id="calcContainer">
    <h2>Калькулятор FIFO</h2>
    <div class="instruction">Добавляйте операции в хронологическом порядке – от первой к последующей.</div>

    <table id="operationsTable">
      <thead>
        <tr>
          <th>№</th>
          <th>Тип</th>
          <th>Кол-во</th>
          <th>Цена</th>
          <th></th>
        </tr>
      </thead>
      <tbody><tr><td colspan="5" class="muted">Нет операций</td></tr></tbody>
    </table>

    <div id="inputBlock">
      <select id="type">
        <option value="Покупка (Приход)">Покупка (Приход)</option>
        <option value="Продажа (Уход)">Продажа (Уход)</option>
        <option value="Списание ЦБ (Уход)">Списание ЦБ (Уход)</option>
      </select>
      <input type="number" id="qty" placeholder="Количество (целое)" min="1" />
      <input type="number" id="price" placeholder="Цена (точка или запятая)" step="0.01" min="0" />
      <div class="actions">
        <button class="btn" id="addBtn">Добавить операцию</button>
      </div>
      <div id="warn" class="muted" aria-live="polite"></div>
    </div>

    <button class="btn" style="background:#00897b;width:100%;margin-top:10px" id="toggleResultsBtn">Показать расчёты</button>
    <div id="resultsBox"></div>

    <div class="actions">
      <button class="btn" style="background:#e5e7eb;color:#111827" id="resetBtn">Очистить всё</button>
      <button class="btn" style="background:#009688" id="exportBtn">Выгрузить</button>
    </div>
  </div>

  <script src="https://html2canvas.hertzen.com/dist/html2canvas.min.js"></script>
  <script>
  (function(){
    const $=(sel,root=document)=>root.querySelector(sel);
    const round=(n,d=2)=>Math.round((n+Number.EPSILON)*10**d)/10**d;

    const els={};
    const bindEls=()=>{
      els.tableBody=$('#operationsTable tbody');
      els.type=$('#type');
      els.qty=$('#qty');
      els.price=$('#price');
      els.add=$('#addBtn');
      els.warn=$('#warn');
      els.toggleResults=$('#toggleResultsBtn');
      els.results=$('#resultsBox');
      els.reset=$('#resetBtn');
      els.export=$('#exportBtn');
    };

    const state={ops:[],lots:[],totalQty:0,avgPrice:0,details:[]};

    const setWarn=(msg='')=>{if(els.warn)els.warn.textContent=msg;};

    const renderRows=()=>{
      if(!els.tableBody)return;
      const body=els.tableBody;
      body.innerHTML=state.ops.length?'':'<tr><td colspan="5" class="muted">Нет операций</td></tr>';
      const frag=document.createDocumentFragment();
      state.ops.forEach((op,i)=>{
        const tr=document.createElement('tr');
        tr.className=op.type.includes('Покупка')?'purchase':(op.type.includes('Продажа')?'sale':'transfer');
        tr.innerHTML=`<td>${i+1}</td><td>${op.type}</td><td>${op.qty}</td><td>${op.price??'-'}</td><td><button class='btn-del' data-i='${i}'>×</button></td>`;
        frag.appendChild(tr);
        const info=document.createElement('tr');
        if(op.type==='Покупка (Приход)'||op.type==='Продажа (Уход)'){
          info.innerHTML=`<td colspan='5' class='info-line'>📊 Остаток после операции: ${op.remain} шт. по средневзвешенной цене ${op.avgRemain}</td>`;
        }else{
          info.innerHTML=`<td colspan='5' class='info-line'>📊 Списано ЦБ: ${op.qty} шт. по средневзвешенной цене: ${op.avgPrice}<br>📊 Остаток после операции: ${op.remain} шт. по средневзвешенной цене ${op.avgRemain}</td>`;
        }
        frag.appendChild(info);
      });
      body.appendChild(frag);
    };

    const recalcTotals=()=>{
      state.totalQty=state.lots.reduce((s,l)=>s+l.qty,0);
      const totalVal=state.lots.reduce((s,l)=>s+l.qty*l.price,0);
      state.avgPrice=state.totalQty?round(totalVal/state.totalQty,2):0;
    };

    const compute=()=>{
      state.lots=[]; state.details=[];
      for(const [index, op] of state.ops.entries()){
        let num=index+1;
        let detailLine=`${num} - Операция: ${op.type}`;
        let lotDetails='';
        if(op.type==='Покупка (Приход)'){
          state.lots.push({qty:op.qty,price:op.price});
          lotDetails+=`: ${op.qty} шт. по цене ${op.price}.`;
        }else if(op.type==='Продажа (Уход)'){
          let remain=op.qty, soldFrom=[];
          while(remain>0&&state.lots.length){
            const lot=state.lots[0];
            const used=Math.min(remain,lot.qty);
            soldFrom.push(` - Продано ${used} шт. по цене ${lot.price}.`);
            lot.qty-=used; remain-=used; if(lot.qty===0){state.lots.shift();}
          }
          lotDetails+=`: ${op.qty} шт.\n${soldFrom.join('\n')}`;
        }else if(op.type==='Списание ЦБ (Уход)'){
          let remain=op.qty,total=0,used=0,writeOffInfo=[];
          for(const lot of state.lots){if(remain<=0)break;const take=Math.min(remain,lot.qty);writeOffInfo.push(` - Списано ${take} шт. по ${lot.price}.`);total+=take*lot.price;used+=take;lot.qty-=take;remain-=take;}
          op.avgPrice=used?round(total/used,2):0;
          lotDetails+=`: ${op.qty} шт.\n${writeOffInfo.join('\n')}\n - Средневзвешенная цена списания: ${op.avgPrice}.`;
        }
        recalcTotals();
        op.remain=state.totalQty; op.avgRemain=state.avgPrice;
        detailLine+=lotDetails+`\n - Остаток после операции: ${op.remain} шт. по средневзвешенной цене: ${op.avgRemain}.`;
        state.details.push(detailLine);
      }
      renderRows();
    };

    const onTypeChange=()=>{if(els.price)els.price.style.display=(els.type.value==='Списание ЦБ (Уход)')?'none':'block';};

    const getAvailable=()=>{
      let available=0;
      for(const op of state.ops){
        if(op.type==='Покупка (Приход)')available+=op.qty;
        else if(op.type==='Продажа (Уход)'||op.type==='Списание ЦБ (Уход)')available-=op.qty;
      }
      return available;
    };

    const addOperation=()=>{
      const qty=Number(els.qty&&els.qty.value);
      const price=Number(els.price&&els.price.value);
      const type=els.type?els.type.value:'Покупка (Приход)';
      if(!qty||qty<=0)return setWarn('Введите корректное количество (>0).');
      if(type!=='Списание ЦБ (Уход)'&&(!price||price<=0))return setWarn('Введите корректную цену (>0).');

      const currentAvail=getAvailable();
      const availAfter=type==='Покупка (Приход)'?(currentAvail+qty):(currentAvail-qty);
      if(availAfter<0)return setWarn(`Недостаточно бумаг: доступно ${currentAvail} шт., пытаетесь ${type.includes('Списание')?'списать':'продать'} ${qty} шт.`);
      setWarn('');

      state.ops.push({type,qty,price:type==='Списание ЦБ (Уход)'?null:price,avgPrice:0,remain:0,avgRemain:0});
      compute();
      if(els.qty)els.qty.value=''; if(els.price)els.price.value='';
    };

    const onTableClick=e=>{
      const btn=e.target.closest('.btn-del'); if(!btn)return;
      const i=Number(btn.dataset.i); if(Number.isNaN(i))return;
      state.ops.splice(i,1); compute();
    };

    const toggleResults=()=>{
      if(!els.results)return;
      if(els.results.style.display==='block')els.results.style.display='none';
      else{
        els.results.style.display='block';
        els.results.textContent=state.details.length?state.details.join('\n\n'):'Нет расчётов';
      }
    };

    const resetAll=()=>{state.ops.length=0;state.lots.length=0;state.details.length=0;renderRows();setWarn('');els.results.textContent='';};

    const exportPng=async()=>{
      const node=$('#calcContainer');if(!node)return;
      const canvas=await html2canvas(node);const a=document.createElement('a');a.download='fifo_result.png';a.href=canvas.toDataURL('image/png');a.click();
    };

    document.addEventListener('DOMContentLoaded',()=>{
      bindEls();
      els.type&&els.type.addEventListener('change',onTypeChange);
      els.add&&els.add.addEventListener('click',addOperation);
      $('#operationsTable')?.addEventListener('click',onTableClick);
      els.toggleResults&&els.toggleResults.addEventListener('click',toggleResults);
      els.reset&&els.reset.addEventListener('click',resetAll);
      els.export&&els.export.addEventListener('click',exportPng);
      onTypeChange();
      renderRows();
    });
  })();
  </script>
</body>
</html>
