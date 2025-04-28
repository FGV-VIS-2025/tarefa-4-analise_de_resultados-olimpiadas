<script>
    import Graph from '../components/Graph.svelte';

    let measure = 'distance';
    let search = '';
    
    // Debug: monitorar mudanças no search
    $: console.log('Search changed:', search);
</script>

  
<svelte:head>
  <title>Tarefa 4 - Olimpíadas</title>
</svelte:head>

<div class='title'>
<h1>🏆  Análise de Resultados - Olimpíadas 🏅</h1> 
<h3>guilherme buss e guilherme carvalho</h3>
<p>A seguinte visualização tem como objetivo analisar a evolução do desempenho de atletas olímpicos com o passar dos anos. 
    Utilizamos o seguinte dataset: <a href="https://www.kaggle.com/datasets/piterfm/olympic-games-medals-19862018?select=olympic_results.csv" target="_blank" rel="noopener noreferrer" class="link-dataset"> Olympic Results (1986-2018) </a>, 
    que contém os resultados (tempo, distânica, peso) de diversas modalidades no intervalo de 1986 até 2018, contendo também o nome 
    dos atletas que realizaram o feito.</p>
</div>

<div class="controls-container">
    <div class="controls">
      <select bind:value={measure}>
        <option value="distance">Distância</option>
        <option value="time">Tempo</option>
        <option value="weight">Peso</option>
      </select>
      
      <input type="text" bind:value={search} placeholder="Filtrar por esporte...">
      <button on:click={() => search = ''}>Limpar</button>
    </div>
</div>

<Graph csvUrl="/olympic_results.csv" {measure} filter={search}/>

<style>
    .title {
        font-size: 35px;
        text-align: Center;
        margin: 0;
    }
    h1 {
        font-family:monospace;
        font-weight: 100;
        font-size: 1em;
        letter-spacing: 1.3px;
    }
    h3 {
        font-family:monospace;
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
        max-width: 900px; /* Mesma largura do gráfico */
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
</style>