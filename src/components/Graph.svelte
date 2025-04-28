<script>
	import { onMount } from 'svelte';
	import * as d3 from 'd3';

	export let csvUrl;
	export let measure;
	export let filter = ''; // Garantir valor padrão

	const m = { t: 20, r: 130, b: 40, l: 60 };
	const W = 900, H = 500;

	let svgEl;
	let data = [];
	let groups = [];
	let x, y, col;
	let ttVisible = false;
	let tt = { x: 0, y: 0, sport: '', athlete: '', value: 0, year: 0 };
	let activeSport = null;

	onMount(async () => {
		data = await d3.csv(csvUrl);
		refresh();
	});

	$: {
		if (data.length) {
			console.log('Filter changed:', filter); // Debug
			refresh();
		}
	}

	function refresh() {
		if (!data.length) return;

		// Filtro simplificado - apenas medalhas de ouro com valores numéricos
		const filteredData = data.filter(d => {
			return d.medal_type === 'GOLD' && !isNaN(Number(d.value_unit));
		});

		const allSports = d3.group(filteredData, d => d.discipline_title);
		
		// Filtro case-insensitive
		const searchTerm = filter ? filter.trim().toLowerCase() : '';
		console.log('Filtering with:', searchTerm); // Debug
		console.log('Sports available:', Array.from(allSports.keys())); // Debug
		
		groups = searchTerm 
			? Array.from(allSports).filter(([sport]) => 
				sport.toLowerCase().includes(searchTerm))
			: Array.from(allSports);

		console.log('Filtered groups:', groups); // Debug
		draw();
	}


	function draw() {
		if (!svgEl) return;

		const svg = d3.select(svgEl);
		svg.selectAll('*').remove();

		if (groups.length === 0) {
			svg.append('text')
				.attr('x', W/2)
				.attr('y', H/2)
				.attr('text-anchor', 'middle')
				.text('Nenhum esporte encontrado');
			return;
		}

		const years = groups.flatMap(([, v]) => v.map(d => {
			const yearMatch = d.slug_game.match(/\d{4}/);
			return yearMatch ? +yearMatch[0] : 0;
		}));
		
		const vals = groups.flatMap(([, v]) => v.map(d => +d.value_unit));

		x = d3.scaleLinear()
			.domain(d3.extent(years))
			.range([m.l, W - m.r]);

		y = d3.scaleLinear()
			.domain([0, Math.max(...vals)])
			.nice()
			.range([H - m.b, m.t]);

		col = d3.scaleOrdinal()
			.domain(groups.map(([s]) => s))
			.range(d3.schemeCategory10);

		const line = d3.line()
			.x(d => x(d.year))
			.y(d => y(d.val));

		groups.forEach(([sport, rows]) => {
			const serie = rows.map(r => ({
				year: +r.slug_game.match(/\d{4}/)[0],
				val: +r.value_unit,
				raw: r,
				sport
			})).sort((a, b) => a.year - b.year);

			// Draw the line
			svg.append('path')
				.datum(serie)
				.attr('fill', 'none')
				.attr('stroke', col(sport))
				.attr('stroke-width', 1.5)
				.attr('opacity', activeSport ? (sport === activeSport ? 1 : 0.2) : 1)
				.attr('d', line);

			// Add interactive hit areas
			svg.selectAll(null)
				.data(serie)
				.enter()
				.append('circle')
				.attr('cx', d => x(d.year))
				.attr('cy', d => y(d.val))
				.attr('r', 10)
				.attr('fill', 'transparent')
				.style('cursor', 'pointer')
				.on('click', (event, d) => {
					event.stopPropagation();
					activeSport = sport;
					tt = { 
						x: event.clientX,
						y: event.clientY,
						sport, 
						athlete: d.raw.athlete_full_name, 
						value: d.val, 
						year: d.year 
					};
					ttVisible = true;
					draw();
				});

			// Only show points and labels for active sport (or all if none active)
			if (!activeSport || sport === activeSport) {
				// Data points
				svg.selectAll(null)
					.data(serie)
					.enter()
					.append('circle')
					.attr('cx', d => x(d.year))
					.attr('cy', d => y(d.val))
					.attr('r', 3)
					.attr('fill', col(sport));

				// Sport label
				const last = serie[serie.length - 1];
				svg.append('text')
					.attr('x', x(last.year) + 4)
					.attr('y', y(last.val))
					.attr('dy', '0.32em')
					.attr('font-size', 10)
					.attr('fill', col(sport))
					.text(sport);
			}
		});

		// Add axes
		svg.append('g')
			.attr('transform', `translate(0,${H - m.b})`)
			.call(d3.axisBottom(x).tickFormat(d3.format('d')));

		svg.append('g')
			.attr('transform', `translate(${m.l},0)`)
			.call(d3.axisLeft(y));
	}

	function handleClickOutside() {
		activeSport = null;
		ttVisible = false;
		draw();
	}
</script>
<div class="graph-container">
	<svg bind:this={svgEl} width={W} height={H}></svg>
  
	{#if ttVisible}
	  <aside class="tt" style:left={`${tt.x + 15}px`} style:top={`${tt.y + 15}px`}>
		<b>{tt.sport}</b><br>
		Ano: {tt.year}<br>
		Atleta: {tt.athlete}<br>
		Resultado: {tt.value} {measure === 'time' ? 's' : measure === 'distance' ? 'm' : 'kg'}<br>
	  </aside>
	{/if}
  </div>
  
  <style>
	.graph-container {
	  display: flex;
	  justify-content: center;
	  width: 100%;
	  margin: 0 auto;
	}
	
	.pane { 
	  position: relative;
	  cursor: default;
	  margin-top: 20px;
	  width: 100%;
	  display: flex;
	  flex-direction: column;
	  align-items: center;
	}
	
	.tt {
	  position: fixed;
	  background: white;
	  border: 1px solid #ddd;
	  border-radius: 4px;
	  padding: 0.5rem;
	  font-size: 0.85rem;
	  max-width: 200px;
	  box-shadow: 0 2px 8px rgba(0,0,0,0.1);
	  pointer-events: none;
	  z-index: 100;
	}
  </style>