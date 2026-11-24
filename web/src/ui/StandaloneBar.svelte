<script lang="ts">
  import ChevronLeft from '@lucide/svelte/icons/chevron-left';
  import LayoutGrid from '@lucide/svelte/icons/layout-grid';
  import Search from '@lucide/svelte/icons/search';
  import X from '@lucide/svelte/icons/x';
  import { t } from '../lib/i18n/index.svelte';
  import { flushSync, tick } from 'svelte';
  import { close, goTab, open, overlays, ui } from '../lib/state/ui.svelte';
  import { backButton } from '../lib/tg';
  import { glass } from './glass';
  import QuickLaunch from './QuickLaunch.svelte';
  import { tabSearches, type TabSearch } from './PullSearch.svelte';

  let { back }: { back: boolean } = $props();
  let quickId = $state(0);
  const expanded = $derived(overlays.some((o) => o.id === quickId && !o.closing));

  // The one search of the web app. No Back: the left pill is Search, and a tap stretches it into a field across the whole
  // bar (the capsule and the menu step aside); it types into the tab's own search (each tab's PullSearch lends its
  // value). Profile has none: the Afisha's. An overlay over the results brings the Back face; the field is back after it.
  let field = $state(false), active = $state.raw<TabSearch | null>(null), tabAt = $state(ui.tab);
  let input = $state<HTMLInputElement>();
  const shown = $derived(field && !back);
  const mine = () => [...tabSearches].find((s) => {
    const tab = s.node?.closest<HTMLElement>('.tab');
    return !tab || tab.classList.contains('on'); // the shown tab (App.svelte), not one away or being drawn unseen
  }) ?? null;

  async function search() {
    if (expanded) close(quickId, true);
    if (ui.tab === 'profile') goTab('afisha');
    else if (!mine()) return;
    tabAt = ui.tab;
    flushSync(() => { field = true; });
    input?.focus({ preventScroll: true }); // inside the tap, or iOS shows no keyboard
    await tick();
    active = mine();
    if (!active) { field = false; return; }
    active.node?.closest('.scroll')?.scrollTo({ top: 0, behavior: 'smooth' });
  }

  function shut() {
    if (active) active.value = '';
    active = null;
    field = false;
    input?.blur();
  }

  function keys(e: KeyboardEvent) {
    if (e.key === 'Escape') shut();
    else if (e.key === 'Enter') input?.blur();
  }

  $effect(() => { if (field && ui.tab !== tabAt) shut(); }); // another tab: its own search, not this one's words
  $effect(() => {
    document.documentElement.toggleAttribute('data-search', shown); // the capsule steps aside (ui/Capsule.svelte)
    return () => document.documentElement.removeAttribute('data-search');
  });

  // the quick launch (owner, Nov 24): the shortcuts unfold under the pill (ui/QuickLaunch); the rare items live in Settings
  function quick(e: MouseEvent) {
    if (expanded) { close(quickId, true); return; }
    quickId = open(QuickLaunch, { anchor: e.currentTarget as HTMLElement });
  }
</script>

<header class="bar">
  <!-- always there (owner, Nov 22): the capsule's glass, a step smaller than it. Its face changes, Search <-> Back: the
       old one leaves a few px towards where it came from, the new one settles in from the other side, the word a beat
       behind its icon. Or it becomes the search field (above). -->
  <div class="glass glass-pill side left" class:back class:field={shown} use:glass={{ bloom: true }}>
    <span class="tone-base" aria-hidden="true"></span>
    <button class="face" type="button" inert={shown} aria-label={back ? t('web.back') : t('web.search')} onclick={() => (back ? backButton.click() : void search())}>
      <span class="ico" aria-hidden="true"><i class="a"><Search size={15} strokeWidth={2.4} /></i><i class="b"><ChevronLeft size={18} strokeWidth={2.4} /></i></span>
      <span class="txt" aria-hidden="true"><i class="a">{t('web.search')}</i><i class="b">{t('web.back')}</i></span>
    </button>
    <label class="fld" inert={!shown}>
      <Search size={15} strokeWidth={2.4} />
      <input bind:this={input} type="search" enterkeyhint="search" autocomplete="off" placeholder={active?.placeholder ?? t('web.search')}
        aria-label={active?.placeholder ?? t('web.search')} value={active?.value ?? ''} oninput={(e) => { if (active) active.value = e.currentTarget.value; }} onkeydown={keys} />
      <button class="x" type="button" aria-label={t('common.close')} onclick={shut}><X size={15} strokeWidth={2.4} /></button>
    </label>
  </div>
  <button class="glass glass-pill side more" class:open={expanded} type="button" inert={shown} aria-label={t('web.quick')} aria-haspopup="menu" aria-expanded={expanded} onclick={quick} use:glass={{ bloom: true }}>
    <span class="tone-base" aria-hidden="true"></span>
    <i class="g"><LayoutGrid size={17} strokeWidth={2.2} /></i><i class="g x"><X size={18} strokeWidth={2.4} /></i>
  </button>
