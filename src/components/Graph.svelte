<script>
    import { onMount } from 'svelte';
    import * as d3 from 'd3';

    export let csvUrl;
    export let measure;
    export let filter = '';

    const m = { t: 60, r: 20, b: 60, l: 60 };
    const W = 900, H = 500;

    let svgEl;
    let data = [];
    let groups = [];
    let x, y, col;
    let ttVisible = false;
    let tt = { x: 0, y: 0, sport: '', athlete: '', value: 0, year: 0 };
    let activeSport = null;
    let hoveredSport = null;

    onMount(async () => {
        data = await d3.csv(csvUrl);
        refresh();
    });

    $: {
		if (data.length && filter !== undefined) {
			refresh();
		}
	}

    function refresh() {
		console.log('Filter:', filter);  // Log filter value
		if (!data.length) return;

		const filteredData = data.filter(d => 
			d.medal_type === 'GOLD' && 
			!isNaN(+d.value_unit)
		);

		const allSports = d3.group(filteredData, d => d.discipline_title);
		
		const searchTerm = filter.trim().toLowerCase();
		console.log('Search Term:', searchTerm);
		
		groups = searchTerm
			? Array.from(allSports).filter(([sport, rows]) => {
				console.log('Sport Name:', sport);  // Log sport name for filtering
				return sport.toLowerCase().includes(searchTerm)
			})
			: Array.from(allSports);

		console.log('Filtered Groups:', groups);  // Log the groups after filtering

		draw();
	}


    function draw() {
        if (!svgEl) return;

        const svg = d3.select(svgEl);
        svg.selectAll('*').remove();

        if (groups.length === 0) {
            svg.append('text')
                .attr('x', W / 2)
                .attr('y', H / 2)
                .attr('text-anchor', 'middle')
                .text(filter ? 'No sports match your search' : 'No data available');
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
            .text('Olympic Results Over Time');

        groups.forEach(([sport, rows]) => {
            const serie = rows.map(r => ({
                year: +r.slug_game.match(/\d{4}/)[0],
                val: +r.value_unit,
                raw: r,
                sport
            })).sort((a, b) => a.year - b.year);

            const maxValue = Math.max(...serie.map(d => d.val));
            const maxPoint = serie.find(d => d.val === maxValue);

            // Draw the line
            svg.append('path')
                .datum(serie)
                .attr('fill', 'none')
                .attr('stroke', col(sport))
                .attr('stroke-width', 2)
                .attr('opacity', activeSport ? (sport === activeSport ? 1 : 0.2) : 0.8)
                .attr('d', line)
                .style('cursor', 'pointer')
                .on('mouseenter', () => {
                    hoveredSport = sport;
                    draw();
                })
                .on('mouseleave', () => {
                    hoveredSport = null;
                    draw();
                })
                .on('click', (event) => {
                    event.stopPropagation();
                    activeSport = sport === activeSport ? null : sport;
                    ttVisible = false;
                    draw();
                });

            // Only show points if hovered or active
            if (sport === hoveredSport || sport === activeSport) {
                // Regular points (excluding max point)
                svg.selectAll(null)
                    .data(serie.filter(d => d !== maxPoint))
                    .enter()
                    .append('circle')
                    .attr('cx', d => x(d.year))
                    .attr('cy', d => y(d.val))
                    .attr('r', 3)
                    .attr('fill', col(sport))
                    .style('cursor', 'pointer')
                    .on('mouseenter', function() {
                        d3.select(this).attr('r', 5);
                    })
                    .on('mouseleave', function() {
                        d3.select(this).attr('r', 3);
                    })
                    .on('click', (event, d) => {
                        event.stopPropagation();
                        showTooltip(event, d, sport);
                    });

                // Record mark (only when hovered/active)
                if (maxPoint) {
                    svg.append('circle')
                        .attr('cx', x(maxPoint.year))
                        .attr('cy', y(maxPoint.val))
                        .attr('r', 6)
                        .attr('fill', 'white')
                        .attr('stroke', col(sport))
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
                            showTooltip(event, maxPoint, sport);
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
            .call(d3.axisLeft(y));
    }

    function showTooltip(event, d, sport) {
        tt = { 
            x: event.clientX,
            y: event.clientY,
            sport, 
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
</script>

<div class="graph-container" on:click={handleClickOutside}>
    <svg bind:this={svgEl} width={W} height={H}></svg>

    {#if ttVisible}
        <aside class="tt" style:left={`${tt.x + 15}px`} style:top={`${tt.y + 15}px`}>
            <b>{tt.sport}</b><br>
            Year: {tt.year}<br>
            Athlete: {tt.athlete}<br>
            Result: {tt.value} {measure === 'time' ? 's' : measure === 'distance' ? 'm' : 'kg'}
        </aside>
    {/if}
</div>

<style>
    .graph-container {
        display: flex;
        justify-content: center;
        width: 100%;
        margin: 0 auto;
        background-color: white;
        padding: 1rem;
        border-radius: 8px;
        box-shadow: 0 2px 10px rgba(0,0,0,0.1);
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