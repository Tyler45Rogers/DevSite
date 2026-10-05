<script>
  import RotateCcw from '@lucide/svelte/icons/rotate-ccw';
  // Svelte 5 (runes mode) + Tailwind + daisyUI (v4 or v5). No other dependencies.
  const NAMES = ['C', 'C♯', 'D', 'D♯', 'E', 'F', 'F♯', 'G', 'G♯', 'A', 'A♯', 'B'];
  // MIDI numbers, low string -> high string: E2 A2 D3 G3 B3 E4
  const STANDARD = [40, 45, 50, 55, 59, 64];
  const PRESETS = [
    { name: 'Drop D', t: [38, 45, 50, 55, 59, 64] },
    { name: 'Double Drop D', t: [38, 45, 50, 55, 59, 62] },
    { name: 'Half step down', t: [39, 44, 49, 54, 58, 63] },
    { name: 'Full step down', t: [38, 43, 48, 53, 57, 62] },
    { name: 'Drop C', t: [36, 43, 48, 53, 57, 62] },
    { name: 'DADGAD', t: [38, 45, 50, 55, 57, 62] },
    { name: 'Open G', t: [38, 43, 50, 55, 59, 62] },
    { name: 'Open D', t: [38, 45, 50, 54, 57, 62] },
    { name: 'Open E', t: [40, 47, 52, 56, 59, 64] }
  ];
  const KNOWN = [{ name: 'Standard', t: STANDARD }, ...PRESETS];

  const FRETS = 15; // frets drawn past the nut
  const BACK_MAX = 3; // max frets drawn *before* the nut (for strings tuned above standard)
  const INLAYS = [3, 5, 7, 9, 15];

  // SVG geometry. The board plus the open-note column sits dead centre; when frets are
  // added before the nut, the whole stage slides right by half their width to stay centred.
  const TOP = 44, GAP = 46, H = 322, SLOT = 56, R = 16, BOARD = 900;
  const M = 210; // blank margin on each side
  const NUT = M + SLOT - 2; // open-note column sits just left of the nut
  const W = 2 * M + SLOT - 2 + BOARD;
  const SCALE = BOARD / (1 - Math.pow(2, -FRETS / 12));
  const fretX = (n) => NUT + SCALE * (1 - Math.pow(2, -n / 12));
  // f > 0: middle of the fret; f = 0: open-string note just left of the nut;
  // f < 0: fixed-width slots further left (frets "before" the nut)
  const noteX = (f) => (f > 0 ? (fretX(f - 1) + fretX(f)) / 2 : NUT - 26 + SLOT * f);

  const pc = (m) => ((m % 12) + 12) % 12;
  const noteName = (m) => NAMES[pc(m)];
  const octave = (m) => Math.floor(m / 12) - 1;
  const stdPCs = new Set(STANDARD.map(pc));

  let tuning = $state([...STANDARD]);
  let showShared = $state(true);
  let active = $state(null); // string index hovered/focused in the tuner

  // Picking a note name keeps the pitch closest to what the string was before,
  // so E → D means "down 2", not "down 10". Use the octave menu to override.
  function setNote(i, name) {
    const target = NAMES.indexOf(name);
    const old = tuning[i];
    let best = old;
    let bestDist = Infinity;
    for (let m = old - 6; m <= old + 6; m++) {
      if (pc(m) === target && Math.abs(m - old) < bestDist) {
        best = m;
        bestDist = Math.abs(m - old);
      }
    }
    tuning = tuning.map((m, k) => (k === i ? best : m));
  }
  function setOctave(i, oct) {
    const m = (oct + 1) * 12 + pc(tuning[i]);
    tuning = tuning.map((x, k) => (k === i ? m : x));
  }

  // Hovering or focusing a string's column spotlights it on the fretboard
  const hover = (i) => ({
    onmouseenter: () => (active = i),
    onmouseleave: () => (active = null),
    onfocusin: () => (active = i),
    onfocusout: () => (active = null)
  });

  // stdFret = the fret on this string that sounds the standard-tuned open note
  // (negative when the string is tuned above standard, i.e. before the nut)
  const rows = $derived(
    tuning.map((m, i) => {
      const delta = m - STANDARD[i];
      const stdFret = -delta;
      let offNote = '';
      if (stdFret > FRETS) offNote = `fret ${stdFret}`;
      else if (delta > BACK_MAX) offNote = `${delta} below nut`;
      return { i, m, std: STANDARD[i], delta, stdFret, offNote, y: TOP + (5 - i) * GAP };
    })
  );
  // How many frets before the nut to draw: enough for the farthest string that's within the limit
  const back = $derived(Math.max(0, ...rows.map((r) => (r.delta >= 1 && r.delta <= BACK_MAX ? r.delta : 0))));
  const frets = $derived(Array.from({ length: back + FRETS + 1 }, (_, k) => k - back));
  const ghostLeft = $derived(noteX(-back) - SLOT / 2);
  const ghostRight = noteX(0) - SLOT / 2;
  const shift = $derived((back * SLOT) / 2);

  const isStandard = $derived(tuning.every((m, i) => m === STANDARD[i]));
  const matched = $derived(KNOWN.find((p) => p.t.every((m, i) => m === tuning[i])));