</header>

<style>
  /* the capsule's own line (its top and 38 px, ui/Capsule.svelte); the pills 34 px on its middle, one width (--pill,
     tokens.css), the capsule the largest thing in the bar, 10 px clear of each */
  .bar { position: absolute; inset: max(8px, calc(var(--safe-top) + (var(--tg-head) - 38px) / 2)) 0 auto; height: 38px;
    z-index: calc(var(--z-sheet) + 7); padding-inline: 12px; display: flex; align-items: center; pointer-events: none; }
  .side { position: relative; flex: none; width: var(--pill); height: 34px; border-radius: var(--r-full); background: none; color: var(--text);
    font: 600 var(--fs-sub)/1 var(--font-text); letter-spacing: -0.005em; pointer-events: auto;
    transition: transform 200ms var(--ease), opacity 180ms var(--ease), width 460ms var(--ease-spring); }
  .side > .tone-base { background: var(--glass-clear); }
  :global(html[data-theme='light'][data-sheet]) .side > .tone-base { background: var(--glass-solid); }
  .side:active { transform: scale(.94); }
  .side :global(svg) { flex: none; }
  .more { position: absolute; right: 12px; top: 2px; display: flex; align-items: center; justify-content: center; }
  .more.away { opacity: 0; transform: translateX(12px) scale(.9); }
  /* the grid turns into the x that folds the shortcuts away, and back */
  .g { position: absolute; display: flex; transition: opacity 180ms var(--ease), transform 420ms var(--ease-spring); }
  .g.x, .open .g:not(.x) { opacity: 0; transform: rotate(-90deg) scale(.6); }
  .open .g.x { opacity: 1; transform: none; }

  .face { display: flex; align-items: center; justify-content: center; gap: 3px; width: var(--pill); height: 100%; padding-inline: 6px;
    color: inherit; font: inherit; letter-spacing: inherit; transition: opacity 160ms var(--ease); }
  .ico, .txt { display: grid; place-items: center; }
  .txt { overflow: hidden; min-width: 0; }
  /* a face coming in: after the old one has begun to go, the word a beat behind its icon, a soft settle */
  .ico > i, .txt > i { grid-area: 1 / 1; font-style: normal; white-space: nowrap;
    transition: opacity 240ms var(--ease) 70ms, transform 480ms var(--ease-spring) 40ms; }
  .ico > i { display: flex; }
  .txt > i { transition-delay: 110ms, 80ms; }
  /* a face not shown waits a few px to its side (Search to the left of Back), the icon a touch small; going there it is quick */
  .ico > .b, .back .ico > .a, .txt > .b, .back .txt > .a { opacity: 0; transition: opacity 130ms var(--ease-in), transform 220ms var(--ease-in); }
  .ico > .b { transform: translateX(5px) scale(.84); }
  .txt > .b { transform: translateX(9px); }
  .back .ico > .a { transform: translateX(-5px) scale(.84); }
  .back .txt > .a { transform: translateX(-9px); }
  .back .ico > .b, .back .txt > .b { opacity: 1; transform: none; transition: opacity 240ms var(--ease) 70ms, transform 480ms var(--ease-spring) 40ms; }
  .back .txt > .b { transition-delay: 110ms, 80ms; }

  /* the search field: the pill itself, stretched across the bar; its face fades where it stands, the field comes up in it */
  .left.field { width: 100%; }
  .left.field:active { transform: none; }
  .field .face { opacity: 0; }
  .fld { position: absolute; inset: 0; display: flex; align-items: center; gap: 6px; padding: 0 2px 0 11px; opacity: 0;
    transition: opacity 140ms var(--ease); }
  .field .fld { opacity: 1; transition: opacity 260ms var(--ease) 120ms; }
  .fld input { flex: 1; min-width: 0; height: 100%; font: 450 var(--fs-row)/1.2 var(--font-text); letter-spacing: -0.012em; color: var(--text); }
  .fld input::placeholder { color: var(--text-3); }
  .fld input::-webkit-search-cancel-button { display: none; }
  .x { position: relative; display: flex; align-items: center; justify-content: center; width: 30px; height: 30px; flex: none; border-radius: var(--r-full); color: var(--text-2); }
  .x::after { content: ''; position: absolute; inset: -7px; } /* a 44 px target */
  @media (prefers-reduced-motion: reduce) {
    .side, .ico > i, .txt > i, .fld { transition: opacity 120ms linear !important; }
    .ico > i, .txt > i, .more.away, .g { transform: none !important; }
    .g { transition: opacity 120ms linear !important; }
  }
</style>
