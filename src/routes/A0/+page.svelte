<script>
  let maxClick = $state(4);
  let cnt = $state(maxClick);

  function onClick() {
    if (cnt > 0) {
      cnt -= 1;
    }
  }

  function resetCounter() {
    cnt = maxClick;
  }
</script>

<svelte:head>
  <title>QinYang Tan's VIS Site</title>
  <meta
    name="description"
    content="CSCI 5609 A0 setup demo showing a reactive click counter in Svelte."
  />
</svelte:head>

<main>
  <h1>QinYang Tan's VIS Site</h1>

  <img
    width="200"
    src="https://github.com/QinyangTan.png"
    alt="QinYang Tan's GitHub profile avatar"
  />

  <div class="controls">
    <label for="max-clicks">You can click up to</label>
    <select id="max-clicks" bind:value={maxClick} onchange={resetCounter}>
      {#each [2, 4, 6] as optionNum}
        <option value={optionNum}>{optionNum}</option>
      {/each}
    </select>
    <span>times</span>
  </div>

  <button onclick={onClick} disabled={cnt === 0}>Click Me</button>

  {#if cnt > 0}
    <p id="info">Remaining Number of Clicks: {cnt}</p>
  {:else}
    <p id="info">No more clicks allowed</p>
  {/if}
</main>

<style>
  :global(body) {
    margin: 0;
    font-family: Arial, Helvetica, sans-serif;
    background: #f7f7f7;
    color: #222;
  }

  main {
    max-width: 560px;
    margin: 4rem auto;
    padding: 2rem;
    text-align: center;
    background: white;
    border-radius: 14px;
    box-shadow: 0 8px 28px rgb(0 0 0 / 10%);
  }

  img {
    border-radius: 12px;
    margin: 1rem 0 1.5rem;
  }

  .controls {
    display: flex;
    justify-content: center;
    align-items: center;
    gap: 0.5rem;
    margin: 1rem 0;
  }

  select {
    font: inherit;
    padding: 0.35rem 0.5rem;
  }

  button {
    background-color: #44aa66;
    color: white;
    font-size: xx-large;
    padding: 10px 20px;
    border: none;
    cursor: pointer;
    border-radius: 5px;
  }

  button:disabled {
    cursor: not-allowed;
    opacity: 0.55;
  }

  #info {
    margin-top: 1.25rem;
    font-weight: 700;
  }
</style>
