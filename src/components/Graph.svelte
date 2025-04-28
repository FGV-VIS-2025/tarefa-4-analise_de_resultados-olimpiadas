<script>
import { onMount } from 'svelte';
import * as d3 from 'd3';

export let csvUrl;
export let measure;
export let filter = '';
export let valueTypes = [];

const m = { t: 60, r: 20, b: 60, l: 60 };
const W = 900, H = 500;

let svgEl, rawData = [], filteredData = [], groups = [], legendGroups = [];
let x, y, col;
let activeSport = null;
let ttVisible = false;
let tt = { x:0, y:0, sport:'', athlete:'', value:0, year:0 };

// variáveis de info-card
let introYear, recordAthlete, recordValue, recordYear;

onMount(async () => {
    rawData = await d3.csv(csvUrl);
    valueTypes = [...new Set(rawData.map(d => d.value_type))].filter(Boolean);
    if (valueTypes.length && !measure) measure = valueTypes[0];
    applyFilters();
});

$: if (rawData.length || measure || filter) applyFilters();

function applyFilters() {
    filteredData = rawData.filter(d =>
    d.medal_type === 'GOLD' &&
    !isNaN(+d.value_unit) &&
    (measure ? d.value_type === measure : true)
    );
    if (filter.trim()) {
    const term = filter.trim().toLowerCase();
    filteredData = filteredData.filter(d =>
        d.event_title.toLowerCase().includes(term)
    );
    }
    groups       = Array.from(d3.group(filteredData, d => d.event_title));
    legendGroups = groups.map(([ev]) => ev);

    // escala de cor fixa pra todos
    col = d3.scaleOrdinal()
    .domain(legendGroups)
    .range(d3.schemeTableau10);

    draw();
}

function selectSport(evName) {
    activeSport = activeSport === evName ? null : evName;
    ttVisible   = false;
    calcSportInfo();
    draw();
}

// calcula introYear e record do esporte ativo
function calcSportInfo() {
    if (!activeSport) return;
    const data = rawData.filter(d =>
    d.event_title === activeSport &&
    d.medal_type === 'GOLD' &&
    !isNaN(+d.value_unit)
    );
    introYear = d3.min(data, d => +d.ano);
    const rec = data.reduce((a,b) => +b.value_unit > +a.value_unit ? b : a);
    recordAthlete = rec.athlete_full_name;
    recordValue   = rec.value_unit;
    recordYear    = rec.ano;
}

function draw() {
    if (!svgEl) return;
    const svg = d3.select(svgEl);
    svg.selectAll('*').remove();

    const displayGroups = activeSport
    ? groups.filter(([ev]) => ev === activeSport)
    : groups;

    if (!displayGroups.length) {
    svg.append('text')
        .attr('x', W/2).attr('y', H/2)
        .attr('text-anchor','middle')
        .text('Sem dados');
    return;
    }

    const years = displayGroups.flatMap(([, v]) => v.map(d => +d.ano));
    const vals  = displayGroups.flatMap(([, v]) => v.map(d => +d.value_unit));

    x = d3.scaleLinear()
    .domain(d3.extent(years)).range([m.l, W-m.r]);
    y = d3.scaleLinear()
    .domain([0, d3.max(vals)]).nice().range([H-m.b, m.t]);

    const line = d3.line()
    .x(d => x(d.year))
    .y(d => y(d.val));

    // título
    svg.append('text')
    .attr('x', W/2).attr('y', 30)
    .attr('text-anchor','middle')
    .attr('font-size','20px')
    .text(`Olympic (${measure})`);

    displayGroups.forEach(([ev, rows]) => {
    const serie = rows.map(r => ({
        year: +r.ano, val: +r.value_unit, raw: r, event: ev
    })).sort((a,b)=>a.year-b.year);

    const maxPoint = serie.reduce((a,b)=> b.val>a.val?b:a);

    // linha
    svg.append('path')
        .datum(serie)
        .attr('fill','none')
        .attr('stroke',col(ev))
        .attr('stroke-width',2)
        .style('cursor','pointer')
        .on('click', evt => { evt.stopPropagation(); selectSport(ev) })
        .attr('d', line);

    // pontos
    if (activeSport===ev) {
        svg.selectAll(null)
        .data(serie)
        .enter().append('circle')
        .attr('cx', d=>x(d.year))
        .attr('cy', d=>y(d.val))
        .attr('r', d=> d===maxPoint?6:3)
        .attr('fill', d=> d===maxPoint?'white':col(ev))
        .attr('stroke', d=> d===maxPoint?col(ev):null)
        .attr('stroke-width', d=> d===maxPoint?2:0)
        .style('cursor','pointer')
        .on('click', (event, d) => {
        event.stopPropagation();
        showTooltip(event, d);
        });
    }
    });

    // eixos
    svg.append('g')
    .attr('transform',`translate(0,${H-m.b})`)
    .call(d3.axisBottom(x).ticks(10).tickFormat(d3.format('d')));
    svg.append('g')
    .attr('transform',`translate(${m.l},0)`)
    .call(d3.axisLeft(y));
}

