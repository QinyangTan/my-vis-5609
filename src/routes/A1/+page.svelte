<script lang="ts">
  import * as d3 from 'd3';
  import { onMount } from 'svelte';
  import { base } from '$app/paths';
  import type { TMovie } from '../../types';
  import { Bar } from '$lib';

  type AnnualTop = { year: number; genre: string; count: number; rank: number };
  type PairCell = { a: string; b: string; count: number };

  let movies: TMovie[] = [];
  let loadError = '';
  let hoveredTop = '';
  let hoveredPair = '';

  async function loadCsv() {
    try {
      const rows = await d3.csv(base + '/summer_movies.csv', (row) => {
        const yearNumber = Number(row.year);
        return {
          tconst: row.tconst ?? '',
          title_type: row.title_type ?? '',
          primary_title: row.primary_title ?? '',
          original_title: row.original_title ?? '',
          year: new Date(yearNumber, 0, 1),
          runtime_minutes: row.runtime_minutes === 'NA' ? 0 : Number(row.runtime_minutes),
          genres: !row.genres || row.genres === 'NA'
            ? []
            : row.genres.split(',').map((d) => d.trim()),
          average_rating: Number(row.average_rating),
          num_votes: Number(row.num_votes)
        } as TMovie;
      });
      movies = rows.filter((d) => Number.isFinite(d.year.getFullYear()) && d.genres.length > 0);
    } catch (error) {
      loadError = error instanceof Error ? error.message : String(error);
    }
  }

  onMount(loadCsv);

  function annualTop3(data: TMovie[]): AnnualTop[] {
    const grouped = d3.group(data, (d) => d.year.getFullYear());
    return Array.from(grouped, ([year, yearMovies]) => {
      const counts = d3.rollups(
        yearMovies.flatMap((d) => d.genres),
        (v) => v.length,
        (genre) => genre
      ).sort((a, b) => d3.descending(a[1], b[1]) || d3.ascending(a[0], b[0]));
      return counts.slice(0, 3).map(([genre, count], index) => ({
        year,
        genre,
        count,
        rank: index + 1
      }));
    }).flat();
  }

  function genrePairs(data: TMovie[]): PairCell[] {
    const map = new Map<string, number>();
    for (const movie of data) {
      const unique = Array.from(new Set(movie.genres)).sort();
      for (let i = 0; i < unique.length; i += 1) {
        for (let j = i + 1; j < unique.length; j += 1) {
          const key = unique[i] + '|||' + unique[j];
          map.set(key, (map.get(key) ?? 0) + 1);
        }
      }
    }
    return Array.from(map, ([key, count]) => {
      const parts = key.split('|||');
      return { a: parts[0], b: parts[1], count };
    });
  }

  $: top3 = annualTop3(movies);
  $: years = Array.from(new Set(top3.map((d) => d.year))).sort(d3.ascending);
  $: topGenreFreq = d3.rollups(top3, (v) => v.length, (d) => d.genre)
    .sort((a, b) => d3.descending(a[1], b[1]));
  $: topGenres = topGenreFreq.map(([genre]) => genre);
  $: pairs = genrePairs(movies);
  $: allGenres = Array.from(new Set(movies.flatMap((d) => d.genres))).sort();
  $: pairLookup = new Map(
    pairs.flatMap((d) => [
      [d.a + '|||' + d.b, d.count] as [string, number],
      [d.b + '|||' + d.a, d.count] as [string, number]
    ])
  );
  $: maxPair = d3.max(pairs, (d) => d.count) ?? 1;
  $: mostPersistent = topGenreFreq[0];
  $: comedyPairs = pairs
    .filter((d) => d.a === 'Comedy' || d.b === 'Comedy')
    .map((d) => ({ genre: d.a === 'Comedy' ? d.b : d.a, count: d.count }))
    .sort((a, b) => d3.descending(a.count, b.count));

  const q1Width = 1120;
  const q1Height = 610;
  const q1Margin = { top: 45, right: 28, bottom: 70, left: 120 };
  $: q1x = d3.scaleBand<number>()
    .domain(years)
    .range([q1Margin.left, q1Width - q1Margin.right])
    .padding(0.04);
  $: q1y = d3.scaleBand<string>()
    .domain(topGenres)
    .range([q1Margin.top, q1Height - q1Margin.bottom])
    .padding(0.08);
  const rankColor = d3.scaleOrdinal<number, string>()
    .domain([1, 2, 3])
    .range(['#7c2d12', '#ea580c', '#fdba74']);

  const q2Width = 940;
  const q2Height = 940;
  const q2Margin = { top: 135, right: 30, bottom: 30, left: 135 };
  $: q2x = d3.scaleBand<string>()
    .domain(allGenres)
    .range([q2Margin.left, q2Width - q2Margin.right])
    .padding(0.04);
  $: q2y = d3.scaleBand<string>()
    .domain(allGenres)
    .range([q2Margin.top, q2Height - q2Margin.bottom])
    .padding(0.04);
  $: pairColor = d3.scaleSequential(d3.interpolateBlues).domain([0, maxPair]);
