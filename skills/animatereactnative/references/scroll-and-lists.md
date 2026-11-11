# Scroll-Driven Animation

Scroll-driven animation has one law: **the scroll offset is the only driver, and every effect is a pure interpolation of it.** No timings fired from `onScroll`, no viewability callbacks toggling state, no springs inside scroll-linked styles. Done this way, the animation scrubs perfectly in both directions at any speed, because it is not "triggered", it is *derived*.

The second law is restraint. The scroll animations that read as expensive use two or three channels per item with small ranges: off-focus scale 0.8 to 0.92, off-focus opacity 0.3 to 0.5, a 15 to 40px arc, ±15° of rotation, ±20px of text drift. Six channels per card plus springs per frame (the catalog's random-people carousel) is the heaviest cell in the catalog, and it doesn't look six times better.

## Wiring the driver: normalize to index space, once

Divide the pixel offset by the full item size in the scroll handler, and every consumer gets clean, size-independent ranges:

```tsx
const { width } = useWindowDimensions(); // or onLayout; never Dimensions.get('screen') (Android counts system bars)
const ITEM_FULL = CARD_W + GAP;          // the EXACT layout footprint of one item

const scrollX = useSharedValue(0);
const onScroll = useAnimatedScrollHandler((e) => {
  scrollX.value = e.contentOffset.x / ITEM_FULL; // index space: 0, 1, 2, ... computed ONCE
});
<Animated.FlatList horizontal onScroll={onScroll} scrollEventThrottle={16}
  snapToInterval={ITEM_FULL} decelerationRate="fast" ... />
```

Now "item `i` is centered" is simply `scrollX.value === i`, and every per-item window is `[i - 1, i, i + 1]`. Changing the card size touches one constant instead of every input range.

- **Divide once.** Don't hand items the pixel offset and let each run `useDerivedValue(() => offset.value / ITEM_FULL)`: that is N identical divisions per frame and N places for the constant to drift.
- `ITEM_FULL` must match the real layout, margins and `contentContainerStyle` gap included, or the ranges drift out of phase and centered items sit slightly off-scale: the classic "almost right" carousel bug. Decks whose cells overlap normalize by the **snap interval**, not the card size (below).
- Centering: `paddingHorizontal: (width - ITEM_FULL) / 2` when each item carries `GAP / 2` margins, `(width - CARD_W) / 2` when the gap comes from `contentContainerStyle`. Vertical: `paddingVertical: (height - ITEM) / 2`.
- `scrollEventThrottle={16}`. `Animated.FlatList` and `Animated.ScrollView` default to 1 when you omit it; a hand-set 32 or 50 updates every second or third frame and steps visibly.
- Inside the handler, assign. `withTiming(v, { duration: 0 })`, with or without a `withDelay` around it, is a slower assignment that lags the finger.
- With a ref and no handler: `useScrollOffset(animatedRef)` (Reanimated 4; `useScrollViewOffset` on v3, now a deprecated alias) gives pixels. Normalize it in one list-level `useDerivedValue`.

## The per-item window

The signature pattern behind almost every scroll animation. Each item cares about a three-point window around its own index:

```tsx
// inside renderItem's component, index i
const stylez = useAnimatedStyle(() => ({
  transform: [
    { scale: interpolate(scrollX.value, [i - 1, i, i + 1], [0.82, 1, 0.82], Extrapolation.CLAMP) },
  ],
  opacity: interpolate(scrollX.value, [i - 1, i, i + 1], [0.5, 1, 0.5], Extrapolation.CLAMP),
}));
```

Read it as: "when I am centered, I am full size; one position away in either direction, I am small and dim." The shape of the output triple tells you what a channel means:

- **Symmetric `[a, 0, a]`**: distance from focus. Arc drop `[40, 0, 40]`, dimming `[0.3, 1, 0.3]`.
- **Antisymmetric `[a, 0, -a]`**: direction of travel. Tilt `['15deg', '0deg', '-15deg']`, text drift `[20, 0, -20]`.
- **Asymmetric `[a, 1, b]`**: arriving differs from leaving. Scale `[0.2, 1, 0]` grows in from small and shrinks away to nothing.

**Every new effect is one more channel on the same window**, never a new mechanism (this is how the carousel recipe series is built):

- `translateY: [26, 0, 26]` sinks neighbors into an **arc**, so the row reads as a curve instead of a strip. Depth 15 to 40px (the circular slider uses `size / 3`, about 31px, plus a 4px focus ring); deeper clips without a taller viewport.
- `rotateY: ['35deg', '0deg', '-35deg']` with `{ perspective: 1000 }` **first** in the transform array turns neighbors toward the focused card: scale + drop + turn is the whole cover-flow effect. Rotation narrows a card by `(w / 2) * (1 - cos θ)` per side (9% of its width at 35°, 15% at 45°), so pull neighbors in by that much on `translateX` or the row looks gappy. A perspective equal to the card width is very strong; 1000 is a calm default.
- `backgroundColor` via `interpolateColor` over the same window highlights the focused card.
- **The window as a gate**: multiply another input by the window's peak and only the focused card responds. `const focus = interpolate(scrollX.value, [i - 1, i, i + 1], [0, 1, 0], Extrapolation.CLAMP)`, then ``rotateY: `${focus * tiltDeg.value}deg` `` lets a device-tilt sensor turn the centered card (cap near 45°) while neighbors stay still at scale 0.9.

Catalog: https://www.animatereactnative.com/post/wallpaper-animated-carousel-react-native-reanimated, https://www.animatereactnative.com/post/parallax-carousel-reanimated-and-sensors

## Asymmetric windows, and the two range traps

Past and future don't have to mirror. The Star Wars casting deck flies the next card in from in front of the camera and lets passed cards recede in even steps. Its cells overlap by half, so the driver counts snap steps, not cards:

```tsx
scrollIdx.value = e.contentOffset.y / (ITEM / 2); // snapToInterval={ITEM / 2}, decelerationRate="fast"

// card i
scale: interpolate(scrollIdx.value, [i - 1, i, i + 1, i + 2, i + 4], [3, 1, 0.8, 0.6, 0.2], Extrapolation.CLAMP),
opacity: interpolate(scrollIdx.value, [i - 1, i, i + 1, i + 2], [0, 1, 0.85, 0.7], Extrapolation.CLAMP),
```

Two traps hide in ranges like these:

1. **Unclamped `EXTEND` that crosses zero.** Without the `i + 4` floor point and `CLAMP`, scale keeps dropping 0.2 per step, passes 0 after about five steps, and the card renders mirrored; opacity goes negative the same way. Any channel heading toward 0 gets a floor point plus `CLAMP`. Keep `EXTEND` only on purpose (pinning and overscroll stretch, below).
2. **Input ranges must ascend.** `interpolate` binary-searches the input range assuming it is sorted. Two descending points happen to work; with three or more it silently picks the wrong segment. Flip the outputs instead: `[1, 0.5] → [a, b]` becomes `[0.5, 1] → [b, a]`.

Catalog: https://www.animatereactnative.com/post/custom-vertical-scroll-list-starwars-like-casting-reanimated2-+-gestures

## Pin, then release (and linger)

A row scrolls normally until it reaches the top, then sticks while it shrinks and fades, and the next rows slide over it. Past the pin line, translate by exactly the overshoot: that cancels the scroll.

```tsx
// row i, whose top sits at i * ITEM in content space
const stylez = useAnimatedStyle(() => {
  const past = Math.max(0, scrollY.value - i * ITEM); // 0 until my top reaches the top edge
  return {
    transform: [
      { translateY: past },                             // cancels the scroll: pinned
      { scale: interpolate(past, [0, ITEM], [1, 0], Extrapolation.CLAMP) },
    ],
    opacity: interpolate(past, [0, ITEM], [1, 0], Extrapolation.CLAMP),
  };
});
```

Later cells paint over earlier ones, so the list reads as a stack with no `zIndex` work. The vertical carousel uses items 60% of the window height and an asymmetric `[1.2, 1, 0]` scale, so incoming cards arrive slightly large. Variable row heights work the same with each top from prefix sums (below). Older code writes this as a 1px unclamped segment, `[s, s + 1] → [0, 1]` with `EXTEND`; `Math.max` says the same thing readably.

The family is one counter-translation at different slopes:

- **Pin inside content** (slope 1): `translateY: Math.max(0, scrollY.value - PIN_AT)`. The expandable carousel pins its header icons at `statusBarHeight - 16` this way, no measuring.
- **Hold, then ride away** (an overlay outside the list): `translateY: -Math.max(0, scrollY.value - RELEASE_AT)`. The bank app's balance holds for `itemSize + 30` px, then leaves with the content.
- **Linger** (slope below 1): the Waze arrival picker counter-translates each row 0.75px per scrolled px near the focus line, so it moves at 25% speed, magnetized, while it fades. Input `[(i - 0.8) * ROW, i * ROW, (i + 0.8) * ROW]`, translateY `[-0.6 * ROW, 0, 0.6 * ROW]`, opacity `[0, 1, 0]`, with `ROW = Math.floor(H / 8)`, `snapToInterval={ROW}`, `paddingVertical: H / 2 - ROW / 2`, `bounces={false}`. Only the focused row shows its detail card; the row highlight is a 5% black fill.

Catalog: https://www.animatereactnative.com/post/vertical-carousel-animation-reanimated, https://www.animatereactnative.com/post/waze-time-of-arrival-reanimated2-+-flatlist

## Detached layers on the same driver

Titles, captions, and backdrops often live **outside** the list, absolutely positioned with `pointerEvents="none"`, yet read the same `scrollX` with the same index windows:

```tsx
// label i of N, stacked absolutely over the carousel
const labelStylez = useAnimatedStyle(() => ({
  opacity: interpolate(scrollX.value, [i - 0.5, i, i + 0.5], [0, 1, 0], Extrapolation.CLAMP),
  transform: [{
    translateX: interpolate(scrollX.value, [i - 1, i, i + 1], [width / 2, 0, -width / 2], Extrapolation.CLAMP),
  }],
}));
```

The opacity window is **narrower** than the movement window: at mid-swipe (`i + 0.5`) the outgoing and incoming labels are both at 0, a clean handoff instead of two half-opaque texts on top of each other. With a full ±1 opacity window both sit at 0.5 mid-swipe, which only reads when the travel is small (the awesome carousel's ±20px with `zIndex: N - i`). Vertical title stacks do the same on `translateY` (`[1.2 * fs, 0, -1.5 * fs]` for a 48pt title with `letterSpacing: -2`).

**Backdrops** (a color or blurred image per page) are one layer behind the list over the full range `[0, 1, 2, ...]`, never part of the item. Don't mount N full-screen blurred images: mount the settled index ±1 (next section), clamp, and keep it quiet (max opacity 0.8 under `blurRadius` 50). When the backdrop only changes at rest, swap it by key: ``<Animated.Image key={`bg-${index}`} entering={FadeIn.duration(1000)} exiting={FadeOut.duration(1000)} />``. Old and new are briefly mounted together, so both sit absolutely in the same box.

Catalog: https://www.animatereactnative.com/post/awesome-reanimated2-carousel, https://www.animatereactnative.com/post/apple-invites-animated-onboarding

## The settled index: round, then cross once

Some things belong in React: which backdrops mount, which detail card renders, an accessibility label, a haptic. Derive the index on the UI thread and cross threads **only when it changes**:

```tsx
useAnimatedReaction(
  () => Math.round(Math.min(Math.max(scrollX.value, 0), COUNT - 1)), // clamp the iOS bounce
  (cur, prev) => {
    if (cur !== prev) scheduleOnRN(setIndex, cur); // the callback runs every frame; this guard makes it once
  },
);
```

- **`Math.round`, not `Math.floor`.** Snapping lands at 2.9999 as often as 3.0001; floor reports 2 while the screen shows card 3. Round what you report and what you display.
- The reaction callback runs on every driver update, with `prev` as `null` the first time. Without the guard, `scheduleOnRN` fires per frame.
- Don't compute it in `onMomentumScrollEnd` + `setState`: the index lags the swipe, and the event can be skipped when a drag ends without momentum.
- `scheduleOnRN` comes from `react-native-worklets`; `runOnJS` still exists in Reanimated 4 as a deprecated alias. A picker's `Haptics.selectionAsync()` per index change goes through the same guard.

Catalog: https://www.animatereactnative.com/post/carousel-pagination-with-timer-reanimated, https://www.animatereactnative.com/post/custom-react-native-picker-reanimated

## One follower for the whole list, never springs per item

A spring or `withRepeat` inside a scroll-linked style restarts every frame, because the style re-runs on every scroll event. `withSpring(interpolate(scrollX.value, ...))` in two styles per item retargets 2N springs per frame, each lagging on its own. When the scroll needs inertia (jelly, velocity skew, a lean), build **one** follower at list level and hand it down:

```tsx
// list level: one spring trails the scroll; the gap is a smoothed velocity
const lazyY = useDerivedValue(() => withSpring(scrollY.value));
const skew = useDerivedValue(() =>
  interpolate(scrollY.value - lazyY.value, [-LAG, 0, LAG], [8, 0, -8], Extrapolation.CLAMP),
);
// every row: transform: [{ skewY: `${skew.value}deg` }]
```

- 8° is the ceiling for the jelly look; past that, text stops reading.
- v4's default spring (damping 120, stiffness 900, mass 4) is critically damped, so the follower never wobbles, and the gap settles at `velocity × damping / stiffness`, about velocity × 0.13s. `LAG` near 650px reproduces the catalog's ±8° at a 5px/ms fling. Check the sign on device.
- A hand-rolled `dy/dt` in `onScroll` sticks at its last value when the finger holds still (no events fire); the follower settles on its own.
- Idle motion on the focused card (a 10px bob over 2000ms, reversing) starts from the settled-index reaction and is cancelled in `onBeginDrag`. It never lives in the scroll-linked style.

The follower-lag trick in general (lean, atan2 on the gap) is in `references/animation-math.md`.

Catalog: https://www.animatereactnative.com/post/jelly-scrolling-reanimated2-+-flatlist

## Carousel checklist

- `horizontal`, `pagingEnabled` (full-width pages) or `snapToInterval={ITEM_FULL}` + `decelerationRate="fast"` (peeking cards).
- **Peek the neighbors**: size cards so ~15% of each neighbor is visible at rest. A rail where only one card shows reads as a pager, and users won't discover the scroll.
- **Parallax inside a card**: the card moves, its image (inside `overflow: 'hidden'`) counter-moves or turns over the same window. The wallpaper carousel turns the image `[15, 0, -15]°` and zooms `[1.6, 1, 1.6]`. Rotated content covers its frame only when `scale ≥ cos θ + (H / W) * sin θ`: 1.42 for a 1:1.76 card at 15°, so 1.6 leaves margin.
- **Variable widths**: measure each item's center once, `snapToOffsets={middles}` with `paddingHorizontal: listWidth / 2`, and build windows in pixels: `[mid - w / 2 - 42, mid, mid + w / 2 + 42] → [0.2, 1, 0.2]`.
- **Ruler and tick pickers**: 8px pitch, `snapToInterval={8}`, `paddingHorizontal: W / 2 - 4`, the needle is a fixed view **outside** the list, edges fade under a gradient overlay, neighbor ticks scale only 0.98, and the live number uses `fontVariant: ['tabular-nums']`.
- **Selection that steps aside**: a focus ring on a snapping picker hides while the list moves and returns at rest. Hide from `onBeginDrag` (100ms), return from `onMomentumEnd` (300ms), opacity and scale together: quick to leave, slow to come back. Each is one guarded assignment of a driver, never a timing per scroll event.
- **Over-scroll reveal**: start at index 1 over an empty index-0 placeholder (`initialScrollIndex={1}` + `getItemLayout`). Scrolling back past the first card grows an overlay card to full screen (scaleX/scaleY from the card-to-screen ratios, radius 30 → 0), and its content fades in only over the last half, which hides the scale distortion.

Catalog: https://www.animatereactnative.com/post/banking-slider-full-screen-slider, https://www.animatereactnative.com/post/react-native-animated-ruler-reanimated

## Pagination, autoplay, and tab indicators

Dots are items too; give each dot the same window over the same driver:

```tsx
const stylez = useAnimatedStyle(() => ({
  transform: [{ scaleX: interpolate(scrollX.value, [i - 1, i, i + 1], [1, 2.4, 1], Extrapolation.CLAMP) }],
  opacity: interpolate(scrollX.value, [i - 1, i, i + 1], [0.4, 1, 0.4], Extrapolation.CLAMP),
}));
```

- **Scale, don't resize.** The one exception: a pill that must **push its neighbors** (an 8px dot becoming a 24px pill while the row reflows) needs `width`. That is fine for ~10 dots in one row, never for anything per list item.
- **Worm**: translate a single active dot by `scrollX.value * DOT_SPACING` instead of animating every dot.
- **Fill-up**: `interpolateColor(scrollX.value, [i - 1, i, i + 1], [OFF, ON, ON])` keeps passed dots lit (an asymmetric window).
- **Indicator ladder** (vertical carousel): 10×2 dashes with 6px gaps over a 5-point window `[i - 2 ... i + 2]`, opacity `[0.2, 0.5, 1, 0.5, 0.2]`, scaleX `[1, 1.4, 2, 1.4, 1]`, `transformOrigin: 'right'` so they grow leftward from the rail.
- When the index is **state** (tap to select, no scroll), spring a copy, `useDerivedValue(() => withSpring(selected), [selected])`, and feed the same windows.

**Autoplay** stays on the UI thread: a timer value fills, its completion advances the index, and a reaction on the index scrolls and restarts the timer.

```tsx
useAnimatedReaction(() => active.value, (i, prev) => {
  if (i === prev) return;
  scheduleOnRN(onIndexChange, i);        // once per page
  scrollTo(listRef, i * width, 0, true); // worklet scroll: no JS ref methods inside worklets
  timer.value = 0;
  timer.value = withDelay(100, withTiming(1, { duration: 3000, easing: Easing.linear }, (done) => {
    if (done) active.value = (i + 1) % COUNT;
  }));
});
// onBeginDrag: cancelAnimation(timer), the finger always wins; the settled index writes active.value
```

The progress fill is `scaleX: timer.value` with `transformOrigin: 'left'`, not `width`. The active pill is 4× a 10px dot on a default spring, inactive dots sit at 0.5 opacity, prev/next controls are 24px.

**Tab indicators** follow the pager's scroll, not the selected-tab state (state set in `onMomentumScrollEnd` lags the swipe). Measure each tab once with `onLayout` into shared arrays, then:

```tsx
const indicatorStylez = useAnimatedStyle(() => ({
  width: interpolate(page.value, INDICES, ws.value), // one view: the width exception again
  transform: [{ translateX: interpolate(page.value, INDICES, xs.value) }],
}));
// center the active tab from the settled-index reaction:
// scrollTo(tabsRef, Math.max(0, xs.value[i] + ws.value[i] / 2 - width / 2), 0, true);
```

For tap-to-switch, the worm tabs add a squash-and-stretch hop (0 → 1 over 300ms): scaleX `[1, 2.2, 1]` and scaleY `[1, 0.5, 1]` over `[0, 0.5, 1]`, area ≈ 1.1, so it reads elastic rather than growing. The bar is 4px tall, radius 2, width `clamp(tabW - 20, 20, tabW)`, 5px under the label; the label color goes `#333 → #999` across `[0.3, 0.7]`.

Catalog: https://www.animatereactnative.com/post/onboarding-pagination-indicator-reanimated, https://www.animatereactnative.com/post/worm-scrollable-tabs

## Headers: collapsing, sticky, folding

The fan-out shape at its clearest (collapsing-sticky-header-react-native-reanimated recipe): the list's pixel scroll drives header height, title position, title scale, subtitle fade, and a divider, all over the **same** `[0, COLLAPSE]` window, so retuning the collapse distance is one constant and everything still finishes together.

```tsx
// shared by two consumers → hoist into a derived value
const headerHeight = useDerivedValue(
  () => interpolate(scrollY.value, [0, 120], [150, 64], Extrapolation.CLAMP),
);

const titleStylez = useAnimatedStyle(() => ({
  transform: [
    { translateY: interpolate(scrollY.value, [0, 120], [156, 74], Extrapolation.CLAMP) },
    { scale: interpolate(scrollY.value, [0, 120], [1, 0.74], Extrapolation.CLAMP) }, // large title → nav title
  ],
}));

const dividerStylez = useAnimatedStyle(() => ({
  transform: [{ translateY: headerHeight.value + HEADER_TOP }], // rides the LIVE bottom edge: computed, not positioned
  opacity: interpolate(scrollY.value, [40, 120], [0, 0.55], Extrapolation.CLAMP), // only exists once you've scrolled
}));
```

There is no "collapsed" boolean anywhere: every property is a pure function of scroll, so fling it, catch it mid-collapse, drag back down, and there is never a wrong frame. Prefer transforming the header (translate a fixed-height header under a clip) over animating `height`; where a real height is needed, keep the header shallow and give the list `contentContainerStyle={{ paddingTop: HEADER_MAX }}` so content starts below it.

- **One constant, many consumers.** The sticky-header example drives 12 properties from one `[0, 0.3 * H]` range: header 40% → 20% of the screen, tabs 20% → 10%, and a full-bleed tab bar turning into a floating card (radius 0 → 20; do the inset as a scaleX on a background layer, not `margin`). The credit-card screen runs four consumers on `[0, 200]`: header 200 → 70, the card number moves, and the footer fades 1 → 0 while drifting `0 → -67`, a third of the scroll speed.
- **Text size is a scale.** `fontSize` 64 → 24 re-lays out text every frame. Render at 64 and scale to 0.375 with `transformOrigin: 'left'` (RN 0.73+).
- **Chrome that appears only once you scroll**: a 0.1 hairline and an `rgba(0,0,0,0.02)` fill fading in over `[0, 60 + statusBar]` (iOS Calls), or a 0.2 shadow that starts only once the first row slides under the header.
- **Stretch on overscroll**: pulling down on iOS gives negative offsets. `scale: interpolate(scrollY.value, [-H, 0], [2, 1], { extrapolateLeft: Extrapolation.EXTEND, extrapolateRight: Extrapolation.CLAMP })` with `transformOrigin: 'top'` grows a header image 1:1 with the pull.

**3D flap header** (bank app): the header folds away on a hinge along its top edge.

```tsx
// static style: { transformOrigin: 'top' } (the hinge); H = 0.4 * width
const flapStylez = useAnimatedStyle(() => ({
  transform: [
    { perspective: H * 5 },
    { rotateX: `${interpolate(scrollY.value, [0, H], [0, 90], Extrapolation.CLAMP)}deg` },
  ],
  opacity: interpolate(scrollY.value, [H / 2, H], [1, 0], Extrapolation.CLAMP), // fade only the second half
}));
```

Full opacity until the fold is half done: one change at a time. The header is an overlay; its measured height (`onLayout`) sizes the `ListHeaderComponent` spacer so content starts below it. The original faked the hinge with `translateY` 0 → −H/2; `transformOrigin` does it directly.

Catalog: https://www.animatereactnative.com/post/bank-app-3d-sticky-scrolling-animation-reanimated2, https://www.animatereactnative.com/post/sticky-header-scrolling-animation

## Bottom sheets as drivers

A sheet library already computes a per-frame driver; use it instead of inventing one. `@gorhom/bottom-sheet` v5 writes `animatedIndex` (the fractional snap index: 0 → 1 between the first two snap points, −1 when closed) and `animatedPosition` (the sheet's top, in px) into shared values you pass in:

```tsx
const sheetIndex = useSharedValue(0);
<BottomSheet snapPoints={[H / 2, H - 64 - insets.top]} animatedIndex={sheetIndex}>...</BottomSheet>

// anything outside the sheet derives from it
const barStylez = useAnimatedStyle(() => ({
  opacity: interpolate(sheetIndex.value, [0, 1], [1, 0], Extrapolation.CLAMP),
  transform: [{ translateY: interpolate(sheetIndex.value, [0, 1], [0, BAR_H], Extrapolation.CLAMP) }],
}));
```

- Prefer `animatedPosition` + `translateY` over interpolating `bottom` or `top` (layout props).
- **Text leaves first.** The Telegram stories sheet nests sub-windows on one index: labels gone by 0.1, non-header avatars by 0.3, header avatars appear over `[0.95, 1]`, and the title shifts −45px. Everything else waits.
- **Sheet-driven deck**: cards stacked while the sheet is low, laid out naturally once it opens. Each card's natural top `y` comes from data (`index * ROW`), not a measurement: `translateY: interpolate(sheetIndex.value, [0, 1], [PEEK * index - y, 0], Extrapolation.CLAMP)`, with `PEEK` 98px (title plus tag row visible).

Catalog: https://www.animatereactnative.com/post/discover-bottomsheet-custom-scroll-animation, https://www.animatereactnative.com/post/telegram-stories-scroll-animation

## Linked panes, nested scrollers, pinned layers

**Follower panes are transforms, not second ScrollViews.** A calendar's day-header row and hours column must track the grid. Two ScrollViews synced through events fight and lag a frame; one scroll value per axis and a plain row inside `overflow: 'hidden'` can't drift:

```tsx
const dayHeaderStylez = useAnimatedStyle(() => ({ transform: [{ translateX: -scrollX.value }] }));
const hoursStylez = useAnimatedStyle(() => ({ transform: [{ translateY: -scrollY.value }] }));
```

Google Calendar numbers: day width `W / 6` (five days plus the hours column), hour rows 1.4× the day width, snap every two days, `bounces={false}` (https://www.animatereactnative.com/post/google-calendar).

**Nested scroll lock** (a horizontal pager of vertical pages): the pager may swipe only while the open page is scrolled to its top. The safe version flips React state once per change:

```tsx
useAnimatedReaction(() => innerY.value <= 0, (atTop, prev) => { // <= 0, not === 0: bounce goes negative
  if (atTop !== prev) scheduleOnRN(setPagerEnabled, atTop);
});
<Animated.FlatList horizontal pagingEnabled scrollEnabled={pagerEnabled} ... />
```

The catalog drives this with `useAnimatedProps(() => ({ scrollEnabled: innerY.value <= 0 }))`, which skips the re-render, but that is Reanimated 3-era and unconfirmed on Reanimated 4 / Fabric: test it on both platforms before relying on it. A close button that `scrollTo`s the page back to 0 unlocks the pager (https://www.animatereactnative.com/post/expandable-carousel). For pans inside scroll views, see `references/gestures.md`.

**Viewport-pinned layer inside scrolling content** (Messenger's gradient bubbles): a bubble's color depends on where it is on screen. Give each bubble a screen-tall gradient that cancels the scroll:

```tsx
// bubble at content top itemY (prefix sums); VIEWPORT_H from the list's onLayout
const fillStylez = useAnimatedStyle(() => ({
  transform: [{ translateY: scrollY.value - itemY }], // the gradient holds still on screen
}));
<View style={{ overflow: 'hidden', borderRadius: 18 }}>
  <Animated.View style={[{ position: 'absolute', top: 0, left: 0, right: 0, height: VIEWPORT_H }, fillStylez]}>
    <LinearGradient colors={['#FD84AA', '#A38CF9', '#09E0FF']} style={{ flex: 1 }} />
  </Animated.View>
  <Text>{text}</Text>
</View>
```

One extra view per own bubble, and virtualization stays intact. The original duplicated the whole list inside a mask: 200 views for 100 messages, no virtualization (https://www.animatereactnative.com/post/facebook-messenger-gradient-conversation).

## Endless marquees

Infinite rails (ambient photo walls, onboarding card loops) can't use index space: the offset grows forever. Wrap in **screen space** with a modulo that is safe for negatives, and advance by time, not frames:

```tsx
const wrap = (x: number, n: number) => { 'worklet'; return ((x % n) + n) % n; }; // -3 % 10 is -3 in JS

useFrameCallback((f) => {
  if (!paused.value) offset.value += (SPEED * (f.timeSincePreviousFrame ?? 0)) / 1000; // px per second
});

// tile i of N, TOTAL = N * ITEM (at least screen width + ITEM; repeat the data if not)
const stylez = useAnimatedStyle(() => ({
  transform: [{ translateX: wrap(i * ITEM - offset.value + ITEM, TOTAL) - ITEM }],
}));
```

- `+= 0.2` per frame is 12px/s at 60Hz and 24px/s at 120Hz. Integrate time instead.
- Drag to scrub: set `paused` in the pan's `onBegin`, write `offset` from the drag, release with `withDecay({ velocity: -e.velocityX })`, and unpause when the decay finishes.
- Restraint: the AI playground wall drifts at 12px/s (one loop ≈ 185s), three rows in alternating directions, tiles 0.45W with 8px gaps, the whole wall rotated −4°, edges faded 15% top and 30% bottom. Apple Invites tilts each card by its screen position, `interpolate(x, [-ITEM, (W - CARD) / 2, W], [-3, 0, 3])` degrees: ±3° across the full width.

Catalog: https://www.animatereactnative.com/post/apple-invites-animated-onboarding, https://www.animatereactnative.com/post/ai-image-playground-react-native-marquee

## Rows that come and go: filters and live feeds

Layout animations do the list work scroll offsets can't: rows arriving, leaving, and the survivors closing the gap. General habits (hoisted instances, identity keys) are in `references/core-concepts.md`; these are the list-specific rules.

**Filtering** is a data swap; keys and one transition do the rest:

```tsx
const _layout = LinearTransition.springify(); // hoisted: built once, one motion character

<Animated.FlatList
  data={results.slice(0, 10)}                    // cap what animates
  keyExtractor={(c) => c.id}                     // identity, never index
  itemLayoutAnimation={_layout}                  // single-column lists only
  renderItem={({ item }) => <Row item={item} />} // Row root: entering={FadeInRight} exiting={FadeOutLeft}
/>
```

- **`itemLayoutAnimation` works only with a single column** (`numColumns` must be 1). It wraps each cell in an `Animated.View` with `layout`, so don't also put `layout` on the row: pick one.
- Enter and exit along the list's axis (`FadeInRight`/`FadeOutLeft` on a horizontal strip, `FadeInDown`/`FadeOutUp` on a column). The presets travel only 25px, which is plenty here.
- Before swapping data, `scrollToOffset({ offset: 0, animated: true })` so the change is visible; debounce the query (500ms) inside a `useMemo`, not rebuilt per render.
- `skipEnteringExitingAnimations` skips row `entering` when the list first mounts and `exiting` when it unmounts. In 4.5.1 any defined value turns it on, `false` included: pass it or omit it.
- Per-row delays use the static form, `FadeInDown.delay(i * 50)`, which builds a fresh instance. Calling `.delay()` on a hoisted instance mutates it, so every row gets the last delay.

**Live feeds** (chat, activity, comments) use an `inverted` list, newest item at index 0 at the bottom:

```tsx
const _arrive = FadeInDown.springify().withInitialValues({ opacity: 0, transform: [{ translateY: 100 }] });

useAnimatedReaction(() => scrollY.value > 0, (away, prev) => {
  if (away !== prev) scheduleOnRN(setUserScrolling, away); // once per flip, never per scroll event
});
<Animated.FlatList inverted onScroll={onScroll}
  itemLayoutAnimation={userScrolling ? undefined : _layout}
  maintainVisibleContentPosition={userScrolling ? { minIndexForVisible: 0 } : undefined} ... />
```

- A chat push wants ~100px of travel, so override the 25px preset with `withInitialValues`.
- Older messages fade with age: opacity `Math.max(0, 1 - index / 6)`, animated with a 250ms timing (a spring overshoots opacity). Once the user scrolls away from the bottom, everything returns to full opacity.
- While the user reads history, turn `itemLayoutAnimation` off: it fights `maintainVisibleContentPosition`, which keeps their place as messages arrive.

Catalog: https://www.animatereactnative.com/post/incoming-chat-messages-animation-reanimated, https://www.animatereactnative.com/post/contacts-filter-layout-animation-+-dialpad

## Lists at frame rate

- The scroll shared value goes to items via prop or context; shared values are stable references and cause no re-renders. Items interpolate from `(scrollX, i)` internally.
- `renderItem` must return a real memoized component (hooks per item live there). No inline hook calls in `renderItem` closures.
- **No layout properties in scroll-linked styles**: `width`, `height`, `margin`, `padding`, `top`/`left`, `fontSize`. The catalog's own offenders and their fixes: card padding 16 → 0 for edge-to-edge (scale a background layer), avatars 70 → 30 via width/height (scale 0.43), a card moved with `left`/`top` (translate).
- **Mount only what's near**: backdrops at the settled index ±1, decks the visible 3 plus 1, filter results capped at 10.
- **Known heights → prefix sums.** Compute offsets from data instead of measuring, and give the list `getItemLayout`:

```tsx
const offsets = useMemo(() => {
  let y = 0;
  return data.map((d) => { const top = y; y += d.height + GAP; return top; });
}, [data]);
// getItemLayout={(_, i) => ({ length: data[i].height + GAP, offset: offsets[i], index: i })}
// item i reaches the top edge at scrollY = offsets[i]: build its window around that
```

- **Leave `CellRendererComponent` alone.** `Animated.FlatList` owns it (it's how `itemLayoutAnimation` works, and the prop is typed `never`). For per-cell styles such as a stack's `zIndex`, 4.5.1 has `CellRendererComponentStyle={({ index }) => ({ zIndex: N - index })}`. On a plain list, a custom cell renderer must forward `props.onLayout` (or the list loses its measurements) and be defined outside render (an inline one remounts every cell). Never clone children to inject measurements; compute from the index.
- FlashList/LegendList work with the same pattern; the offset driver does not care which list renders the items.
- Entering animations on items are fine on initial mount; disable or simplify them for items mounting during fast scroll on low-end Android.

## Scroll-triggered (as opposed to scroll-linked)

"Animate the card in when it enters the viewport" is the one legitimate discrete case. Derive it on the UI thread, fire once:

```tsx
const shown = useSharedValue(0);
useAnimatedReaction(
  () => scrollY.value > itemTop - VIEWPORT_H * 0.8,
  (visible, was) => {
    if (visible && !was) shown.value = withTiming(1, { duration: 400 });
  },
);
```

Still no `setState`, still interruption-safe, and `shown` is just another driver for the item's interpolations.

## Real-world examples

Interactive versions with live previews and full generated code, at keyframer.dev/recipes/:

- The per-item window, vertical: `react-native-carousel-animation-flatlist-reanimated`
- Flip one axis + the arc + cover flow: `horizontal-carousel-animation-react-native`
- Varying item sizes: `carousel-different-item-heights-react-native` · free scrolling: `free-scroll-carousel-react-native`
- The five-consumer fan-out: `collapsing-sticky-header-react-native-reanimated`

Complete catalog implementations are linked at the end of each section above (animatereactnative.com).
