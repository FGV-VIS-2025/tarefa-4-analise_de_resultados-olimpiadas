<script>
    import { onMount } from 'svelte';
    import * as d3 from 'd3';

    export let csvUrl;
    export let measure;
    export let filter = '';
    export let valueTypes = [];

    const m = { t: 60, r: 20, b: 60, l: 60 };
    const W = 900, H = 500;

    let svgEl;
    let rawData = [];
    let filteredData = [];
    let groups = [];
    let x, y, col;
    let ttVisible = false;
    let tt = { x: 0, y: 0, sport: '', athlete: '', value: 0, year: 0 };
    let activeSport = null;
    let hoveredSport = null;
	let legendGroups = [];

    onMount(async () => {
        rawData = await d3.csv(csvUrl);
        valueTypes = [...new Set(rawData.map(d => d.value_type))].filter(Boolean);
        if (valueTypes.length && !measure) measure = valueTypes[0];
        applyFilters();
    });

    $: {
		// sempre que rawData, measure ou filter mudar, refaz o filtro e redesenha
		if (rawData.length || measure || filter) {
			applyFilters();
		}
	}

    function applyFilters() {
        // Apply measure filter
        filteredData = rawData.filter(d => 
            d.medal_type === 'GOLD' && 
            !isNaN(+d.value_unit) &&
            (measure ? d.value_type === measure : true)
        );

        // Apply event title filter
        if (filter.trim()) {
            const searchTerm = filter.trim().toLowerCase();
            filteredData = filteredData.filter(d => 
                d.event_title.toLowerCase().includes(searchTerm)
            );
        }

        // Group the filtered data
        groups = Array.from(d3.group(filteredData, d => d.event_title));
		legendGroups = groups.map(([ev]) => ev);
        draw();
    }

    function draw() {
        if (!svgEl) return;

        const svg = d3.select(svgEl);
        svg.selectAll('*').remove();

        if (!groups.length) {
            svg.append('text')
                .attr('x', W / 2)
                .attr('y', H / 2)
                .attr('text-anchor', 'middle')
                .text(getNoDataMessage());
            return;
        }

        const years = groups.flatMap(([, v]) => v.map(d => +d.ano));
        const vals = groups.flatMap(([, v]) => v.map(d => +d.value_unit));

        x = d3.scaleLinear()
            .domain(d3.extent(years))
            .range([m.l, W - m.r]);

        y = d3.scaleLinear()
            .domain([0, Math.max(...vals)]).nice()
            .range([H - m.b, m.t]);

        col = d3.scaleOrdinal()
            .domain(groups.map(([s]) => s))
            .range(d3.schemeTableau10);

        const line = d3.line()
            .x(d => x(d.year))
            .y(d => y(d.val));

        // Chart title
        svg.append('text')
            .attr('x', W / 2)
            .attr('y', 30)
            .attr('text-anchor', 'middle')
            .attr('font-size', '20px')
            .attr('fill', '#333')
            .text(`Olympic Results Over Time (${measure})`);

        groups.forEach(([event, rows]) => {
            const serie = rows.map(r => ({
                year: +r.ano,
                val: +r.value_unit,
                raw: r,
                event
            })).sort((a, b) => a.year - b.year);

            const maxValue = Math.max(...serie.map(d => d.val));
            const maxPoint = serie.find(d => d.val === maxValue);

            // Draw the line
            svg.append('path')
                .datum(serie)
                .attr('fill', 'none')
                .attr('stroke', col(event))
                .attr('stroke-width', 2)
                .attr('opacity', activeSport ? (event === activeSport ? 1 : 0.2) : 0.8)
                .attr('d', line)
                .style('cursor', 'pointer')
                .on('mouseenter', () => {
                    hoveredSport = event;
                    draw();
                })
                .on('mouseleave', () => {
                    hoveredSport = null;
                    draw();
                })
                .on('click', (event) => {
                    event.stopPropagation();
                    activeSport = event === activeSport ? null : event;
                    ttVisible = false;
                    draw();
                });

            if (event === hoveredSport || event === activeSport) {
                // Regular points
                svg.selectAll(null)
                    .data(serie.filter(d => d !== maxPoint))
                    .enter()
                    .append('circle')
                    .attr('cx', d => x(d.year))
                    .attr('cy', d => y(d.val))
                    .attr('r', 3)
                    .attr('fill', col(event))
                    .style('cursor', 'pointer')
                    .on('mouseenter', function() {
                        d3.select(this).attr('r', 5);
                    })
                    .on('mouseleave', function() {
                        d3.select(this).attr('r', 3);
                    })
                    .on('click', (event, d) => {
                        event.stopPropagation();
                        showTooltip(event, d);
                    });

                // Record mark
                if (maxPoint) {
                    svg.append('circle')
                        .attr('cx', x(maxPoint.year))
                        .attr('cy', y(maxPoint.val))
                        .attr('r', 6)
                        .attr('fill', 'white')
                        .attr('stroke', col(event))
                        .attr('stroke-width', 2)
                        .style('cursor', 'pointer')
                        .on('mouseenter', function() {
                            d3.select(this).attr('r', 8);
                        })
                        .on('mouseleave', function() {
                            d3.select(this).attr('r', 6);
                        })
                        .on('click', (event) => {
                            event.stopPropagation();
                            showTooltip(event, maxPoint);
                        })
                        .append('title')
                        .text(`Record: ${maxPoint.val} (${maxPoint.year})`);
                }
            }
        });

        // Axes
        svg.append('g')
            .attr('transform', `translate(0,${H - m.b})`)
            .call(d3.axisBottom(x).ticks(10).tickFormat(d3.format('d')))
            .call(g => g.append('text')
                .attr('x', W / 2)
                .attr('y', 40)
                .attr('fill', '#333')
                .attr('text-anchor', 'middle')
                .attr('font-weight', 'bold')
                .text('Year'));

        svg.append('g')
            .attr('transform', `translate(${m.l},0)`)
            .call(d3.axisLeft(y))
            .call(g => g.append('text')
                .attr('transform', 'rotate(-90)')
                .attr('y', -40)
                .attr('x', -H / 2)
                .attr('fill', '#333')
                .attr('text-anchor', 'middle')
                .attr('font-weight', 'bold')
                .text(measure === 'TIME' ? 'Time (seconds)' : 
                     measure === 'DISTANCE' ? 'Distance (meters)' : 
                     measure === 'WEIGHT' ? 'Weight (kg)' : 'Value'));
    }

    function showTooltip(event, d) {
        tt = { 
            x: event.clientX,
            y: event.clientY,
            sport: d.raw.event_title, 
            athlete: d.raw.athlete_full_name, 
            value: d.val, 
            year: d.year 
        };
        ttVisible = true;
    }

    function handleClickOutside() {
        activeSport = null;
        ttVisible = false;
        draw();
    }

    function getNoDataMessage() {
        if (!measure) return 'Please select a measurement type';
        if (filter && !filteredData.length) return 'No events match your search';
        return 'No data available for current filters';
    }
