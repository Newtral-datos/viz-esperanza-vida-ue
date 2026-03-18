<script>
  import { onMount } from 'svelte';
  import * as d3 from 'd3';

  let svgEl, containerEl;
  let tooltip = {
    visible: false, x: 0, y: 0,
    nombre: '', pais: '',
    valor: null, valor_h: null, valor_m: null,
    serie: [], serie_h: [], serie_m: []
  };
  let legendData = [];
  let maxVal;
  let isMobile = false;

  const CW = 240, CH = 80, PAD = { l: 28, r: 8, t: 6, b: 18 };
  const IH = CH - PAD.t - PAD.b;
  const fmt = v => d3.format('.1f')(v).replace('.', ',');
  const GLOBAL_X1 = 1990, GLOBAL_X2 = 2024;
  let globalY1 = 65, globalY2 = 90;

  function buildPath(serie, px, py) {
    if (!serie?.length) return '';
    let path = '';
    for (let i = 0; i < serie.length; i++) {
      const [yr, v] = serie[i];
      const cmd = (i === 0 || yr - serie[i-1][0] > 1) ? 'M' : 'L';
      path += `${cmd}${px(yr).toFixed(1)},${py(v).toFixed(1)} `;
    }
    return path;
  }

  function lastPoint(serie, px, py) {
    if (!serie?.length) return null;
    const [yr, v] = serie[serie.length - 1];
    return { x: px(yr), y: py(v) };
  }

  $: {
    const iw = CW - PAD.l - PAD.r;
    const px = yr  => PAD.l + (yr - GLOBAL_X1) / (GLOBAL_X2 - GLOBAL_X1) * iw;
    const py = val => PAD.t + IH - (val - globalY1) / (globalY2 - globalY1) * IH;
    const mid = (globalY1 + globalY2) / 2;
    chart = {
      px, py,
      pathT:  buildPath(tooltip.serie,   px, py),
      pathH:  buildPath(tooltip.serie_h, px, py),
      pathM:  buildPath(tooltip.serie_m, px, py),
      lastT:  lastPoint(tooltip.serie,   px, py),
      lastH:  lastPoint(tooltip.serie_h, px, py),
      lastM:  lastPoint(tooltip.serie_m, px, py),
      ticks:  [1990, 2000, 2010, 2024].map(y => ({ x: px(y), label: y })),
      yticks: [globalY1, mid, globalY2].map(v => ({ y: py(v), label: String(v.toFixed(0)) })),
    };
  }
  let chart = {};

  const palette = ['#D2F7EC', '#A3F2DA', '#7CECC7', '#01f3b3', '#00A983', '#006C50'];

  // Datos cargados una sola vez; el mapa se redibuja si cambia el tamaño
  let _geojson, _world, _data, _valueMap, _colorScale;

  function drawMap(width, height) {
    if (!_geojson || !svgEl) return;

    const svg = d3.select(svgEl);
    svg.selectAll('*').remove();
    svg.attr('width', width).attr('height', height).style('cursor', 'grab');

    const projection = d3.geoMercator()
      .center([15, 52])
      .scale(Math.min(width, height) * (isMobile ? 0.68 : 0.59))
      .translate([width / 2, height / 2]);

    const path = d3.geoPath().projection(projection);
    const g = svg.append('g');

    g.append('g')
      .selectAll('path')
      .data(_world.features)
      .join('path')
        .attr('d', path)
        .attr('fill', '#d4d4d4')
        .attr('stroke', '#494949')
        .attr('stroke-width', 0.4);

    g.append('g')
      .selectAll('path')
      .data(_geojson.features)
      .join('path')
        .attr('d', path)
        .attr('fill', d => {
          const info = _valueMap.get(d.properties.NUTS_ID);
          return info?.valor != null ? _colorScale(info.valor) : '#ccc';
        })
        .attr('stroke', '#494949')
        .attr('stroke-width', isMobile ? 0.5 : 0.3)
        .on('pointermove', (event, d) => {
          if (event.pointerType === 'touch') return; // touch → usar pointerdown
          const info = _valueMap.get(d.properties.NUTS_ID);
          tooltip = {
            visible: true,
            x: event.clientX + 12, y: event.clientY - 28,
            nombre:  d.properties.NUTS_NAME || d.properties.NUTS_ID,
            pais:    info?.pais    ?? '',
            valor:   info?.valor   ?? null,
            valor_h: info?.valor_h ?? null,
            valor_m: info?.valor_m ?? null,
            serie:   info?.serie   ?? [],
            serie_h: info?.serie_h ?? [],
            serie_m: info?.serie_m ?? [],
          };
        })
        .on('pointerleave', (event) => {
          if (event.pointerType === 'touch') return;
          tooltip = { ...tooltip, visible: false };
        })
        .on('pointerdown', (event, d) => {
          if (event.pointerType !== 'touch') return;
          const info = _valueMap.get(d.properties.NUTS_ID);
          tooltip = {
            visible: true,
            x: 0, y: 0,
            nombre:  d.properties.NUTS_NAME || d.properties.NUTS_ID,
            pais:    info?.pais    ?? '',
            valor:   info?.valor   ?? null,
            valor_h: info?.valor_h ?? null,
            valor_m: info?.valor_m ?? null,
            serie:   info?.serie   ?? [],
            serie_h: info?.serie_h ?? [],
            serie_m: info?.serie_m ?? [],
          };
          event.stopPropagation();
        });

    const zoom = d3.zoom()
      .scaleExtent([1, 12])
      .on('zoom', (event) => {
        g.attr('transform', event.transform);
        svg.style('cursor', event.transform.k > 1 ? 'grabbing' : 'grab');
      });

    svg.call(zoom);
    svg.on('pointerdown.dismiss', (event) => {
      if (event.pointerType === 'touch' && event.target === svgEl) {
        tooltip = { ...tooltip, visible: false };
      }
    });

    d3.select('#reset-zoom').on('click', () => {
      svg.transition().duration(400).call(zoom.transform, d3.zoomIdentity);
    });
  }

  onMount(async () => {
    isMobile = window.matchMedia('(hover: none)').matches;

    const [geojson, world, data] = await Promise.all([
      fetch('https://gisco-services.ec.europa.eu/distribution/v2/nuts/geojson/NUTS_RG_20M_2021_4326_LEVL_2.geojson').then(r => r.json()),
      fetch('https://raw.githubusercontent.com/holtzy/D3-graph-gallery/master/DATA/world.geojson').then(r => r.json()),
      fetch(import.meta.env.BASE_URL + 'data.json').then(r => r.json())
    ]);

    _geojson = geojson;
    _world   = world;
    _data    = data;
    _valueMap = new Map(data.map(d => [d.nuts2, d]));

    const allVals = data.flatMap(d => [
      ...d.serie.map(s => s[1]),
      ...d.serie_h.map(s => s[1]),
      ...d.serie_m.map(s => s[1])
    ]);
    globalY1 = Math.floor(d3.min(allVals));
    globalY2 = Math.ceil(d3.max(allVals));

    const values = data.map(d => d.valor).filter(v => v != null);
    const min = d3.min(values);
    maxVal = d3.max(values);
    const step = (maxVal - min) / 5;
    legendData = palette.map((color, i) => ({ color, label: fmt(min + step * i) }));
    _colorScale = d3.scaleLinear()
      .domain(palette.map((_, i) => min + step * i))
      .range(palette);

    const ro = new ResizeObserver(entries => {
      const { width, height } = entries[0].contentRect;
      if (width > 0 && height > 0) drawMap(width, height);
    });
    ro.observe(containerEl);

    return () => ro.disconnect();
  });
