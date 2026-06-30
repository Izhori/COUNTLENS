<!DOCTYPE html>
<html>
<head>
<meta charset="utf-8">
<meta name="viewport" content="width=device-width, initial-scale=1">
<script src="./support.js"></script>
</head>
<body>
<x-dc>
<helmet>
  <script src="https://cdn.jsdelivr.net/npm/xlsx-js-style@1.2.0/dist/xlsx.bundle.js"></script>
  <script src="https://cdn.jsdelivr.net/npm/exifr@7.1.3/dist/full.umd.js"></script>
  <style>
    @keyframes spin { to { transform: rotate(360deg); } }
    @keyframes barpulse { 0%,100%{ opacity:.35 } 50%{ opacity:1 } }
    @keyframes pop { 0%{ transform:scale(.5); opacity:0 } 100%{ transform:scale(1); opacity:1 } }
    @keyframes fadeup { from{ transform:translateY(10px); opacity:0 } to{ transform:translateY(0); opacity:1 } }
  </style>
</helmet>
<x-import component-from-global-scope="IOSDevice" from="./ios-frame.jsx" dark="{{ dark }}" width="402" height="874" hint-size="402px,874px">{{ app }}</x-import>
</x-dc>
<script type="text/x-dc" data-dc-script data-props="{&quot;$preview&quot;:{&quot;width&quot;:&quot;402px&quot;,&quot;height&quot;:&quot;874px&quot;},&quot;variant&quot;:{&quot;editor&quot;:&quot;enum&quot;,&quot;options&quot;:[&quot;step&quot;,&quot;single&quot;],&quot;default&quot;:&quot;step&quot;,&quot;tsType&quot;:&quot;'step'|'single'&quot;},&quot;dark&quot;:{&quot;editor&quot;:&quot;boolean&quot;,&quot;default&quot;:false,&quot;tsType&quot;:&quot;boolean&quot;},&quot;accent&quot;:{&quot;editor&quot;:&quot;color&quot;,&quot;default&quot;:&quot;#4F46E5&quot;,&quot;tsType&quot;:&quot;string&quot;}}">
class Component extends DCLogic {
  state = {
    screen: 'capture',
    imgSrc: null,
    imgAspect: 0.66,
    items: [],
    selected: [],
    prices: {},
    records: [],
    addMode: null,
    place: '',
    geoNote: '',
    memo: '',
    capturedAt: '',
    error: '',
    busy: false,
  };

  palette = ['#5B8DEF','#F2994A','#27AE60','#EB5757','#9B51E0','#2D9CDB','#F2C94C','#EB5FA0','#56CCF2','#6FCF97','#F2785C','#7C5CFF'];

  get variant(){ return this.props.variant || 'step'; }
  get dark(){ return this.props.dark ?? (this.variant === 'single'); }
  get accent(){ return this.props.accent || (this.dark ? '#34D399' : '#4F46E5'); }

  theme(){
    return this.dark
      ? { bg:'#0B0B0E', panel:'#16161B', card:'#1E1E25', fg:'#F4F4F7', sub:'#9A9AA8', line:'rgba(255,255,255,0.09)', accent:this.accent, accentFg:'#06210F', shadow:'0 8px 30px rgba(0,0,0,0.5)' }
      : { bg:'#FFFFFF', panel:'#F6F6F8', card:'#FFFFFF', fg:'#0D0D12', sub:'#73737E', line:'rgba(0,0,0,0.08)', accent:this.accent, accentFg:'#FFFFFF', shadow:'0 8px 30px rgba(0,0,0,0.10)' };
  }

  labels(){ return [...new Set(this.state.items.map(i => i.label))]; }
  colorFor(label){ const ls = this.labels(); const i = ls.indexOf(label); return this.palette[(i<0?0:i) % this.palette.length]; }
  countOf(label){ return this.state.items.filter(i => i.label === label).length; }
  selectedTotal(){ return this.state.items.filter(i => this.state.selected.includes(i.label)).length; }
  unitPrice(label){ const p = this.state.prices[label]; return (p == null || isNaN(p)) ? 0 : Number(p); }
  lineTotal(label){ return this.unitPrice(label) * this.countOf(label); }
  grandAmount(){ return this.state.selected.reduce((a, l) => a + this.lineTotal(l), 0); }
  yen(n){ return '¥' + Number(n || 0).toLocaleString('ja-JP'); }

  nowStr(){ const d=new Date(); const p=n=>String(n).padStart(2,'0'); return `${d.getFullYear()}/${p(d.getMonth()+1)}/${p(d.getDate())} ${p(d.getHours())}:${p(d.getMinutes())}`; }

  parseJSON(t){
    let s = String(t).trim();
    s = s.replace(/```json/gi,'').replace(/```/g,'').trim();
    const a = s.indexOf('{'), b = s.lastIndexOf('}');
    if(a>=0 && b>a) s = s.slice(a,b+1);
    return JSON.parse(s);
  }

