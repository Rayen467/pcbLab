<script>
  import { onMount } from 'svelte';

  const version = '0.4.0';
  const modes = ['Schematic', 'PCB', 'Simulator', '3D', 'BOM', 'Fabrication', 'Rules', 'Release'];
  const menus = ['File', 'Edit', 'View', 'Place', 'Route', 'Inspect', 'Tools', 'Manufacture'];
  const catalog = [
    { code: 'V', name: 'DC Source', value: '5 V', group: 'Sources', footprint: 'TerminalBlock_2P' },
    { code: 'R', name: 'Resistor', value: '330 Ω', group: 'Passives', footprint: 'R_0805' },
    { code: 'C', name: 'Capacitor', value: '1 µF', group: 'Passives', footprint: 'C_0805' },
    { code: 'L', name: 'Inductor', value: '10 µH', group: 'Passives', footprint: 'L_0805' },
    { code: 'D', name: 'Diode', value: '1N4148', group: 'Semiconductors', footprint: 'SOD-123' },
    { code: 'LED', name: 'LED', value: 'Red 2 V', group: 'Semiconductors', footprint: 'LED_0603' },
    { code: 'Q', name: 'N-MOSFET', value: '2N7002', group: 'Semiconductors', footprint: 'SOT-23' },
    { code: 'U', name: 'Op-Amp', value: 'LM358', group: 'IC', footprint: 'SOIC-8' },
    { code: 'SW', name: 'Switch', value: 'SPST', group: 'Input', footprint: 'SW_THT' },
    { code: 'J', name: 'Connector', value: '2 Pin', group: 'Connectors', footprint: 'HDR_1x02' },
    { code: 'GND', name: 'Ground', value: '0 V', group: 'Power', footprint: '—' },
    { code: 'TP', name: 'Test Point', value: 'TP', group: 'Debug', footprint: 'TestPoint_1mm' }
  ];

  const toolsets = {
    Schematic: ['Select', 'Place', 'Wire', 'Bus', 'Net label', 'Junction', 'No connect', 'Power', 'Measure', 'Annotate'],
    PCB: ['Select', 'Route', 'Via', 'Zone', 'Keepout', 'Dimension', 'Measure', 'Tune', 'Layer swap', 'Ratsnest'],
    Simulator: ['Run', 'Stop', 'Probe', 'Cursor A', 'Cursor B', 'Add trace', 'Measurements'],
    '3D': ['Orbit', 'Pan', 'Zoom', 'Measure', 'Section', 'Explode', 'Reset'],
    BOM: ['Refresh', 'Group', 'MPN', 'Supplier', 'Cost', 'Export CSV'],
    Fabrication: ['Preflight', 'Gerber', 'Drill', 'Pick & Place', 'Assembly', 'Archive'],
    Rules: ['Electrical', 'Clearance', 'Track width', 'Via', 'Differential', 'Mask', 'Silkscreen'],
    Release: ['Snapshot', 'Compare', 'Tag', 'Notes', 'Package']
  };

  let mode = 'Schematic';
  let activeTool = 'Select';
  let leftTab = 'Library';
  let rightOpen = true;
  let leftOpen = true;
  let query = '';
  let zoom = 100;
  let grid = 10;
  let selectedId = 'R2';
  let routed = false;
  let drcErrors = 2;
  let running = false;
  let consoleOpen = true;
  let toast = '';
  let savedAt = 'Unsaved';
  let simType = 'Operating Point';
  let activeLayer = 'F.Cu';
  let showRatsnest = true;
  let layerVisibility = { 'F.Cu': true, 'B.Cu': true, 'F.Silk': true, 'Edge.Cuts': true, 'Ratsnest': true };
  let patchLog = [
    { v: '0.4.0', title: 'Workspace Pro', text: 'Click-to-place, responsive panels, expanded EDA toolbars, rules and release center.' },
    { v: '0.3.1', title: 'Deploy fix', text: 'SvelteKit app shell and Vercel build repair.' },
    { v: '0.3.0', title: 'EDA flow', text: 'Schematic, PCB, simulator, 3D, BOM and fabrication workspaces.' }
  ];

  let counters = { V: 2, R: 3, C: 1, L: 1, D: 4, LED: 4, Q: 1, U: 1, SW: 1, J: 1, GND: 5, TP: 1 };
  let components = [
    { id: 'V1', code: 'V', name: 'DC Source', value: '5 V', footprint: 'TerminalBlock_2P', sx: 17, sy: 48, px: 16, py: 58, rot: 0 },
    { id: 'R2', code: 'R', name: 'Resistor', value: '330 Ω', footprint: 'R_0805', sx: 47, sy: 28, px: 46, py: 28, rot: 0 },
    { id: 'D3', code: 'LED', name: 'LED', value: 'Red 2 V', footprint: 'LED_0603', sx: 78, sy: 47, px: 76, py: 58, rot: 0 },
    { id: 'G4', code: 'GND', name: 'Ground', value: '0 V', footprint: '—', sx: 48, sy: 72, px: 50, py: 76, rot: 0 }
  ];

  const nets = [
    { name: 'VCC', pins: 'V1.1, R2.1', color: '#e2b95d' },
    { name: 'N_LED', pins: 'R2.2, D3.1', color: '#40d8bd' },
    { name: 'GND / 0', pins: 'D3.2, V1.2', color: '#8ba1b5' }
  ];

  $: filteredCatalog = catalog.filter((p) => `${p.code} ${p.name} ${p.value} ${p.group}`.toLowerCase().includes(query.toLowerCase()));
  $: selectedPart = components.find((p) => p.id === selectedId) || null;
  $: pcbParts = components.filter((p) => p.footprint && p.footprint !== '—');
  $: routePercent = routed ? 100 : Math.min(88, Math.max(28, pcbParts.length * 14));
  $: currentTools = toolsets[mode] || [];

  onMount(() => {
    try {
      const raw = localStorage.getItem('pcblab-project-v04');
      if (raw) {
        const saved = JSON.parse(raw);
        if (Array.isArray(saved.components) && saved.components.length) components = saved.components;
        if (saved.savedAt) savedAt = saved.savedAt;
      }
    } catch (e) {
      console.warn('PCB Lab restore skipped', e);
    }
  });

  function notify(message) {
    toast = message;
    window.clearTimeout(notify.t);
    notify.t = window.setTimeout(() => toast = '', 2200);
  }

  function selectMode(next) {
    mode = next;
    activeTool = toolsets[next]?.[0] || 'Select';
  }

  function addPart(part) {
    const n = counters[part.code] || 1;
    counters = { ...counters, [part.code]: n + 1 };
    const id = `${part.code}${n}`;
    const i = components.length;
    const next = {
      id,
      code: part.code,
      name: part.name,
      value: part.value,
      footprint: part.footprint,
      sx: 24 + ((i * 11) % 55),
      sy: 32 + ((i * 9) % 40),
      px: 20 + ((i * 13) % 58),
      py: 26 + ((i * 12) % 48),
      rot: 0
    };
    components = [...components, next];
    selectedId = id;
    notify(`${id} ditambahkan langsung · klik area kerja untuk memindahkan`);
  }

  function removeSelected() {
    if (!selectedPart) return;
    components = components.filter((p) => p.id !== selectedId);
    selectedId = components[0]?.id || '';
    notify('Komponen dihapus');
  }

  function rotateSelected() {
    if (!selectedPart) return;
    components = components.map((p) => p.id === selectedId ? { ...p, rot: (p.rot + 90) % 360 } : p);
    notify(`${selectedId} diputar 90°`);
  }

  function placeSelected(event, surface) {
    if (!selectedPart || event.target.closest('.eda-node, .footprint')) return;
    const rect = event.currentTarget.getBoundingClientRect();
    const x = Math.max(5, Math.min(92, ((event.clientX - rect.left) / rect.width) * 100));
    const y = Math.max(8, Math.min(88, ((event.clientY - rect.top) / rect.height) * 100));
    components = components.map((p) => p.id === selectedId ? { ...p, [surface === 'pcb' ? 'px' : 'sx']: x, [surface === 'pcb' ? 'py' : 'sy']: y } : p);
  }

  function updateSelected(field, value) {
    if (!selectedPart) return;
    components = components.map((p) => p.id === selectedId ? { ...p, [field]: value } : p);
  }

  function runDrc() {
    drcErrors = routed ? 0 : Math.max(1, Math.min(6, pcbParts.length - 1));
    notify(drcErrors === 0 ? 'DRC bersih · 0 pelanggaran' : `DRC: ${drcErrors} item perlu diperiksa`);
  }

  function routeBoard() {
    routed = true;
    drcErrors = 0;
    notify('Interactive route state: complete');
  }

  function simulate() {
    running = true;
    mode = 'Simulator';
    activeTool = 'Run';
    setTimeout(() => {
      running = false;
      notify(`${simType} selesai · browser linear solver`);
    }, 650);
  }

  function saveProject() {
    const time = new Date().toLocaleTimeString('id-ID', { hour: '2-digit', minute: '2-digit' });
    savedAt = `Saved ${time}`;
    localStorage.setItem('pcblab-project-v04', JSON.stringify({ components, savedAt }));
    notify('Project tersimpan di browser');
  }

  function download(name, content, type = 'application/json') {
    const blob = new Blob([content], { type });
    const url = URL.createObjectURL(blob);
    const a = document.createElement('a');
    a.href = url;
    a.download = name;
    a.click();
    URL.revokeObjectURL(url);
  }

  function exportProject() {
    download('pcblab-project.json', JSON.stringify({ version, components, nets }, null, 2));
    notify('Project JSON diekspor');
  }

  function exportBom() {
    const rows = [['Ref','Part','Value','Footprint'], ...components.map((p) => [p.id, p.name, p.value, p.footprint])];
    download('pcblab-bom.csv', rows.map((r) => r.map((x) => `"${String(x).replaceAll('"','""')}"`).join(',')).join('\n'), 'text/csv');
    notify('BOM CSV diekspor');
  }

  function toggleLayer(name) {
    layerVisibility = { ...layerVisibility, [name]: !layerVisibility[name] };
    if (name === 'Ratsnest') showRatsnest = layerVisibility[name];
  }

  function toolAction(tool) {
    activeTool = tool;
    if (tool === 'Run') simulate();
    if (tool === 'Delete') removeSelected();
    if (tool === 'Rotate') rotateSelected();
    if (tool === 'Export CSV') exportBom();
    if (tool === 'Gerber' || tool === 'Drill' || tool === 'Pick & Place') notify(`${tool}: exporter pipeline disiapkan di Fabrication`);
  }

  function handleKey(event) {
    const tag = event.target?.tagName?.toLowerCase();
    if (tag === 'input' || tag === 'select' || tag === 'textarea') return;
    if (event.key === 'Delete') removeSelected();
    if (event.key.toLowerCase() === 'r') rotateSelected();
    if (event.key.toLowerCase() === 'w') { mode = 'Schematic'; activeTool = 'Wire'; }
    if (event.key.toLowerCase() === 'x') activeLayer = activeLayer === 'F.Cu' ? 'B.Cu' : 'F.Cu';
    if (event.ctrlKey && event.key.toLowerCase() === 's') { event.preventDefault(); saveProject(); }
  }
