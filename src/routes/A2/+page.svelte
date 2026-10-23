<script lang="ts">
  import * as d3 from 'd3';
  import { onMount } from 'svelte';
  import { base } from '$app/paths';
  import type { TMovie } from '../../types';

  type Metric = 'year' | 'average_rating' | 'runtime_minutes' | 'num_votes';
  let movies: TMovie[] = [];
  let error = '';
  let xMetric: Metric = 'year';
  let yMetric: Metric = 'average_rating';
  let sizeMetric: Metric = 'num_votes';
  let selected: TMovie | null = null;
  let selectedGenre = 'All';
  let brushStart: number | null = null;
  let brushEnd: number | null = null;
  let dragging = false;

  const fields: {key: Metric; label: string}[] = [
    {key:'year',label:'Release year'}, {key:'average_rating',label:'IMDb rating'},
    {key:'runtime_minutes',label:'Runtime (minutes)'}, {key:'num_votes',label:'Number of votes'}
  ];
  const W = 940, scatterH = 450, lineH = 260;
  const L = 72, R = 30, T = 25, B = 54;
  const value = (m: TMovie, key: Metric) => key === 'year' ? m.year.getFullYear() : m[key];
  const fmt = (v:number) => Number.isInteger(v) ? d3.format(',')(v) : d3.format('.2f')(v);

  onMount(async () => {
    try {
      const rows = await d3.csv(base + '/summer_movies.csv', row => ({
        tconst: row.tconst || '',
        title_type: row.title_type || '',
        primary_title: row.primary_title || '',
        original_title: row.original_title || '',
        year: new Date(Number(row.year), 0, 1),
        runtime_minutes: row.runtime_minutes === 'NA' ? 0 : Number(row.runtime_minutes),
        genres: !row.genres || row.genres === 'NA' ? [] : row.genres.split(',').map(s=>s.trim()),
        average_rating: Number(row.average_rating),
        num_votes: Number(row.num_votes)
      } as TMovie));
      movies = rows.filter(m => Number.isFinite(m.year.getFullYear()) && Number.isFinite(m.average_rating) && m.num_votes > 0);
    } catch (e) { error = e instanceof Error ? e.message : String(e); }
  });

  $: years = movies.map(m => m.year.getFullYear());
  $: minYear = d3.min(years) ?? 1990;
  $: maxYear = d3.max(years) ?? 2025;
  $: yearly = d3.rollups(movies, v => d3.mean(v, d => d.average_rating) ?? 0, m => m.year.getFullYear())
    .sort((a,b) => a[0] - b[0]);
  $: genres = Array.from(new Set(movies.flatMap(m=>m.genres))).sort();
  $: rangeStart = brushStart === null || brushEnd === null ? minYear : Math.min(brushStart, brushEnd);
  $: rangeEnd = brushStart === null || brushEnd === null ? maxYear : Math.max(brushStart, brushEnd);
  $: visible = movies.filter(m => m.year.getFullYear() >= rangeStart && m.year.getFullYear() <= rangeEnd && (selectedGenre === 'All' || m.genres.includes(selectedGenre)));
  $: xExtent = d3.extent(movies, m => value(m, xMetric)) as [number,number];
  $: yExtent = d3.extent(movies, m => value(m, yMetric)) as [number,number];
  $: xDomain = xExtent[0] === xExtent[1] ? [xExtent[0]-1,xExtent[1]+1] : xExtent;
  $: yDomain = yExtent[0] === yExtent[1] ? [yExtent[0]-1,yExtent[1]+1] : yExtent;
  $: xScale = d3.scaleLinear().domain(xDomain).nice().range([L,W-R]);
  $: yScale = d3.scaleLinear().domain(yDomain).nice().range([scatterH-B,T]);
  $: sizeExtent = d3.extent(movies, m=>value(m,sizeMetric)) as [number,number];
  $: sizeScale = d3.scaleSqrt().domain(sizeExtent[0] === sizeExtent[1] ? [0,sizeExtent[1]+1] : sizeExtent).range([3,11]);
  $: yearScale = d3.scaleLinear().domain([minYear,maxYear]).range([L,W-R]);
  $: trendScale = d3.scaleLinear().domain([0,10]).range([lineH-B,T]);
  $: trendPath = d3.line<[number,number]>().x(d=>yearScale(d[0])).y(d=>trendScale(d[1]))(yearly) ?? '';
  $: genreStats = d3.rollups(movies.flatMap(m=>m.genres.map(g=>({genre:g,movie:m}))),
    v=>({count:v.length, rating:d3.mean(v,d=>d.movie.average_rating)??0, votes:d3.mean(v,d=>d.movie.num_votes)??0}), d=>d.genre)
    .filter(d=>d[1].count>=5).sort((a,b)=>b[1].count-a[1].count).slice(0,12);
  $: genreTrend = d3.rollups(movies.filter(m=>selectedGenre==='All'||m.genres.includes(selectedGenre)),
    v=>d3.mean(v,m=>m.average_rating)??0, m=>m.year.getFullYear()).sort((a,b)=>a[0]-b[0]);
  $: genreTrendPath = d3.line<[number,number]>().x(d=>yearScale(d[0])).y(d=>trendScale(d[1]))(genreTrend)??'';

  function brushYear(e:PointerEvent): number {
    const box=(e.currentTarget as SVGRectElement).getBoundingClientRect();
    const px=L+Math.max(0,Math.min(1,(e.clientX-box.left)/box.width))*(W-R-L);
    return Math.max(minYear,Math.min(maxYear,Math.round(yearScale.invert(px))));
  }
  function startBrush(e:PointerEvent) {
    dragging=true;
    brushStart=brushYear(e); brushEnd=brushStart;
    (e.currentTarget as SVGRectElement).setPointerCapture(e.pointerId);
  }
  function moveBrush(e:PointerEvent) { if(dragging) brushEnd=brushYear(e); }
  function endBrush(e:PointerEvent) { if(dragging) brushEnd=brushYear(e); dragging=false; }
  function clearFilters(){ brushStart=null;brushEnd=null;selectedGenre='All'; }
