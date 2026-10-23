<script lang="ts">
  import * as d3 from 'd3';
  import { onMount } from 'svelte';
  import { base } from '$app/paths';
  import type { TMovie } from '../../types';

  type Attribute = 'average_rating' | 'num_votes' | 'runtime_minutes' | 'year';
  type Annual = { year: number; count: number; mean: number };
  type GenreStat = { genre: string; count: number; mean: number };
  const attrs: { value: Attribute; label: string }[] = [
    { value: 'average_rating', label: 'IMDb rating (0–10)' },
    { value: 'num_votes', label: 'Audience votes (log₁₀)' },
    { value: 'runtime_minutes', label: 'Runtime (minutes)' },
    { value: 'year', label: 'Release year' }
  ];

  let movies: TMovie[] = [];
  let error = '';
  let xAttr: Attribute = 'num_votes';
  let yAttr: Attribute = 'average_rating';
  let sizeAttr: Attribute = 'runtime_minutes';
  let chosen: TMovie | null = null;
  let chosenGenre = '';
  let yearRange: [number, number] | null = null;
  let brushTarget: SVGGElement | null = null;
  let brushControl: d3.BrushBehavior<unknown> | null = null;

  const scatter = { w: 980, h: 440, l: 76, r: 30, t: 28, b: 70 };
  const trend = { w: 980, h: 245, l: 76, r: 30, t: 22, b: 45 };
  const genres = { w: 980, h: 315, l: 76, r: 30, t: 22, b: 95 };
  const attributeLabel = (attr: Attribute) => attrs.find((d) => d.value === attr)?.label ?? attr;
  const number = (n: number) => Math.round(n * 100) / 100;
  const metric = (d: TMovie, attr: Attribute) =>
    attr === 'year' ? d.year.getFullYear() :
    attr === 'num_votes' ? Math.log10(1 + d.num_votes) : d[attr];
  const valid = (d: TMovie) => Number.isFinite(d.average_rating)
    && d.average_rating >= 0 && d.average_rating <= 10
    && Number.isFinite(d.num_votes) && d.num_votes >= 0
    && Number.isFinite(d.runtime_minutes) && d.runtime_minutes > 0
    && Number.isFinite(d.year.getFullYear());
  const scale = (vals: number[], to: [number, number]) => {
    const extent = d3.extent(vals) as [number, number];
    const min = Number.isFinite(extent[0]) ? extent[0] : 0;
    const max = Number.isFinite(extent[1]) ? extent[1] : min + 1;
    const pad = (max - min || 1) * .04;
    return d3.scaleLinear().domain([min - pad, max + pad]).range(to);
  };

  onMount(async () => {
    try {
      const raw = await d3.csv(base + '/summer_movies.csv', (r) => ({
        tconst: r.tconst ?? '',
        title_type: r.title_type ?? '',
        primary_title: r.primary_title ?? '',
        original_title: r.original_title ?? '',
        year: new Date(Number(r.year), 0, 1),
        runtime_minutes: Number(r.runtime_minutes),
        genres: !r.genres || r.genres === 'NA' ? [] : r.genres.split(',').map((g) => g.trim()),
        average_rating: Number(r.average_rating),
        num_votes: Number(r.num_votes)
      } as TMovie));
      movies = raw.filter(valid);
    } catch (e) {
      error = e instanceof Error ? e.message : String(e);
    }
  });

  $: allYears = Array.from(new Set(movies.map((m) => m.year.getFullYear()))).sort(d3.ascending);
  $: firstYear = allYears[0] ?? 1946;
  $: lastYear = allYears[allYears.length - 1] ?? 2025;
  $: selectedStart = yearRange?.[0] ?? firstYear;
  $: selectedEnd = yearRange?.[1] ?? lastYear;
  $: timeMovies = movies.filter((m) => m.year.getFullYear() >= selectedStart && m.year.getFullYear() <= selectedEnd);
  $: linkedMovies = timeMovies.filter((m) => !chosenGenre || m.genres.includes(chosenGenre));

  $: yearly = d3.rollups(movies,
    (items) => ({ count: items.length, mean: d3.mean(items, (m) => m.average_rating) ?? 0 }),
    (m) => m.year.getFullYear()).map(([year, value]) => ({ year, ...value } as Annual)).sort((a, b) => a.year - b.year);
  $: yearScale = d3.scaleLinear().domain([firstYear, lastYear]).range([trend.l, trend.w - trend.r]);
  $: countScale = scale(yearly.map((d) => d.count), [trend.h - trend.b, trend.t]);
  $: linePath = d3.line<Annual>().x((d) => yearScale(d.year)).y((d) => countScale(d.count))(yearly) ?? '';
  $: ratingTrendScale = d3.scaleLinear().domain([0, 10]).range([trend.h - trend.b, trend.t]);
  $: meanRatingPath = d3.line<Annual>().x((d) => yearScale(d.year)).y((d) => ratingTrendScale(d.mean))(yearly) ?? '';

  $: xScale = scale(movies.map((m) => metric(m, xAttr)), [scatter.l, scatter.w - scatter.r]);
  $: yScale = scale(movies.map((m) => metric(m, yAttr)), [scatter.h - scatter.b, scatter.t]);
  $: sizeScale = d3.scaleSqrt()
    .domain(d3.extent(movies, (m) => metric(m, sizeAttr)) as [number, number])
    .range([3.3, 11]);
  $: allGenres = d3.rollups(timeMovies.flatMap((m) => m.genres), (values) => values.length, (g) => g)
    .sort((a, b) => b[1] - a[1]).slice(0, 12).map(([g]) => g);
  $: genreStats = allGenres.map((genre) => {
    const filtered = timeMovies.filter((m) => m.genres.includes(genre));
    return { genre, count: filtered.length, mean: d3.mean(filtered, (m) => m.average_rating) ?? 0 } as GenreStat;
  });
  $: genreX = d3.scaleBand<string>().domain(allGenres).range([genres.l, genres.w - genres.r]).padding(.22);
  $: genreY = d3.scaleLinear().domain([0, 10]).range([genres.h - genres.b, genres.t]);

  function resetFilters() {
    yearRange = null;
    chosen = null;
    chosenGenre = '';
    if (brushTarget && brushControl) d3.select(brushTarget).call(brushControl.move, null);
  }

  function brushYears(node: SVGGElement) {
    const selection = d3.select(node);
    brushTarget = node;
    const brush = d3.brushX<unknown>().extent([[trend.l, trend.t], [trend.w - trend.r, trend.h - trend.b]])
      .on('brush end', (event) => {
        if (!event.sourceEvent) return;
        if (!event.selection) { yearRange = null; return; }
        const [a, b] = event.selection as [number, number];
        yearRange = [
          Math.max(firstYear, Math.ceil(yearScale.invert(a))),
          Math.min(lastYear, Math.floor(yearScale.invert(b)))
        ];
        chosen = null;
      });
    brushControl = brush;
    selection.call(brush);
    return { destroy() { selection.on('.brush', null); brushTarget = null; brushControl = null; } };
  }
