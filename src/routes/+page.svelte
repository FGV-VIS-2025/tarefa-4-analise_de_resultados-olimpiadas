<script>
    import Graph from '../components/Graph.svelte';

    let measure = '';
    let search = '';
    let valueTypes = [];
    
    $: graphKey = measure + search;
</script>
  
<svelte:head>
  <title>Tarefa 4 - Olimpíadas</title>
</svelte:head>

<div class='title'>
<h1>🏆 Olympic Results Analysis 🏅</h1> 
<h3>Guilherme Buss and Guilherme Carvalho</h3>
<p>This visualization analyzes the evolution of Olympic athletes' performance over time. 
    Dataset: <a href="https://www.kaggle.com/datasets/piterfm/olympic-games-medals-19862018?select=olympic_results.csv" target="_blank" rel="noopener noreferrer" class="link-dataset">Olympic Results (1986-2018)</a>, 
    containing results (time, distance, weight) of various events from 1986 to 2018, including athlete names.</p>
</div>

<div class="controls-container">
    <div class="controls">
      <select bind:value={measure}>
        <option value="" disabled selected>Select measurement type</option>
        {#each valueTypes as type}
          <option value={type}>{type.toLowerCase()}</option>
        {/each}
      </select>
      
      <input 
        type="text" 
        bind:value={search} 
        placeholder="Search events..." 
        disabled={!measure}
      >
      <button on:click={() => search = ''} disabled={!measure}>Clear</button>
    </div>
</div>

<Graph 
    csvUrl="/df_processed.csv" 
    bind:valueTypes
    {measure}
    filter={search} 
    key={graphKey}
/>

<footer></footer>

<style>
    .title {
        font-size: 35px;
        text-align: center;
        margin: 0;
    }
    h1 {
        font-family: monospace;
        font-weight: 100;
        font-size: 1em;
        letter-spacing: 1.3px;
    }
    h3 {
        font-family: monospace;
        font-weight: 100;
        font-size: 0.6em;
        letter-spacing: 0.3px;
    }
    p {
        font-size: 0.5em;
        width: 70%;
        margin: 0 auto;
        text-align: center;
    }
    .controls-container {
        display: flex;
        justify-content: center;
        width: 100%;
    }
    
    .controls {
        display: flex;
        gap: 10px;
        margin: 20px 0;
        max-width: 900px;
        width: 100%;
    }
    
    .controls select, .controls input {
        padding: 8px;
        border: 1px solid #ccc;
        border-radius: 4px;
    }
    
    .controls input {
        flex-grow: 1;
        min-width: 200px;
    }
    
    .controls button {
        padding: 8px 16px;
        background-color: #f0f0f0;
        border: 1px solid #ccc;
        border-radius: 4px;
        cursor: pointer;
    }

    footer {
        background-image: url('/olympic-games-svgrepo-com.svg'); 
        position: relative;
        background-repeat: repeat-x;
        background-size:50px 30px;
        height: 30px;
        top: 10px;  
        margin-top: 2%;
    }
</style>