</script>

<svelte:window onkeydown={handleKey} />

<svelte:head>
  <title>PCB Lab — Engineering EDA Workspace</title>
  <meta name="description" content="PCB Lab engineering workspace for schematic capture, PCB layout, simulation, rules, BOM and fabrication." />
</svelte:head>

<div class="app">
  <header class="topbar">
    <div class="brandline">
      <button class="panel-toggle" onclick={() => leftOpen = !leftOpen} title="Toggle library">☰</button>
      <div class="brand-icon">⌁</div>
      <div class="brand">PCB<span>Lab</span></div>
      <span class="build">ENGINEERING BUILD · v{version}</span>
      <button class="project"><i></i><strong>Indikator LED · 5 V</strong><span>⌄</span></button>
    </div>
    <div class="actions">
      <span class="autosave">{savedAt}</span>
      <button class="btn subtle" onclick={() => notify('New project workspace ready')}>New</button>
      <button class="btn subtle" onclick={saveProject}>Save</button>
      <button class="btn subtle" onclick={exportProject}>Export</button>
      <button class="btn run" onclick={simulate}>{running ? 'Running…' : '▶ Simulate'}</button>
      <button class="panel-toggle" onclick={() => rightOpen = !rightOpen} title="Toggle inspector">☷</button>
    </div>
  </header>

  <div class="menubar">
    <div class="menu-items">
      {#each menus as item}<button onclick={() => notify(`${item} menu ready for command mapping`)}>{item}</button>{/each}
    </div>
    <div class="menu-state"><span class="live-dot"></span>Project local <b>•</b> Grid {grid} mil <b>•</b> {activeLayer}</div>
  </div>

  <div class:left-collapsed={!leftOpen} class:right-collapsed={!rightOpen} class="shell">
    <aside class="library">
      <div class="side-tabs">
        {#each ['Library','Project','History'] as tab}<button class:active={leftTab===tab} onclick={() => leftTab=tab}>{tab}</button>{/each}
      </div>

      {#if leftTab === 'Library'}
        <div class="side-head"><div><span>COMPONENT LIBRARY</span><h2>Parts</h2></div><b>{catalog.length}</b></div>
        <label class="search"><span>⌕</span><input bind:value={query} placeholder="Search symbol, value, group" /><kbd>/</kbd></label>
        <div class="library-note">Click a part to add it directly. No drag & drop.</div>
        <div class="parts">
          {#each filteredCatalog as part}
            <button class="part" onclick={() => addPart(part)} title={`Add ${part.name}`}>
              <span class="part-icon">{part.code}</span>
              <span><strong>{part.name}</strong><small>{part.value} · {part.group}</small></span>
              <em>＋</em>
            </button>
          {/each}
        </div>
      {:else if leftTab === 'Project'}
        <div class="tree-wrap">
          <div class="tree-title">PROJECT TREE</div>
          <button class="tree active">▾ ◫ Indikator LED · 5 V</button>
          <button class="tree indent">⌁ Sheet 01 · Main</button>
          <button class="tree indent">▦ PCB · Main Board</button>
          <button class="tree indent">◈ 3D · Assembly</button>
          <button class="tree indent">☷ BOM · {components.length} items</button>
          <button class="tree indent">⬡ Fabrication · Rev A</button>
          <div class="tree-title gap">OUTPUTS</div>
          <button class="tree indent">Gerber X2</button><button class="tree indent">NC Drill</button><button class="tree indent">BOM / CPL</button>
        </div>
      {:else}
        <div class="history-wrap">
          <div class="tree-title">PATCH HISTORY</div>
          {#each patchLog as p}
            <article class="patch"><div><b>v{p.v}</b><span>{p.title}</span></div><p>{p.text}</p></article>
          {/each}
        </div>
      {/if}
    </aside>

    <main class="workbench">
      <nav class="tabs" aria-label="EDA workspaces">
        {#each modes as item}
          <button class:active={mode === item} onclick={() => selectMode(item)}>
            <span>{item === 'Schematic' ? '⌁' : item === 'PCB' ? '▦' : item === 'Simulator' ? '∿' : item === '3D' ? '◈' : item === 'BOM' ? '☷' : item === 'Fabrication' ? '⬡' : item === 'Rules' ? '⚙' : '↟'}</span>{item}
          </button>
        {/each}
      </nav>

      <div class="toolbar-scroll">
        <div class="toolbar">
          <div class="tools">
            {#each currentTools as tool}
              <button class:active={activeTool===tool} onclick={() => toolAction(tool)} title={tool}>
                <span>{tool === 'Select' ? '↖' : tool === 'Wire' || tool === 'Route' ? '⌁' : tool === 'Via' ? '◉' : tool === 'Zone' ? '▧' : tool === 'Measure' ? '⌖' : tool === 'Run' ? '▶' : tool === 'Stop' ? '■' : tool === 'Probe' ? '⌁' : tool === 'Orbit' ? '⟳' : tool === 'Pan' ? '✥' : tool === 'Annotate' ? 'A' : '◇'}</span><small>{tool}</small>
              </button>
            {/each}
          </div>
          <div class="tool-right">
            <button class="rule" onclick={runDrc}><span class:bad={drcErrors>0}>●</span> DRC {drcErrors}</button>
            <button class="rule" onclick={() => notify('ERC: 0 errors · 1 advisory')}>✓ ERC</button>
            <button class="fit" onclick={() => zoom=100}>⛶ Fit</button>
          </div>
        </div>
      </div>

      <div class="content">
        {#if mode === 'Schematic'}
          <section class="canvas schematic" onclick={(e) => placeSelected(e, 'schematic')}>
            <div class="canvas-topline"><span>SCHEMATIC / MAIN</span><div><b>Snap {grid} mil</b><b>Orthogonal</b><b>ERC Live</b></div></div>
            <svg class="wires" viewBox="0 0 1000 620" preserveAspectRatio="none" aria-hidden="true">
              <polyline points="170,300 310,300 310,175 470,175" />
              <polyline points="530,175 710,175 710,300 810,300" />
              <polyline points="810,300 810,450 500,450 170,450 170,300" />
              <circle cx="310" cy="300" r="4"/><circle cx="710" cy="300" r="4"/>
            </svg>
            {#each components as part}
              <button class="eda-node" class:selected={selectedId===part.id} style={`left:${part.sx}%;top:${part.sy}%;transform:translate(-50%,-50%) rotate(${part.rot}deg)`} onclick={(e) => { e.stopPropagation(); selectedId = part.id; }}>
                <span class="ref">{part.id}</span>
                <div class="symbol">{part.code === 'R' ? '─[▰]─' : part.code === 'C' ? '─│ │─' : part.code === 'GND' ? '⏚' : part.code === 'LED' ? '─▷│↗' : part.code === 'V' ? '⊕' : part.code}</div>
                <b>{part.value}</b><small>{part.name}</small>
              </button>
            {/each}
            <div class="canvas-hud"><span><b>Click canvas</b> move selected</span><span><b>W</b> Wire</span><span><b>R</b> Rotate</span><span><b>Del</b> Delete</span></div>
          </section>
        {:else if mode === 'PCB'}
          <section class="canvas pcbstage" onclick={(e) => placeSelected(e, 'pcb')}>
            <div class="canvas-topline"><span>PCB EDITOR / MAIN</span><div><b>{activeLayer}</b><b>2 Layer</b><b>FR-4 1.6 mm</b><b>1 oz</b></div></div>
            <div class="layer-strip">
              {#each ['F.Cu','B.Cu','F.Silk','Edge.Cuts','Ratsnest'] as layer}
                <button class:off={!layerVisibility[layer]} class:active={activeLayer===layer} onclick={(e) => { e.stopPropagation(); if(layer==='F.Cu'||layer==='B.Cu') activeLayer=layer; toggleLayer(layer); }}><i class={`layer-dot ${layer.replace('.','-')}`}></i>{layer}</button>
              {/each}
            </div>
            <div class="board" class:routed onclick={(e) => placeSelected(e, 'pcb')}>
              <i class="hole h1"></i><i class="hole h2"></i><i class="hole h3"></i><i class="hole h4"></i>
              {#each pcbParts as part}
                <button class="footprint" class:selected={selectedId===part.id} style={`left:${part.px}%;top:${part.py}%;transform:translate(-50%,-50%) rotate(${part.rot}deg)`} onclick={(e) => { e.stopPropagation(); selectedId=part.id; }}>
                  <span>{part.id}</span><i></i><i></i><small>{part.footprint}</small>
                </button>
              {/each}
              {#if layerVisibility['F.Cu']}<div class="trace tr1"></div><div class="trace tr2"></div>{/if}
              {#if layerVisibility['B.Cu']}<div class="trace bottom tr3"></div>{/if}
              {#if showRatsnest && !routed}<svg class="rats" viewBox="0 0 700 390"><line x1="125" y1="225" x2="330" y2="115"/><line x1="330" y1="115" x2="555" y2="235"/></svg>{/if}
            </div>
            <div class="pcb-sidecard"><span>ROUTING STATUS</span><strong>{routePercent}%</strong><small>{routed ? '0 unrouted' : `${Math.max(1, pcbParts.length-1)} unrouted connections`}</small><div class="progress"><i style={`width:${routePercent}%`}></i></div><button onclick={(e) => {e.stopPropagation(); routeBoard();}}>Complete route</button></div>
          </section>
        {:else if mode === 'Simulator'}
          <section class="panel-view simulator">
            <div class="view-title"><div><span>SIMULATION WORKBENCH</span><h2>{simType}</h2><p>Local browser solver · circuit model view</p></div><div class="sim-actions"><select bind:value={simType}><option>Operating Point</option><option>Transient</option><option>AC Sweep</option><option>DC Sweep</option></select><button class="btn run" onclick={simulate}>▶ Run</button></div></div>
            <div class="metrics"><article><span>V(source)</span><strong>5.000 <i>V</i></strong><small>DC source</small></article><article><span>I(R2)</span><strong>9.091 <i>mA</i></strong><small>Series current</small></article><article><span>P(R2)</span><strong>27.27 <i>mW</i></strong><small>Dissipation</small></article><article><span>V(D3)</span><strong>2.000 <i>V</i></strong><small>LED model</small></article></div>
            <div class="scope"><div class="scope-head"><span>Waveform viewer</span><div><b class="cyan"></b>V(in)<b class="purple"></b>V(led)</div></div><svg viewBox="0 0 900 300"><path class="wave a" d="M0 240 C80 235 90 70 175 65 S280 65 340 65 S480 65 560 65 S700 65 900 65"/><path class="wave b" d="M0 245 C100 242 150 180 240 175 S420 174 520 174 S700 174 900 174"/></svg><div class="scope-footer"><span>Cursor A 2.50 ms / 4.92 V</span><span>ΔT 1.20 ms</span><span>ΔV 2.91 V</span></div></div>
          </section>
        {:else if mode === '3D'}
          <section class="panel-view three"><div class="view-title"><div><span>3D ASSEMBLY</span><h2>Board mechanical preview</h2><p>Assembly orientation, board thickness and component envelope.</p></div><div class="sim-actions"><button class="btn subtle">Top</button><button class="btn subtle">Bottom</button><button class="btn subtle">Reset camera</button></div></div><div class="three-scene"><div class="board3d"><div class="chip3d c1">J1</div><div class="chip3d c2">R2</div><div class="chip3d c3">LED</div><div class="chip3d c4">U1</div></div><div class="axis">Z ↑<br/><span>Y ↙ · X ↗</span></div><div class="three-info"><b>Board</b><span>70 × 39 mm</span><b>Thickness</b><span>1.6 mm</span><b>Max Z</b><span>8.4 mm</span></div></div></section>
        {:else if mode === 'BOM'}
          <section class="panel-view"><div class="view-title"><div><span>BILL OF MATERIALS</span><h2>Project components</h2><p>Part mapping, footprint verification and sourcing metadata.</p></div><button class="btn subtle" onclick={exportBom}>Export CSV</button></div><div class="table-wrap"><table><thead><tr><th>Ref</th><th>Part</th><th>Value</th><th>Footprint</th><th>Qty</th><th>Status</th></tr></thead><tbody>{#each components as part}<tr><td>{part.id}</td><td>{part.name}</td><td>{part.value}</td><td>{part.footprint}</td><td>1</td><td><span class:warn={part.footprint==='—'} class="ok">{part.footprint==='—' ? 'Virtual' : 'Mapped'}</span></td></tr>{/each}</tbody></table></div></section>
        {:else if mode === 'Fabrication'}
          <section class="panel-view fabrication"><div class="view-title"><div><span>MANUFACTURING</span><h2>Fabrication package</h2><p>Pre-flight and manufacturing outputs in one release flow.</p></div><button class="btn run" onclick={() => notify(routed ? 'Preflight passed · package manifest ready' : 'Finish routing before package generation')}>Run preflight</button></div><div class="fab-grid"><article><span>01</span><h3>DRC / Preflight</h3><p>Clearance, track width, drills, shorts and board edge.</p><b class:done={routed}>{routed ? '✓ Ready' : 'Routing required'}</b></article><article><span>02</span><h3>Gerber X2 + Drill</h3><p>Copper, mask, silkscreen, paste and Excellon mapping.</p><b>Exporter pipeline</b></article><article><span>03</span><h3>BOM + CPL</h3><p>Assembly part list and placement coordinates.</p><b>{pcbParts.length} placed footprints</b></article><article><span>04</span><h3>Release archive</h3><p>Revision notes, checksum and manufacturing snapshot.</p><b>Rev A</b></article></div></section>
        {:else if mode === 'Rules'}
          <section class="panel-view"><div class="view-title"><div><span>DESIGN RULES</span><h2>Board constraints</h2><p>Central rule stack for routing and fabrication.</p></div><button class="btn run" onclick={runDrc}>Run DRC</button></div><div class="rules-grid"><label><span>Minimum clearance</span><input value="0.20 mm" /></label><label><span>Minimum track width</span><input value="0.20 mm" /></label><label><span>Preferred track width</span><input value="0.25 mm" /></label><label><span>Via diameter / drill</span><input value="0.60 / 0.30 mm" /></label><label><span>Copper to edge</span><input value="0.30 mm" /></label><label><span>Silkscreen clearance</span><input value="0.15 mm" /></label></div><div class="rule-summary"><article><b>Electrical</b><span>Net class · power · differential pairs</span><strong>3 rules</strong></article><article><b>Physical</b><span>Width · clearance · hole · edge</span><strong>6 rules</strong></article><article><b>Manufacturing</b><span>Mask · paste · silk</span><strong>4 rules</strong></article></div></section>
        {:else}
          <section class="panel-view"><div class="view-title"><div><span>RELEASE CENTER</span><h2>Development & project releases</h2><p>Track PCB project snapshots and PCB Lab application patches.</p></div><button class="btn subtle" onclick={() => notify('Snapshot Rev A created locally')}>Create snapshot</button></div><div class="release-grid"><div class="release-card"><span>CURRENT BUILD</span><h3>PCB Lab v{version}</h3><p>Professional workspace pass: click-to-place, responsive layout, expanded tools and rule/release workspaces.</p><div class="chips"><b>SvelteKit</b><b>Vercel</b><b>Local save</b></div></div><div class="release-card"><span>PROJECT REVISION</span><h3>Indikator LED · Rev A</h3><p>{components.length} symbols · {pcbParts.length} footprints · {nets.length} nets · DRC {drcErrors}</p><div class="chips"><b>Board 70×39</b><b>2 layers</b><b>FR-4</b></div></div></div><div class="timeline">{#each patchLog as p}<article><b>v{p.v}</b><div><h4>{p.title}</h4><p>{p.text}</p></div></article>{/each}</div></section>
        {/if}
      </div>

      <footer class="statusbar"><div><span class="green-dot"></span><b>PCB Lab Core</b><span>{mode} workspace</span><span>{components.length} symbols</span><span>{nets.length} nets</span></div><div><button onclick={() => grid = grid === 10 ? 5 : 10}>Grid {grid} mil</button><button onclick={() => zoom=Math.max(50,zoom-10)}>−</button><b>{zoom}%</b><button onclick={() => zoom=Math.min(200,zoom+10)}>＋</button><button onclick={() => consoleOpen=!consoleOpen}>{consoleOpen ? 'Hide log' : 'Show log'}</button></div></footer>
      {#if consoleOpen}<div class="console"><span>[core]</span> v{version} <span>[project]</span> {components.length} symbols / {pcbParts.length} footprints <span>[erc]</span> 0 errors <span>[drc]</span> {drcErrors} findings <span>[sim]</span> local linear solver <span>[fab]</span> {routed ? 'preflight eligible' : 'routing incomplete'}</div>{/if}
    </main>

    <aside class="inspector">
      <section>
        <div class="inspector-title"><span>PROPERTIES</span><b>⋯</b></div>
        {#if selectedPart}
          <div class="selection"><small>SELECTED</small><h3>{selectedPart.id}</h3><span>{selectedPart.name}</span></div>
          <label class="field"><span>Reference</span><input value={selectedPart.id} oninput={(e) => { const old=selectedId; const val=e.currentTarget.value; components=components.map((p)=>p.id===old?{...p,id:val}:p); selectedId=val; }} /></label>
          <label class="field"><span>Value</span><input value={selectedPart.value} oninput={(e) => updateSelected('value', e.currentTarget.value)} /></label>
          <label class="field"><span>Footprint</span><input value={selectedPart.footprint} oninput={(e) => updateSelected('footprint', e.currentTarget.value)} /></label>
          <div class="prop-actions"><button onclick={rotateSelected}>↻ Rotate</button><button onclick={() => updateSelected('rot',0)}>0°</button><button class="danger" onclick={removeSelected}>Delete</button></div>
        {:else}<div class="empty-side">Select a component to inspect.</div>{/if}
      </section>

      <section>
        <div class="inspector-title"><span>DESIGN CHECKS</span><b class:good>{drcErrors===0 ? 'A' : '!'}</b></div>
        <div class="checks"><button onclick={() => notify('ERC: 0 errors')}><i class="pass">✓</i><span><b>ERC</b><small>Electrical rules clean</small></span></button><button onclick={runDrc}><i class:warn={drcErrors>0} class="pass">{drcErrors>0?'!':'✓'}</i><span><b>DRC</b><small>{drcErrors} findings</small></span></button><button><i class="pass">✓</i><span><b>Footprints</b><small>{pcbParts.length} physical mappings</small></span></button></div>
      </section>

      <section>
        <div class="inspector-title"><span>NET INSPECTOR</span><b>{nets.length}</b></div>
        <div class="net-list">{#each nets as net}<button onclick={() => notify(`${net.name}: ${net.pins}`)}><i style={`background:${net.color}`}></i><span><b>{net.name}</b><small>{net.pins}</small></span><em>›</em></button>{/each}</div>
      </section>

      <section>
        <div class="inspector-title"><span>LAYER STACK</span><b>2 Cu</b></div>
        <div class="layer-list">{#each ['F.Cu','B.Cu','F.Silk','Edge.Cuts','Ratsnest'] as layer}<button onclick={() => toggleLayer(layer)} class:off={!layerVisibility[layer]}><i class={`layer-dot ${layer.replace('.','-')}`}></i><span>{layer}</span><b>{layerVisibility[layer]?'ON':'OFF'}</b></button>{/each}</div>
      </section>
    </aside>
  </div>

  {#if toast}<div class="toast">✓ {toast}</div>{/if}
</div>

<style>
  :global(*){box-sizing:border-box} :global(html),:global(body){margin:0;width:100%;height:100%;background:#060b11;color:#dce6f2;font-family:Inter,ui-sans-serif,system-ui,-apple-system,BlinkMacSystemFont,"Segoe UI",sans-serif} :global(body){overflow:hidden} :global(button),:global(input),:global(select){font:inherit} :global(button){cursor:pointer} :global(:root){color-scheme:dark}
  .app{height:100dvh;min-height:0;background:radial-gradient(circle at 46% -20%,#123044 0,#09131d 34%,#070c12 64%);display:flex;flex-direction:column;overflow:hidden}.topbar{height:54px;flex:0 0 54px;display:flex;align-items:center;justify-content:space-between;gap:10px;padding:0 10px;border-bottom:1px solid #1c2b39;background:#08121cd9;backdrop-filter:blur(18px);z-index:10}.brandline,.actions{display:flex;align-items:center;gap:7px;min-width:0}.brand-icon{width:31px;height:31px;border-radius:8px;background:linear-gradient(145deg,#2de2c4,#0a8f82);display:grid;place-items:center;color:#031f1b;font-size:20px;font-weight:900;box-shadow:0 0 22px #16d5bc30;flex:none}.brand{font-weight:900;letter-spacing:-.045em;font-size:16px;white-space:nowrap}.brand span{color:#49dcc7}.build{font-size:8px;letter-spacing:.1em;font-weight:800;padding:4px 6px;border:1px solid #285146;border-radius:5px;color:#59d9b9;background:#0b211b;white-space:nowrap}.project{border:0;background:transparent;color:#afc2d2;display:flex;align-items:center;gap:7px;padding:7px 8px;border-radius:7px;min-width:0}.project:hover{background:#101c28}.project i{width:6px;height:6px;border-radius:50%;background:#38d7a4;box-shadow:0 0 10px #38d7a4;flex:none}.project strong{font-size:11px;white-space:nowrap;overflow:hidden;text-overflow:ellipsis}.project span{font-size:11px}.autosave{color:#657a8e;font-size:9px;margin-right:4px;white-space:nowrap}.btn,.panel-toggle{border:1px solid #273848;background:#101a25;color:#c4d1dc;padding:7px 10px;border-radius:7px;font-size:10px;font-weight:750}.btn:hover,.panel-toggle:hover{border-color:#446079;background:#142333}.btn.subtle{background:#0d1620}.btn.run{background:linear-gradient(180deg,#1eb49b,#108373);border-color:#2bc9b0;color:#fff;box-shadow:0 4px 18px #13a68d30}.panel-toggle{width:31px;height:31px;padding:0;display:grid;place-items:center}.menubar{height:29px;flex:0 0 29px;border-bottom:1px solid #172532;background:#08111a;display:flex;align-items:center;justify-content:space-between;padding:0 10px;gap:10px;overflow:hidden}.menu-items{display:flex;min-width:0;overflow-x:auto;scrollbar-width:none}.menu-items button{border:0;background:transparent;color:#71869a;font-size:9px;padding:6px 8px;white-space:nowrap}.menu-items button:hover{color:#d1dee8;background:#101c28}.menu-state{font-size:8px;color:#53697c;display:flex;gap:6px;align-items:center;white-space:nowrap}.live-dot{width:6px;height:6px;border-radius:50%;background:#49d4ad;box-shadow:0 0 8px #49d4ad}.shell{display:grid;grid-template-columns:242px minmax(0,1fr) 258px;min-height:0;flex:1;transition:grid-template-columns .2s ease}.shell.left-collapsed{grid-template-columns:0 minmax(0,1fr) 258px}.shell.right-collapsed{grid-template-columns:242px minmax(0,1fr) 0}.shell.left-collapsed.right-collapsed{grid-template-columns:0 minmax(0,1fr) 0}.library,.inspector{background:#09121bf2;overflow:auto;min-width:0;min-height:0}.library{border-right:1px solid #1c2b38}.inspector{border-left:1px solid #1c2b38}.left-collapsed .library,.right-collapsed .inspector{overflow:hidden;border:0}.side-tabs{display:flex;position:sticky;top:0;background:#09131d;z-index:2;border-bottom:1px solid #1b2a37}.side-tabs button{flex:1;border:0;border-bottom:2px solid transparent;background:transparent;color:#60768a;font-size:8px;font-weight:800;padding:10px 4px}.side-tabs button.active{color:#d9e6ef;border-color:#38cdb6}.side-head{display:flex;justify-content:space-between;align-items:end;padding:15px 14px 8px}.side-head span,.tree-title,.view-title span,.inspector-title span{font-size:8px;letter-spacing:.14em;font-weight:850;color:#5c758b}.side-head h2{font-size:15px;margin:3px 0 0}.side-head>b,.inspector-title>b{font-size:9px;background:#112130;color:#88a6bd;border:1px solid #253c4d;border-radius:10px;padding:2px 7px}.search{display:flex;align-items:center;margin:7px 11px 8px;border:1px solid #223747;background:#0d1823;border-radius:8px;padding:0 8px;gap:7px;color:#698398}.search input{background:transparent;border:0;outline:0;color:#d7e2eb;min-width:0;width:100%;padding:8px 0;font-size:10px}.search kbd{font-size:8px;border:1px solid #324658;border-radius:4px;padding:1px 4px}.library-note{margin:0 12px 9px;padding:7px 8px;border:1px solid #1f423b;background:#0c211d;color:#6fcfba;border-radius:6px;font-size:8px;line-height:1.35}.parts{padding:0 7px 14px}.part{width:100%;border:1px solid transparent;background:transparent;color:#bdcad6;padding:6px;border-radius:7px;display:grid;grid-template-columns:34px 1fr 18px;align-items:center;text-align:left}.part:hover{background:#0f1d29;border-color:#213849}.part-icon{width:29px;height:29px;border:1px solid #2a4759;background:#102330;border-radius:6px;display:grid;place-items:center;color:#69ddc8;font:750 9px ui-monospace}.part strong{display:block;font-size:10px}.part small{display:block;color:#62798c;font-size:8px;margin-top:2px;white-space:nowrap;overflow:hidden;text-overflow:ellipsis}.part em{font-style:normal;color:#5a768c}.tree-wrap,.history-wrap{padding:14px 10px}.tree-title{margin:4px 4px 8px}.tree-title.gap{margin-top:20px}.tree{width:100%;text-align:left;border:1px solid transparent;background:transparent;color:#8fa4b5;padding:8px;border-radius:6px;font-size:9px}.tree:hover,.tree.active{background:#0f1d2a;border-color:#203646;color:#d6e3ec}.tree.indent{padding-left:21px}.patch{border:1px solid #1f3141;background:#0d1822;border-radius:8px;padding:9px;margin-bottom:8px}.patch div{display:flex;gap:7px;align-items:center}.patch b{font:800 9px ui-monospace;color:#50d5bd}.patch span{font-size:9px;font-weight:800}.patch p{margin:6px 0 0;color:#657b8e;font-size:8px;line-height:1.4}.workbench{min-width:0;min-height:0;display:flex;flex-direction:column;background:#09111a}.tabs{height:43px;flex:0 0 43px;display:flex;align-items:end;border-bottom:1px solid #1d2b38;background:#0a131d;padding:0 8px;overflow-x:auto;scrollbar-width:thin}.tabs button{height:100%;border:0;border-bottom:2px solid transparent;background:transparent;color:#6f8599;padding:0 11px;display:flex;gap:6px;align-items:center;font-size:9px;font-weight:750;white-space:nowrap}.tabs button.active{color:#e0edf5;border-bottom-color:#36cbb4;background:linear-gradient(0deg,#12332b68,transparent)}.tabs button span{font-size:12px}.toolbar-scroll{height:47px;flex:0 0 47px;overflow-x:auto;overflow-y:hidden;border-bottom:1px solid #182633;background:#0b141e;scrollbar-width:thin}.toolbar{min-width:max-content;height:46px;display:flex;justify-content:space-between;align-items:center;padding:0 9px;gap:14px}.tools,.tool-right{display:flex;align-items:center;gap:3px}.tools button,.fit,.rule{border:1px solid transparent;background:transparent;color:#758b9e;border-radius:6px;min-width:34px;height:31px;display:flex;align-items:center;justify-content:center;gap:5px;font-size:12px}.tools button:hover,.fit:hover,.rule:hover{background:#132331;border-color:#273d50;color:#d3e0e9}.tools button.active{background:#14362f;border-color:#286e60;color:#65dfc7}.tools small{font-size:7px;white-space:nowrap}.rule,.fit{font-size:8px;border-color:#203547;padding:0 8px}.rule span{color:#4fd2af}.rule span.bad{color:#ff826f}.content{flex:1;min-height:0;display:flex;overflow:hidden}.canvas{position:relative;flex:1;min-width:0;min-height:0;overflow:hidden;background-color:#0a131c;background-image:radial-gradient(#253746 1px,transparent 1px),radial-gradient(#162635 .7px,transparent .7px);background-size:20px 20px,10px 10px}.canvas-topline{position:absolute;z-index:3;left:12px;top:10px;right:12px;display:flex;justify-content:space-between;align-items:center;color:#607689;font:750 8px ui-monospace;pointer-events:none}.canvas-topline div{display:flex;gap:5px}.canvas-topline b{padding:4px 6px;border:1px solid #263745;border-radius:5px;background:#0b151fd9}.wires{position:absolute;inset:0;width:100%;height:100%;pointer-events:none}.wires polyline{fill:none;stroke:#64d9c4;stroke-width:2}.wires circle{fill:#68dec8}.eda-node{position:absolute;z-index:2;border:1px solid #20313e;background:#0b1721;color:#d5e2eb;border-radius:8px;min-width:94px;padding:8px 10px;box-shadow:0 9px 22px #0006}.eda-node:hover,.eda-node.selected{border-color:#31b7a4;box-shadow:0 0 0 1px #1d5d54,0 14px 34px #0008}.eda-node .ref{position:absolute;top:-14px;left:0;color:#8097a9;font:750 8px ui-monospace}.eda-node .symbol{height:22px;display:grid;place-items:center;color:#73decf;font:800 12px ui-monospace}.eda-node b{display:block;font:750 9px ui-monospace;color:#cbe1e6;margin-top:4px}.eda-node small{color:#60788c;font-size:7px}.canvas-hud{position:absolute;z-index:4;left:12px;bottom:10px;display:flex;gap:5px;flex-wrap:wrap}.canvas-hud span{font-size:7px;color:#5e7487;background:#09131ddd;border:1px solid #213545;border-radius:5px;padding:4px 6px}.canvas-hud b{color:#a1b4c3}.pcbstage{display:flex;align-items:center;justify-content:center}.layer-strip{position:absolute;z-index:5;left:12px;top:37px;display:flex;gap:4px;flex-wrap:wrap;max-width:70%}.layer-strip button,.layer-list button{border:1px solid #213746;background:#0b1721;color:#8094a5;border-radius:5px;font-size:7px;padding:4px 6px;display:flex;align-items:center;gap:5px}.layer-strip button.active{border-color:#c05848;color:#e3ad9d}.layer-strip button.off,.layer-list button.off{opacity:.42}.layer-dot{width:7px;height:7px;border-radius:2px;background:#cc5e4d}.layer-dot.B-Cu{background:#507fd1}.layer-dot.F-Silk{background:#e4e8ea}.layer-dot.Edge-Cuts{background:#d8b759}.layer-dot.Ratsnest{background:#65d8c3}.board{position:relative;width:min(72%,760px);max-height:72%;aspect-ratio:1.8;border:2px solid #d4b557;border-radius:7px;background:linear-gradient(140deg,#15503e,#0d392e);box-shadow:0 30px 70px #0009,inset 0 0 45px #09261e}.hole{position:absolute;width:12px;height:12px;border:2px solid #c6aa63;border-radius:50%;background:#09100f}.h1{left:12px;top:12px}.h2{right:12px;top:12px}.h3{left:12px;bottom:12px}.h4{right:12px;bottom:12px}.footprint{position:absolute;z-index:3;border:1px solid #dbbd66;background:#183f34dd;color:#e3cf8e;width:90px;height:56px;border-radius:4px;font:750 8px ui-monospace;text-align:center;padding-top:5px}.footprint.selected{outline:2px solid #59ddc6;outline-offset:3px}.footprint>i{position:absolute;width:10px;height:10px;border-radius:50%;border:2px solid #e2bd5f;background:#283827;bottom:7px}.footprint>i:first-of-type{left:18px}.footprint>i:last-of-type{right:18px}.footprint small{display:block;color:#9ab6a8;font-size:6px;margin-top:3px;white-space:nowrap;overflow:hidden;text-overflow:ellipsis}.trace{position:absolute;height:5px;background:#c75d4a;box-shadow:0 0 0 1px #9f4235;border-radius:5px;transform-origin:left center}.trace.bottom{background:#4b72c3;box-shadow:0 0 0 1px #395ea7}.tr1{left:19%;top:55%;width:31%;transform:rotate(-34deg)}.tr2{left:48%;top:29%;width:34%;transform:rotate(29deg)}.tr3{right:22%;bottom:30%;width:27%;transform:rotate(130deg)}.rats{position:absolute;inset:0;width:100%;height:100%;pointer-events:none}.rats line{stroke:#ead56a;stroke-width:1;stroke-dasharray:5 4;opacity:.82}.pcb-sidecard{position:absolute;z-index:4;right:13px;bottom:13px;width:172px;padding:11px;border:1px solid #253b4c;background:#09151fdd;border-radius:9px;backdrop-filter:blur(8px)}.pcb-sidecard span{font-size:7px;color:#5e788c}.pcb-sidecard strong{display:block;font-size:23px;color:#58dbc1;margin:3px 0}.pcb-sidecard small{font-size:7px;color:#778c9e}.progress{height:4px;background:#132330;border-radius:4px;margin:8px 0;overflow:hidden}.progress i{display:block;height:100%;background:#44cdb6}.pcb-sidecard button{width:100%;border:1px solid #2a6054;background:#12352c;color:#69dbc3;border-radius:6px;padding:6px;font-size:8px}.panel-view{flex:1;min-width:0;min-height:0;padding:20px;overflow:auto;background:linear-gradient(160deg,#0b141e,#091018)}.view-title{display:flex;align-items:center;justify-content:space-between;gap:10px;margin-bottom:16px}.view-title h2{margin:4px 0 2px;font-size:20px}.view-title p{margin:0;color:#607589;font-size:9px}.sim-actions{display:flex;gap:6px;align-items:center}.sim-actions select{background:#0d1924;border:1px solid #263b4c;color:#b9c9d7;border-radius:7px;padding:7px 9px;font-size:9px}.metrics{display:grid;grid-template-columns:repeat(4,minmax(130px,1fr));gap:9px}.metrics article,.fab-grid article{background:#0d1924;border:1px solid #203243;border-radius:10px;padding:13px}.metrics span{font-size:8px;color:#71889b}.metrics strong{display:block;font:750 21px ui-monospace;color:#dcebf3;margin:8px 0}.metrics strong i{font-size:9px;color:#66d7c2;font-style:normal}.metrics small{color:#51697c;font-size:7px}.scope{margin-top:11px;background:#08111a;border:1px solid #1d3040;border-radius:10px;overflow:hidden}.scope-head{height:37px;border-bottom:1px solid #1b2c3a;display:flex;align-items:center;justify-content:space-between;padding:0 11px;color:#778da1;font-size:8px}.scope-head div{display:flex;align-items:center;gap:6px}.scope-head b{width:8px;height:2px}.cyan{background:#46e4ca}.purple{background:#a57af5}.scope svg{width:100%;height:245px;background-image:linear-gradient(#132330 1px,transparent 1px),linear-gradient(90deg,#132330 1px,transparent 1px);background-size:45px 45px}.wave{fill:none;stroke-width:2}.wave.a{stroke:#47e4cb}.wave.b{stroke:#a97df7}.scope-footer{display:flex;gap:18px;padding:7px 11px;border-top:1px solid #1a2a38;color:#597083;font:7px ui-monospace}.three-scene{height:74%;min-height:360px;display:grid;place-items:center;perspective:900px;position:relative}.board3d{position:relative;width:min(56vw,500px);aspect-ratio:1.85;border:4px solid #bcae5e;border-radius:10px;background:linear-gradient(135deg,#17654a,#0b3d2d);transform:rotateX(58deg) rotateZ(-25deg);box-shadow:25px 45px 40px #0008,inset 0 0 70px #06241a}.chip3d{position:absolute;background:#111820;border:2px solid #506b5e;color:#9ecbb2;padding:16px 24px;border-radius:4px;box-shadow:12px 18px 14px #0008;font:750 10px ui-monospace}.chip3d.c1{left:8%;top:42%}.chip3d.c2{left:43%;top:16%}.chip3d.c3{right:9%;bottom:17%;background:#a52924;color:#ffd5ca}.chip3d.c4{right:38%;bottom:21%;padding:23px 32px}.axis{position:absolute;right:15px;bottom:15px;color:#5c7588;font:750 9px ui-monospace}.axis span{color:#809cb0}.three-info{position:absolute;left:12px;bottom:12px;display:grid;grid-template-columns:auto auto;gap:5px 12px;border:1px solid #213545;background:#09141ddd;border-radius:8px;padding:9px;font-size:7px}.three-info b{color:#60798d}.three-info span{color:#9eb2c2}.table-wrap{overflow:auto;border:1px solid #1f3140;border-radius:9px}.table-wrap table{width:100%;border-collapse:collapse;background:#0c1620;font-size:9px;min-width:650px}.table-wrap th,.table-wrap td{padding:10px 12px;border-bottom:1px solid #1b2b38;text-align:left}.table-wrap th{font-size:7px;color:#60778a;background:#0f1b27;letter-spacing:.08em}.table-wrap td{color:#a9bbc9}.ok,.warn{padding:3px 6px;border-radius:5px;font-size:7px}.ok{background:#0e2a22;color:#61d6b6}.warn{background:#332915;color:#e1bf6a}.fab-grid{display:grid;grid-template-columns:repeat(2,minmax(220px,1fr));gap:9px}.fab-grid article>span{color:#47647a;font:750 9px ui-monospace}.fab-grid h3{font-size:13px;margin:7px 0}.fab-grid p{color:#687e90;font-size:9px;line-height:1.45;min-height:28px}.fab-grid b{font-size:8px;color:#e1b865}.fab-grid b.done{color:#5bd2af}.rules-grid{display:grid;grid-template-columns:repeat(3,minmax(180px,1fr));gap:9px}.rules-grid label{display:block;border:1px solid #203243;background:#0d1924;border-radius:9px;padding:10px}.rules-grid span{display:block;font-size:8px;color:#6e8599;margin-bottom:7px}.rules-grid input{width:100%;background:#09131d;border:1px solid #263c4e;color:#c8d6e0;border-radius:6px;padding:8px;font:9px ui-monospace}.rule-summary{display:grid;grid-template-columns:repeat(3,1fr);gap:9px;margin-top:10px}.rule-summary article{border:1px solid #1f3140;background:#0b1620;border-radius:9px;padding:11px}.rule-summary b,.rule-summary span,.rule-summary strong{display:block}.rule-summary b{font-size:10px}.rule-summary span{font-size:8px;color:#647b8e;margin:5px 0}.rule-summary strong{font-size:8px;color:#57d5bd}.release-grid{display:grid;grid-template-columns:repeat(2,minmax(260px,1fr));gap:10px}.release-card{border:1px solid #213342;background:#0d1823;border-radius:11px;padding:15px}.release-card>span{font-size:7px;letter-spacing:.13em;color:#5d778c;font-weight:800}.release-card h3{margin:6px 0;font-size:15px}.release-card p{color:#687e90;font-size:9px;line-height:1.5}.chips{display:flex;gap:5px;flex-wrap:wrap}.chips b{font-size:7px;padding:4px 6px;border:1px solid #285146;background:#0c221d;color:#5fd3b6;border-radius:5px}.timeline{margin-top:12px;border-left:1px solid #264050;padding-left:14px}.timeline article{display:grid;grid-template-columns:54px 1fr;gap:8px;margin:0 0 12px}.timeline article>b{font:800 9px ui-monospace;color:#4ed6be}.timeline h4{font-size:10px;margin:0 0 3px}.timeline p{font-size:8px;color:#657c8e;margin:0}.statusbar{height:28px;flex:0 0 28px;border-top:1px solid #1b2a37;background:#08121a;display:flex;align-items:center;justify-content:space-between;padding:0 8px;gap:8px;overflow-x:auto;white-space:nowrap}.statusbar>div{display:flex;align-items:center;gap:8px;font-size:7px;color:#61788b}.statusbar button{border:0;background:transparent;color:#72899c;font-size:7px}.statusbar b{color:#91a8ba}.green-dot{width:6px;height:6px;border-radius:50%;background:#42d4aa}.console{height:27px;flex:0 0 27px;display:flex;align-items:center;gap:7px;padding:0 9px;overflow-x:auto;white-space:nowrap;background:#050b11;border-top:1px solid #13212c;color:#536a7d;font:7px ui-monospace}.console span{color:#4fcdb7}.inspector section{padding:12px;border-bottom:1px solid #1b2a37}.inspector-title{display:flex;align-items:center;justify-content:space-between;margin-bottom:9px}.inspector-title>b.good{color:#64d7b8;background:#0e2a22;border-color:#285649}.selection{border:1px solid #223847;background:#0c1822;border-radius:8px;padding:9px;margin-bottom:10px}.selection small{font-size:7px;color:#5c7387}.selection h3{margin:4px 0 1px;font:800 17px ui-monospace;color:#dce8f0}.selection span{font-size:8px;color:#6b8194}.field{display:block;margin:8px 0}.field span{display:block;font-size:7px;color:#60778a;margin-bottom:4px}.field input{width:100%;background:#0b1620;border:1px solid #263b4c;color:#c8d7e1;border-radius:6px;padding:7px 8px;font-size:9px;outline:none}.field input:focus{border-color:#36a894}.prop-actions{display:flex;gap:5px;margin-top:9px}.prop-actions button{flex:1;border:1px solid #263b4b;background:#0d1822;color:#91a6b6;border-radius:6px;padding:6px;font-size:7px}.prop-actions button.danger{color:#e18a7b;border-color:#51342f}.empty-side{color:#60778a;font-size:8px;padding:10px}.checks button,.net-list button{width:100%;border:0;background:transparent;color:#a8bac8;display:flex;align-items:center;gap:8px;text-align:left;padding:6px 0}.checks i{width:18px;height:18px;border-radius:50%;display:grid;place-items:center;font-style:normal;font-size:8px;background:#0d2a22;border:1px solid #25594c;color:#59d2b2}.checks i.warn{background:#332715;border-color:#654e20;color:#e5bd5e}.checks span,.net-list span{min-width:0;flex:1}.checks b,.net-list b{display:block;font-size:8px}.checks small,.net-list small{display:block;color:#60778a;font-size:7px;margin-top:2px;white-space:nowrap;overflow:hidden;text-overflow:ellipsis}.net-list>button>i{width:6px;height:6px;border-radius:50%;flex:none}.net-list em{font-style:normal;color:#536b7e}.layer-list{display:grid;gap:4px}.layer-list button{width:100%;justify-content:flex-start;padding:6px}.layer-list button span{flex:1;text-align:left}.layer-list button b{font-size:6px;color:#668196}.toast{position:fixed;z-index:50;right:18px;bottom:45px;background:#10251f;border:1px solid #2f6d5e;color:#75e0c7;border-radius:8px;padding:9px 12px;box-shadow:0 15px 45px #0008;font-size:9px;font-weight:750}
  @media(max-width:1180px){.shell{grid-template-columns:220px minmax(0,1fr) 230px}.build,.autosave{display:none}.rules-grid{grid-template-columns:repeat(2,minmax(180px,1fr))}.metrics{grid-template-columns:repeat(2,minmax(150px,1fr))}}
  @media(max-width:930px){.shell{grid-template-columns:0 minmax(0,1fr) 0}.library,.inspector{display:none}.project{max-width:190px}.topbar{padding:0 7px}.actions .btn.subtle:nth-of-type(1){display:none}.menu-state{display:none}.fab-grid,.release-grid,.rule-summary{grid-template-columns:1fr}.rules-grid{grid-template-columns:1fr 1fr}.board{width:86%}.pcb-sidecard{width:145px}.panel-view{padding:14px}}
  @media(max-width:650px){.project,.build{display:none}.actions{gap:4px}.btn{padding:7px 8px}.tabs button{padding:0 9px}.tools small{display:none}.metrics,.rules-grid{grid-template-columns:1fr}.view-title{align-items:flex-start;flex-direction:column}.scope svg{height:210px}.board{width:94%;max-height:64%}.pcb-sidecard{bottom:8px;right:8px}.canvas-topline div{display:none}.console{display:none}.panel-view{padding:11px}.three-scene{min-height:300px}.board3d{width:78vw}.scope-footer{gap:8px;overflow:auto}}
</style>