function showTooltip(event, d) {
  tt = {
    x: event.clientX,
    y: event.clientY,
    sport: d.event,
    athlete: d.raw.athlete_full_name,
    value: d.val,
    year: d.year
  };
  ttVisible = true;
}

function handleClickOutside() {
    activeSport = null;
    ttVisible   = false;
    draw();
}

function getNoDataMessage(){ return ''; }
</script>
  
<style>
  .dashboard {
    display: grid;
    grid-template-columns: 200px 1fr 200px;
    gap: 1rem;
    align-items: start;
    justify-content: center;  
    max-width: 1200px;         
    margin: 0 auto;          
    padding: 1rem;
  }

  .info-card,
  .chart-container,
  .legend {
    background: #fff;
    padding: 1rem;
    border-radius: 8px;
    box-shadow: 0 2px 10px rgba(0,0,0,.1);
  }

  .info-card {
    font-size: 0.7rem;
  }

  .legend {
    max-height: 500px;
    overflow-y: auto;
    font-size: .85rem;
  }

  .legend-item {
    display: flex;
    align-items: center;
    font-size: 0.65rem;
    margin-bottom: 6px;
    padding: 1px 2px;
    border-radius: 4px;
    cursor: pointer;
    transition: background-color .2s, transform .2s;
  }
  .legend-item:hover {
    background-color: rgba(0,0,0,.05);
  }
  .legend-item.active {
    background-color: rgba(212,175,55,0.2);
    transform: scale(1.03);
  }

  .swatch {
    width: 14px;
    height: 14px;
    border-radius: 3px;
    margin-right: 6px;
    transition: transform .2s, opacity .2s;
    opacity: 0.8;
  }
  .legend-item.active .swatch {
    transform: scale(1.4);
    opacity: 1;
  }

  .tt {
  position: fixed;
  background: white;
  border: 1px solid #ccc;
  border-radius: 4px;
  padding: 0.5rem;
  font-size: 0.8rem;
  box-shadow: 0 2px 6px rgba(0,0,0,0.15);
  pointer-events: none;
  z-index: 1000;
  max-width: 150px;
}

</style>
  
<div class="dashboard" on:click={handleClickOutside}>
<!-- Info-card -->
<aside class="info-card">
    {#if activeSport}
    <h2>{activeSport}</h2>
    <p><b>Introduzido em:</b> {introYear}</p>
    <p><b>Recorde:</b> {recordValue} ({measure.toLowerCase()})</p>
    <p><b>Atleta:</b> {recordAthlete} em {recordYear}</p>
    {:else}
    <p>Clique na legenda ou no gráfico para ver detalhes.</p>
    {/if}
</aside>

<!-- Gráfico -->
<div class="chart-container">
    <svg bind:this={svgEl} width={W} height={H}></svg>
</div>

<!-- Legenda -->
{#if legendGroups.length}
<aside class="legend">
    {#each legendGroups as ev}
    <div
        class="legend-item"
        class:active={activeSport === ev}
        on:click|stopPropagation={() => selectSport(ev)}>
        <span class="swatch" style="background:{col(ev)}"></span>
        {ev}
    </div>
    {/each}
</aside>
{/if}

{#if ttVisible}
  <aside class="tt" style="left:{tt.x + 10}px; top:{tt.y + 10}px">
    <b>{tt.sport}</b><br>
    Ano: {tt.year}<br>
    Atleta: {tt.athlete}<br>
    Resultado: {tt.value}{measure==='TIME'?' s':measure==='DISTANCE'?' m':measure==='WEIGHT'?' kg':''}
  </aside>
{/if}
</div>
  