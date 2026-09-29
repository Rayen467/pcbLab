<script>
  const modes = ['Schematic', 'PCB', 'Simulator', '3D', 'BOM', 'Fabrication'];
  const parts = [
    ['V','Sumber DC','5 V'], ['R','Resistor','330 Ω'], ['C','Kapasitor','1 µF'], ['D','Dioda','700 mV'],
    ['LED','LED','2 V'], ['SW','Sakelar','Ideal'], ['GND','Ground','0 V'], ['J','Junction','Net']
  ];
  let mode = 'Schematic';
  let activeTool = 'select';
  let selected = 'R2';
  let running = false;
  let zoom = 100;
  let query = '';
  let consoleOpen = true;
  let toast = '';
  let routed = false;
  let drcErrors = 0;
  let savedAt = 'Belum disimpan';

  const filtered = () => parts.filter((p) => p.join(' ').toLowerCase().includes(query.toLowerCase()));
  function notify(message) {
    toast = message;
    setTimeout(() => toast = '', 1800);
  }
  function simulate() {
    running = true;
    mode = 'Simulator';
    setTimeout(() => { running = false; notify('Simulasi DC selesai · 9.09 mA'); }, 700);
  }
  function runDrc() {
    drcErrors = routed ? 0 : 2;
    notify(routed ? 'DRC bersih · 0 pelanggaran' : 'DRC menemukan 2 unrouted net');
  }
  function autoRoute() {
    routed = true;
    drcErrors = 0;
    notify('Prototype autoroute selesai');
  }
  function saveProject() {
    savedAt = new Date().toLocaleTimeString('id-ID', {hour:'2-digit', minute:'2-digit'});
    notify('Project tersimpan lokal');
  }
</script>

<svelte:head>
  <title>PCB Lab — Pro EDA Studio</title>
  <meta name="description" content="PCB Lab prototype: schematic, SPICE simulation, PCB routing, DRC, BOM and fabrication workflow." />
</svelte:head>

