<script lang="ts">
  // The web app's quick launch (StandaloneBar's right pill): the shortcuts students actually use, from any tab in two
  // taps. A shelf of glass under the pill; the shelf grows out of it, the shortcuts fan out of it one after another (the
  // nearest first), and each screen opens out of the pill and closes back into it (the iris), since the shelf is gone by
  // then. An overlay like Menu (a tap outside, Escape close it), but the left pill keeps its face: it's no page to go
  // Back from. Calm motion: a plain fade.
  import type { Component } from 'svelte';
  import { onMount, untrack } from 'svelte';
  import MapIcon from '@lucide/svelte/icons/map';
  import Play from '@lucide/svelte/icons/play';
  import Images from '@lucide/svelte/icons/images';
  import Newspaper from '@lucide/svelte/icons/newspaper';
  import { calm } from './fold';
  import { close, removeOverlay, type Overlay } from '../lib/state/ui.svelte';
  import { data } from '../lib/state/data.svelte';
  import { sheets, openMapFrom, irisFrom } from '../lib/state/sheets';
  import { t } from '../lib/i18n/index.svelte';
  import { haptic } from '../lib/tg';

  let { overlay, anchor }: { overlay: Overlay; anchor: HTMLElement } = $props();
  untrack(() => { overlay.holdBack = true; });

  // the iris readers take the pressed control from the click; here it is the pill (the shelf leaves as the screen comes)
  const fromPill = () => ({ currentTarget: anchor }) as unknown as MouseEvent;
  type Item = { icon: Component<any>; label: string; go: () => void };
  const items: Item[] = [
    { icon: MapIcon, label: t('map.open'), go: () => openMapFrom(fromPill()) },
    ...(data.events.some((e) => e.status !== 'cancelled') ? [{ icon: Play, label: t('web.videos'), go: () => { irisFrom(fromPill()); sheets.feed(); } }] : []),
    { icon: Images, label: t('gallery.title'), go: () => { irisFrom(fromPill()); sheets.gallery(); } },
    { icon: Newspaper, label: t('page.news'), go: () => { irisFrom(fromPill()); sheets.news(); } },
  ];

  let shelf: HTMLDivElement, gone = false;

  onMount(() => {
    const a = anchor.getBoundingClientRect(), s = shelf.getBoundingClientRect();
    const ox = a.left + a.width / 2 - s.left, oy = a.top + a.height / 2 - s.top;
    shelf.style.transformOrigin = `${ox}px ${oy}px`;
    shelf.querySelector<HTMLElement>('button')?.focus({ preventScroll: true });
    if (calm()) { shelf.animate({ opacity: [0, 1] }, { duration: 160, easing: 'ease-out' }); return; }
    // the shelf: out of the pill with a soft spring; its glass is in at once, so the tiles ride on it
    shelf.animate({ transform: ['translateY(-6px) scale(0.32, 0.22)', 'none'], borderRadius: ['17px', 'var(--r-xl)'] },
      { duration: 520, easing: 'cubic-bezier(.3, 1.25, .45, 1)' });
    shelf.animate({ opacity: [0, 1] }, { duration: 140, easing: 'ease-out' });
    // the shortcuts: each from the pill's middle to its place, the nearest first, a beat apart
    const tiles = [...shelf.querySelectorAll<HTMLElement>('.tile')];
    tiles.reverse().forEach((el, i) => {
      const r = el.getBoundingClientRect();
      const dx = (a.left + a.width / 2 - (r.left + r.width / 2)) * 0.55, dy = (a.top + a.height / 2 - (r.top + r.height / 2)) * 0.55;
      el.animate({ transform: [`translate(${dx}px, ${dy}px) scale(0.5)`, 'none'], opacity: [0, 1, 1] },
        { duration: 560, delay: 30 + i * 38, easing: 'cubic-bezier(.3, 1.35, .5, 1)', fill: 'backwards' });
    });
  });

  // back into the pill, quick (an exit is shorter than its entrance)
  function leave() {
    if (gone) return;
    gone = true;
    const done = () => removeOverlay(overlay.id);
    if (calm()) { shelf.animate({ opacity: [1, 0] }, { duration: 140, easing: 'ease-in', fill: 'forwards' }).onfinish = done; return; }
    shelf.animate({ transform: ['none', 'translateY(-4px) scale(0.4, 0.3)'], opacity: [1, 0.9, 0] },
      { duration: 210, easing: 'cubic-bezier(.4, 0, .7, .2)', fill: 'forwards' }).onfinish = done;
  }
  $effect(() => { if (overlay.closing) leave(); });

  function pick(it: Item) {
    if (gone) return;
    haptic.select();
    it.go(); // in the tap's frame: the chunks are warmed with the app (App warmPages, warmMap)
    close(overlay.id, false);
  }
</script>

<div class="layer">
  <div class="catch" onclick={() => close(overlay.id, true)} role="presentation"></div>
  <div class="shelf glass glass-sheet" role="menu" tabindex="-1" aria-label={t('web.quick')} bind:this={shelf}>
    {#each items as it (it.label)}
      {@const Icon = it.icon}
      <button class="tile" type="button" role="menuitem" onclick={() => pick(it)}>
        <span class="ic"><Icon size={22} strokeWidth={2} /></span>
        <span class="lb">{it.label}</span>
      </button>
    {/each}
  </div>
</div>

<style>
  .layer { position: absolute; inset: 0; z-index: calc(var(--z-sheet) + 6); } /* under the bar (its pill closes it again) */
  .catch { position: absolute; inset: 0; }
  /* 8 px under the bar's line (StandaloneBar), its right edge on the pill's */
  .shelf { position: absolute; right: 12px; top: calc(max(8px, calc(var(--safe-top) + (var(--tg-head) - 38px) / 2)) + 46px);
    display: flex; padding: 10px 8px 9px; border-radius: var(--r-xl); background-color: var(--menu-bg); max-width: calc(100% - 24px); }
  .shelf:focus { outline: none; }
  .tile { display: flex; flex-direction: column; align-items: center; gap: 6px; width: 70px; padding: 2px 0; border-radius: var(--r-m);
    color: var(--text); -webkit-tap-highlight-color: transparent; }
  .ic { display: flex; align-items: center; justify-content: center; width: 50px; height: 50px; border-radius: var(--r-full);
    background: var(--fill); transition: transform 160ms var(--ease), background-color 160ms var(--ease); }
  .tile:active .ic { transform: scale(.9); background: var(--fill-2); }
  .tile:focus-visible { outline: none; }
  .tile:focus-visible .ic { box-shadow: 0 0 0 2px var(--accent); }
  @media (hover: hover) { .tile:hover .ic { background: var(--fill-2); } }
  .lb { max-width: 100%; overflow: hidden; text-overflow: ellipsis; white-space: nowrap; font: 500 var(--fs-cap)/1.2 var(--font-text);
    letter-spacing: -0.005em; color: var(--text-2); }
</style>
