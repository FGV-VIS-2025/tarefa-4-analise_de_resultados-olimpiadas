<script>
	import { onMount, onDestroy } from 'svelte';
	import * as d3 from 'd3';

	/* props ---------------------------------------------------------- */
	export let csvUrl;
	export let measure     = '';
	export let searchQuery = '';      /* usado pela página */
	export let valueTypes  = [];

	/* layout --------------------------------------------------------- */
	const margin = { top: 60, right: 20, bottom: 60, left: 60 };
	const chartHeight = 500;      
	let chartWidth = 600;

	let svgRef, chartContainerRef, ro;   /* ResizeObserver */

	/* data & state --------------------------------------------------- */
	let rawData=[], filtered=[], groups=[], legend=[];
	let xScale,yScale,color;

	let activeEvent=null;
	let introYear, recordRow, recAth, recVal, recYear;
	let recPhoto='', recFlag='';

	/* hover ---------------------------------------------------------- */
	let hoverVisible=false;
	let hover={sport:'',athlete:'',year:0,value:0,country:'',photo:'',flag:''};
	const cache=new Map();

	/* --------------------------- mount ------------------------------ */
	onMount(async ()=>{
		rawData = await d3.csv(csvUrl);
		valueTypes=[...new Set(rawData.map(d=>d.value_type))].filter(Boolean);
		if(!measure && valueTypes.length) measure=valueTypes[0];

		updateWidth();
		if(typeof ResizeObserver!=='undefined'){
			ro=new ResizeObserver(updateWidth);
			ro.observe(chartContainerRef);
		}
		filterData();
	});
	onDestroy(()=>ro?.disconnect());

	/* reactivos */
	$: { rawData; measure; searchQuery; rawData.length && filterData(); }

	/* ------------------------- helpers ----------------------------- */
	function updateWidth(){
		if(!chartContainerRef) return;
		const w=chartContainerRef.clientWidth;
		chartWidth=Math.max(300, w - margin.left - margin.right);
		draw();
	}

	function filterData() {
    filtered = rawData.filter(r =>
      r.medal_type === 'GOLD' &&
      !isNaN(+r.value_unit) &&
      (!measure || r.value_type === measure)
    );

    // Apply search filter
    if (searchQuery.trim()) {
      const q = searchQuery.trim().toLowerCase();
      filtered = filtered.filter(r => r.event_title.toLowerCase().includes(q));
    }

    // Group filtered data
    groups = Array.from(d3.group(filtered, r => r.event_title));
    legend = groups.map(([ev]) => ev);
    color = d3.scaleOrdinal().domain(legend).range(d3.schemeTableau10);

    draw();
  }

  $: { rawData, measure, searchQuery, rawData.length && filterData(); }

	function selectEvent(ev){
		activeEvent = activeEvent===ev ? null : ev;
		hoverVisible=false;
		calcInfo();
		draw();
	}

	function calcInfo(){
		if(!activeEvent) return;
		const rows = rawData.filter(r=>r.event_title===activeEvent && r.medal_type==='GOLD');
		introYear = d3.min(rows,r=>+r.ano);
		recordRow = measure==='TIME'
			? rows.reduce((a,b)=>+b.value_unit<+a.value_unit?b:a)
			: rows.reduce((a,b)=>+b.value_unit>+a.value_unit?b:a);

		recAth = recordRow.athlete_full_name;
		recVal = recordRow.value_unit;
		recYear= recordRow.ano;

		const k=recordRow.athlete_url||recordRow.athlete_link||'';
		if(k && !cache.has(k)){
			fetch(k).then(r=>r.text()).then(h=>{
				const d=new DOMParser().parseFromString(h,'text/html');
				cache.set(k,{
					photo:d.querySelector('section picture img')?.src||'',
					flag :d.querySelector('section img[alt][src*="noc"]')?.src||''
				});
				const o=cache.get(k); recPhoto=o.photo; recFlag=o.flag;
			}).catch(()=>cache.set(k,{photo:'',flag:''}));
		}else if(k){const o=cache.get(k)||{}; recPhoto=o.photo; recFlag=o.flag;}
	}

	async function showHover(row){
		hover={sport:row.event,athlete:row.raw.athlete_full_name,
		       year:row.year,value:row.val,country:row.raw.country_name,
		       photo:'',flag:''};
		const k=row.raw.athlete_url||row.raw.athlete_link||'';
		if(k && !cache.has(k)){
			try{
				const html=await (await fetch(k)).text();
				const d=new DOMParser().parseFromString(html,'text/html');
				cache.set(k,{
					photo:d.querySelector('section picture img')?.src||'',
					flag :d.querySelector('section img[alt][src*="noc"]')?.src||''
				});
			}catch{cache.set(k,{photo:'',flag:''});}
		}
		if(k){const o=cache.get(k)||{}; hover.photo=o.photo; hover.flag=o.flag;}
		hoverVisible=true;
	}
	const hideHover=()=>hoverVisible=false;
	const resetAll = ()=>{activeEvent=null; hoverVisible=false; draw();};

	/* --------------------------- draw ------------------------------ */
	function draw(){
		if(!svgRef) return;
		const svg=d3.select(svgRef)
			.attr('width',chartWidth+margin.left+margin.right)
			.attr('height',chartHeight);
		svg.selectAll('*').remove();

		const useGroups = activeEvent
			? groups.filter(([ev])=>ev===activeEvent)
			: groups;

		const years = useGroups.flatMap(([,v])=>v.map(r=>+r.ano));
		const vals  = useGroups.flatMap(([,v])=>v.map(r=>+r.value_unit));
		if(!years.length){
			svg.append('text')
				.attr('x',(chartWidth+margin.left+margin.right)/2)
				.attr('y',chartHeight/2)
				.attr('text-anchor','middle')
				.text('Sem dados');
			return;
		}

		xScale=d3.scaleLinear().domain(d3.extent(years))
			.range([margin.left, margin.left+chartWidth]);
		yScale=d3.scaleLinear().domain([0, d3.max(vals)]).nice()
			.range([chartHeight-margin.bottom, margin.top]);

		const line=d3.line().x(d=>xScale(d.year)).y(d=>yScale(d.val));

		svg.append('text')
			.attr('x',(chartWidth+margin.left+margin.right)/2)
			.attr('y',30)
			.attr('text-anchor','middle')
			.attr('font-size','20px')
			.text(`Olympic (${measure})`);

		useGroups.forEach(([ev,rows])=>{
			const series=rows.map(r=>({year:+r.ano,val:+r.value_unit,raw:r,event:ev}))
				.sort((a,b)=>a.year-b.year);

			svg.append('path').datum(series)
				.attr('fill','none').attr('stroke',color(ev)).attr('stroke-width',2)
				.attr('d',line)
				.style('cursor','pointer')
				.on('click',e=>{e.stopPropagation();selectEvent(ev);});

			svg.selectAll(null).data(series).enter().append('circle')
				.attr('cx',d=>xScale(d.year)).attr('cy',d=>yScale(d.val))
				.attr('r',8).attr('fill','transparent')
				.on('mouseenter',(_,d)=>showHover(d))
				.on('mouseleave',hideHover)
				.on('click',e=>{e.stopPropagation();selectEvent(ev);});
		});

		svg.append('g')
			.attr('transform',`translate(0,${chartHeight-margin.bottom})`)
			.call(d3.axisBottom(xScale).ticks(10).tickFormat(d3.format('d')));
		svg.append('g')
			.attr('transform',`translate(${margin.left},0)`)
			.call(d3.axisLeft(yScale));
	}
