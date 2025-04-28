<script>
	import { onMount } from 'svelte';
	import * as d3 from 'd3';

	/* props */
	export let csvUrl;
	export let measure = '';
	export let filter  = '';
	export let valueTypes = [];

	/* layout */
	const margin      = { top: 60, right: 20, bottom: 60, left: 60 };
	const chartWidth  = 900;
	const chartHeight = 500;

	/* data */
	let svgRef;
	let rawData        = [];
	let filteredData   = [];
	let eventGroups    = [];
	let legendItems    = [];

	/* scales */
	let xScale, yScale, colorScale;

	/* state */
	let tooltipVisible = false;
	let tooltipData    = { x: 0, y: 0, sport: '', athlete: '', value: 0, year: 0 };
	let activeEvent    = null;
	let hoveredEvent   = null;

	onMount(async () => {
		rawData = await d3.csv(csvUrl);
		valueTypes = [...new Set(rawData.map(d => d.value_type))].filter(Boolean);
		if (!measure && valueTypes.length) measure = valueTypes[0];
		updateFilteredData();
	});

	$: { rawData; measure; filter; rawData.length && updateFilteredData(); }

	function updateFilteredData() {
		filteredData = rawData
			.filter(d =>
				d.medal_type === 'GOLD' &&
				!isNaN(+d.value_unit) &&
				(!measure || d.value_type === measure)
			);

		if (filter.trim()) {
			const q = filter.trim().toLowerCase();
			filteredData = filteredData.filter(d => d.event_title.toLowerCase().includes(q));
		}

		eventGroups = Array.from(d3.group(filteredData, d => d.event_title));
		legendItems = eventGroups.map(([ev]) => ev);
		draw();
	}

	function draw() {
		if (!svgRef) return;

		const svg = d3.select(svgRef);
		svg.selectAll('*').remove();

		if (!eventGroups.length) {
			svg.append('text')
				.attr('x', chartWidth / 2)
				.attr('y', chartHeight / 2)
				.attr('text-anchor', 'middle')
				.text('Sem dados');
			return;
		}

		const years  = eventGroups.flatMap(([, v]) => v.map(d => +d.ano));
		const values = eventGroups.flatMap(([, v]) => v.map(d => +d.value_unit));

		xScale = d3.scaleLinear()
			.domain(d3.extent(years))
			.range([margin.left, chartWidth - margin.right]);

		yScale = d3.scaleLinear()
			.domain([0, Math.max(...values)]).nice()
			.range([chartHeight - margin.bottom, margin.top]);

		colorScale = d3.scaleOrdinal()
			.domain(eventGroups.map(([ev]) => ev))
			.range(d3.schemeTableau10);

		const line = d3.line()
			.x(d => xScale(d.year))
			.y(d => yScale(d.val));

		/* title */
		svg.append('text')
			.attr('x', chartWidth / 2)
			.attr('y', 30)
			.attr('text-anchor', 'middle')
			.attr('font-size', '20px')
			.attr('fill', 'var(--primary)')
			.text(`Resultados Olímpicos (${measure})`);

		/* series */
		eventGroups.forEach(([ev, rows]) => {
			const series   = rows.map(r => ({ year: +r.ano, val: +r.value_unit, raw: r }));
			const maxValue = Math.max(...series.map(d => d.val));
			const maxPoint = series.find(d => d.val === maxValue);

			svg.append('path')
				.datum(series)
				.attr('fill', 'none')
				.attr('stroke', colorScale(ev))
				.attr('stroke-width', 2)
				.attr('opacity', activeEvent ? (ev === activeEvent ? 1 : 0.2) : 0.9)
				.attr('d', line)
				.style('cursor', 'pointer')
				.on('mouseenter', () => { hoveredEvent = ev; draw(); })
				.on('mouseleave', () => { hoveredEvent = null; draw(); })
				.on('click', e => {
					e.stopPropagation();
					activeEvent = ev === activeEvent ? null : ev;
					tooltipVisible = false;
					draw();
				});

			if (ev === hoveredEvent || ev === activeEvent) {
				/* points */
				svg.selectAll(null)
					.data(series.filter(d => d !== maxPoint))
					.enter()
					.append('circle')
					.attr('cx', d => xScale(d.year))
					.attr('cy', d => yScale(d.val))
					.attr('r', 3)
					.attr('fill', colorScale(ev))
					.style('cursor', 'pointer')
					.on('mouseenter', function () { d3.select(this).attr('r', 5); })
					.on('mouseleave', function () { d3.select(this).attr('r', 3); })
					.on('click', (e, d) => { e.stopPropagation(); showTooltip(e, d, ev); });

				/* record */
				svg.append('circle')
					.attr('cx', xScale(maxPoint.year))
					.attr('cy', yScale(maxPoint.val))
					.attr('r', 6)
					.attr('fill', '#fff')
					.attr('stroke', colorScale(ev))
					.attr('stroke-width', 2)
					.style('cursor', 'pointer')
					.on('click', e => { e.stopPropagation(); showTooltip(e, maxPoint, ev); })
					.append('title')
					.text(`Recorde: ${maxPoint.val} (${maxPoint.year})`);
			}
		});

		/* grid */
		svg.append('g')
			.attr('transform', `translate(${margin.left},0)`)
			.call(
				d3.axisLeft(yScale)
					.tickSize(-(chartWidth - margin.left - margin.right))
					.tickFormat('')
			)
			.selectAll('line')
			.attr('stroke', 'var(--grid)');

		/* axes */
		svg.append('g')
			.attr('transform', `translate(0,${chartHeight - margin.bottom})`)
			.call(d3.axisBottom(xScale).ticks(10).tickFormat(d3.format('d')))
			.call(g => g.append('text')
				.attr('x', chartWidth / 2)
				.attr('y', 40)
				.attr('fill', 'var(--text)')
				.attr('text-anchor', 'middle')
				.attr('font-weight', 'bold')
				.text('Ano'));

		svg.append('g')
			.attr('transform', `translate(${margin.left},0)`)
			.call(d3.axisLeft(yScale))
			.call(g => g.append('text')
				.attr('transform', 'rotate(-90)')
				.attr('y', -40)
				.attr('x', -chartHeight / 2)
				.attr('fill', 'var(--text)')
				.attr('text-anchor', 'middle')
				.attr('font-weight', 'bold')
				.text(
					measure === 'TIME'      ? 'Tempo (s)'     :
					measure === 'DISTANCE'  ? 'Distância (m)' :
					measure === 'WEIGHT'    ? 'Peso (kg)'     :
					'Valor'
				));
	}

	function showTooltip(event, d, ev) {
		tooltipData = {
			x: event.clientX,
			y: event.clientY,
			sport: ev,
			athlete: d.raw.athlete_full_name,
			value: d.val,
			year: d.year
		};
		tooltipVisible = true;
	}

	function resetSelection() {
		activeEvent = null;
		tooltipVisible = false;
		draw();
	}