</script>

<svelte:head>
  <title>CSCI 5609 A2 | Interactive Analysis</title>
  <meta name="description" content="Linked interactive analysis of summer-movie ratings, audience engagement and genres." />
</svelte:head>

<main>
  <header>
    <p class="eyebrow">CSCI 5609 · ASSIGNMENT 2</p>
    <h1>Summer Movie Explorer</h1>
    <p>Linked views for exploring ratings, audience attention, genre, and release trends.</p>
    <nav><a href={base + '/A1'}>A1 Visual Encodings</a><a href={base + '/'}>Course home</a></nav>
  </header>

  {#if error}
    <section class="card error">Unable to load the movie dataset: {error}</section>
  {:else if !movies.length}
    <section class="card">Loading movie dataset…</section>
  {:else}
    <section class="stats">
      <div><strong>{movies.length}</strong><span>valid movie records</span></div>
      <div><strong>{linkedMovies.length}</strong><span>records in active selection</span></div>
      <div><strong>{selectedStart}–{selectedEnd}</strong><span>brushed release years</span></div>
      <div><strong>{chosenGenre || 'All genres'}</strong><span>genre cross-filter</span></div>
    </section>

    <section class="card">
      <div class="section-head">
        <div><p class="eyebrow">01 / Time brush</p><h2>When were these movies released?</h2></div>
        <button onclick={resetFilters}>Clear selections</button>
      </div>
      <p>Movie counts (blue, left axis) and average IMDb ratings (orange, right axis). <b>Drag a horizontal interval directly in the chart</b> to restrict the scatter plot and the genre comparison. Click outside the interval to reset.</p>
      <svg viewBox={`0 0 ${trend.w} ${trend.h}`} role="img" aria-label="Line chart of movie count by year with interactive horizontal brush">
        {#each countScale.ticks(5) as tick}
          <line x1={trend.l} y1={countScale(tick)} x2={trend.w - trend.r} y2={countScale(tick)} stroke="#e2e8f0" />
          <text x={trend.l - 10} y={countScale(tick) + 4} text-anchor="end" class="tick">{tick}</text>
        {/each}
        {#each yearScale.ticks(10) as tick}
          <text x={yearScale(tick)} y={trend.h - 14} text-anchor="middle" class="tick">{tick}</text>
        {/each}
        {#each ratingTrendScale.ticks(5) as tick}
          <text x={trend.w - trend.r + 7} y={ratingTrendScale(tick)+4} text-anchor="start" class="tick">{tick}</text>
        {/each}
        <path d={linePath} fill="none" stroke="#2563eb" stroke-width="3" stroke-linecap="round" stroke-linejoin="round"/>
        <path d={meanRatingPath} fill="none" stroke="#ea580c" stroke-width="2.5" stroke-dasharray="7 5" stroke-linecap="round"/>
        <g class="year-brush" use:brushYears></g>
      </svg>
    </section>

    <section class="card">
      <div class="section-head">
        <div><p class="eyebrow">02 / Interactive scatter</p><h2>What predicts a movie’s rating?</h2></div>
        <span class="pill">{linkedMovies.length} movies shown</span>
      </div>
      <p>Choose independent data dimensions for horizontal position, vertical position and circle radius. Click a circle to inspect the exact record on the right. Opacity reduces overplotting.</p>
      <div class="controls">
        <label>X position<select bind:value={xAttr}>{#each attrs as a}<option value={a.value}>{a.label}</option>{/each}</select></label>
        <label>Y position<select bind:value={yAttr}>{#each attrs as a}<option value={a.value}>{a.label}</option>{/each}</select></label>
        <label>Circle size<select bind:value={sizeAttr}>{#each attrs as a}<option value={a.value}>{a.label}</option>{/each}</select></label>
      </div>
      <div class="scatter-grid">
        <svg viewBox={`0 0 ${scatter.w} ${scatter.h}`} role="img" aria-label="Interactive scatter plot of summer movies">
          {#each yScale.ticks(6) as tick}
            <line x1={scatter.l} y1={yScale(tick)} x2={scatter.w-scatter.r} y2={yScale(tick)} stroke="#e2e8f0" />
            <text x={scatter.l - 8} y={yScale(tick)+4} text-anchor="end" class="tick">{number(tick)}</text>
          {/each}
          {#each xScale.ticks(7) as tick}
            <text x={xScale(tick)} y={scatter.h - scatter.b + 25} text-anchor="middle" class="tick">{number(tick)}</text>
          {/each}
          {#each linkedMovies as m (m.tconst)}
            <circle cx={xScale(metric(m, xAttr))} cy={yScale(metric(m, yAttr))}
              r={Math.max(3, Number(sizeScale(metric(m, sizeAttr))))}
              fill={chosen?.tconst === m.tconst ? '#f97316' : '#2563eb'}
              stroke={chosen?.tconst === m.tconst ? '#9a3412' : '#ffffff'}
              stroke-width={chosen?.tconst === m.tconst ? 2.5 : 0.6}
              fill-opacity={chosen?.tconst === m.tconst ? 1 : .28}
              onclick={() => chosen = m}
              role="button" tabindex="0"
              onkeydown={(e) => { if (e.key === 'Enter' || e.key === ' ') chosen = m; }}>
              <title>{m.primary_title} ({m.year.getFullYear()}) · Rating {m.average_rating}</title>
            </circle>
          {/each}
          <text x={scatter.w / 2} y={scatter.h - 7} class="axis" text-anchor="middle">{attributeLabel(xAttr)}</text>
          <text x="20" y={scatter.h / 2} transform={`rotate(-90 20 ${scatter.h / 2})`} class="axis" text-anchor="middle">{attributeLabel(yAttr)}</text>
        </svg>
        <aside>
          <h3>Selected movie</h3>
          {#if chosen && linkedMovies.some((m) => m.tconst === chosen?.tconst)}
            <strong class="movie-title">{chosen.primary_title}</strong>
            <dl>
              <dt>Year</dt><dd>{chosen.year.getFullYear()}</dd>
              <dt>Rating</dt><dd>{chosen.average_rating.toFixed(1)} / 10</dd>
              <dt>Votes</dt><dd>{chosen.num_votes.toLocaleString()}</dd>
              <dt>Runtime</dt><dd>{chosen.runtime_minutes} min</dd>
              <dt>Genres</dt><dd>{chosen.genres.join(', ')}</dd>
              <dt>IMDb ID</dt><dd>{chosen.tconst}</dd>
            </dl>
          {:else}
            <p>Click a dot in the scatter plot to display its exact metadata here.</p>
          {/if}
        </aside>
      </div>
    </section>

    <section class="card">
      <div class="section-head">
        <div><p class="eyebrow">03 / Self-directed EDA · linked second view</p><h2>Do genres explain differences in audience ratings?</h2></div>
        {#if chosenGenre}<button onclick={() => { chosenGenre = ''; chosen = null; }}>Show all genres</button>{/if}
      </div>
      <p>Mean IMDb ratings for the 12 most common genres in the brushed period. <b>Click a genre bar</b> to cross-filter the scatter plot above; films with multiple genres may appear in multiple bars.</p>
      <svg viewBox={`0 0 ${genres.w} ${genres.h}`} role="img" aria-label="Linked bar chart of average rating by genre">
        {#each genreY.ticks(6) as tick}
          <line x1={genres.l} y1={genreY(tick)} x2={genres.w-genres.r} y2={genreY(tick)} stroke="#e2e8f0"/>
          <text x={genres.l-8} y={genreY(tick)+4} class="tick" text-anchor="end">{tick}</text>
        {/each}
        {#each genreStats as g}
          <rect x={genreX(g.genre)} y={genreY(g.mean)} width={genreX.bandwidth()}
            height={Math.max(0, genreY(0)-genreY(g.mean))}
            rx="4" fill={chosenGenre === g.genre ? '#ea580c' : '#0f766e'}
            fill-opacity={chosenGenre && chosenGenre !== g.genre ? .35 : 1}
            role="button" tabindex="0" onclick={() => { chosenGenre = chosenGenre === g.genre ? '' : g.genre; chosen = null; }}
            onkeydown={(e) => { if (e.key === 'Enter' || e.key === ' ') { chosenGenre = chosenGenre === g.genre ? '' : g.genre; chosen = null; } }}>
            <title>{g.genre}: mean rating {g.mean.toFixed(2)}, n={g.count}</title>
          </rect>
          <text x={(genreX(g.genre) ?? 0) + genreX.bandwidth()/2} y={genreY(g.mean)-6} text-anchor="middle" class="bar-value">{g.mean.toFixed(1)}</text>
          <text x={(genreX(g.genre) ?? 0) + genreX.bandwidth()/2} y={genres.h-genres.b+16}
            transform={`rotate(-42 ${(genreX(g.genre) ?? 0) + genreX.bandwidth()/2} ${genres.h-genres.b+16})`}
            text-anchor="end" class="tick">{g.genre}</text>
        {/each}
      </svg>
      <p class="note">Dataset: <code>summer_movies.csv</code>. Descriptive associations, not causal claims. Vote counts are logarithmically transformed only for position; the details panel shows raw counts.</p>
    </section>
  {/if}
  <footer><a href="https://github.com/QinyangTan/my-vis-5609">Source repository</a> · <span>CSCI 5609 / A2</span></footer>
</main>

<style>
  :global(body){background:#f4f7fb;color:#172438;font:16px/1.5 system-ui,-apple-system,sans-serif;margin:0}
  main{max-width:1260px;padding:32px 24px 64px;margin:auto;display:grid;gap:20px}
  header{padding:12px 4px}header p{color:#52657d;margin:5px 0}h1{font-size:clamp(2.2rem,4.3vw,3.5rem);letter-spacing:-.04em;margin:2px 0}
  h2{font-size:1.5rem;margin:0 0 8px}h3{margin-top:0}.eyebrow{color:#2563eb;font-size:.76rem;letter-spacing:.15em;font-weight:800;text-transform:uppercase}
  nav{display:flex;gap:20px;margin-top:12px}a{color:#1d4ed8;text-decoration:none;font-weight:600}
  .card{border:1px solid #dce4ee;border-radius:18px;background:white;padding:24px;box-shadow:0 7px 28px #17243809}
  .card p{color:#536478;margin:0 0 15px}.section-head{display:flex;justify-content:space-between;align-items:center;gap:16px;flex-wrap:wrap}
  .stats{display:grid;grid-template-columns:repeat(4,1fr);gap:14px}.stats div{background:#132d4d;color:white;border-radius:13px;padding:18px;display:grid}
  .stats strong{font-size:1.5rem;line-height:1.5}.stats span{font-size:.8rem;color:#c7d9f2}
  button{background:white;border:1px solid #b2c6e0;color:#17477d;padding:9px 15px;border-radius:9px;cursor:pointer;font:inherit;font-weight:600}
  button:hover{background:#eff6ff}svg{width:100%;height:auto;display:block}
  .tick{font-size:11px;fill:#587087}.axis{font-size:14px;font-weight:700;fill:#334155}.bar-value{font-size:11px;fill:#164e63;font-weight:700}
  .controls{display:grid;grid-template-columns:repeat(3,minmax(0,1fr));gap:12px;margin:14px 0}
  label{display:grid;gap:6px;font-size:13px;color:#334155;font-weight:700}
  select{font:inherit;padding:9px;border-radius:8px;border:1px solid #bfccdd;background:white;max-width:100%}
  .scatter-grid{display:grid;grid-template-columns:minmax(0,3fr) minmax(185px,1fr);gap:14px;align-items:start}
  aside{border:1px solid #d5e2f1;border-radius:14px;padding:16px;background:#f8fbff;min-height:230px}
  .movie-title{font-size:1.1rem;display:block;margin-bottom:12px}
  dl{display:grid;grid-template-columns:1fr 1fr;gap:7px 9px;margin:0;font-size:13px;overflow-wrap:anywhere}
  dt{color:#60728a}dd{margin:0;font-weight:700;text-align:right}
  .pill{background:#e7f0fd;color:#1e40af;border-radius:30px;padding:6px 12px;font-size:12px;font-weight:700}
  circle,rect[role="button"]{cursor:pointer;transition:fill-opacity .12s}
  circle:hover{fill-opacity:1;stroke:#1e40af;stroke-width:2}
  .note{font-size:13px;margin-top:12px!important}.error{color:#b91c1c}footer{text-align:center;color:#64748b}
  @media(max-width:830px){main{padding:15px}.stats{grid-template-columns:repeat(2,1fr)}.scatter-grid{grid-template-columns:1fr}.controls{grid-template-columns:1fr}.card{padding:15px}}
</style>
