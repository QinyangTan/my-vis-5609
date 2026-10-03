<script lang="ts">
  import type { TMovie } from '../types';
  import * as d3 from 'd3';

  export let movies: TMovie[] = [];
  export let width = 920;
  export let height = 430;

  let selectedGenre = '';

  $: counts = d3.rollups(
    movies.flatMap((movie) => movie.genres),
    (values) => values.length,
    (genre) => genre
  ).sort((a, b) => d3.descending(a[1], b[1]));

  $: margin = { top: 20, right: 20, bottom: 115, left: 58 };
  $: innerWidth = width - margin.left - margin.right;
  $: innerHeight = height - margin.top - margin.bottom;
  $: x = d3.scaleBand<string>()
    .domain(counts.map(([genre]) => genre))
    .range([0, innerWidth])
    .padding(0.16);
  $: y = d3.scaleLinear()
    .domain([0, d3.max(counts, (d) => d[1]) ?? 1])
    .nice()
    .range([innerHeight, 0]);
  $: yTicks = y.ticks(5);
</script>

<section class="card">
  <h2>Genre distribution</h2>
  <p class="subtitle">Number of summer-title movies containing each genre. Hover a bar to focus it.</p>
  {#if movies.length}
    <svg viewBox={"0 0 " + width + " " + height} role="img" aria-label="Bar chart of genre counts">
      <g transform={"translate(" + margin.left + "," + margin.top + ")"}>
        {#each yTicks as tick}
          <line x1="0" x2={innerWidth} y1={y(tick)} y2={y(tick)} class="grid" />
          <text x="-10" y={y(tick) + 4} text-anchor="end" class="axis-label">{tick}</text>
        {/each}

        {#each counts as [genre, count]}
          <g>
            <rect
              class="bar"
              x={x(genre)}
              y={y(count)}
              width={x.bandwidth()}
              height={innerHeight - y(count)}
              opacity={!selectedGenre || selectedGenre === genre ? 0.92 : 0.28}
              on:mouseenter={() => (selectedGenre = genre)}
              on:mouseleave={() => (selectedGenre = '')}
            >
              <title>{genre}: {count} movies</title>
            </rect>
            <text
              x={(x(genre) ?? 0) + x.bandwidth() / 2}
              y={y(count) - 6}
              text-anchor="middle"
              class="count"
              opacity={!selectedGenre || selectedGenre === genre ? 1 : 0.3}
            >{count}</text>
            <text
              x={(x(genre) ?? 0) + x.bandwidth() / 2}
              y={innerHeight + 14}
              transform={"rotate(45 " + ((x(genre) ?? 0) + x.bandwidth() / 2) + " " + (innerHeight + 14) + ")"}
              text-anchor="start"
              class="genre-label"
            >{genre}</text>
          </g>
        {/each}

        <line x1="0" x2={innerWidth} y1={innerHeight} y2={innerHeight} class="axis" />
        <line x1="0" x2="0" y1="0" y2={innerHeight} class="axis" />
        <text x={innerWidth / 2} y={innerHeight + 98} text-anchor="middle" class="axis-title">Genre</text>
        <text transform={"translate(-44," + innerHeight / 2 + ") rotate(-90)"} text-anchor="middle" class="axis-title">Movie count</text>
      </g>
    </svg>
  {:else}
    <p>Loading data...</p>
  {/if}
</section>

<style>
  .card { background:#fff; border:1px solid #e5e7eb; border-radius:16px; padding:1.25rem; box-shadow:0 8px 24px rgb(15 23 42 / 6%); }
  h2 { margin:0; }
  .subtitle { color:#64748b; margin:.35rem 0 1rem; }
  svg { width:100%; height:auto; overflow:visible; }
  .bar { fill:#2563eb; transition:opacity .15s ease, fill .15s ease; }
  .bar:hover { fill:#1d4ed8; }
  .grid { stroke:#e5e7eb; stroke-width:1; }
  .axis { stroke:#64748b; }
  .axis-label,.genre-label,.count { fill:#475569; font-size:11px; }
  .count { font-weight:700; }
  .axis-title { fill:#0f172a; font-size:13px; font-weight:700; }
</style>