</script>

<svelte:head>
  <title>CSCI 5609 A2 — Interactive Analysis</title>
  <meta name="description" content="Interactive linked dashboard for summer movie ratings" />
</svelte:head>

<main>
  <header>
    <p class="eyebrow">CSCI 5609 · Assignment 2</p>
    <h1>Summer Movies — Interactive Analysis</h1>
    <p>Explore how ratings, audience size, runtime, release year, and genre relate. Click a dot to inspect a film; drag on the trend to brush years; click a genre to link views.</p>
    <div class="stats"><span>{movies.length} movies loaded</span><span>{visible.length} in current selection</span><button on:click={clearFilters}>Reset filters</button></div>
  </header>

  {#if error}<div class="error">Dataset could not be loaded: {error}</div>{/if}
  {#if !movies.length && !error}<p>Loading movie dataset…</p>{/if}

  {#if movies.length}
    <section class="card">
      <h2>1. Movie scatterplot</h2>
      <p>Choose the x, y, and size encodings. Dots are semi-transparent so dense clusters remain visible; brushed-out years and other genres are excluded.</p>
      <div class="controls">
        <label>X axis <select bind:value={xMetric}>{#each fields as f}<option value={f.key}>{f.label}</option>{/each}</select></label>
        <label>Y axis <select bind:value={yMetric}>{#each fields as f}<option value={f.key}>{f.label}</option>{/each}</select></label>
        <label>Dot size <select bind:value={sizeMetric}>{#each fields as f}<option value={f.key}>{f.label}</option>{/each}</select></label>
      </div>
      <div class="two-col">
        <svg viewBox={'0 0 '+W+' '+scatterH} role="img" aria-label="Interactive movie scatterplot">
          {#each xScale.ticks(8) as tick}
            <line x1={xScale(tick)} x2={xScale(tick)} y1={T} y2={scatterH-B} stroke="#e2e8f0"/>
            <text x={xScale(tick)} y={scatterH-B+21} text-anchor="middle" class="tick">{fmt(tick)}</text>
          {/each}
          {#each yScale.ticks(6) as tick}
            <line x1={L} x2={W-R} y1={yScale(tick)} y2={yScale(tick)} stroke="#e2e8f0"/>
            <text x={L-9} y={yScale(tick)+4} text-anchor="end" class="tick">{fmt(tick)}</text>
          {/each}
          {#each visible as movie (movie.tconst)}
            <circle cx={xScale(value(movie,xMetric))} cy={yScale(value(movie,yMetric))}
              r={sizeScale(value(movie,sizeMetric))}
              fill={selected?.tconst===movie.tconst?'#ea580c':'#2563eb'}
              stroke={selected?.tconst===movie.tconst?'#9a3412':'white'}
              stroke-width={selected?.tconst===movie.tconst?2:0.4} opacity={selected?.tconst===movie.tconst?1:0.48}
              on:click={() => selected=movie} role="button" tabindex="0"
              on:keydown={(e)=>{if(e.key==='Enter')selected=movie}}>
              <title>{movie.primary_title} ({movie.year.getFullYear()}), rating {movie.average_rating}</title>
            </circle>
          {/each}
          <text x={W/2} y={scatterH-6} text-anchor="middle" class="axis">{fields.find(f=>f.key===xMetric)?.label}</text>
          <text x="16" y={scatterH/2} transform={'rotate(-90 16 '+scatterH/2+')'} text-anchor="middle" class="axis">{fields.find(f=>f.key===yMetric)?.label}</text>
        </svg>
        <aside class="details">
          <h3>Selected movie</h3>
          {#if selected}
            <h4>{selected.primary_title}</h4>
            <dl>
              <dt>Year</dt><dd>{selected.year.getFullYear()}</dd>
              <dt>Rating</dt><dd>{selected.average_rating.toFixed(1)} / 10</dd>
              <dt>Votes</dt><dd>{d3.format(',')(selected.num_votes)}</dd>
              <dt>Runtime</dt><dd>{selected.runtime_minutes} min</dd>
              <dt>Genres</dt><dd>{selected.genres.join(', ')}</dd>
            </dl>
          {:else}<p>Click any blue dot to inspect its title, year, rating, votes, runtime and genres.</p>{/if}
        </aside>
      </div>
    </section>

    <section class="card">
      <h2>2. Brush the rating trend to filter the scatterplot</h2>
      <p>Drag horizontally across the plot. The highlighted interval determines which years appear above. Click “Reset filters” to show all years again.</p>
      <svg viewBox={'0 0 '+W+' '+lineH} role="img" aria-label="Yearly mean movie rating with interactive year brush">
        {#each trendScale.ticks(5) as tick}
          <line x1={L} x2={W-R} y1={trendScale(tick)} y2={trendScale(tick)} stroke="#e2e8f0"/>
          <text x={L-10} y={trendScale(tick)+4} text-anchor="end" class="tick">{tick}</text>
        {/each}
        {#each yearScale.ticks(10) as tick}
          <text x={yearScale(tick)} y={lineH-B+22} text-anchor="middle" class="tick">{Math.round(tick)}</text>
        {/each}
        <path d={trendPath} fill="none" stroke="#0f766e" stroke-width="3"/>
        {#if brushStart!==null && brushEnd!==null}
          <rect x={yearScale(rangeStart)} y={T} width={Math.max(2,yearScale(rangeEnd)-yearScale(rangeStart))}
            height={lineH-B-T} fill="#38bdf8" opacity=".24" pointer-events="none"/>
        {/if}
        <rect x={L} y={T} width={W-R-L} height={lineH-B-T} fill="transparent"
          style="cursor:crosshair;touch-action:none"
          on:pointerdown={startBrush} on:pointermove={moveBrush} on:pointerup={endBrush} />
        <text x={W/2} y={lineH-7} text-anchor="middle" class="axis">Release year</text>
      </svg>
      <p class="selection">Selected years: <strong>{rangeStart}–{rangeEnd}</strong> · {visible.length} movies shown in the scatterplot.</p>
    </section>

    <section class="card">
      <h2>3. Linked genre exploration</h2>
      <p>Exploratory question: Do popular genres consistently receive higher ratings? Select a genre in the bar chart; the adjacent annual-rating trend and main scatterplot update together. Bars show mean rating; counts show the amount of evidence.</p>
      <div class="two-panels">
        <div>
          <h3>Average rating by genre</h3>
          {#each genreStats as [genre,stat]}
            <button class:selected={selectedGenre===genre} class="genre-row" on:click={()=>selectedGenre=selectedGenre===genre?'All':genre}>
              <span class="genre-name">{genre}</span>
              <span class="bar"><span style={'width:'+(stat.rating/10*100)+'%'}></span></span>
              <strong>{stat.rating.toFixed(2)}</strong><small>n={stat.count}</small>
            </button>
          {/each}
        </div>
        <div>
          <h3>{selectedGenre==='All'?'All genres':selectedGenre} — annual average rating</h3>
          <svg viewBox={'0 0 '+W+' '+lineH} role="img" aria-label="Linked yearly genre rating trend">
            {#each trendScale.ticks(5) as tick}
              <line x1={L} x2={W-R} y1={trendScale(tick)} y2={trendScale(tick)} stroke="#e2e8f0"/>
              <text x={L-9} y={trendScale(tick)+4} text-anchor="end" class="tick">{tick}</text>
            {/each}
            {#each yearScale.ticks(9) as tick}
              <text x={yearScale(tick)} y={lineH-B+22} text-anchor="middle" class="tick">{Math.round(tick)}</text>
            {/each}
            <path d={genreTrendPath} fill="none" stroke="#ea580c" stroke-width="3"/>
            {#each genreTrend as [year,rating]}
              <circle cx={yearScale(year)} cy={trendScale(rating)} r="3" fill="#ea580c"><title>{year}: {rating.toFixed(2)}</title></circle>
            {/each}
            <text x={W/2} y={lineH-7} text-anchor="middle" class="axis">Release year</text>
          </svg>
          <p class="selection">Click another genre to compare; click the selected genre again to reset.</p>
        </div>
      </div>
    </section>
    <footer>CSCI 5609 · A2 · QinYang Tan · <a href={base+'/A1'}>Previous assignment</a></footer>
  {/if}
</main>

<style>
  :global(body){margin:0;background:#f1f5f9;color:#0f172a;font-family:Inter,system-ui,sans-serif}
  main{max-width:1280px;margin:auto;padding:2rem 1.3rem 5rem;display:grid;gap:1.2rem}
  header{background:#0f172a;color:white;padding:2rem;border-radius:18px}
  h1{font-size:clamp(2rem,4vw,3.4rem);margin:.2rem 0}
  h2{font-size:1.45rem;margin:.2rem 0 .7rem}
  h3{font-size:1.1rem;margin:.2rem 0 1rem}
  p{line-height:1.5;color:#64748b} header p{color:#cbd5e1;max-width:850px}
  .eyebrow{text-transform:uppercase;letter-spacing:.12em;font-size:.8rem;font-weight:800;color:#67e8f9}
  .stats{display:flex;gap:1rem;align-items:center;flex-wrap:wrap;margin-top:1.3rem}
  .stats span{background:#1e293b;border-radius:9px;padding:.6rem .8rem}
  button{cursor:pointer}
  .stats button{border:0;background:#38bdf8;color:#082f49;font-weight:700;border-radius:9px;padding:.65rem 1rem}
  .card{background:white;border:1px solid #e2e8f0;border-radius:16px;padding:1.4rem;box-shadow:0 6px 20px #0f172a0c}
  .controls{display:flex;gap:1rem;flex-wrap:wrap;margin:1rem 0}
  .controls label{font-weight:700;font-size:.88rem;display:grid;gap:.35rem}
  select{padding:.55rem;border:1px solid #cbd5e1;border-radius:8px;background:white}
  .two-col{display:grid;grid-template-columns:minmax(0,4fr) minmax(190px,1fr);gap:1.2rem}
  svg{width:100%;height:auto;min-width:0}
  .tick{font-size:13px;fill:#64748b}.axis{font-size:14px;font-weight:700;fill:#334155}
  .details{background:#f8fafc;padding:1rem;border-radius:12px}
  .details h4{font-size:1.25rem;margin:.5rem 0}
  dl{display:grid;grid-template-columns:80px 1fr;gap:.5rem;font-size:.9rem}
  dt{font-weight:700;color:#475569}dd{margin:0;overflow-wrap:anywhere}
  .selection{padding:.7rem 1rem;background:#ecfeff;border-left:4px solid #0891b2;border-radius:7px}
  .two-panels{display:grid;grid-template-columns:1fr 1.2fr;gap:1.4rem}
  .genre-row{width:100%;display:grid;grid-template-columns:85px 1fr 38px 50px;align-items:center;gap:.5rem;border:0;background:transparent;padding:.32rem;border-radius:6px;text-align:left}
  .genre-row:hover,.genre-row.selected{background:#ffedd5}
  .genre-name{font-size:.82rem}.genre-row small{color:#64748b}
  .bar{background:#e2e8f0;height:12px;border-radius:10px;overflow:hidden}.bar span{display:block;background:#ea580c;height:100%}
  .error{background:#fee2e2;color:#991b1b;padding:1rem}
  footer{text-align:center;color:#64748b}footer a{color:#2563eb}
  @media(max-width:850px){.two-col,.two-panels{grid-template-columns:1fr}}
</style>