<div class="app">
  <header class="topbar">
    <div class="brandline">
      <div class="brand-icon">⌁</div>
      <div class="brand">PCB<span>Lab</span></div>
      <div class="sep"></div>
      <button class="project"><i></i><strong>Indikator LED · 5 V</strong><span>⌄</span></button>
      <span class="badge">PRO LAB</span>
    </div>
    <div class="actions">
      <span class="autosave">{savedAt}</span>
      <button class="btn subtle" onclick={() => notify('Project baru siap')}>Baru</button>
      <button class="btn subtle" onclick={saveProject}>Simpan</button>
      <button class="btn subtle" onclick={() => notify('Ekspor Gerber akan tersedia di tahap fabrication engine')}>Ekspor</button>
      <button class="btn run" onclick={simulate}>{running ? 'Menjalankan…' : '▶ Simulasikan'}</button>
    </div>
  </header>

  <div class="shell">
    <aside class="library">
      <div class="side-head">
        <div><span>PUSTAKA</span><h2>Komponen</h2></div><b>8</b>
      </div>
      <label class="search"><span>⌕</span><input bind:value={query} placeholder="Cari komponen" /><kbd>/</kbd></label>
      <div class="section-label">SCHEMATIC · BASIC MODELS</div>
      <div class="parts">
        {#each filtered() as part}
          <button class="part" onclick={() => { selected = part[0] + '1'; notify(part[1] + ' dipilih'); }}>
            <span class="part-icon">{part[0]}</span>
            <span><strong>{part[1]}</strong><small>{part[2]}</small></span>
            <em>＋</em>
          </button>
        {/each}
      </div>
      <div class="learn">
        <div class="learn-kicker">WORKFLOW</div>
        <h3>Dari ide ke PCB</h3>
        <div class="steps">
          <span class="done">1</span><p><b>Schematic</b><small>Hubungkan net & pin</small></p>
          <span class="done">2</span><p><b>ERC</b><small>Validasi elektrikal</small></p>
          <span class="active">3</span><p><b>PCB</b><small>Placement & routing</small></p>
          <span>4</span><p><b>Fabrication</b><small>Gerber, BOM, CPL</small></p>
        </div>
      </div>
    </aside>

    <main class="workbench">
      <nav class="tabs">
        {#each modes as item}
          <button class:active={mode === item} onclick={() => mode = item}>
            <span>{item === 'Schematic' ? '⌁' : item === 'PCB' ? '▦' : item === 'Simulator' ? '∿' : item === '3D' ? '◈' : item === 'BOM' ? '☷' : '⬡'}</span>{item}
          </button>
        {/each}
      </nav>

      <div class="toolbar">
        <div class="tools">
          <button class:active={activeTool==='select'} onclick={() => activeTool='select'} title="Select">↖<small>Select</small></button>
          <button class:active={activeTool==='wire'} onclick={() => activeTool='wire'} title="Wire">⌁<small>Wire</small></button>
          <button class:active={activeTool==='pan'} onclick={() => activeTool='pan'} title="Pan">✥<small>Pan</small></button>
          <div class="vsep"></div>
          <button title="Undo">↶</button><button title="Redo">↷</button><button title="Rotate">⤾</button><button title="Duplicate">⧉</button><button title="Delete">⌫</button>
        </div>
        <div class="tool-right">
          <button class="rule" onclick={runDrc}><span class:bad={drcErrors>0}>●</span> DRC {drcErrors}</button>
          <button class="rule" onclick={() => notify('ERC: 0 error, 1 warning')}>✓ ERC</button>
          <button class="fit" onclick={() => zoom=100}>⛶ Fit</button>
        </div>
      </div>

      {#if mode === 'Schematic'}
        <section class="canvas schematic">
          <div class="sheet-tag">SCHEMATIC / 01</div>
          <svg class="wires" viewBox="0 0 1000 620" preserveAspectRatio="none">
            <polyline points="190,300 310,300 310,205 438,205" />
            <polyline points="560,205 705,205 705,300 820,300" />
            <polyline points="820,300 820,430 500,430" />
            <polyline points="500,430 190,430 190,300" />
            <circle cx="310" cy="300" r="5"/><circle cx="705" cy="300" r="5"/>
          </svg>
          <button class="component source" class:selected={selected==='V1'} onclick={() => selected='V1'}>
            <span class="ref">V1</span><div class="symbol circle">＋<br/>−</div><b>5 V</b><small>DC</small>
          </button>
          <button class="component resistor" class:selected={selected==='R2'} onclick={() => selected='R2'}>
            <span class="ref">R2</span><div class="res-symbol">▱▰▱</div><b>330 Ω</b><small>±1%</small>
          </button>
          <button class="component led" class:selected={selected==='D3'} onclick={() => selected='D3'}>
            <span class="ref">D3</span><div class="symbol led-symbol">▷│ ↗↗</div><b>2.0 V</b><small>LED Red</small>
          </button>
          <button class="component ground" class:selected={selected==='G4'} onclick={() => selected='G4'}>
            <span class="ref">G4</span><div class="gnd">⏚</div><b>GND</b><small>Net 0</small>
          </button>
          <div class="canvas-hud"><span><b>W</b> Wire</span><span><b>R</b> Rotate</span><span><b>Del</b> Delete</span><span><b>Space</b> Pan</span></div>
        </section>
      {:else if mode === 'PCB'}
        <section class="canvas pcbstage">
          <div class="pcb-head"><span>BOARD / MAIN</span><div><b>2 Layer</b><b>FR-4 1.6 mm</b><b>1 oz Cu</b></div></div>
          <div class="board">
            <i class="hole h1"></i><i class="hole h2"></i><i class="hole h3"></i><i class="hole h4"></i>
            <div class="fp fp1"><span>V1</span><i></i><i></i></div>
            <div class="fp fp2"><span>R2</span><i></i><i></i></div>
            <div class="fp fp3"><span>D3</span><i></i><i></i></div>
            <div class="trace tr1"></div><div class="trace tr2"></div><div class="trace tr3"></div>
            {#if !routed}<svg class="rats" viewBox="0 0 700 390"><line x1="150" y1="200" x2="350" y2="120"/><line x1="350" y1="120" x2="535" y2="240"/></svg>{/if}
          </div>
          <div class="pcb-sidecard"><span>ROUTING</span><strong>{routed ? '100%' : '33%'}</strong><small>{routed ? '0 unrouted' : '2 unrouted nets'}</small><button onclick={autoRoute}>⚡ Prototype Autoroute</button></div>
        </section>
      {:else if mode === 'Simulator'}
        <section class="panel-view simulator">
          <div class="view-title"><div><span>SPICE ANALYSIS</span><h2>Operating Point</h2><p>Prototype result model · NGSpice integration next.</p></div><button class="btn run" onclick={simulate}>▶ Run analysis</button></div>
          <div class="metrics"><article><span>V(source)</span><strong>5.000 <i>V</i></strong><small>DC source</small></article><article><span>I(R2)</span><strong>9.091 <i>mA</i></strong><small>Series current</small></article><article><span>P(R2)</span><strong>27.27 <i>mW</i></strong><small>Dissipation</small></article><article><span>V(D3)</span><strong>2.000 <i>V</i></strong><small>Linearized LED</small></article></div>
          <div class="scope"><div class="scope-head"><span>Waveform viewer</span><div><b class="cyan"></b>V(in)<b class="purple"></b>V(led)</div></div><svg viewBox="0 0 900 300"><path class="wave a" d="M0 240 C80 235 90 70 175 65 S280 65 340 65 S480 65 560 65 S700 65 900 65"/><path class="wave b" d="M0 245 C100 242 150 180 240 175 S420 174 520 174 S700 174 900 174"/></svg></div>
        </section>
      {:else if mode === '3D'}
        <section class="panel-view three"><div class="view-title"><div><span>3D PREVIEW</span><h2>Board visualization</h2><p>Prototype mechanical preview.</p></div><button class="btn subtle">Reset camera</button></div><div class="three-scene"><div class="board3d"><div class="chip3d c1">V1</div><div class="chip3d c2">R2</div><div class="chip3d c3">LED</div></div><div class="axis">Z<br/><span>Y ─ X</span></div></div></section>
      {:else if mode === 'BOM'}
        <section class="panel-view"><div class="view-title"><div><span>BILL OF MATERIALS</span><h2>Project components</h2><p>Part mapping and manufacturing readiness.</p></div><button class="btn subtle" onclick={() => notify('BOM CSV prototype exported')}>Export CSV</button></div><table><thead><tr><th>Ref</th><th>Part</th><th>Value</th><th>Footprint</th><th>Qty</th><th>Status</th></tr></thead><tbody><tr><td>V1</td><td>DC Input</td><td>5 V</td><td>TerminalBlock 2P</td><td>1</td><td><span class="ok">Mapped</span></td></tr><tr><td>R2</td><td>Resistor</td><td>330 Ω</td><td>R_0805</td><td>1</td><td><span class="ok">Mapped</span></td></tr><tr><td>D3</td><td>LED Red</td><td>2 V</td><td>LED_0603</td><td>1</td><td><span class="warn">Verify MPN</span></td></tr></tbody></table></section>
      {:else}
        <section class="panel-view fabrication"><div class="view-title"><div><span>FABRICATION</span><h2>Manufacturing package</h2><p>Pre-flight checklist before real PCB production.</p></div><button class="btn run" onclick={() => notify('Run DRC dulu sebelum generate package')}>Generate package</button></div><div class="fab-grid"><article><span>01</span><h3>DRC</h3><p>Clearance, width, drill, short, board edge.</p><b class:done={routed}>{routed ? '✓ Ready' : 'Needs routing'}</b></article><article><span>02</span><h3>Gerber + Drill</h3><p>Copper, mask, silkscreen and Excellon.</p><b>Planned</b></article><article><span>03</span><h3>BOM + CPL</h3><p>Part list and pick-and-place coordinates.</p><b>Planned</b></article><article><span>04</span><h3>Release</h3><p>Versioned manufacturing archive.</p><b>Planned</b></article></div></section>
      {/if}

      <footer class="statusbar"><div><span class="green-dot"></span><b>PCB Lab Core</b><span>Prototype engine online</span></div><div><span>Grid 10 mil</span><button onclick={() => zoom=Math.max(50,zoom-10)}>−</button><b>{zoom}%</b><button onclick={() => zoom=Math.min(180,zoom+10)}>＋</button><button onclick={() => consoleOpen=!consoleOpen}>{consoleOpen ? 'Hide log' : 'Show log'}</button></div></footer>
      {#if consoleOpen}<div class="console"><span>[core]</span> project loaded · 4 symbols · 3 nets <span>[erc]</span> 0 errors <span>[pcb]</span> {routed ? 'routing complete' : '2 unrouted'} <span>[sim]</span> model: prototype-linear</div>{/if}
    </main>

    <aside class="inspector">
      <section><div class="inspector-title"><span>PROPERTIES</span><b>⋯</b></div><div class="selection"><small>SELECTED</small><h3>{selected}</h3><span>{selected==='R2' ? 'Resistor' : selected==='D3' ? 'LED' : selected==='V1' ? 'DC Source' : selected==='G4' ? 'Ground' : 'Component'}</span></div><label>Reference<input value={selected} /></label><label>Value<input value={selected==='R2' ? '330 Ω' : selected==='V1' ? '5 V' : selected==='D3' ? '2 V' : '0 V'} /></label><label>Footprint<select><option>{selected==='R2' ? 'R_0805' : selected==='D3' ? 'LED_0603' : 'Default'}</option><option>Custom...</option></select></label></section>
      <section><div class="inspector-title"><span>DESIGN CHECKS</span><b class="health">A</b></div><div class="checks"><p><i class="pass">✓</i><span><b>ERC</b><small>Electrical rules clean</small></span></p><p><i class:pass={routed} class:idle={!routed}>{routed ? '✓' : '!'}</i><span><b>DRC</b><small>{routed ? '0 violations' : 'Run after routing'}</small></span></p><p><i class="pass">✓</i><span><b>Footprints</b><small>3 of 3 assigned</small></span></p></div></section>
      <section><div class="inspector-title"><span>NET INSPECTOR</span><b>3</b></div><div class="nets"><button><i class="power"></i><span><b>VCC</b><small>2 pins · 5.0 V</small></span><em>›</em></button><button><i class="signal"></i><span><b>N_LED</b><small>2 pins · 2.0 V</small></span><em>›</em></button><button><i class="groundnet"></i><span><b>GND / 0</b><small>2 pins · reference</small></span><em>›</em></button></div></section>
      <div class="roadmap"><span>ENGINE ROADMAP</span><p><b>Next:</b> real netlist core → NGSpice/WASM → interactive router → full DRC → Gerber.</p></div>
    </aside>
  </div>

  {#if toast}<div class="toast">✓ {toast}</div>{/if}
</div>

<style>
  :global(*){box-sizing:border-box} :global(body){margin:0;background:#070b12;color:#dce6f2;font-family:Inter,ui-sans-serif,system-ui,-apple-system,BlinkMacSystemFont,"Segoe UI",sans-serif;overflow:hidden} :global(button),:global(input),:global(select){font:inherit}
  :global(:root){color-scheme:dark}.app{height:100vh;background:radial-gradient(circle at 45% -20%,#132638 0,#08111b 35%,#070b12 62%);display:flex;flex-direction:column}.topbar{height:58px;display:flex;align-items:center;justify-content:space-between;padding:0 14px;border-bottom:1px solid #1a2634;background:#09111bd9;backdrop-filter:blur(18px);z-index:5}.brandline,.actions{display:flex;align-items:center;gap:8px}.brand-icon{width:31px;height:31px;border-radius:9px;background:linear-gradient(145deg,#2df0ce,#0b8f85);display:grid;place-items:center;color:#03221d;font-size:20px;font-weight:900;box-shadow:0 0 24px #15d4bd33}.brand{font-weight:850;letter-spacing:-.04em;font-size:16px}.brand span{color:#44dac5}.sep{width:1px;height:24px;background:#263241;margin:0 4px}.project{border:0;background:transparent;color:#b8c8d8;display:flex;align-items:center;gap:8px;padding:7px 9px;border-radius:8px}.project:hover{background:#111c28}.project i{width:7px;height:7px;border-radius:50%;background:#37d7a3;box-shadow:0 0 10px #37d7a3}.project strong{font-size:12px}.badge{font-size:9px;letter-spacing:.12em;font-weight:800;padding:4px 7px;border:1px solid #265549;border-radius:5px;color:#5fe0bd;background:#0c201b}.autosave{color:#5f7285;font-size:10px;margin-right:8px}.btn{border:1px solid #263646;background:#101a26;color:#c3d1df;padding:7px 11px;border-radius:8px;font-size:11px;font-weight:700;cursor:pointer}.btn:hover{border-color:#40576d;background:#142334}.btn.subtle{background:#0d1620}.btn.run{background:linear-gradient(180deg,#1fae98,#118473);border-color:#2ccbb3;color:white;box-shadow:0 4px 16px #13a58d2c}.shell{display:grid;grid-template-columns:236px minmax(500px,1fr) 252px;min-height:0;flex:1}.library,.inspector{background:#09111af0;overflow:auto}.library{border-right:1px solid #1b2835}.inspector{border-left:1px solid #1b2835}.side-head{display:flex;justify-content:space-between;align-items:end;padding:18px 15px 9px}.side-head span,.section-label,.learn-kicker,.view-title span,.inspector-title span,.roadmap>span{font-size:9px;letter-spacing:.13em;font-weight:800;color:#536b81}.side-head h2{font-size:16px;margin:3px 0 0}.side-head>b,.inspector-title>b{font-size:10px;background:#122130;color:#83a2bc;border:1px solid #243a4c;border-radius:10px;padding:2px 7px}.search{display:flex;align-items:center;margin:7px 12px 14px;border:1px solid #213445;background:#0d1722;border-radius:8px;padding:0 8px;gap:7px;color:#647e94}.search input{background:transparent;border:0;outline:0;color:#d6e1eb;min-width:0;width:100%;padding:8px 0;font-size:11px}.search kbd{font-size:9px;border:1px solid #304254;border-radius:4px;padding:1px 4px}.section-label{padding:0 15px 7px}.parts{padding:0 8px}.part{width:100%;border:1px solid transparent;background:transparent;color:#b9c8d6;padding:7px;border-radius:8px;display:grid;grid-template-columns:34px 1fr 18px;align-items:center;text-align:left;cursor:pointer}.part:hover{background:#0f1d2a;border-color:#1f3446}.part-icon{width:29px;height:29px;border:1px solid #2b4658;background:#102230;border-radius:6px;display:grid;place-items:center;color:#67dbc7;font:700 10px ui-monospace}.part strong{display:block;font-size:11px}.part small{display:block;color:#60778b;font-size:9px;margin-top:2px}.part em{font-style:normal;color:#506d83}.learn{margin:15px 10px;border-top:1px solid #1b2a38;padding:15px 5px}.learn h3{font-size:13px;margin:4px 0 12px}.steps{display:grid;grid-template-columns:25px 1fr;gap:7px 8px;align-items:center}.steps>span{width:22px;height:22px;border-radius:50%;border:1px solid #2c3c4c;color:#607589;display:grid;place-items:center;font-size:9px}.steps>span.done{color:#52d8b6;border-color:#24594e;background:#0b221d}.steps>span.active{color:#fff;border-color:#2b9c8c;background:#176d61}.steps p{margin:0}.steps b{display:block;font-size:10px}.steps small{display:block;font-size:8px;color:#62778a;margin-top:1px}.workbench{min-width:0;display:flex;flex-direction:column;background:#0a111a}.tabs{height:43px;display:flex;align-items:end;border-bottom:1px solid #1d2a37;background:#0a131d;padding:0 10px}.tabs button{height:100%;border:0;border-bottom:2px solid transparent;background:transparent;color:#6d8398;padding:0 12px;display:flex;gap:6px;align-items:center;font-size:10px;font-weight:700}.tabs button.active{color:#e0edf6;border-bottom-color:#34cbb4;background:linear-gradient(0deg,#12302966,transparent)}.tabs button span{font-size:13px}.toolbar{height:48px;display:flex;justify-content:space-between;align-items:center;padding:0 10px;border-bottom:1px solid #182532;background:#0b141e}.tools,.tool-right{display:flex;align-items:center;gap:4px}.tools button,.fit,.rule{border:1px solid transparent;background:transparent;color:#72889b;border-radius:6px;min-width:31px;height:31px;display:flex;align-items:center;justify-content:center;gap:5px;font-size:13px}.tools button:hover,.fit:hover,.rule:hover{background:#132231;border-color:#263b4f;color:#d2e0eb}.tools button.active{background:#15352f;border-color:#286e60;color:#65dfc7}.tools small{font-size:8px}.vsep{height:22px;width:1px;background:#23313f;margin:0 4px}.rule{font-size:9px;border-color:#203346;padding:0 8px}.rule span{color:#4fd2af}.rule span.bad{color:#ff866f}.fit{font-size:9px;border-color:#203346;padding:0 8px}.canvas{position:relative;flex:1;min-height:0;overflow:hidden;background-color:#0b131d;background-image:radial-gradient(#253646 1px,transparent 1px),radial-gradient(#172635 .7px,transparent .7px);background-size:20px 20px,10px 10px}.schematic{min-height:0}.wires{position:absolute;inset:0;width:100%;height:100%}.wires polyline{fill:none;stroke:#66d8c4;stroke-width:2}.wires circle{fill:#6ce3cd}.component{position:absolute;border:1px solid transparent;background:#0b151f;color:#d4e2ed;border-radius:9px;min-width:100px;padding:9px 12px;box-shadow:0 8px 20px #0005;cursor:pointer}.component:hover,.component.selected{border-color:#32b7a4;box-shadow:0 0 0 1px #1d5e55,0 12px 35px #0007}.component .ref{position:absolute;top:-15px;left:0;color:#7890a3;font:700 9px ui-monospace}.component b{display:block;font:700 10px ui-monospace;color:#cbe4e7;margin-top:6px}.component small{color:#5f778b;font-size:8px}.source{left:14%;top:40%}.resistor{left:43%;top:22%}.led{right:13%;top:40%}.ground{left:45%;bottom:17%}.symbol,.res-symbol,.gnd{height:28px;display:grid;place-items:center;color:#75decf;font:700 14px ui-monospace}.symbol.circle{width:28px;border:1px solid #73d6c7;border-radius:50%;line-height:11px;margin:auto}.led-symbol{white-space:nowrap}.gnd{font-size:25px}.sheet-tag{position:absolute;right:14px;bottom:13px;color:#425a6f;font:700 9px ui-monospace;letter-spacing:.1em}.canvas-hud{position:absolute;left:15px;bottom:12px;display:flex;gap:6px}.canvas-hud span{font-size:8px;color:#5b7184;background:#0a131de8;border:1px solid #1f3343;border-radius:5px;padding:4px 6px}.canvas-hud b{color:#9eb2c3}.pcbstage{display:flex;align-items:center;justify-content:center}.pcb-head{position:absolute;left:14px;top:13px;right:14px;display:flex;justify-content:space-between;color:#5e7386;font:700 9px ui-monospace}.pcb-head div{display:flex;gap:6px}.pcb-head b{padding:4px 7px;border:1px solid #243544;border-radius:5px;background:#0c1620}.board{position:relative;width:min(72%,700px);aspect-ratio:1.8;border:2px solid #d5b656;border-radius:7px;background:linear-gradient(140deg,#164b3c,#0e3a2f);box-shadow:0 30px 70px #0009,inset 0 0 45px #09261e}.hole{position:absolute;width:12px;height:12px;border:2px solid #c6ab63;border-radius:50%;background:#09100f}.h1{left:12px;top:12px}.h2{right:12px;top:12px}.h3{left:12px;bottom:12px}.h4{right:12px;bottom:12px}.fp{position:absolute;border:1px solid #dbbd66;background:#183f34aa;color:#e3cf8e;width:84px;height:54px;border-radius:4px;font:700 9px ui-monospace;text-align:center;padding-top:5px}.fp i{position:absolute;width:10px;height:10px;border-radius:50%;border:2px solid #e2bd5f;background:#283827;bottom:7px}.fp i:first-of-type{left:18px}.fp i:last-of-type{right:18px}.fp1{left:14%;top:43%}.fp2{left:44%;top:21%;transform:rotate(-8deg)}.fp3{right:15%;bottom:24%}.trace{position:absolute;height:5px;background:#c75c49;box-shadow:0 0 0 1px #9f4235;border-radius:5px;transform-origin:left center}.tr1{left:20%;top:50%;width:29%;transform:rotate(-33deg)}.tr2{left:50%;top:28%;width:34%;transform:rotate(29deg)}.tr3{right:22%;bottom:30%;width:26%;transform:rotate(130deg)}.rats{position:absolute;inset:0;width:100%;height:100%}.rats line{stroke:#e6d46a;stroke-width:1;stroke-dasharray:5 4;opacity:.8}.pcb-sidecard{position:absolute;right:16px;bottom:16px;width:160px;padding:12px;border:1px solid #24394a;background:#0a151fdd;border-radius:10px}.pcb-sidecard span{font-size:8px;color:#587388}.pcb-sidecard strong{display:block;font-size:24px;color:#57dbc0;margin:3px 0}.pcb-sidecard small{font-size:8px;color:#74899b}.pcb-sidecard button{width:100%;margin-top:9px;border:1px solid #285c51;background:#113229;color:#67d9c1;border-radius:6px;padding:6px;font-size:8px}.panel-view{flex:1;min-height:0;padding:22px;overflow:auto;background:linear-gradient(160deg,#0b131d,#091018)}.view-title{display:flex;align-items:center;justify-content:space-between;margin-bottom:18px}.view-title h2{margin:4px 0 2px;font-size:21px}.view-title p{margin:0;color:#607589;font-size:10px}.metrics{display:grid;grid-template-columns:repeat(4,1fr);gap:10px}.metrics article,.fab-grid article{background:#0d1823;border:1px solid #1f3040;border-radius:11px;padding:14px}.metrics span{font-size:9px;color:#71879b}.metrics strong{display:block;font:700 22px ui-monospace;color:#dcebf3;margin:8px 0}.metrics strong i{font-size:10px;color:#66d7c2;font-style:normal}.metrics small{color:#506779;font-size:8px}.scope{margin-top:12px;background:#08111a;border:1px solid #1c2f3f;border-radius:11px;overflow:hidden}.scope-head{height:39px;border-bottom:1px solid #1b2c3a;display:flex;align-items:center;justify-content:space-between;padding:0 12px;color:#768da1;font-size:9px}.scope-head div{display:flex;align-items:center;gap:6px}.scope-head b{width:8px;height:2px}.cyan{background:#46e4ca}.purple{background:#a57af5}.scope svg{width:100%;height:260px;background-image:linear-gradient(#132330 1px,transparent 1px),linear-gradient(90deg,#132330 1px,transparent 1px);background-size:45px 45px}.wave{fill:none;stroke-width:2}.wave.a{stroke:#47e4cb}.wave.b{stroke:#a97df7}.three-scene{height:75%;display:grid;place-items:center;perspective:900px;position:relative}.board3d{position:relative;width:480px;height:260px;border:4px solid #bcae5e;border-radius:10px;background:linear-gradient(135deg,#17654a,#0b3d2d);transform:rotateX(58deg) rotateZ(-25deg);box-shadow:25px 45px 40px #0008,inset 0 0 70px #06241a}.chip3d{position:absolute;background:#111820;border:2px solid #506b5e;color:#9ecbb2;padding:18px 28px;border-radius:4px;box-shadow:12px 18px 14px #0008;font:700 11px ui-monospace}.chip3d.c1{left:50px;top:105px}.chip3d.c2{left:205px;top:45px}.chip3d.c3{right:45px;bottom:50px;background:#a52924;color:#ffd5ca}.axis{position:absolute;right:15px;bottom:15px;color:#5c7588;font:700 10px ui-monospace}.axis span{color:#809cb0}table{width:100%;border-collapse:collapse;background:#0c1620;border:1px solid #1f3040;border-radius:9px;overflow:hidden;font-size:10px}th,td{padding:11px 13px;border-bottom:1px solid #1b2b38;text-align:left}th{font-size:8px;color:#60778a;background:#0f1b27;letter-spacing:.08em}td{color:#a9bbc9}.ok,.warn{padding:3px 6px;border-radius:5px;font-size:8px}.ok{background:#0e2a22;color:#61d6b6}.warn{background:#332915;color:#e1bf6a}.fab-grid{display:grid;grid-template-columns:repeat(2,1fr);gap:10px}.fab-grid article span{color:#446177;font:700 10px ui-monospace}.fab-grid h3{font-size:14px;margin:7px 0}.fab-grid p{font-size:9px;color:#64798b;line-height:1.5}.fab-grid b{font-size:8px;color:#71889b}.fab-grid b.done{color:#5dd7b7}.statusbar{height:30px;background:#09111a;border-top:1px solid #1c2a37;display:flex;justify-content:space-between;align-items:center;padding:0 9px;font-size:8px;color:#50677a}.statusbar>div{display:flex;align-items:center;gap:9px}.statusbar b{color:#859bae}.statusbar button{border:0;background:transparent;color:#6f8597;font-size:8px}.green-dot{width:6px;height:6px;border-radius:50%;background:#4dd4a9;box-shadow:0 0 8px #4dd4a9}.console{height:26px;background:#050a10;border-top:1px solid #15212b;padding:7px 10px;color:#516678;font:8px ui-monospace;white-space:nowrap;overflow:hidden}.console span{color:#42cdb3;margin-left:10px}.inspector section{padding:15px 13px;border-bottom:1px solid #1a2834}.inspector-title{display:flex;align-items:center;justify-content:space-between;margin-bottom:11px}.inspector-title .health{background:#11382e;color:#6adabd;border-color:#28594e}.selection{border:1px solid #203345;background:#0d1924;border-radius:9px;padding:10px;margin-bottom:10px}.selection small{font-size:8px;color:#547085}.selection h3{font:800 18px ui-monospace;margin:3px 0}.selection span{font-size:9px;color:#708599}.inspector label{display:block;color:#647b8e;font-size:8px;margin-top:8px}.inspector input,.inspector select{width:100%;margin-top:4px;border:1px solid #203646;background:#0a141e;color:#c1d1dc;border-radius:6px;padding:7px;font-size:9px;outline:0}.checks p{display:flex;align-items:center;gap:9px;margin:8px 0}.checks i{width:20px;height:20px;border-radius:50%;display:grid;place-items:center;font-style:normal;font-size:9px}.checks .pass{background:#0f2c24;color:#55d7b4;border:1px solid #1d5749}.checks .idle{background:#2c2517;color:#caa84d;border:1px solid #554620}.checks b,.checks small{display:block}.checks b{font-size:9px}.checks small{font-size:8px;color:#5e7487;margin-top:1px}.nets button{width:100%;border:0;background:transparent;color:#a8bac7;display:grid;grid-template-columns:8px 1fr 8px;text-align:left;align-items:center;gap:8px;padding:7px 2px}.nets i{width:6px;height:6px;border-radius:50%}.power{background:#f0c060}.signal{background:#57d8be}.groundnet{background:#7d91a3}.nets b{display:block;font-size:9px}.nets small{font-size:8px;color:#587084}.nets em{font-style:normal;color:#435c70}.roadmap{margin:12px;border:1px solid #1d3444;background:linear-gradient(145deg,#0e1f2b,#0c181f);border-radius:10px;padding:11px}.roadmap p{font-size:8px;line-height:1.5;color:#647d90;margin:6px 0 0}.roadmap b{color:#56d5bb}.toast{position:fixed;right:22px;bottom:22px;background:#10271f;border:1px solid #2d6658;color:#78dfc6;border-radius:9px;padding:10px 14px;font-size:10px;box-shadow:0 20px 50px #0008;z-index:20}
  @media(max-width:1050px){.shell{grid-template-columns:210px 1fr}.inspector{display:none}.actions .autosave,.actions .subtle:nth-of-type(1){display:none}}@media(max-width:760px){:global(body){overflow:auto}.app{height:auto;min-height:100vh}.shell{display:block}.library{display:none}.workbench{height:calc(100vh - 58px)}.topbar{padding:0 8px}.project,.badge,.actions .subtle{display:none}.tabs{overflow:auto}.tabs button{flex:0 0 auto}.metrics{grid-template-columns:repeat(2,1fr)}.fab-grid{grid-template-columns:1fr}.component{transform:scale(.85)}.board{width:88%}}
</style>