</script>

<svelte:head>
  <title>CSCI 5609 A1 - Visual Encodings</title>
  <meta name="description" content="Summer Movies visual encoding assignment by QinYang Tan" />
</svelte:head>

<main>
  <header class="hero">
    <div>
      <p class="eyebrow">CSCI 5609 - A1</p>
      <h1>Summer Movies: Visual Encodings</h1>
      <p>QinYang Tan - annual genre rankings and genre co-occurrence</p>
    </div>
    <a href={base + '/A0'}>View A0</a>
  </header>

  {#if loadError}
    <div class="error">Failed to load data: {loadError}</div>
  {:else}
    <section class="summary">
      <div><strong>{movies.length || '...'}</strong><span>titles</span></div>
      <div><strong>{allGenres.length || '...'}</strong><span>genres</span></div>
      <div>
        <strong>{years.length ? years[0] + '-' + years[years.length - 1] : '...'}</strong>
        <span>years represented</span>
      </div>
    </section>

    <Bar {movies} />

    <section class="card">
      <div class="section-head">
        <div>
          <p class="eyebrow">Question 1</p>
          <h2>How do the annual top 3 genres change over time?</h2>
          <p>Each colored cell marks a genre that ranked in the top 3 for that year. Darker orange means a higher rank.</p>
        </div>
        <div class="legend">
          <span><i style="background:#7c2d12"></i>#1</span>
          <span><i style="background:#ea580c"></i>#2</span>
          <span><i style="background:#fdba74"></i>#3</span>
        </div>
      </div>

      {#if top3.length}
        <svg viewBox={"0 0 " + q1Width + " " + q1Height} role="img" aria-label="Heatmap of annual top three genres">
          {#each topGenres as genre}
            <text
              x={q1Margin.left - 10}
              y={(q1y(genre) ?? 0) + q1y.bandwidth()/2 + 4}
              text-anchor="end"
              class="label"
            >{genre}</text>
          {/each}

          {#each years as year, i}
            {#if i % 5 === 0 || i === years.length - 1}
              <text
                x={(q1x(year) ?? 0) + q1x.bandwidth()/2}
                y={q1Height - q1Margin.bottom + 23}
                text-anchor="middle"
                class="label"
              >{year}</text>
            {/if}
          {/each}

          {#each top3 as d}
            <rect
              x={q1x(d.year)}
              y={q1y(d.genre)}
              width={q1x.bandwidth()}
              height={q1y.bandwidth()}
              rx="2"
              fill={rankColor(d.rank)}
              opacity={!hoveredTop || hoveredTop === d.genre ? 1 : 0.22}
              on:mouseenter={() => (hoveredTop = d.genre)}
              on:mouseleave={() => (hoveredTop = '')}
            >
              <title>{d.year}: #{d.rank} {d.genre} ({d.count} movies)</title>
            </rect>
          {/each}
          <text x={q1Width/2} y={q1Height - 10} text-anchor="middle" class="axis-title">Release year</text>
        </svg>
        <p class="insight">
          <strong>Insight:</strong>
          {#if mostPersistent}
            {mostPersistent[0]} appears in the annual top 3 in {mostPersistent[1]} different years,
            the most persistent genre in this dataset.
          {/if}
          Hover any row to trace one genre across time.
        </p>
      {:else}
        <p>Loading visualization...</p>
      {/if}
    </section>

    <section class="card">
      <div class="section-head">
        <div>
          <p class="eyebrow">Question 2</p>
          <h2>Which genres frequently co-occur?</h2>
          <p>The symmetric matrix shows how many movies contain each pair of genres. Darker blue means more co-occurrences.</p>
        </div>
      </div>

      {#if allGenres.length}
        <svg viewBox={"0 0 " + q2Width + " " + q2Height} role="img" aria-label="Genre co-occurrence matrix">
          {#each allGenres as genre}
            <text
              x={q2Margin.left - 8}
              y={(q2y(genre) ?? 0) + q2y.bandwidth()/2 + 4}
              text-anchor="end"
              class="matrix-label"
            >{genre}</text>
            <text
              x={(q2x(genre) ?? 0) + q2x.bandwidth()/2}
              y={q2Margin.top - 8}
              transform={"rotate(-55 " + ((q2x(genre) ?? 0) + q2x.bandwidth()/2) + " " + (q2Margin.top - 8) + ")"}
              text-anchor="start"
              class="matrix-label"
            >{genre}</text>
          {/each}

          {#each allGenres as a}
            {#each allGenres as b}
              {@const count = a === b ? 0 : (pairLookup.get(a + '|||' + b) ?? 0)}
              {@const pairKey = [a, b].sort().join(' + ')}
              <rect
                x={q2x(b)}
                y={q2y(a)}
                width={q2x.bandwidth()}
                height={q2y.bandwidth()}
                fill={a === b ? '#f1f5f9' : pairColor(count)}
                stroke="#ffffff"
                stroke-width="0.6"
                opacity={!hoveredPair || hoveredPair === pairKey ? 1 : 0.35}
                on:mouseenter={() => (hoveredPair = pairKey)}
                on:mouseleave={() => (hoveredPair = '')}
              >
                <title>{a} + {b}: {count} movies</title>
              </rect>
            {/each}
          {/each}
        </svg>
        <p class="insight">
          <strong>Insight:</strong>
          {#if comedyPairs.length}
            Comedy most often co-occurs with {comedyPairs[0].genre} ({comedyPairs[0].count} movies),
            followed by {comedyPairs[1]?.genre} ({comedyPairs[1]?.count}).
          {/if}
          Hover a matrix cell to inspect a pair.
        </p>
      {:else}
        <p>Loading visualization...</p>
      {/if}
    </section>
  {/if}

  <footer>
    <a href="https://github.com/QinyangTan/my-vis-5609" target="_blank" rel="noreferrer">Repository</a>
    <a href="https://qinyangtan.github.io/my-vis-5609/A1" target="_blank" rel="noreferrer">Published A1</a>
  </footer>
</main>

<style>
  :global(body) {
    margin:0;
    font-family:Inter,ui-sans-serif,system-ui,-apple-system,BlinkMacSystemFont,"Segoe UI",sans-serif;
    background:#f8fafc;
    color:#0f172a;
  }
  main { max-width:1180px; margin:0 auto; padding:2rem 1.25rem 4rem; display:grid; gap:1.4rem; }
  .hero { display:flex; justify-content:space-between; gap:1rem; align-items:end; padding:1rem 0 .5rem; }
  h1 { font-size:clamp(2rem,4vw,3.4rem); margin:.1rem 0 .45rem; letter-spacing:-.04em; }
  h2 { margin:.1rem 0 .4rem; font-size:1.55rem; }
  p { line-height:1.55; }
  .hero p { color:#64748b; margin:0; }
  .hero a, footer a { color:#1d4ed8; font-weight:700; text-decoration:none; }
  .eyebrow { text-transform:uppercase; letter-spacing:.12em; font-size:.76rem; font-weight:800; color:#ea580c !important; margin:0; }
  .summary { display:grid; grid-template-columns:repeat(3,1fr); gap:1rem; }
  .summary div { background:#0f172a; color:white; border-radius:14px; padding:1rem 1.2rem; display:flex; flex-direction:column; }
  .summary strong { font-size:1.55rem; }
  .summary span { color:#cbd5e1; font-size:.85rem; margin-top:.2rem; }
  .card { background:white; border:1px solid #e5e7eb; border-radius:16px; padding:1.25rem; box-shadow:0 8px 24px rgb(15 23 42 / 6%); overflow:auto; }
  .section-head { display:flex; justify-content:space-between; gap:1rem; align-items:start; margin-bottom:.5rem; }
  .section-head p:not(.eyebrow) { color:#64748b; margin:.2rem 0; }
  .legend { display:flex; gap:.8rem; white-space:nowrap; padding-top:1.4rem; }
  .legend span { display:flex; gap:.35rem; align-items:center; font-size:.85rem; color:#475569; }
  .legend i { width:14px; height:14px; border-radius:3px; display:inline-block; }
  svg { width:100%; height:auto; min-width:760px; }
  .label,.matrix-label { fill:#475569; font-size:11px; }
  .axis-title { fill:#0f172a; font-size:13px; font-weight:700; }
  .insight { background:#fff7ed; border-left:4px solid #f97316; padding:.75rem 1rem; border-radius:8px; margin:.8rem 0 0; }
  .error { background:#fef2f2; color:#991b1b; padding:1rem; border-radius:10px; }
  footer { display:flex; gap:1.2rem; justify-content:center; padding-top:.6rem; }
  @media (max-width:720px) {
    .summary { grid-template-columns:1fr; }
    .hero,.section-head { flex-direction:column; align-items:start; }
    .legend { padding-top:0; }
  }
</style>