  // 出力JSONが途中で切れていても、完成済みの検出枠を救い出す
  parseDetection(text){
    try{
      const j = this.parseJSON(text);
      if(j && Array.isArray(j.objects)) return { objects: j.objects, prices: (j.prices && typeof j.prices === 'object') ? j.prices : {} };
    }catch(_){}
    const objects = [];
    const re = /"label"\s*:\s*"([^"]+)"\s*,\s*"box"\s*:\s*\[\s*(-?\d+(?:\.\d+)?)\s*,\s*(-?\d+(?:\.\d+)?)\s*,\s*(-?\d+(?:\.\d+)?)\s*,\s*(-?\d+(?:\.\d+)?)\s*\]/g;
    let m;
    while((m = re.exec(text))){ objects.push({ label: m[1], box: [Number(m[2]), Number(m[3]), Number(m[4]), Number(m[5])] }); }
    const prices = {};
    const pi = text.indexOf('"prices"');
    if(pi >= 0){ const seg = text.slice(pi + 8); const pe = /"([^"]+)"\s*:\s*(\d+)/g; let pm; while((pm = pe.exec(seg))){ prices[pm[1]] = Number(pm[2]); } }
    return { objects, prices };
  }

  onFile = (e) => {
    const f = e.target.files && e.target.files[0];
    if(!f) return;
    this.extractGeo(f);
    const r = new FileReader();
    r.onload = () => this.loadSrc(r.result);
    r.readAsDataURL(f);
    e.target.value = '';
  };

  async extractGeo(file){
    this.setState({ place:'', geoNote:'' });
    try{
      if(!window.exifr) return;
      const g = await window.exifr.gps(file);
      if(!g || g.latitude == null || g.longitude == null) return;
      const coords = `${g.latitude.toFixed(5)}, ${g.longitude.toFixed(5)}`;
      this.setState({ place: coords, geoNote:'写真のGPS情報から自動取得（座標）' });
      try{
        const res = await fetch(`https://nominatim.openstreetmap.org/reverse?format=json&zoom=18&accept-language=ja&lat=${g.latitude}&lon=${g.longitude}`);
        if(res.ok){
          const j = await res.json();
          if(j && j.display_name) this.setState({ place: j.display_name, geoNote:'写真のGPS情報から自動取得（推定住所）' });
        }
      }catch(_){}
    }catch(_){}
  }

  fitDataUrl(img){
    const LIMIT = 245000;
    const dims = [1400, 1200, 1024, 880, 760, 640, 520];
    const quals = [0.82, 0.7, 0.6, 0.5, 0.42];
    let last = null;
    for(const d of dims){
      const sc = Math.min(1, d / Math.max(img.width, img.height));
      const cw = Math.max(1, Math.round(img.width*sc)), ch = Math.max(1, Math.round(img.height*sc));
      const cv = document.createElement('canvas'); cv.width = cw; cv.height = ch;
      cv.getContext('2d').drawImage(img, 0, 0, cw, ch);
      for(const q of quals){
        const url = cv.toDataURL('image/jpeg', q);
        const b64len = url.length - url.indexOf(',') - 1;
        last = { url, w:cw, h:ch };
        if(b64len <= LIMIT) return last;
      }
    }
    return last;
  }
  loadSrc(src){
    const img = new Image();
    img.onload = () => { const fit = this.fitDataUrl(img); this.setState({ imgSrc: fit.url, imgAspect: fit.h/fit.w || 0.66 }); this.detect(fit.url); };
    img.onerror = () => { this.setState({ screen:'capture', error:'画像を読み込めませんでした。別のファイルをお試しください。' }); };
    img.src = src;
  }

  drawSample = () => {
    const c = document.createElement('canvas'); c.width = 960; c.height = 640;
    const x = c.getContext('2d');
    const g = x.createLinearGradient(0,0,0,640); g.addColorStop(0,'#ECE5D8'); g.addColorStop(1,'#DED4C2');
    x.fillStyle = g; x.fillRect(0,0,960,640);
    x.strokeStyle = 'rgba(120,90,50,0.18)'; x.lineWidth = 14;
    for(let i=1;i<3;i++){ x.beginPath(); x.moveTo(0,i*213); x.lineTo(960,i*213); x.stroke(); }
    const fruit = (cx,cy,r,col,dark,leaf) => {
      const grd = x.createRadialGradient(cx-r*0.35,cy-r*0.35,r*0.2,cx,cy,r);
      grd.addColorStop(0, col); grd.addColorStop(1, dark);
      x.fillStyle = grd; x.beginPath(); x.arc(cx,cy,r,0,7); x.fill();
      x.fillStyle = 'rgba(255,255,255,0.35)'; x.beginPath(); x.ellipse(cx-r*0.35,cy-r*0.4,r*0.22,r*0.13,-0.6,0,7); x.fill();
      x.strokeStyle = '#6B4A2B'; x.lineWidth = r*0.12; x.beginPath(); x.moveTo(cx,cy-r); x.lineTo(cx+r*0.1,cy-r*1.3); x.stroke();
      if(leaf){ x.fillStyle = '#4E8A3A'; x.beginPath(); x.ellipse(cx+r*0.35,cy-r*1.15,r*0.32,r*0.16,0.5,0,7); x.fill(); }
    };
    const rows = [
      { col:'#E0463B', dark:'#9E261F', leaf:true, n:7 },
      { col:'#F0A23A', dark:'#C06A12', leaf:false, n:6 },
      { col:'#8CBF4B', dark:'#5C8A28', leaf:true, n:5 },
    ];
    rows.forEach((row, ri) => {
      for(let i=0;i<row.n;i++){
        const cx = 130 + i*125 + (Math.random()*24-12);
        const cy = 150 + ri*180 + (Math.random()*22-11);
        fruit(cx, cy, 52, row.col, row.dark, row.leaf);
      }
    });
    this.loadSrc(c.toDataURL('image/jpeg', 0.9));
  };

  async detect(src){
    this.setState({ screen:'analyzing', error:'' });
    try{
      const media = src.substring(5, src.indexOf(';'));
      const b64 = src.split(',')[1];
      const prompt = '画像内の数えられる物体を1つずつ検出してください。出力はJSONのみ、説明や```は不要。形式: {"objects":[{"label":"日本語の短い名詞(単数)","box":[x,y,w,h]}], "prices":{"<label>":推定単価}}。座標は画像全体を0〜1000に正規化した整数で、boxは[左上x, 左上y, 幅, 高さ]。見た目が同じ物は同じlabelに統一し、物体ごとに必ず1エントリ作成。最大50個まで。pricesには各labelごとの日本円での推定単価(1個あたり、整数)を入れてください。';
      const res = await window.claude.complete({ messages:[{ role:'user', content:[
        { type:'image', source:{ type:'base64', media_type: media, data: b64 } },
        { type:'text', text: prompt },
      ]}]});
      const parsed = this.parseDetection(res);
      let id = 1;
      const items = parsed.objects
        .filter(o => Array.isArray(o.box) && o.box.length === 4)
        .map(o => ({ id: id++, label: String(o.label || '物体').trim() || '物体', box: o.box.map(n => Math.max(0, Math.min(1000, Number(n)||0))) }));
      const labels = [...new Set(items.map(i => i.label))];
      const prices = {};
      Object.keys(parsed.prices || {}).forEach(k => { const v = Number(parsed.prices[k]); if(!isNaN(v)) prices[String(k).trim()] = Math.max(0, Math.round(v)); });
      this.setState({
        items,
        selected: labels,
        prices,
        capturedAt: this.nowStr(),
        screen: this.variant === 'single' ? 'result' : 'review',
      });
    }catch(e){
      this.setState({ error:'検出に失敗しました（' + ((e && e.message) || 'error') + '）。もう一度お試しください。', screen:'capture' });
    }
  }

  reset = () => this.setState({ screen:'capture', imgSrc:null, items:[], selected:[], prices:{}, addMode:null, place:'', geoNote:'', memo:'', error:'' });
  setPrice = (label, v) => this.setState(s => ({ prices: { ...s.prices, [label]: v === '' ? undefined : Math.max(0, Math.round(Number(v) || 0)) } }));

  storeKey(){ return 'countlens_records_' + this.variant; }
  componentDidMount(){ try{ const s = localStorage.getItem(this.storeKey()); if(s){ const r = JSON.parse(s); if(Array.isArray(r)) this.setState({ records: r }); } }catch(_){} }
  persist(records){ try{ localStorage.setItem(this.storeKey(), JSON.stringify(records)); }catch(_){} }

  recordsCount(){ return this.state.records.reduce((a,r)=>a+r.total,0); }
  recordsAmount(){ return this.state.records.reduce((a,r)=>a+r.amount,0); }

  addRecord = () => {
    const sel = this.state.selected.filter(l => this.countOf(l) > 0);
    if(!sel.length){ this.setState({ error:'リストに追加できる対象がありません。' }); return; }
    const rows = sel.map(l => ({ label:l, count:this.countOf(l), unit:this.unitPrice(l), amount:this.lineTotal(l), color:this.colorFor(l) }));
    const rec = { id: Date.now(), capturedAt: this.state.capturedAt, place: this.state.place, memo: this.state.memo,
      rows, total: rows.reduce((a,r)=>a+r.count,0), amount: rows.reduce((a,r)=>a+r.amount,0) };
    const records = [...this.state.records, rec];
    this.persist(records);
    this.setState({ records, screen:'list', imgSrc:null, items:[], selected:[], prices:{}, addMode:null, place:'', geoNote:'', memo:'', error:'' });
  };
  deleteRecord = (id) => { const records = this.state.records.filter(r => r.id !== id); this.persist(records); this.setState({ records }); };
  clearRecords = () => { this.persist([]); this.setState({ records: [] }); };
  goList = () => this.setState({ screen:'list', error:'' });
  goSummary = () => this.setState({ screen:'summary', error:'' });
  aggregate(){
    const agg = {};
    this.state.records.forEach(rec => rec.rows.forEach(row => {
      if(!agg[row.label]) agg[row.label] = { count:0, amount:0, color: row.color };
      agg[row.label].count += row.count; agg[row.label].amount += row.amount;
    }));
    const labels = Object.keys(agg).sort((a,b) => agg[b].count - agg[a].count);
    const totC = labels.reduce((x,l) => x + agg[l].count, 0);
    const totA = labels.reduce((x,l) => x + agg[l].amount, 0);
    return { agg, labels, totC, totA };
  }
  toggleSelect = (label) => this.setState(s => ({ selected: s.selected.includes(label) ? s.selected.filter(l=>l!==label) : [...s.selected, label] }));
  goResult = () => this.setState({ screen:'result' });
  setAddMode = (label) => this.setState(s => ({ addMode: s.addMode === label ? null : label }));
  deleteItem = (id) => this.setState(s => ({ items: s.items.filter(i => i.id !== id) }));

  addAt = (e) => {
    if(!this.state.addMode) return;
    const r = e.currentTarget.getBoundingClientRect();
    const fx = (e.clientX - r.left) / r.width;
    const fy = (e.clientY - r.top) / r.height;
    const w = 90, h = 90;
    const box = [ Math.max(0,Math.min(1000-w, fx*1000 - w/2)), Math.max(0,Math.min(1000-h, fy*1000 - h/2)), w, h ];
    const id = (Math.max(0, ...this.state.items.map(i=>i.id)) + 1);
    this.setState(s => ({ items:[...s.items, { id, label: s.addMode, box, manual:true }] }));
  };

  saveExcel = () => {
    if(!window.XLSX){ this.setState({ error:'Excelライブラリの読込中です。数秒後に再度お試しください。' }); return; }
    const recs = this.state.records;
    if(!recs.length){ this.setState({ error:'保存する計測結果がありません。先にリストへ追加してください。' }); return; }
    const X = window.XLSX;
    // ── columns: object table first, meta (datetime/place/memo) on the right ──
    const headers = ['No', '対象物名', '数量', '単価(円)', '金額(円)', '撮影日時', '場所', 'メモ'];
    const aoa = [
      ['CountLens ・ 物体数量計測レポート'],
      [`作成日時 ${this.nowStr()}　／　記録件数 ${recs.length} 件`],
      headers,
    ];
    const merges = [ {s:{r:0,c:0},e:{r:0,c:7}}, {s:{r:1,c:0},e:{r:1,c:7}} ];
    const rowShade = {};
    let r = 3;
    recs.forEach((rec, i) => {
      const start = r;
      rec.rows.forEach((row, j) => {
        aoa.push([ j===0 ? i+1 : '', row.label, row.count, row.unit, row.amount,
          j===0 ? rec.capturedAt : '', j===0 ? (rec.place||'') : '', j===0 ? (rec.memo||'') : '' ]);
        rowShade[r] = i % 2; r++;
      });
      const end = r - 1;
      if(end > start){ [0,5,6,7].forEach(c => merges.push({ s:{r:start,c}, e:{r:end,c} })); }
    });
    const totalRow = r;
    aoa.push(['合計', '', this.recordsCount(), '', this.recordsAmount(), '', '', '']);
    merges.push({ s:{r:totalRow,c:0}, e:{r:totalRow,c:1} });
    merges.push({ s:{r:totalRow,c:5}, e:{r:totalRow,c:7} });

    const ws = X.utils.aoa_to_sheet(aoa);
    ws['!merges'] = merges;
    ws['!cols'] = [{wch:5},{wch:16},{wch:8},{wch:12},{wch:13},{wch:19},{wch:22},{wch:24}];
    const rows = [{hpt:28},{hpt:18},{hpt:24}];
    for(let i=3;i<=totalRow;i++) rows.push({hpt:20});
    ws['!rows'] = rows;

    const thin = { style:'thin', color:{ rgb:'D6D5E3' } };
    const med  = { style:'medium', color:{ rgb:'8E8BD6' } };
    const bAll = (b) => ({ top:b, bottom:b, left:b, right:b });
    const set = (R, C, s, z) => { const ref = X.utils.encode_cell({ r:R, c:C }); if(!ws[ref]) ws[ref] = { t:'s', v:'' }; ws[ref].s = s; if(z) ws[ref].z = z; };

    for(let C=0;C<8;C++) set(0, C, { font:{ bold:true, sz:14, color:{rgb:'FFFFFF'} }, fill:{ patternType:'solid', fgColor:{rgb:'4F46E5'} }, alignment:{ horizontal:'left', vertical:'center' } });
    for(let C=0;C<8;C++) set(1, C, { font:{ sz:10, color:{rgb:'5B5B6B'} }, fill:{ patternType:'solid', fgColor:{rgb:'EFEEF8'} }, alignment:{ horizontal:'left', vertical:'center' } });
    for(let C=0;C<8;C++) set(2, C, { font:{ bold:true, sz:11, color:{rgb:'FFFFFF'} }, fill:{ patternType:'solid', fgColor:{rgb:'3F3D9E'} }, alignment:{ horizontal:'center', vertical:'center', wrapText:true }, border:bAll(thin) });

    for(let R=3;R<totalRow;R++){
      const bg = rowShade[R] ? 'F3F2FC' : 'FFFFFF';
      for(let C=0;C<8;C++){
        const right = (C===2 || C===3 || C===4);
        const center = (C===0);
        set(R, C, {
          font:{ sz:11, color:{rgb:'2A2A33'} },
          fill:{ patternType:'solid', fgColor:{rgb:bg} },
          alignment:{ vertical:'center', horizontal: right?'right':center?'center':'left', wrapText: C>=5 },
          border: bAll(thin),
        }, (C===3||C===4) ? '"¥"#,##0' : (C===2 ? '#,##0' : undefined));
      }
    }

    for(let C=0;C<8;C++){
      const right = (C===2 || C===4);
      set(totalRow, C, {
        font:{ bold:true, sz:11.5, color:{rgb:'2E2C7A'} },
        fill:{ patternType:'solid', fgColor:{rgb:'E1DFFA'} },
        alignment:{ vertical:'center', horizontal: C===0?'left':right?'right':'center' },
        border:{ top:med, bottom:med, left:thin, right:thin },
      }, C===4 ? '"¥"#,##0' : (C===2 ? '#,##0' : undefined));
    }

    // ── summary sheet: per-object aggregation (no record No) ──
    const agg = {};
    recs.forEach(rec => rec.rows.forEach(row => { if(!agg[row.label]) agg[row.label] = { count:0, amount:0 }; agg[row.label].count += row.count; agg[row.label].amount += row.amount; }));
    const sLabels = Object.keys(agg).sort((a,b) => agg[b].count - agg[a].count);
    const sAoa = [
      ['CountLens ・ 対象物別 集計'],
      [`作成日時 ${this.nowStr()}　／　記録件数 ${recs.length} 件　／　対象物 ${sLabels.length} 種類`],
      ['対象物名', '数量', '単価(平均/円)', '金額(円)'],
    ];
    sLabels.forEach(l => { const a = agg[l]; sAoa.push([ l, a.count, a.count ? Math.round(a.amount / a.count) : 0, a.amount ]); });
    const sTotalRow = sAoa.length;
    sAoa.push(['合計', sLabels.reduce((x,l)=>x+agg[l].count,0), '', sLabels.reduce((x,l)=>x+agg[l].amount,0)]);
    const ws2 = X.utils.aoa_to_sheet(sAoa);
    ws2['!merges'] = [ {s:{r:0,c:0},e:{r:0,c:3}}, {s:{r:1,c:0},e:{r:1,c:3}} ];
    ws2['!cols'] = [{wch:22},{wch:10},{wch:16},{wch:14}];
    const sRows = [{hpt:28},{hpt:18},{hpt:24}]; for(let i=3;i<=sTotalRow;i++) sRows.push({hpt:20}); ws2['!rows'] = sRows;
    const set2 = (R, C, s, z) => { const ref = X.utils.encode_cell({ r:R, c:C }); if(!ws2[ref]) ws2[ref] = { t:'s', v:'' }; ws2[ref].s = s; if(z) ws2[ref].z = z; };
    for(let C=0;C<4;C++) set2(0, C, { font:{ bold:true, sz:14, color:{rgb:'FFFFFF'} }, fill:{ patternType:'solid', fgColor:{rgb:'4F46E5'} }, alignment:{ horizontal:'left', vertical:'center' } });
    for(let C=0;C<4;C++) set2(1, C, { font:{ sz:10, color:{rgb:'5B5B6B'} }, fill:{ patternType:'solid', fgColor:{rgb:'EFEEF8'} }, alignment:{ horizontal:'left', vertical:'center' } });
    for(let C=0;C<4;C++) set2(2, C, { font:{ bold:true, sz:11, color:{rgb:'FFFFFF'} }, fill:{ patternType:'solid', fgColor:{rgb:'3F3D9E'} }, alignment:{ horizontal:'center', vertical:'center', wrapText:true }, border:bAll(thin) });
    for(let R=3;R<sTotalRow;R++){
      const bg = ((R-3) % 2) ? 'F3F2FC' : 'FFFFFF';
      for(let C=0;C<4;C++){
        const right = (C >= 1);
        set2(R, C, {
          font:{ sz:11, color:{rgb:'2A2A33'} },
          fill:{ patternType:'solid', fgColor:{rgb:bg} },
          alignment:{ vertical:'center', horizontal: right?'right':'left' },
          border: bAll(thin),
        }, (C===2||C===3) ? '"¥"#,##0' : (C===1 ? '#,##0' : undefined));
      }
    }
    for(let C=0;C<4;C++){
      const right = (C===1 || C===3);
      set2(sTotalRow, C, {
        font:{ bold:true, sz:11.5, color:{rgb:'2E2C7A'} },
        fill:{ patternType:'solid', fgColor:{rgb:'E1DFFA'} },
        alignment:{ vertical:'center', horizontal: C===0?'left':right?'right':'center' },
        border:{ top:med, bottom:med, left:thin, right:thin },
      }, C===3 ? '"¥"#,##0' : (C===1 ? '#,##0' : undefined));
    }

    const wb = X.utils.book_new();
    X.utils.book_append_sheet(wb, ws2, '対象物別集計');
    X.utils.book_append_sheet(wb, ws, '記録明細');
    X.writeFile(wb, `計測結果_${this.nowStr().replace(/[\/: ]/g,'')}.xlsx`);
  };

  // ── render helpers ───────────────────────────────────────────
  renderApp(){
    const c = this.theme();
    const wrap = (children) => React.createElement('div', { style:{ position:'relative', height:'100%', background:c.bg, color:c.fg, fontFamily:'-apple-system, "Hiragino Sans", system-ui, sans-serif', overflowY:'auto', WebkitFontSmoothing:'antialiased' } }, children);
    const s = this.state.screen;
    if(s === 'capture') return wrap(this.renderCapture(c));
    if(s === 'analyzing') return wrap(this.renderAnalyzing(c));
    if(s === 'review') return wrap(this.renderReview(c));
    if(s === 'list') return wrap(this.renderList(c));
    if(s === 'summary') return wrap(this.renderSummary(c));
    return wrap(this.renderResult(c));
  }

  renderCapture(c){
    const h = React.createElement;
    const camRef = React.createRef(); const libRef = React.createRef();
    const bigBtn = (label, sub, onClick, primary) => h('button', { onClick, style:{
      display:'flex', alignItems:'center', gap:14, width:'100%', textAlign:'left', cursor:'pointer',
      padding:'18px 20px', borderRadius:20, border:'none', marginBottom:14,
      background: primary ? c.accent : c.card, color: primary ? c.accentFg : c.fg,
      boxShadow: primary ? 'none' : `inset 0 0 0 1px ${c.line}`,
    }}, [
      h('div',{ key:'i', style:{ width:42, height:42, borderRadius:12, flexShrink:0, display:'flex', alignItems:'center', justifyContent:'center',
        background: primary ? 'rgba(255,255,255,0.2)' : c.panel, fontSize:22 } }, sub.icon),
      h('div',{ key:'t' }, [
        h('div',{ key:'a', style:{ fontSize:17, fontWeight:650, letterSpacing:'-0.01em' } }, label),
        h('div',{ key:'b', style:{ fontSize:13, marginTop:2, opacity:0.75 } }, sub.text),
      ]),
    ]);
    return h('div',{ style:{ padding:'68px 22px 30px', minHeight:'100%', boxSizing:'border-box', display:'flex', flexDirection:'column' } }, [
      h('div',{ key:'badge', style:{ fontSize:12, fontWeight:700, letterSpacing:'0.12em', color:c.accent, marginBottom:10 } }, 'COUNTLENS'),
      h('h1',{ key:'h', style:{ fontSize:32, lineHeight:1.12, fontWeight:740, letterSpacing:'-0.03em', margin:'0 0 10px' } }, ['写真に写る物を,', h('br',{key:'br'}), '数える。']),
      h('p',{ key:'p', style:{ fontSize:15, lineHeight:1.5, color:c.sub, margin:'0 0 24px', maxWidth:300 } }, '撮影するだけで対象を自動検出。写真を追加しながら計測リストにためて、まとめてExcelに保存できます。'),
      this.state.records.length ? h('button',{ key:'rec', onClick:this.goList, style:{ display:'flex', alignItems:'center', gap:12, width:'100%', textAlign:'left', cursor:'pointer', border:'none', background:c.panel, borderRadius:16, padding:'13px 15px', marginBottom:8, boxShadow:`inset 0 0 0 1px ${c.line}` } }, [
        h('div',{ key:'i', style:{ width:38, height:38, borderRadius:11, background:c.accent, color:c.accentFg, display:'flex', alignItems:'center', justifyContent:'center', fontSize:16, fontWeight:800, flexShrink:0 } }, this.state.records.length),
        h('div',{ key:'t', style:{ flex:1 } }, [
          h('div',{ key:'a', style:{ fontSize:15, fontWeight:680, color:c.fg } }, '計測リスト'),
          h('div',{ key:'b', style:{ fontSize:12.5, color:c.sub, marginTop:1 } }, `合計 ${this.recordsCount()}個 ・ ${this.yen(this.recordsAmount())}`),
        ]),
        h('span',{ key:'c', style:{ fontSize:20, color:c.sub, fontWeight:300 } }, '›'),
      ]) : null,
      this.state.error ? h('div',{ key:'e', style:{ background:'rgba(235,87,87,0.12)', color:'#EB5757', padding:'12px 14px', borderRadius:12, fontSize:13, marginBottom:16 } }, this.state.error) : null,
      h('div',{ key:'btns', style:{ marginTop:'auto' } }, [
        bigBtn('写真を撮影', { icon:'📷', text:'カメラで対象を撮る' }, () => camRef.current && camRef.current.click(), true),
        bigBtn('ライブラリから選択', { icon:'🖼', text:'保存済みの写真を読み込む' }, () => libRef.current && libRef.current.click()),
        bigBtn('サンプルで試す', { icon:'✨', text:'デモ用の写真で検出を体験' }, this.drawSample),
      ]),
      h('input',{ key:'ci', ref:camRef, type:'file', accept:'image/*', capture:'environment', onChange:this.onFile, style:{ display:'none' } }),
      h('input',{ key:'li', ref:libRef, type:'file', accept:'image/*', onChange:this.onFile, style:{ display:'none' } }),
    ]);
  }

  renderAnalyzing(c){
    const h = React.createElement;
    return h('div',{ style:{ minHeight:'100%', display:'flex', flexDirection:'column', alignItems:'center', justifyContent:'center', padding:40, boxSizing:'border-box' } }, [
      this.state.imgSrc ? h('div',{ key:'img', style:{ width:'100%', borderRadius:24, overflow:'hidden', position:'relative', marginBottom:30, boxShadow:c.shadow } }, [
        h('img',{ key:'i', src:this.state.imgSrc, style:{ width:'100%', display:'block', filter:'saturate(0.9)' } }),
        h('div',{ key:'scan', style:{ position:'absolute', inset:0, background:`linear-gradient(180deg, transparent, ${c.accent}22)`, } }),
      ]) : null,
      h('div',{ key:'sp', style:{ width:34, height:34, borderRadius:'50%', border:`3px solid ${c.line}`, borderTopColor:c.accent, animation:'spin 0.8s linear infinite', marginBottom:18 } }),
      h('div',{ key:'t', style:{ fontSize:17, fontWeight:650 } }, '対象物を検出中…'),
      h('div',{ key:'s', style:{ fontSize:13, color:c.sub, marginTop:6 } }, 'AIが写真の中の物体を解析しています'),
    ]);
  }

  renderPhoto(c, editable){
    const h = React.createElement;
    const visible = editable
      ? this.state.items.filter(i => this.state.selected.includes(i.label))
      : this.state.items;
    const counters = {};
    const boxes = visible.map(it => {
      const [x,y,w,ht] = it.box;
      const col = this.colorFor(it.label);
      counters[it.label] = (counters[it.label]||0) + 1;
      const badge = this.variant === 'single';
      const label = badge ? String(counters[it.label]) : it.label;
      return h('div',{ key:it.id,
        onClick:(e)=>{ e.stopPropagation(); if(editable) this.deleteItem(it.id); },
        style:{ position:'absolute', left:(x/10)+'%', top:(y/10)+'%', width:(w/10)+'%', height:(ht/10)+'%',
          border:`2px solid ${col}`, borderRadius: badge?'50%':6, background:col+'1f',
          boxShadow:`0 0 0 1px rgba(0,0,0,0.15)`, cursor: editable?'pointer':'default', animation:'pop 0.25s ease both' } }, [
        h('div',{ key:'l', style:{ position:'absolute', top: badge?'50%':-9, left: badge?'50%':-1,
          transform: badge?'translate(-50%,-50%)':'none',
          background:col, color:'#fff', fontSize:badge?11:10, fontWeight:700, lineHeight:1,
          padding: badge?'0':'2px 5px', borderRadius: badge?'50%':5, minWidth:badge?16:'auto', height:badge?16:'auto',
          display:'flex', alignItems:'center', justifyContent:'center', whiteSpace:'nowrap', boxShadow:'0 1px 3px rgba(0,0,0,0.3)' } }, label),
      ]);
    });
    return h('div',{ onClick: editable ? this.addAt : undefined,
      style:{ position:'relative', width:'100%', borderRadius:20, overflow:'hidden', boxShadow:c.shadow, cursor: editable && this.state.addMode ? 'crosshair' : 'default' } }, [
      h('img',{ key:'img', src:this.state.imgSrc, draggable:false, style:{ width:'100%', display:'block', pointerEvents:'none' } }),
      ...boxes,
    ]);
  }

  typeRow(c, label, { checkbox, showPrice }){
    const h = React.createElement;
    const col = this.colorFor(label);
    const on = this.state.selected.includes(label);
    const dim = checkbox && !on;
    return h('div',{ key:label,
      style:{ borderRadius:14, marginBottom:8, background: c.card, boxShadow:`inset 0 0 0 1px ${c.line}`, overflow:'hidden', opacity: dim ? 0.6 : 1 } }, [
      h('div',{ key:'top', onClick: checkbox ? ()=>this.toggleSelect(label) : undefined,
        style:{ display:'flex', alignItems:'center', gap:12, padding:'13px 14px', cursor: checkbox?'pointer':'default' } }, [
        checkbox ? h('div',{ key:'cb', style:{ width:22, height:22, borderRadius:7, flexShrink:0, display:'flex', alignItems:'center', justifyContent:'center',
          background: on ? c.accent : 'transparent', boxShadow: on ? 'none' : `inset 0 0 0 2px ${c.line}`, color:c.accentFg, fontSize:13, fontWeight:900 } }, on ? '✓' : '') : null,
        h('span',{ key:'dot', style:{ width:12, height:12, borderRadius:4, background:col, flexShrink:0 } }),
        h('span',{ key:'l', style:{ flex:1, fontSize:16, fontWeight:600 } }, label),
        h('span',{ key:'n', style:{ fontSize:17, fontWeight:740, fontVariantNumeric:'tabular-nums' } }, this.countOf(label)),
        h('span',{ key:'u', style:{ fontSize:12, color:c.sub, marginLeft:-4 } }, '個'),
      ]),
      showPrice ? h('div',{ key:'pr', style:{ display:'flex', alignItems:'center', gap:9, padding:'10px 14px', background:c.panel, borderTop:`1px solid ${c.line}` } }, [
        h('span',{ key:'pl', style:{ fontSize:12.5, color:c.sub, fontWeight:600 } }, '単価'),
        h('div',{ key:'pin', style:{ display:'flex', alignItems:'center', gap:1, background:c.card, borderRadius:9, boxShadow:`inset 0 0 0 1px ${c.line}`, padding:'5px 10px' } }, [
          h('span',{ key:'y', style:{ fontSize:13, color:c.sub } }, '¥'),
          h('input',{ key:'i', type:'number', inputMode:'numeric', value: this.state.prices[label] ?? '', placeholder:'0',
            onClick:e=>e.stopPropagation(), onChange:e=>this.setPrice(label, e.target.value),
            style:{ width:62, border:'none', background:'transparent', color:c.fg, fontSize:15, fontWeight:650, fontFamily:'inherit', outline:'none', MozAppearance:'textfield' } }),
        ]),
        h('span',{ key:'x', style:{ fontSize:12.5, color:c.sub } }, '× ' + this.countOf(label)),
        h('div',{ key:'amt', style:{ marginLeft:'auto', textAlign:'right' } }, [
          h('div',{ key:'al', style:{ fontSize:10, color:c.sub, fontWeight:600, letterSpacing:'0.02em' } }, '金額'),
          h('div',{ key:'av', style:{ fontSize:15.5, fontWeight:740, color:c.fg, fontVariantNumeric:'tabular-nums' } }, this.yen(this.lineTotal(label))),
        ]),
      ]) : null,
    ]);
  }

  header(c, title, sub, right){
    const h = React.createElement;
    return h('div',{ style:{ padding:'58px 20px 8px', display:'flex', alignItems:'flex-end', justifyContent:'space-between', gap:10 } }, [
      h('div',{ key:'l' }, [
        h('div',{ key:'t', style:{ fontSize:26, fontWeight:740, letterSpacing:'-0.03em' } }, title),
        sub ? h('div',{ key:'s', style:{ fontSize:13.5, color:c.sub, marginTop:3 } }, sub) : null,
      ]),
      right || null,
    ]);
  }

  renderReview(c){
    const h = React.createElement;
    const labels = this.labels();
    const n = this.state.selected.length;
    return h('div',{ style:{ paddingBottom:96 } }, [
      this.header(c, '検出結果', `${this.state.items.length}個 ・ ${labels.length}種類を検出`,
        h('button',{ key:'r', onClick:this.reset, style:{ border:'none', background:c.panel, color:c.sub, fontSize:13, fontWeight:600, padding:'8px 13px', borderRadius:10, cursor:'pointer' } }, 'やり直す')),
      h('div',{ key:'ph', style:{ padding:'12px 20px 4px' } }, this.renderPhoto(c, false)),
      h('div',{ key:'lbl', style:{ padding:'16px 24px 6px', fontSize:13, fontWeight:700, letterSpacing:'0.04em', color:c.sub } }, '計測する対象を選択'),
      h('div',{ key:'list', style:{ padding:'0 16px' } }, labels.length ? labels.map(l => this.typeRow(c, l, { checkbox:true })) :
        h('div',{ style:{ color:c.sub, fontSize:14, padding:'10px 8px' } }, '対象が検出されませんでした。やり直すか、次の画面で手動追加できます。')),
      h('div',{ key:'foot', style:{ position:'absolute', left:0, right:0, bottom:0, padding:'14px 20px 30px',
        background:`linear-gradient(180deg, transparent, ${c.bg} 38%)` } },
        h('button',{ onClick:this.goResult, disabled:n===0,
          style:{ width:'100%', padding:'17px', borderRadius:16, border:'none', cursor: n?'pointer':'default',
            background: n? c.accent : c.panel, color: n? c.accentFg : c.sub, fontSize:16.5, fontWeight:700, boxShadow: n?c.shadow:'none' } },
          n ? `選択した対象を計測 (${n}種類)` : '1つ以上選択してください')),
    ]);
  }

  renderResult(c){
    const h = React.createElement;
    const single = this.variant === 'single';
    const sel = this.state.selected.filter(l => this.countOf(l) > 0);
    const total = this.selectedTotal();
    const addLabels = (this.state.selected.length ? this.state.selected : ['対象物']);
    const field = (label, value, onChange, area) => h('div',{ key:label, style:{ marginBottom:12 } }, [
      h('div',{ key:'l', style:{ fontSize:12.5, fontWeight:650, color:c.sub, marginBottom:6 } }, label),
      h(area?'textarea':'input',{ key:'i', value, onChange:e=>onChange(e.target.value), rows: area?2:undefined,
        placeholder: area?'気づいた点など':'例: 第2倉庫 A-3棚',
        style:{ width:'100%', boxSizing:'border-box', resize:'none', fontFamily:'inherit',
          background:c.card, color:c.fg, border:'none', boxShadow:`inset 0 0 0 1px ${c.line}`,
          borderRadius:12, padding:'12px 14px', fontSize:15 } }),
    ]);
    return h('div',{ style:{ paddingBottom:100 } }, [
      this.header(c, '計測結果', this.state.capturedAt,
        h('button',{ key:'r', onClick:this.reset, style:{ border:'none', background:c.panel, color:c.sub, fontSize:13, fontWeight:600, padding:'8px 13px', borderRadius:10, cursor:'pointer' } }, 'やり直す')),
      // total
      h('div',{ key:'tot', style:{ margin:'10px 20px 4px', padding:'16px 20px', borderRadius:20, background:c.accent, color:c.accentFg, display:'flex', alignItems:'stretch', gap:16, boxShadow:c.shadow } }, [
        h('div',{ key:'q', style:{ flex:1 } }, [
          h('div',{ key:'l', style:{ fontSize:13, fontWeight:650, opacity:0.85 } }, '合計数量'),
          h('div',{ key:'v', style:{ marginTop:2 } }, [ h('span',{ key:'n', style:{ fontSize:32, fontWeight:800, fontVariantNumeric:'tabular-nums' } }, total), h('span',{ key:'u', style:{ fontSize:15, fontWeight:650, marginLeft:3 } }, '個') ]),
        ]),
        h('div',{ key:'d', style:{ width:1, background:c.accentFg, opacity:0.22 } }),
        h('div',{ key:'a', style:{ flex:1.2, textAlign:'right' } }, [
          h('div',{ key:'l', style:{ fontSize:13, fontWeight:650, opacity:0.85 } }, '推定総額'),
          h('div',{ key:'v', style:{ marginTop:2, fontSize:27, fontWeight:800, fontVariantNumeric:'tabular-nums', letterSpacing:'-0.01em' } }, this.yen(this.grandAmount())),
        ]),
      ]),
      h('div',{ key:'ph', style:{ padding:'10px 20px 2px' } }, this.renderPhoto(c, true)),
      // edit toolbar
      h('div',{ key:'edit', style:{ padding:'12px 20px 2px' } }, [
        h('div',{ key:'hint', style:{ fontSize:12.5, color:c.sub, marginBottom:9 } }, this.state.addMode ? `「${this.state.addMode}」を写真上でタップして追加 ・ ボックスをタップで削除` : 'ボックスをタップで削除。手動で追加するには種類を選択:'),
        h('div',{ key:'chips', style:{ display:'flex', gap:8, flexWrap:'wrap' } }, addLabels.map(l => {
          const on = this.state.addMode === l;
          return h('button',{ key:l, onClick:()=>this.setAddMode(l), style:{ border:'none', cursor:'pointer',
            padding:'7px 13px', borderRadius:999, fontSize:13, fontWeight:650,
            background: on ? c.accent : c.card, color: on ? c.accentFg : c.fg, boxShadow:`inset 0 0 0 1px ${c.line}` } }, '＋ '+l);
        })),
      ]),
      // type list
      h('div',{ key:'tl', style:{ padding:'18px 24px 6px', fontSize:13, fontWeight:700, letterSpacing:'0.04em', color:c.sub } }, single ? '対象を選択 / 内訳' : '内訳'),
      h('div',{ key:'rows', style:{ padding:'0 16px' } }, this.labels().map(l => this.typeRow(c, l, { checkbox: single, showPrice: true }))),
      // meta
      h('div',{ key:'meta', style:{ padding:'18px 20px 8px' } }, [
        h('div',{ key:'h', style:{ fontSize:13, fontWeight:700, letterSpacing:'0.04em', color:c.sub, marginBottom:10 } }, '記録情報'),
        h('div',{ key:'place', style:{ marginBottom:12 } }, [
          h('div',{ key:'l', style:{ fontSize:12.5, fontWeight:650, color:c.sub, marginBottom:6, display:'flex', alignItems:'center', gap:6 } }, [
            '場所',
            this.state.geoNote ? h('span',{ key:'b', style:{ fontSize:10.5, fontWeight:700, color:c.accent, background:c.accent+'1f', padding:'2px 7px', borderRadius:6 } }, '📍 自動取得') : null,
          ]),
          h('input',{ key:'i', value:this.state.place, onChange:e=>this.setState({place:e.target.value}), placeholder:'例: 第2倉庫 A-3棚',
            style:{ width:'100%', boxSizing:'border-box', fontFamily:'inherit', background:c.card, color:c.fg, border:'none', boxShadow:`inset 0 0 0 1px ${c.line}`, borderRadius:12, padding:'12px 14px', fontSize:15 } }),
          this.state.geoNote ? h('div',{ key:'n', style:{ fontSize:11.5, color:c.sub, marginTop:5 } }, this.state.geoNote) : null,
        ]),
        field('メモ', this.state.memo, v=>this.setState({memo:v}), true),
      ]),
      this.state.error ? h('div',{ key:'e', style:{ margin:'0 20px', background:'rgba(235,87,87,0.12)', color:'#EB5757', padding:'12px 14px', borderRadius:12, fontSize:13 } }, this.state.error) : null,
      // actions
      h('div',{ key:'foot', style:{ position:'absolute', left:0, right:0, bottom:0, padding:'12px 20px 28px', background:`linear-gradient(180deg, transparent, ${c.bg} 34%)`, display:'flex', flexDirection:'column', gap:8 } }, [
        h('button',{ key:'add', onClick:this.addRecord, style:{ width:'100%', padding:'16px', borderRadius:16, border:'none', cursor:'pointer',
          background:c.accent, color:c.accentFg, fontSize:16, fontWeight:700, display:'flex', alignItems:'center', justifyContent:'center', gap:7, boxShadow:c.shadow } },
          [ h('span',{key:'i',style:{fontSize:19, marginTop:-2}},'＋'), 'この結果を計測リストに追加' ]),
        h('button',{ key:'view', onClick:this.goList, style:{ width:'100%', padding:'11px', borderRadius:14, border:'none', cursor:'pointer', background:'transparent', color:c.sub, fontSize:13.5, fontWeight:650 } },
          this.state.records.length ? `計測リスト (${this.state.records.length}件) を見る / Excel保存` : 'リスト・Excel保存へ'),
      ]),
    ]);
  }

  recordCard(c, r, i){
    const h = React.createElement;
    return h('div',{ key:r.id, style:{ background:c.card, borderRadius:16, boxShadow:`inset 0 0 0 1px ${c.line}`, padding:'13px 15px', marginBottom:10, animation:'fadeup .3s ease both' } }, [
      h('div',{ key:'top', style:{ display:'flex', alignItems:'center', gap:9, marginBottom: r.place ? 7 : 10 } }, [
        h('span',{ key:'no', style:{ fontSize:11.5, fontWeight:800, color:c.accent, background:c.accent+'1f', padding:'3px 8px', borderRadius:7 } }, '#'+(i+1)),
        h('span',{ key:'dt', style:{ flex:1, fontSize:13, color:c.sub, fontWeight:600 } }, r.capturedAt),
        h('button',{ key:'del', onClick:()=>this.deleteRecord(r.id), style:{ border:'none', background:'transparent', color:c.sub, fontSize:18, lineHeight:1, cursor:'pointer', padding:'2px 4px' } }, '×'),
      ]),
      r.place ? h('div',{ key:'pl', style:{ fontSize:12.5, color:c.sub, marginBottom:10, overflow:'hidden', textOverflow:'ellipsis', whiteSpace:'nowrap' } }, '📍 '+r.place) : null,
      h('div',{ key:'chips', style:{ display:'flex', flexWrap:'wrap', gap:7, marginBottom:11 } }, r.rows.map(row =>
        h('span',{ key:row.label, style:{ display:'inline-flex', alignItems:'center', gap:6, fontSize:13, fontWeight:600, background:c.panel, borderRadius:8, padding:'5px 9px' } }, [
          h('span',{ key:'d', style:{ width:9, height:9, borderRadius:3, background:row.color } }),
          h('span',{ key:'t' }, row.label),
          h('span',{ key:'n', style:{ fontWeight:740, fontVariantNumeric:'tabular-nums' } }, row.count),
        ]))),
      h('div',{ key:'sum', style:{ display:'flex', alignItems:'baseline', justifyContent:'space-between', borderTop:`1px solid ${c.line}`, paddingTop:10 } }, [
        h('span',{ key:'q', style:{ fontSize:13.5, color:c.sub, fontWeight:600 } }, r.total + '個'),
        h('span',{ key:'a', style:{ fontSize:17, fontWeight:740, fontVariantNumeric:'tabular-nums' } }, this.yen(r.amount)),
      ]),
    ]);
  }

  renderList(c){
    const h = React.createElement;
    const recs = this.state.records;
    const empty = recs.length === 0;
    return h('div',{ style:{ paddingBottom:120, minHeight:'100%' } }, [
      this.header(c, '計測リスト', empty ? 'まだ記録がありません' : `${recs.length}件の記録`,
        empty ? null : h('button',{ key:'cl', onClick:this.clearRecords, style:{ border:'none', background:c.panel, color:c.sub, fontSize:13, fontWeight:600, padding:'8px 13px', borderRadius:10, cursor:'pointer' } }, '全消去')),
      empty ? null : h('div',{ key:'tot', style:{ margin:'10px 20px 16px', padding:'16px 20px', borderRadius:20, background:c.accent, color:c.accentFg, display:'flex', alignItems:'stretch', gap:16, boxShadow:c.shadow } }, [
        h('div',{ key:'q', style:{ flex:1 } }, [
          h('div',{ key:'l', style:{ fontSize:13, fontWeight:650, opacity:0.85 } }, '総数量'),
          h('div',{ key:'v', style:{ marginTop:2 } }, [ h('span',{ key:'n', style:{ fontSize:32, fontWeight:800, fontVariantNumeric:'tabular-nums' } }, this.recordsCount()), h('span',{ key:'u', style:{ fontSize:15, fontWeight:650, marginLeft:3 } }, '個') ]),
        ]),
        h('div',{ key:'d', style:{ width:1, background:c.accentFg, opacity:0.22 } }),
        h('div',{ key:'a', style:{ flex:1.2, textAlign:'right' } }, [
          h('div',{ key:'l', style:{ fontSize:13, fontWeight:650, opacity:0.85 } }, '総額'),
          h('div',{ key:'v', style:{ marginTop:2, fontSize:27, fontWeight:800, fontVariantNumeric:'tabular-nums', letterSpacing:'-0.01em' } }, this.yen(this.recordsAmount())),
        ]),
      ]),
      empty ? null : h('div',{ key:'sumwrap', style:{ padding:'0 20px 16px' } },
        h('button',{ onClick:this.goSummary, style:{ display:'flex', alignItems:'center', justifyContent:'center', gap:8, width:'100%', padding:'14px', borderRadius:14, border:'none', cursor:'pointer', background:c.card, color:c.accent, fontSize:15, fontWeight:700, boxShadow:`inset 0 0 0 1.5px ${c.accent}` } },
          [ h('span',{ key:'i', style:{ fontSize:17 } }, '📊'), '集計結果を確認' ])),
      empty
        ? h('div',{ key:'emp', style:{ margin:'28px 20px', padding:'40px 24px', borderRadius:20, background:c.card, boxShadow:`inset 0 0 0 1px ${c.line}`, textAlign:'center', color:c.sub } }, [
            h('div',{ key:'i', style:{ fontSize:34, marginBottom:12 } }, '🗒'),
            h('div',{ key:'t', style:{ fontSize:14.5, lineHeight:1.6 } }, '写真を計測して「計測リストに追加」すると、ここに記録がたまります。'),
          ])
        : h('div',{ key:'list', style:{ padding:'0 18px' } }, recs.map((r,i) => this.recordCard(c, r, i))),
      this.state.error ? h('div',{ key:'e', style:{ margin:'8px 20px 0', background:'rgba(235,87,87,0.12)', color:'#EB5757', padding:'12px 14px', borderRadius:12, fontSize:13 } }, this.state.error) : null,
      h('div',{ key:'foot', style:{ position:'absolute', left:0, right:0, bottom:0, padding:'12px 20px 28px', background:`linear-gradient(180deg, transparent, ${c.bg} 34%)`, display:'flex', gap:10 } }, [
        h('button',{ key:'add', onClick:this.reset, style:{ flex:1, padding:'16px', borderRadius:16, border:'none', cursor:'pointer', background:c.card, color:c.fg, fontSize:15, fontWeight:700, boxShadow:`inset 0 0 0 1px ${c.line}` } }, '＋ 写真を追加'),
        empty ? null : h('button',{ key:'save', onClick:this.saveExcel, style:{ flex:1.3, padding:'16px', borderRadius:16, border:'none', cursor:'pointer', background:c.fg, color:c.bg, fontSize:15, fontWeight:700, display:'flex', alignItems:'center', justifyContent:'center', gap:7, boxShadow:c.shadow } }, [ h('span',{key:'i',style:{fontSize:17}},'⤓'), 'まとめてExcel保存' ]),
      ]),
    ]);
  }

  renderSummary(c){
    const h = React.createElement;
    const { agg, labels, totC, totA } = this.aggregate();
    const empty = labels.length === 0;
    const back = h('button',{ onClick:this.goList, style:{ border:'none', background:c.panel, color:c.sub, fontSize:13, fontWeight:600, padding:'8px 13px', borderRadius:10, cursor:'pointer' } }, '← 戻る');
    const num = (txt, w, { bold, color, sz }) => h('div',{ style:{ width:w, textAlign:'right', fontSize:sz||13, fontWeight:bold?740:600, color:color||c.fg, fontVariantNumeric:'tabular-nums', flexShrink:0 } }, txt);
    return h('div',{ style:{ paddingBottom:110, minHeight:'100%' } }, [
      this.header(c, '対象物別集計', empty ? '記録がありません' : `${labels.length}種類 ・ ${this.state.records.length}件の記録`, back),
      empty
        ? h('div',{ key:'emp', style:{ margin:'28px 20px', padding:'40px 24px', borderRadius:20, background:c.card, boxShadow:`inset 0 0 0 1px ${c.line}`, textAlign:'center', color:c.sub, fontSize:14.5, lineHeight:1.6 } }, '計測結果をリストに追加すると、ここで対象物ごとの集計を確認できます。')
        : h('div',{ key:'card', style:{ margin:'10px 20px', borderRadius:18, overflow:'hidden', boxShadow:`inset 0 0 0 1px ${c.line}` } }, [
            h('div',{ key:'th', style:{ display:'flex', alignItems:'center', gap:8, padding:'12px 14px', background:c.accent, color:c.accentFg } }, [
              h('div',{ key:'l', style:{ flex:1, fontSize:12.5, fontWeight:700 } }, '対象物'),
              num('数量', 44, { bold:true, color:c.accentFg, sz:12.5 }),
              num('平均単価', 68, { bold:true, color:c.accentFg, sz:12.5 }),
              num('金額', 82, { bold:true, color:c.accentFg, sz:12.5 }),
            ]),
            ...labels.map((l, i) => h('div',{ key:l, style:{ display:'flex', alignItems:'center', gap:8, padding:'12px 14px', background: i%2 ? c.panel : c.card, borderTop:`1px solid ${c.line}` } }, [
              h('div',{ key:'lab', style:{ flex:1, display:'flex', alignItems:'center', gap:9, minWidth:0 } }, [
                h('span',{ key:'d', style:{ width:11, height:11, borderRadius:4, background: agg[l].color || c.accent, flexShrink:0 } }),
                h('span',{ key:'t', style:{ fontSize:14, fontWeight:650, overflow:'hidden', textOverflow:'ellipsis', whiteSpace:'nowrap' } }, l),
              ]),
              num(agg[l].count, 44, { bold:true, sz:16 }),
              num(this.yen(agg[l].count ? Math.round(agg[l].amount / agg[l].count) : 0), 68, { color:c.sub, sz:12.5 }),
              num(this.yen(agg[l].amount), 82, { bold:true, sz:14 }),
            ])),
            h('div',{ key:'tr', style:{ display:'flex', alignItems:'center', gap:8, padding:'14px', background:c.accent+'14', borderTop:`2px solid ${c.accent}` } }, [
              h('div',{ key:'l', style:{ flex:1, fontSize:14, fontWeight:740 } }, '合計'),
              num(totC, 44, { bold:true, sz:16 }),
              num('', 68, {}),
              num(this.yen(totA), 82, { bold:true, color:c.accent, sz:15 }),
            ]),
          ]),
      h('div',{ key:'foot', style:{ position:'absolute', left:0, right:0, bottom:0, padding:'12px 20px 28px', background:`linear-gradient(180deg, transparent, ${c.bg} 34%)`, display:'flex', gap:10 } }, [
        h('button',{ key:'b', onClick:this.goList, style:{ flex:1, padding:'16px', borderRadius:16, border:'none', cursor:'pointer', background:c.card, color:c.fg, fontSize:15, fontWeight:700, boxShadow:`inset 0 0 0 1px ${c.line}` } }, 'リストへ戻る'),
        empty ? null : h('button',{ key:'s', onClick:this.saveExcel, style:{ flex:1.3, padding:'16px', borderRadius:16, border:'none', cursor:'pointer', background:c.fg, color:c.bg, fontSize:15, fontWeight:700, display:'flex', alignItems:'center', justifyContent:'center', gap:7, boxShadow:c.shadow } }, [ h('span',{ key:'i', style:{ fontSize:17 } }, '⤓'), 'Excel保存' ]),
      ]),
    ]);
  }

  renderVals(){
    return { app: this.renderApp(), dark: this.dark };
  }
}
</script>
</body>
</html>