</script>

<div class="dashboard" on:click={resetAll}>
  <div class="cards-column">
	<!-- info-card -->
	<aside class="info-card">
		{#if activeEvent}
			<h2>{activeEvent}</h2>
			<p><b>Introduzido em:</b> {introYear}</p>
			<p><b>Recorde mundial:</b> {recVal} ({measure.toLowerCase()})</p>
			<p><b>Atleta:</b> {recAth} – {recYear}</p>
			{#if recPhoto}<img class="ath-img" src={recPhoto} alt={recAth}/>{/if}
			<p><b>País:</b> {recordRow.country_name}
				{#if recFlag}<img class="flag" src={recFlag} alt={recordRow.country_name}/>{/if}
			</p>
		{:else}
			<p>Clique na legenda ou no gráfico para ver detalhes.</p>
		{/if}
	</aside>

	<!-- hover-card -->
	<aside class="hover-card">
		{#if hoverVisible}
			<h3>Resultado do ponto</h3>
			<p><b>{hover.sport}</b> – {hover.year}</p>
			{#if hover.photo}<img class="ath-img" src={hover.photo} alt={hover.athlete}/>{/if}
			<p><b>Atleta:</b> {hover.athlete}</p>
			<p><b>País:</b> {hover.country}
				{#if hover.flag}<img class="flag" src={hover.flag} alt={hover.country}/>{/if}
			</p>
			<p><b>Resultado:</b> {hover.value}
				{measure==='TIME'?' s':measure==='DISTANCE'?' m':measure==='WEIGHT'?' kg':''}
			</p>
		{:else}
			<p>Passe o mouse sobre um ponto para ver o resultado.</p>
		{/if}
	</aside>
  </div>

	<!-- gráfico -->
	<div class="chart-container" bind:this={chartContainerRef}>
		<svg bind:this={svgRef}></svg>
	</div>

	<!-- legenda -->
	<aside class="legend">
		{#each legend as ev}
			<div class="legend-item" role="button" tabindex="0"
			     class:active={activeEvent===ev}
			     on:click|stopPropagation={()=>selectEvent(ev)}>
				<span class="swatch" style="background:{color(ev)}"></span>{ev}
			</div>
		{/each}
	</aside>
</div>

<style>
.dashboard{
	display:grid;
	grid-template-columns:200px 1fr 200px;
	grid-template-rows: 1fr 0 0;
	gap:1rem;
	padding:1rem;
	width:100%;
	box-sizing:border-box;
}

.info-card,
.hover-card{
	min-height:250px;
	max-height:300px;   /* nunca somam mais que 480px */
	overflow-y:auto;
	font-size:0.75rem;
}
.info-card,
.hover-card{
	flex: 1 1 48%;                  /* divide espaço quase pela metade    */
	overflow-y: auto;
	font-size: .75rem;
	padding: 1rem;
	border-radius: var(--radius);
	background: var(--card);
	box-shadow: 0 2px 10px rgba(0,0,0,.1);
}
.chart-container{grid-column:2;grid-row:1/span 3;min-width:0;}

.info-card,.hover-card,.chart-container,.legend{
	background:var(--card);padding:1rem;border-radius:var(--radius);
	box-shadow:0 2px 10px rgba(0,0,0,.1);
}

/* svg */
.chart-container svg{width:100%;height:500px;}
.legend{max-height:500px;overflow-y:auto;}

.ath-img{width:60px;border-radius:6px;display:block;margin:.4rem 0;}
.flag{width:14px;height:9px;margin-left:3px;vertical-align:middle;}

.legend-item{
	display:flex;align-items:center;margin-bottom:6px;
	cursor:pointer;border-radius:4px;padding:2px 4px;
	transition:background-color .2s,transform .2s;white-space:normal;
}
.legend-item:hover{background:rgba(0,0,0,.05);}
.legend-item.active{background:rgba(212,175,55,.25);transform:scale(1.03);}
.legend-item .swatch{width:14px;height:14px;border-radius:3px;margin-right:6px;}
.legend-item.active .swatch{transform:scale(1.3);}

.chart-container svg text      {font-family:Poppins,sans-serif;}
.chart-container svg .tick line,
.chart-container svg .domain   {stroke:var(--grid);}
.chart-container svg .tick text{font-size:.75rem;}

.cards-column{
	grid-column: 1;                 /* primeira coluna da grade */
	grid-row:    1 / span 3;        /* ocupa toda a altura da grade */
	display: flex;
	flex-direction: column;
	height: 500px;                  /* igual à altura do SVG              */
	max-height: 500px;
	gap: .5rem;
}
</style>