</script>

<div class="graph-wrapper" on:click={resetSelection}>
	<svg bind:this={svgRef} width={chartWidth} height={chartHeight}></svg>

	{#if legendItems.length && colorScale}
		<aside class="legend">
			{#each legendItems as ev}
				<div	class="legend-item"
						on:click|stopPropagation={() => activeEvent = ev}>
					<span	class="swatch"
							style="background-color:{colorScale(ev)}"></span>{ev}
				</div>
			{/each}
		</aside>
	{/if}

	{#if tooltipVisible}
		<aside	class="tooltip"
				style:left={`${tooltipData.x + 15}px`}
				style:top={`${tooltipData.y + 15}px`}>
			<b>{tooltipData.sport}</b><br>
			Ano: {tooltipData.year}<br>
			Atleta: {tooltipData.athlete}<br>
			Resultado: {tooltipData.value}
			{measure === 'TIME' ? ' s' : measure === 'DISTANCE' ? ' m' : measure === 'WEIGHT' ? ' kg' : ''}
		</aside>
	{/if}
</div>

<style>
.graph-wrapper {
	display: flex;
	gap: 1.5rem;
	justify-content: center;
	background: var(--card);
	padding: 1rem 1.5rem;
	border-radius: var(--radius);
	box-shadow: 0 4px 12px rgba(0, 0, 0, .04);
	border: 1px solid #e5e7eb;
}

svg {
	font-family: Poppins, sans-serif;
}

svg text {
	fill: var(--text);
}

svg .domain,
svg .tick line {
	stroke: var(--grid);
}

svg .tick text {
	font-size: .75rem;
}

.legend {
	max-height: 500px;
	overflow-y: auto;
	padding-right: 4px;
	font-size: .8rem;
}

.legend-item {
	display: flex;
	align-items: center;
	margin-bottom: 6px;
	cursor: pointer;
	user-select: none;
}

.legend-item:hover {
	opacity: .7;
}

.swatch {
	width: 14px;
	height: 14px;
	border-radius: 3px;
	margin-right: 6px;
	border: 1px solid rgba(0, 0, 0, .15);
}

.tooltip {
	position: fixed;
	background: var(--card);
	border: 1px solid var(--accent);
	border-radius: var(--radius);
	padding: .75rem;
	font-size: .8rem;
	pointer-events: none;
	box-shadow: 0 6px 16px rgba(0, 0, 0, .15);
	z-index: 100;
	max-width: 220px;
}
</style>