</script>

<div class="graph-wrapper" on:click={handleClickOutside}>
	<svg bind:this={svgEl} width={W} height={H}></svg>

	<!-- LEGENDA -->
	{#if legendGroups.length}
		<aside class="legend">
			{#each legendGroups as ev}
			<div class="legend-item" on:click|stopPropagation={() => activeSport = ev}>
				<span class="swatch" style="background-color:{col(ev)}"></span>
				{ev}
			</div>
			{/each}
		</aside>
	{/if}

	{#if ttVisible}
		<aside class="tt" style:left={`${tt.x + 15}px`} style:top={`${tt.y + 15}px`}>
		<b>{tt.sport}</b><br>
		Year: {tt.year}<br>
		Athlete: {tt.athlete}<br>
		Result: {tt.value}
		{measure === 'TIME' ? ' s' :
			measure === 'DISTANCE' ? ' m' :
			measure === 'WEIGHT' ? ' kg' : ''}
		</aside>
	{/if}
</div>
  

<style>
    .graph-wrapper{
		display:flex;
		gap:1rem;
		width:100%;
		justify-content:center;
		background:#fff;
		padding:1rem;
		border-radius:8px;
		box-shadow:0 2px 10px rgba(0,0,0,.1);
		}

		.legend{
		max-height:500px;   /* igual à altura do SVG */
		overflow-y:auto;
		padding-right:4px;  /* espaço p/ scrollbar */
		font-size:.8rem;
		}

		.legend-item{
		display:flex;
		align-items:center;
		margin-bottom:4px;
		}

		.swatch{
		width:14px;
		height:14px;
		border-radius:3px;
		margin-right:6px;
		}

    .tt {
        position: fixed;
        background: white;
        border: 1px solid #ccc;
        border-radius: 6px;
        padding: 0.75rem;
        font-size: 0.9rem;
        box-shadow: 0px 6px 12px rgba(0,0,0,0.2);
        pointer-events: none;
        z-index: 100;
        max-width: 220px;
    }
    svg {
        font-family: 'Helvetica Neue', sans-serif;
    }
</style>