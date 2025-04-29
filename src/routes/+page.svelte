<script>
	import Graph from '../components/Graph.svelte';

	let measure    = '';
	let search     = '';
	let valueTypes = [];
    let selectedEvent = '';  

	$: graphKey = measure + search;	/* force remount */

	import { base } from '$app/paths';
  	let csvUrl = `${base}/df_processed.csv`;
</script>

<svelte:head>
	<title>Tarefa 4 – Olimpíadas</title>

	<link	href="https://fonts.googleapis.com/css2?family=Poppins:wght@300;400;600&display=swap"
			rel="stylesheet">

	<style>
	:root {
		--bg:     #f8f9fa;
		--card:   #fff;
		--primary:#003366;
		--accent: #d4af37;
		--text:   #242424;
		--grid:   rgba(0, 0, 0, .06);
		--radius: 12px;
	}

	html, body {
		margin: 0;
		padding: 0;
		background: var(--bg);
		color: var(--text);
		font-family: Poppins, sans-serif;
	}

	a { color: var(--accent); }
	</style>
</svelte:head>

<div class="page">
	<div class="title">
		<h1>🏆 Olympic Results Analysis 🏅</h1>
		<h3>Guilherme Buss & Guilherme Carvalho</h3>
		<p>
			Visualização da evolução do olímpico.<br>
			Dataset:
			<a href="https://www.kaggle.com/datasets/piterfm/olympic-games-medals-19862018?select=olympic_results.csv"
			   target="_blank"
			   rel="noopener">
				Olympic Results (1986-2018)
			</a>
		</p>
	</div>

	<div class="controls-container">
		<div class="controls">
			<select bind:value={measure}>
				<option value="" disabled selected>tipo de medida</option>
				{#each valueTypes as t}
					<option value={t}>{t.toLowerCase()}</option>
				{/each}
			</select>

			<input type="text" bind:value={search} placeholder="Buscar evento…" disabled={!measure}>

			<button on:click={() => search = ''} disabled={!measure}>Limpar</button>
		</div>
	</div>

	{#key graphKey}
		<Graph {csvUrl} bind:valueTypes {measure} searchQuery={search}/>
	{/key}

    {#if selectedEvent}
    <h2 style="text-align:center;margin:2rem 0">
        🔍 {selectedEvent}
        <button style="margin-left:10px"
                on:click={() => selectedEvent = ''}>voltar</button>
    </h2>

    <Graph
		{csvUrl}
		bind:valueTypes
		{measure}
		searchQuery={search}/> 
    {/if}
</div>

<style>
.page {
	max-width: 1380px;
	margin: auto;
	padding: 3rem 1rem;
}

.title h1 {
	font-weight: 600;
	color: var(--primary);
	margin: .2em 0;
}

.title h3 {
	font-weight: 300;
	color: #555;
	margin: .2em 0;
	font-size: 1rem;
}

.title p {
	font-size: .9rem;
	color: #555;
	text-align: center;
}

.controls-container {
	display: flex;
	justify-content: center;
	margin: 1.5rem 0;
}

.controls {
	display: flex;
	gap: 10px;
	max-width: 900px;
	width: 100%;
}

.controls select,
.controls input,
.controls button {
	padding: .6rem .8rem;
	font-size: .9rem;
	border: 1px solid #ccc;
	border-radius: var(--radius);
}

.controls select,
.controls input {
	background: #fff;
	flex: 1;
	min-width: 200px;
}

.controls button {
	background: var(--primary);
	color: #fff;
	cursor: pointer;
	transition: background .2s;
}

.controls button:disabled { background: #888; }
.controls button:not(:disabled):hover { background: #00264d; }
</style>