</script>

<main>
  <header>
    <h1>Esperanza de vida por regiones de la UE</h1>
    <p class="subtitle">Con datos comparados entre hombres y mujeres</p>
    <div class="legend">
      {#each legendData as { color, label }, i}
        <div class="swatch">
          <div class="swatch-color" style="background:{color}"></div>
          <span>{label}{i === legendData.length - 1 && maxVal ? '–' + fmt(maxVal) : ''}</span>
        </div>
      {/each}
    </div>
  </header>

  <div class="map-container" bind:this={containerEl}>
    <svg bind:this={svgEl}></svg>
    <button id="reset-zoom" title="Resetear zoom">⊙</button>
  </div>

  <!-- Desktop: solo si visible | Mobile: siempre en DOM, se muestra/oculta -->
  {#if tooltip.visible || isMobile}
    <div
      class="tooltip"
      class:tooltip-mobile={isMobile}
      class:tooltip-hidden={isMobile && !tooltip.visible}
      style={isMobile ? '' : `left:${tooltip.x}px;top:${tooltip.y}px`}
    >
      {#if isMobile}
        <button class="tt-close" on:click={() => tooltip = { ...tooltip, visible: false }}>✕</button>
      {/if}
      <div class="tt-header">
        <strong>{tooltip.nombre}</strong>
        <span class="tt-pais">{tooltip.pais}</span>
      </div>
      <div class="tt-valores">
        <div class="tt-val-item">
          <span class="dot" style="background:#00A983"></span>
          <span class="tt-val-label">Total</span>
          <span class="tt-val-num" style="color:#01f3b3">{tooltip.valor != null ? fmt(tooltip.valor) : '—'}</span>
        </div>
        <div class="tt-val-item">
          <span class="dot" style="background:#6900ff"></span>
          <span class="tt-val-label">Mujeres</span>
          <span class="tt-val-num" style="color:#a855f7">{tooltip.valor_m != null ? fmt(tooltip.valor_m) : '—'}</span>
        </div>
        <div class="tt-val-item">
          <span class="dot" style="background:#eaea40"></span>
          <span class="tt-val-label">Hombres</span>
          <span class="tt-val-num" style="color:#eaea40">{tooltip.valor_h != null ? fmt(tooltip.valor_h) : '—'}</span>
        </div>
      </div>
      {#if chart.pathT || chart.pathH || chart.pathM}
        <svg width={CW} height={CH} style="display:block;margin-top:6px">
          {#each chart.yticks as t}
            <line x1={PAD.l} x2={CW - PAD.r} y1={t.y} y2={t.y} stroke="#555" stroke-width="0.4" stroke-dasharray="2,2"/>
            <text x={PAD.l - 3} y={t.y + 3.5} text-anchor="end" font-size="8" fill="#888">{t.label}</text>
          {/each}
          {#if chart.pathH}<path d={chart.pathH} fill="none" stroke="#eaea40" stroke-width="1.4" stroke-linejoin="round" opacity="0.9"/>{/if}
          {#if chart.pathM}<path d={chart.pathM} fill="none" stroke="#6900ff" stroke-width="1.4" stroke-linejoin="round" opacity="0.9"/>{/if}
          {#if chart.pathT}<path d={chart.pathT} fill="none" stroke="#00A983" stroke-width="1.8" stroke-linejoin="round"/>{/if}
          {#if chart.lastH}<circle cx={chart.lastH.x} cy={chart.lastH.y} r="2.5" fill="#eaea40"/>{/if}
          {#if chart.lastM}<circle cx={chart.lastM.x} cy={chart.lastM.y} r="2.5" fill="#6900ff"/>{/if}
          {#if chart.lastT}<circle cx={chart.lastT.x} cy={chart.lastT.y} r="3"   fill="#00A983"/>{/if}
          {#each chart.ticks as t}
            <line x1={t.x} x2={t.x} y1={PAD.t + IH} y2={PAD.t + IH + 3} stroke="#666" stroke-width="0.5"/>
            <text x={t.x} y={CH - 3} text-anchor="middle" font-size="8" fill="#888">{t.label}</text>
          {/each}
        </svg>
      {/if}
    </div>
  {/if}
</main>


<style>
  :global(*, *::before, *::after) { box-sizing: border-box; }
  :global(body) { margin: 0; padding: 0; font-family: sans-serif; background: white; overflow: hidden; }

  main {
    display: flex;
    flex-direction: column;
    width: 100vw;
    height: 100dvh;
    position: relative;
  }

  .tooltip-hidden { display: none; }

  /* Tooltip mobile: overlay pegado al fondo del main, encima del mapa */
  .tooltip-mobile {
    position: absolute !important;
    bottom: 0; left: 0; right: 0;
    border-radius: 12px 12px 0 0;
    border-top: none;
    padding: 14px 16px 20px;
    font-size: 14px;
    min-width: unset;
    z-index: 20;
  }

  header {
    display: flex;
    flex-direction: column;
    align-items: center;
    padding: 0.6rem 1rem 0.4rem;
    flex-shrink: 0;
  }

  h1 {
    font-size: clamp(0.85rem, 2.5vw, 1.2rem);
    color: #222;
    margin: 0 0 0.1rem;
    text-align: center;
  }
  .subtitle {
    font-size: clamp(0.62rem, 1.5vw, 0.75rem);
    color: #777;
    font-style: italic;
    margin: 0 0 0.4rem;
    text-align: center;
  }

  .legend {
    display: flex;
    flex-wrap: wrap;
    justify-content: center;
    gap: 4px;
  }

  .swatch { display: flex; flex-direction: column; align-items: center; gap: 2px; }
  .swatch-color { width: clamp(24px, 4vw, 36px); height: 10px; border-radius: 2px; }
  .swatch span { font-size: clamp(8px, 1.8vw, 10px); color: #444; }

  .map-container {
    flex: 1;
    position: relative;
    background: white;
    overflow: hidden;
    min-height: 0;
  }

  svg { display: block; }

  #reset-zoom {
    position: absolute;
    bottom: 12px; right: 12px;
    width: clamp(32px, 8vw, 40px);
    height: clamp(32px, 8vw, 40px);
    border: none; border-radius: 6px;
    background: white;
    box-shadow: 0 1px 4px rgba(0,0,0,0.25);
    cursor: pointer;
    font-size: clamp(14px, 4vw, 18px);
    touch-action: manipulation;
  }
  #reset-zoom:hover { background: #eee; }

  /* Tooltip desktop */
  .tooltip {
    position: fixed;
    background: rgba(20,20,20,0.93);
    color: white;
    padding: 8px 10px;
    border-radius: 6px;
    font-size: 12px;
    pointer-events: none;
    z-index: 10;
    min-width: 200px;
  }

  /* Tooltip mobile: panel inferior */
  .tooltip-mobile {
    position: fixed;
    bottom: 0; left: 0; right: 0;
    border-radius: 12px 12px 0 0;
    padding: 16px;
    pointer-events: all;
    font-size: 14px;
    min-width: unset;
    z-index: 20;
  }

  .tt-close {
    position: absolute;
    top: 10px; right: 12px;
    background: none; border: none;
    color: #aaa; font-size: 16px;
    cursor: pointer; padding: 4px;
  }

  .tt-header { display: flex; flex-direction: column; margin-bottom: 6px; }
  .tt-pais { font-size: 10px; color: #aaa; }

  .tt-valores { display: flex; flex-direction: column; gap: 3px; margin-bottom: 2px; }
  .tt-val-item { display: flex; align-items: center; gap: 6px; }
  .dot { width: 8px; height: 8px; border-radius: 50%; flex-shrink: 0; }
  .tt-val-label { flex: 1; color: #ccc; font-size: 11px; }
  .tt-val-num { font-weight: bold; font-size: 13px; }

  @media (max-width: 480px) {
    .tt-val-label { font-size: 13px; }
    .tt-val-num   { font-size: 15px; }
  }
</style>