</script>

<section class="tc w-full px-4 py-6 flex flex-col gap-6">
  <header class="flex flex-col items-center gap-3 text-center">
    <div class="flex items-center gap-3">
      <h2 class="text-2xl font-semibold">Tuning vs. standard</h2>
      <span class={matched ? 'badge badge-primary' : 'badge badge-outline'}>
        {matched ? matched.name : 'Custom tuning'}
      </span>
    </div>
    <div class="flex flex-wrap justify-center items-center gap-3">
      {#each PRESETS as p}
        <button
          type="button"
          class={`btn border-2 border-gray-300 font-bold text-2xl rounded-full ${matched === p ? 'btn-primary' : 'btn-outline btn-primary'}`}
          onclick={() => (tuning = [...p.t])}
        >
          {p.name}
        </button>
      {/each}
    </div>
  </header>

  <div class="boardwrap">
    <svg
      viewBox={`0 0 ${W} ${H}`}
      role="img"
      aria-label="Fretboard showing the notes of your tuning, with the standard-tuning pitch marked on each string"
    >
      <defs>
        <linearGradient id="tc-wood" x1="0" y1="0" x2="0" y2="1">
          <stop offset="0" style="stop-color: var(--wood-a)" />
          <stop offset="1" style="stop-color: var(--wood-b)" />
        </linearGradient>
      </defs>

      <g class="stage" style={`transform: translateX(${shift}px)`}>
      <!-- frets before the nut: a dashed, tinted strip -->
      {#if back > 0}
        <rect x={ghostLeft} y={TOP - 22} width={ghostRight - ghostLeft} height={5 * GAP + 44} rx="5" class="ghost" />
        {#each Array(back) as _, k}
          <line x1={noteX(-(k + 1)) + SLOT / 2} x2={noteX(-(k + 1)) + SLOT / 2} y1={TOP - 22} y2={TOP + 5 * GAP + 22} class="ghostline" />
        {/each}
        <text x={ghostRight} y={TOP - 30} text-anchor="end" class="ghosttag">before the nut</text>
      {/if}

      <rect x={NUT} y={TOP - 22} width={BOARD} height={5 * GAP + 44} rx="5" fill="url(#tc-wood)" class="board" />

      {#each INLAYS as n}
        <circle cx={noteX(n)} cy={TOP + 2.5 * GAP} r="7" class="inlay" />
      {/each}
      <circle cx={noteX(12)} cy={TOP + 1.5 * GAP} r="7" class="inlay" />
      <circle cx={noteX(12)} cy={TOP + 3.5 * GAP} r="7" class="inlay" />

      {#each [...INLAYS, 12].sort((a, b) => a - b) as n}
        <text x={noteX(n)} y={H - 12} class="fretnum" text-anchor="middle">{n}</text>
      {/each}
      {#each Array(back) as _, k}
        <text x={noteX(-(k + 1))} y={H - 12} class="fretnum" text-anchor="middle">−{k + 1}</text>
      {/each}

      {#each Array(FRETS) as _, k}
        <line x1={fretX(k + 1)} x2={fretX(k + 1)} y1={TOP - 22} y2={TOP + 5 * GAP + 22} class="fret" />
      {/each}
      <line x1={NUT} x2={NUT} y1={TOP - 22} y2={TOP + 5 * GAP + 22} class="nut" />

      {#each rows as r (r.i)}
        <line
          x1={NUT}
          x2={NUT + BOARD}
          y1={r.y}
          y2={r.y}
          class="string"
          class:wound={r.i < 3}
          class:dim={active !== null && active !== r.i}
          class:hot={active === r.i}
          style={`stroke-width:${3.6 - r.i * 0.48}`}
        />
        <text x={ghostLeft - 18} y={r.y + 5} text-anchor="middle" class="snum" class:hot={active === r.i}>{6 - r.i}</text>
      {/each}

      {#each rows as r (r.i)}
        <g class="notes" class:dim={active !== null && active !== r.i}>
          {#each frets as f}
            {@const p = r.m + f}
            {@const isTarget = f === r.stdFret}
            {@const isShared = showShared && !isTarget && stdPCs.has(pc(p))}
            <circle
              cx={noteX(f)}
              cy={r.y}
              r={R}
              class="dot"
              class:free={f <= 0}
              class:target={isTarget}
              class:shared={isShared}
            />
            <text
              x={noteX(f)}
              y={r.y + 5}
              text-anchor="middle"
              class="note"
              class:free={f <= 0}
              class:strong={isShared}
              class:ontarget={isTarget}
            >
              {noteName(p)}
            </text>
          {/each}
        </g>
        {#if r.offNote}
          <text x={NUT + BOARD + 18} y={r.y + 5} class="offtext">
            <title>{`String ${6 - r.i}: standard note is ${r.offNote}`}</title>
            {r.offNote}
          </text>
        {/if}
      {/each}
      </g>
    </svg>
  </div>

  <div class="flex flex-wrap justify-center gap-x-6 gap-y-2 text-sm opacity-80">
    <span class="flex items-center gap-2"><i class="sw target"></i> Fret that sounds the string's standard note</span>
    <label class="flex items-center gap-2 cursor-pointer">
      <input type="checkbox" class="checkbox checkbox-xs checkbox-primary" bind:checked={showShared} />
      <i class="sw shared"></i> Other notes from standard tuning (E A D G B)
    </label>
  </div>

  <div class="tuner mx-auto w-full max-w-3xl" role="group" aria-label="Tuning, low string to high string">
    <div class="row head">
      <div class="rl">String</div>
      {#each rows as r (r.i)}
        <div class="cell" {...hover(r.i)}><span class="chip" class:hot={active === r.i}>{6 - r.i}</span></div>
      {/each}
    </div>

    <div class="row">
      <div class="rl">Standard</div>
      {#each rows as r (r.i)}
        <div class="cell std" {...hover(r.i)}>{noteName(r.std)}<sub>{octave(r.std)}</sub></div>
      {/each}
    </div>

    <div class="row">
      <div class="rl">Frets to tune</div>
      {#each rows as r (r.i)}
        <div class="cell" {...hover(r.i)}>
          <span
            class={`pill badge badge-outline ${r.delta > 0 ? 'badge-success' : r.delta < 0 ? 'badge-error' : 'badge-ghost'}`}
          >
            {#if r.delta === 0}
              0
            {:else}
              <span aria-hidden="true">{r.delta > 0 ? '▲' : '▼'}</span>
              {Math.abs(r.delta)}
              <span class="sr-only">{r.delta > 0 ? 'tune up' : 'tune down'}</span>
            {/if}
          </span>
        </div>
      {/each}
    </div>

    <div class="row">
      <div class="rl">Your tuning</div>
      {#each rows as r (r.i)}
        <div class="cell" {...hover(r.i)}>
          <div class="pair">
            <select
              class="select select-sm"
              aria-label={`String ${6 - r.i} note`}
              value={noteName(r.m)}
              onchange={(e) => setNote(r.i, e.currentTarget.value)}
            >
              {#each NAMES as n}<option value={n}>{n}</option>{/each}
            </select>
            <select
              class="select select-sm"
              aria-label={`String ${6 - r.i} octave`}
              value={octave(r.m)}
              onchange={(e) => setOctave(r.i, +e.currentTarget.value)}
            >
              {#each [0, 1, 2, 3, 4, 5] as o}<option value={o}>{o}</option>{/each}
            </select>
            <button
              type="button"
              class="btn btn-primary btn-xs border-2 border-gray-300 font-bold rounded-full"
              aria-label={`Reset string ${6 - r.i} to standard`}
              title="Reset this string"
              disabled={r.delta === 0}
              onclick={() => (tuning = tuning.map((m, k) => (k === r.i ? STANDARD[k] : m)))}
            >
              <RotateCcw size={14} />
            </button>
          </div>
        </div>
      {/each}
    </div>
  </div>

  <div class="flex justify-center">
    <button
      type="button"
      class="btn btn-primary border-2 border-gray-300 font-bold text-2xl rounded-full"
      disabled={isStandard}
      onclick={() => (tuning = [...STANDARD])}
    >
      <RotateCcw size={22} /> Reset to standard
    </button>
  </div>
</section>

<style>
  .tc {
    /* Pull colors from the active daisyUI theme (v5 names first, v4 as fallback) */
    --c-content: var(--color-base-content, oklch(var(--bc)));
    --c-primary: var(--color-primary, oklch(var(--p)));
    --c-primary-content: var(--color-primary-content, oklch(var(--pc)));
    --c-line: var(--color-base-300, oklch(var(--b3)));
    /* Fretboard palette, light themes (default): honey maple with dark ink */
    --wood-a: #dcaa6c;
    --wood-b: #c28f50;
    --nut: #f6ecd4;
    --fret: #8d8576;
    --inlay: #5a3a1c;
    --inlay-op: 0.22;
    --str: #6f6a60;
    --str-wound: #7b4f1d;
    --str-op: 0.8;
    --shared-stroke: #5a3a1c;
    --shared-fill: rgba(90, 58, 28, 0.12);
    --note: #4a2f17;
    --note-op: 0.85;
    --note-strong: #2a1608;
  }
  /* Dark theme: deeper walnut with light ink */
  :global([data-theme='dark']) .tc {
    --wood-a: #805838;
    --wood-b: #5d3c27;
    --nut: #eadfc8;
    --fret: #dcd6ca;
    --inlay: #f1e5cc;
    --inlay-op: 0.3;
    --str: #ece8df;
    --str-wound: #e2b66e;
    --str-op: 0.5;
    --shared-stroke: #f0dfb8;
    --shared-fill: rgba(240, 223, 184, 0.16);
    --note: #ead9bf;
    --note-op: 0.8;
    --note-strong: #f3e9d8;
  }

  /* Fretboard */
  .boardwrap {
    overflow-x: auto;
  }
  svg {
    display: block;
    width: 100%;
    min-width: 900px;
    height: auto;
  }
  .stage {
    transition: transform 0.35s ease;
  }
  .ghost {
    fill: color-mix(in oklab, var(--c-content) 7%, transparent);
    stroke: color-mix(in oklab, var(--c-content) 35%, transparent);
    stroke-dasharray: 5 4;
  }
  .ghostline {
    stroke: color-mix(in oklab, var(--c-content) 35%, transparent);
    stroke-dasharray: 5 4;
  }
  .ghosttag {
    font-size: 14px;
    fill: var(--c-content);
    opacity: 0.65;
  }
  .board {
    stroke: rgba(0, 0, 0, 0.3);
    stroke-width: 1.5;
    filter: drop-shadow(0 6px 12px rgba(0, 0, 0, 0.28));
  }
  .inlay {
    fill: var(--inlay);
    opacity: var(--inlay-op);
  }
  .fret {
    stroke: var(--fret);
    stroke-width: 2;
    opacity: 0.85;
  }
  .nut {
    stroke: var(--nut);
    stroke-width: 8;
    filter: drop-shadow(0 0 1.5px rgba(0, 0, 0, 0.55));
  }
  .string {
    stroke: var(--str);
    opacity: var(--str-op);
    transition: opacity 0.2s;
  }
  .string.wound {
    stroke: var(--str-wound);
  }
  .string.hot {
    opacity: 1;
  }
  .string.dim {
    opacity: 0.12;
  }
  .notes {
    transition: opacity 0.2s;
  }
  .notes.dim {
    opacity: 0.25;
  }
  .dot {
    fill: transparent;
    stroke: transparent;
    stroke-width: 1.5;
    transition: fill 0.3s, stroke 0.3s;
  }
  .dot.shared {
    stroke: var(--shared-stroke);
    fill: var(--shared-fill);
  }
  .dot.free.shared {
    stroke: color-mix(in oklab, var(--c-content) 50%, transparent);
    fill: color-mix(in oklab, var(--c-content) 8%, transparent);
  }
  .dot.target {
    fill: var(--c-primary);
    stroke: var(--c-primary);
    filter: drop-shadow(0 0 6px color-mix(in oklab, var(--c-primary) 60%, transparent));
  }
  /* Labels sit on the wood, except open-string and before-the-nut labels, which sit on the page */
  .note {
    font-size: 15px;
    fill: var(--note);
    opacity: var(--note-op);
    transition: fill 0.3s, opacity 0.3s;
    pointer-events: none;
  }
  .note.strong {
    opacity: 1;
    fill: var(--note-strong);
  }
  .note.free {
    opacity: 1;
    fill: var(--c-content);
  }
  .note.ontarget {
    opacity: 1;
    fill: var(--c-primary-content);
    font-weight: 700;
  }
  .fretnum,
  .snum {
    font-size: 15px;
    fill: var(--c-content);
    opacity: 0.6;
  }
  .snum.hot {
    fill: var(--c-primary);
    opacity: 1;
    font-weight: 700;
  }
  .offtext {
    font-size: 15px;
    font-weight: 600;
    fill: var(--c-primary);
  }

  .sw {
    display: inline-block;
    width: 0.9rem;
    height: 0.9rem;
    border-radius: 50%;
  }
  .sw.target {
    background: var(--c-primary);
  }
  .sw.shared {
    border: 1.5px solid var(--shared-stroke);
    background: var(--wood-a);
  }

  /* Tuner: horizontal rows, one column per string (low → high), centered */
  .row {
    display: grid;
    grid-template-columns: 7.5rem repeat(6, minmax(0, 1fr));
    align-items: center;
    padding: 0.55rem 0;
    border-bottom: 1px solid var(--c-line);
  }
  .row:last-child {
    border-bottom: 0;
  }
  .row.head {
    padding: 0.4rem 0;
  }
  .rl {
    font-size: 0.85rem;
    opacity: 0.7;
  }
  .cell {
    display: flex;
    justify-content: center;
    align-items: center;
    padding: 0 0.2rem;
  }
  .chip {
    width: 1.5rem;
    height: 1.5rem;
    border-radius: 50%;
    display: grid;
    place-items: center;
    font-size: 0.8rem;
    border: 1px solid var(--c-line);
    transition: background 0.2s, color 0.2s;
  }
  .chip.hot {
    background: var(--c-primary);
    border-color: var(--c-primary);
    color: var(--c-primary-content);
    font-weight: 700;
  }
  .std {
    font-size: 1.3rem;
    font-variant-numeric: tabular-nums;
  }
  .std sub {
    font-size: 0.6em;
    opacity: 0.65;
  }
  .pill {
    min-width: 3.4rem;
    font-weight: 600;
    font-variant-numeric: tabular-nums;
    white-space: nowrap;
  }
  .pair {
    display: flex;
    flex-direction: column;
    align-items: center;
    gap: 0.3rem;
    width: 100%;
  }
  .pair :global(select) {
    width: 100%;
    max-width: 5rem;
  }

  @media (max-width: 520px) {
    .row {
      grid-template-columns: 4.5rem repeat(6, minmax(0, 1fr));
    }
    .rl {
      font-size: 0.75rem;
    }
    .pill {
      min-width: 0;
      padding-inline: 0.35rem;
    }
    .pair :global(select) {
      padding-inline: 0.35rem;
      background-image: none;
    }
  }
  @media (prefers-reduced-motion: reduce) {
    .dot,
    .note,
    .string,
    .notes,
    .stage,
    .chip {
      transition: none;
    }
  }
</style>