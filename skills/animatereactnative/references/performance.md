# Performance: Holding 60-120fps

Frame budget: 16.6ms at 60fps, 8.3ms at 120fps. An animation drops frames for one of three reasons: work on the wrong thread, layout passes per frame, or React re-renders per frame. Audit in that order.

Polish and frame rate come from the same habit. The catalog animations that look most expensive are the cheapest to run: one shared value, a few interpolations, small numbers. This file covers both halves: what the good examples spend (the budget), and how the rest overspend, grouped by cause, each with its fix.

## The audit checklist (when something "feels janky")

1. **Is anything crossing to the JS thread per frame?** Search the animation path for `scheduleOnRN`/`runOnJS`, `setState`, `console.log`, or reading `sharedValue.value` outside a worklet. Any of these inside `onScroll`, `onUpdate`, `useAnimatedReaction`, or `useFrameCallback` is the bug.
2. **Is a layout property animating?** `width`, `height`, `top`, `left`, `margin`, `padding`, `flex`, `fontSize` trigger layout every frame. Replace with `transform`.
3. **Is React re-rendering during the animation?** Add a render log to the component and its list parent. Re-renders per frame usually come from state derived from animation progress.
4. **Is an animation restarted every frame?** A `withSpring` or `withRepeat` built inside a scroll-linked style or handler starts over on every event, once per item.
5. **Is it actually slow, or is it a debug build?** Profile in release. Debug builds are not representative.
6. **How much is live?** Practical limits: ~500 animated components on iOS, ~100 on low-end Android. Count views and shared values per item against the budget below; past the limits, reduce scope or move the scene to Skia.

## The shape that holds frame rate

- **One shared value per interaction**, in semantic units: 0 → 1, or index space (`offset / ITEM_FULL`, normalized once, where the value is written). A gesture, a scroll, or a timing writes it; nothing else does.
- **Everything else is derived.** Styles interpolate the driver, a value with two consumers is hoisted into one `useDerivedValue`, and React state changes only at discrete moments (a page settles, a sheet snaps).
- **Few properties, small ranges.** Two or three channels per element, with numbers like the ones in the budget below. Each extra channel is per-frame work times every item, and usually reads as noise anyway.
- **Name each knob once.** One window for every consumer (sticky-header: 12 properties on one `[0, 0.3H]` window), one duration for every piece of a morph (fab-button: one 500ms `LinearTransition`), one stagger unit with durations as multiples of it (dots-ripple: in 5×, out 3×). Retuning is one edit and nothing drifts.
- **One typed, memoized component per animated thing.** Hooks live per item, the parent renders once, and the shared value travels as a prop (a stable reference, so no re-renders):

```tsx
type DotProps = { index: number; progress: SharedValue<number> };

const Dot = memo(function Dot({ index, progress }: DotProps) {
  const stylez = useAnimatedStyle(() => {
    const range = [index - 1, index, index + 1];
    return {
      opacity: interpolate(progress.value, range, [0.3, 1, 0.3], Extrapolation.CLAMP),
      transform: [{ scaleX: interpolate(progress.value, range, [1, 2, 1], Extrapolation.CLAMP) }],
    };
  });
  return <Animated.View style={[styles.dot, stylez]} />;
});
```

The same shape scales into reusable pieces: a `<Stagger>` wrapper that hands each child its delay (https://www.animatereactnative.com/post/reanimated-custom-stagger-component), and custom entering worklets typed as `EntryExitAnimationFunction` rather than silenced with `@ts-ignore` (https://www.animatereactnative.com/post/announcements-react-native-reanimated-custom-layout-animation).

## Budget: what the good examples spend

Counted in the catalog's Reanimated-only animations. Use them as a sanity check: if your version of an effect needs ten times the shared values, the structure is off, not the device.

- **1 value → 7 bars.** Leaderboard: after the rows land, one `grow` value raises every bar (each derives `withDelay(i * 50, withSpring(grow.value))`); only the winner gets color. https://www.animatereactnative.com/post/leaderboard-animation-reanimated
- **3 values → 100 tiles.** Isometric press grid: finger x, finger y, and a press spring gate. Each tile derives one lift that feeds its face and its body (300 views, 200 of them animated). https://www.animatereactnative.com/post/isometric-press-grid
- **1 sheet index → a whole screen.** Telegram stories flies avatars into the header, clears labels by 10% of the travel, and shifts the title, all on sub-windows of the sheet's index; discover-bottomsheet drives a card deck, heading, and container the same way. Read the sheet library's own `animatedIndex` instead of adding a value. https://www.animatereactnative.com/post/telegram-stories-scroll-animation, https://www.animatereactnative.com/post/discover-bottomsheet-custom-scroll-animation
- **1 scroll value → 12 properties.** Sticky header: header 40% → 20% of the screen, tabs, and a full-bleed tab bar turning into a floating card, on one window. https://www.animatereactnative.com/post/sticky-header-scrolling-animation
- **1 float → a deck.** Tinder poses 3 visible cards (plus 1 hidden) from one fractional index; stack cards derives every depth step from the same kind of float. https://www.animatereactnative.com/post/tinder-swiper-animation-reanimated-+-gesture-handler, https://www.animatereactnative.com/post/react-native-stack-cards-animation
- **1 spring gate → 5 properties.** Balloon slider's knob press: border 12 → 2 (the dot opens into a ring), knob scale, balloon opacity, scale, and a 48px rise. https://www.animatereactnative.com/post/ballon-slider-reanimated-gestures-sensors-and-math
- **1 value → 1 transform.** Shimmer text moves a single 90px band. https://www.animatereactnative.com/post/shimmer-text
- **Counter-examples, fixed below:** dots-ripple (~312 dots plus 312 `Pressable`s, ~937 shared values, 624 never read) and the messenger gradient (200 views for 100 messages).

**Small numbers that read as polished:**

| Knob | Value | Seen in |
|---|---|---|
| Neighbor scale | 0.98 on ruler ticks (2%), 0.92 on off-focus rows | https://www.animatereactnative.com/post/react-native-animated-ruler-reanimated, https://www.animatereactnative.com/post/perplexity-vertical-list-animation-reanimated |
| Press and pop | 0.9 squish on press; 1.1 pop plus a 2px label rise on select | https://www.animatereactnative.com/post/squishy-button-reanimated-layout-animation, https://www.animatereactnative.com/post/drag-tasks-react-native-reanimated2-animation |
| Stack depth | 22px, 0.05 scale, and 1/6 opacity per card | https://www.animatereactnative.com/post/react-native-stack-cards-animation |
| Off-focus opacity | 0.3 for cards, 0.25 for indicator lines, 0.15 for unselected bars | https://www.animatereactnative.com/post/banking-slider-full-screen-slider, https://www.animatereactnative.com/post/arc-carousel-reanimated, https://www.animatereactnative.com/post/custom-bank-spendings-custom-barchart-+-number-animation-reanimated2 |
| Chrome | 2% black header fill with a 0.1 hairline; 5% row highlight; scroll shadow up to 0.2 | https://www.animatereactnative.com/post/react-native-ios-calls-(all-+-missed)-animation, https://www.animatereactnative.com/post/waze-time-of-arrival-reanimated2-+-flatlist |
| Rotation | ±3° across the full screen width, ±15° inner image tilt, ±20° card peeks, skew capped at 8° | https://www.animatereactnative.com/post/apple-invites-animated-onboarding, https://www.animatereactnative.com/post/wallpaper-animated-carousel-react-native-reanimated, https://www.animatereactnative.com/post/jelly-scrolling-reanimated2-+-flatlist |
| Travel | ±20px for swapping labels, 10px nudge on a month label, 26px max lift, 10px idle bob over 2000ms | https://www.animatereactnative.com/post/awesome-reanimated2-carousel, https://www.animatereactnative.com/post/random-people-carousel-animation-reanimated2-+-flatlist |
| Timing | 100ms to step aside, 300ms to come back; one 500ms constant for a whole morph | https://www.animatereactnative.com/post/country-flag-selector-reanimated2-+-moti, https://www.animatereactnative.com/post/fab-button-animation-reanimated-3 |
| Stagger | 50ms for tiles and bars, 70ms for text lines, ~140ms for full rows | https://www.animatereactnative.com/post/reanimated-custom-stagger-component |
| Ambient loops | 12px/s marquee, 18°/s orbit at 0.3 alpha, 1300ms shimmer sweep with a 300ms rest | https://www.animatereactnative.com/post/ai-image-playground-react-native-marquee, https://www.animatereactnative.com/post/lava-lamp-background-animation |
| How much moves | one highlighted bar of seven; only the first visible title gets an entrance; at most 10 animated filter results | https://www.animatereactnative.com/post/leaderboard-animation-reanimated, https://www.animatereactnative.com/post/vertical-carousel-animation-reanimated, https://www.animatereactnative.com/post/contacts-filter-layout-animation-+-dialpad |
| Order | text leaves first (labels gone by 10% of the sheet's travel); one change at a time (the 3D fold is half done before anything fades) | https://www.animatereactnative.com/post/telegram-stories-scroll-animation, https://www.animatereactnative.com/post/bank-app-3d-sticky-scrolling-animation-reanimated2 |

## Wrong/right catalogue, by cause

### Layout per frame

`width`, `height`, `left`, `top`, `margin`, `padding`, `flex`, and `fontSize` re-run layout for the subtree on every frame. Lay the element out once at its final (usually largest) size and move it with transforms.

**Size change:**

```tsx
// WRONG: layout every frame
width: interpolate(progress.value, [0, 1], [56, 240])

// RIGHT: same visual, fast path
transform: [{ scaleX: interpolate(progress.value, [0, 1], [1, 240 / 56]) }]
// (design the element at its largest size and scale down when content allows;
// scaling up strokes/text distorts them)
```

**Bars and fills: scale from an edge.** A fixed-size view plus `transformOrigin` (RN 0.73+) keeps one edge planted:

```tsx
// WRONG: height looped per bar (24 bars × 11 buttons = 264 layout animations)
height: withRepeat(withTiming(MAX_H, { duration: 400 }), -1, true)

// RIGHT: full-height bar, scaled from its base
<Animated.View style={[{ height: MAX_H, transformOrigin: 'bottom' }, barStylez]} />
const barStylez = useAnimatedStyle(() => ({
  transform: [{ scaleY: interpolate(level.value, [0, 1], [0.1, 1]) }],
}));
```

A progress fill is the same move on the other axis: `scaleX` with `transformOrigin: 'left'`. https://www.animatereactnative.com/post/squishy-button-reanimated-layout-animation, https://www.animatereactnative.com/post/carousel-pagination-with-timer-reanimated

**Text: scale, don't resize the font.** `fontSize` 64 → 24 in a collapsing header re-measures text every frame: render at 64 and scale to 0.375 with `transformOrigin: 'left'`. A `fontSize` spring layered on an existing 1.7 scale adds layout for nothing; the scale carries it alone. https://www.animatereactnative.com/post/contacts-sticky-header-+-actions-concept-reanimated2, https://www.animatereactnative.com/post/slider-with-fancy-animated-text

**Moving boxes: translate fixed pieces.** A scanner frame animating `left`/`top`/`width`/`height`/`borderRadius` becomes four fixed-size corner brackets that only translate to the box's corners (each rotated k × 90°). https://www.animatereactnative.com/post/qrcode-scanner-bounding-box

**Split views: counter-translate a clip window.** A before/after compare that resizes two images with `flex` reflows and shifts the crop. Instead: an `overflow: 'hidden'` window at `translateX: x - W`, its image at `translateX: W - x` (net zero, the picture stays still), the handle at `translateX: x`, with x clamped to `[0, W]`. https://www.animatereactnative.com/post/slide-to-compare-image-reanimated2-and-gesturehandler

**Margins and padding that open up.** A keyboard-aware `marginBottom` becomes `translateY`; card padding 16 → 0 on scroll, or a header's `left`/`top`/`height`, becomes a translated and scaled background layer, leaving the content's box alone. https://www.animatereactnative.com/post/expandable-carousel, https://www.animatereactnative.com/post/credit-card-scroll-animation

**When layout IS the effect, keep it small.** A pagination pill that pushes ~10 neighboring dots aside, a single 24px dot whose round caps `scaleX` would squash, three bars whose sibling reflow is the animation: shallow trees, never per list item, never under a scrolling list. For a jump between two sizes, toggle the static style and let one hoisted `LinearTransition` animate the frame (fab-button: width 64 → 85% of the screen, `borderRadius` fixed at 32 so the circle becomes a panel without animating the radius). https://www.animatereactnative.com/post/onboarding-pagination-indicator-reanimated, https://www.animatereactnative.com/post/processing-loop-react-native-reanimated2, https://www.animatereactnative.com/post/fab-button-animation-reanimated-3

### Animations restarted every frame

**Springs and loops inside scroll-linked styles.** A spring created inside a per-item style restarts on every scroll event, once per item per style:

```tsx
// WRONG: N items × 2 styles = 2N springs restarted per frame
transform: [{ translateY: withSpring(interpolate(scrollX.value, [i - 1, i, i + 1], [40, 0, 40])) }]

// RIGHT: one follower for the list, items interpolate it
const lazyX = useDerivedValue(() => withSpring(scrollX.value)); // once, in the list
transform: [{ translateY: interpolate(lazyX.value, [i - 1, i, i + 1], [40, 0, 40], Extrapolation.CLAMP) }]
```

The same carousel picks an idle 10px bob (`withRepeat`) with a ternary inside the style; start it from a reaction on the settled index instead, and cancel it on drag. One list-level follower is also all the jelly skew needs (`scrollY - lazyY`, capped at 8°). https://www.animatereactnative.com/post/random-people-carousel-animation-reanimated2-+-flatlist, https://www.animatereactnative.com/post/jelly-scrolling-reanimated2-+-flatlist

**Decorative loops: CSS animations.** A loop started from inside `useDerivedValue` that captures a `Math.random()` evaluated in render restarts, with new values, on every render. A Reanimated 4 CSS animation needs no shared value and nothing to cancel; memoize any randomness:

```tsx
const ORBIT = { from: { transform: [{ rotate: '0deg' }] }, to: { transform: [{ rotate: '360deg' }] } };

<Animated.View style={{
  animationName: ORBIT,
  animationDuration: '20s',
  animationIterationCount: 'infinite',
  animationTimingFunction: 'linear',
}} />
```

https://www.animatereactnative.com/post/lava-lamp-background-animation

**`withTiming(v, { duration: 0 })` is an assignment.** `sv.value = v` does the same without allocating an animation. `withDelay(50, withTiming(y, { duration: 0 }))` in a scroll handler queues a stale write per event; write directly. https://www.animatereactnative.com/post/incoming-chat-messages-animation-reanimated

### Thread crossings

**State from a per-frame source:**

```tsx
// WRONG: a JS call on every scroll event
onScroll: (e) => { scheduleOnRN(setIsScrolling, e.contentOffset.y > 0); },

// RIGHT: derive the boolean on the UI thread, cross only when it flips
useAnimatedReaction(
  () => scrollY.value > 0,
  (now, prev) => { if (now !== prev) scheduleOnRN(setIsScrolling, now); },
);
```

If only visuals depend on it, skip React entirely and keep it a style (`opacity: progress.value > 0.5 ? 1 : 0`). `prev` is `null` on the first run. For "current page", prepare `Math.round(scrollX.value)`: momentum can settle at 2.9999, which `Math.floor` turns into 2. Comparing against React state inside the worklet (`if (i !== activeIndex)`) reads a stale copy and fires every frame; let `prev` do the comparison. `scheduleOnRN` is for discrete moments (haptics on a snap, navigation after a dismiss), once per event. https://www.animatereactnative.com/post/incoming-chat-messages-animation-reanimated

**Reading `.value` on the JS thread.** In render, 4.5.1 warns (in dev) and the number is stale by the next frame; in a handler, a value the UI thread has changed is fetched with a synchronous call into the UI runtime. Catalog cases: `data[activeIndex.value]` and `value={text.value}` in render, `scrollEnabled={y.value === 0}`, `[value.value]` as effect deps (https://www.animatereactnative.com/post/custom-bank-spendings-custom-barchart-+-number-animation-reanimated2). Derive instead (`useDerivedValue`, `useAnimatedProps`), or mirror into state with the reaction above. Toggles are the most common case:

```tsx
// WRONG: reads UI state in a JS handler; a tap while closing is ignored (0.6 !== 0)
onPress={() => { progress.value = withTiming(progress.value === 0 ? 1 : 0); }}

// RIGHT: JS owns the intent, the driver chases it
const open = useRef(false);
onPress={() => {
  open.current = !open.current;
  progress.value = withTiming(open.current ? 1 : 0, { duration: 350 });
}}
```

Steppers follow the same rule: retarget an integer target (`target + 1`), never `withTiming(index.value + 1)` from the in-flight value.

**Choreography on JS timers.** Per-character `setTimeout`s writing shared values, or `sleep()` chains before a scroll, slip whenever the JS thread is busy. One driver with precomputed windows runs it on the UI thread: the scramble text works as one 1200ms linear `withTiming`, a lock time per character, and `useAnimatedProps` on a non-editable `TextInput`. https://www.animatereactnative.com/post/hacker-text-greater-react-native-reanimated

**JS-thread APIs inside worklets.** `listRef.current?.scrollToOffset()` inside `useAnimatedReaction`, or `setNativeProps({ contentOffset })`: call `scrollTo(animatedRef, x, y, animated)` from the worklet (`scheduleOnUI(scrollTo, animatedRef, x, 0, true)` from JS), or move the content with a transform.

**`console.log` in a worklet** is a `scheduleOnRN` to the JS console with serialized arguments. In a per-frame worklet, that is a thread crossing every frame: delete it before measuring.

### Repeated or wasted work

**One division, repeated per item.** A `useDerivedValue` per item computing `scroll / ITEM_SIZE` does the same math N times a frame. Normalize once in the scroll handler (`scrollX.value = e.contentOffset.x / ITEM_FULL`) and items read index space directly (`references/scroll-and-lists.md`). https://www.animatereactnative.com/post/carousel-animation-x-and-y-rotation

**`measure()` every frame.** It reads layout synchronously on the UI thread and returns `null` when the view can't be measured (a virtualized cell that isn't mounted). Sizes that don't change go through `onLayout` once into a shared value (a slider's track width). Measure per frame only when the source itself moves: telegram stories measures avatars mid-flight because they sit in a horizontal `ScrollView`; a static source is measured once. https://www.animatereactnative.com/post/slider-with-fancy-animated-text, https://www.animatereactnative.com/post/telegram-stories-scroll-animation

**Mount only what's near the viewport.** A carousel that mounts every full-screen, blurred backdrop pays for N images to show one. Window them to `i - 1 .. i + 1` from a reaction on the rounded index; each mounted backdrop still interpolates the scroll itself:

```tsx
const [center, setCenter] = useState(0);
useAnimatedReaction(
  () => Math.round(scrollX.value),
  (i, prev) => { if (i !== prev) scheduleOnRN(setCenter, i); },
);
{images.map((uri, i) => Math.abs(i - center) <= 1
  ? <Backdrop key={uri} index={i} uri={uri} scrollX={scrollX} />
  : null)}
```

Decks do the same: `slice(active, active + 4)`, with state changing only when the rounded index flips. https://www.animatereactnative.com/post/wallpaper-animated-carousel-react-native-reanimated, https://www.animatereactnative.com/post/tinder-swiper-animation-reanimated-+-gesture-handler

**Shared values nobody reads, touch targets per cell.** The dots grid keeps three shared values per dot (two never read) and a `Pressable` per dot. One origin value plus one gesture on the whole grid does the job:

```tsx
const tap = useMemo(() => Gesture.Tap().onEnd((e) => {
  origin.value = { col: Math.floor(e.x / CELL), row: Math.floor(e.y / CELL) };
}), []);
```

https://www.animatereactnative.com/post/dots-ripple-reanimated

**Duplicating a list to mask it.** Rendering every message twice (once inside a gradient mask) doubles the views and defeats virtualization. Instead, each bubble clips (`overflow: 'hidden'`) a screen-tall gradient translated by `scrollY - itemY`: one extra view per bubble, and it works in a virtualized list. https://www.animatereactnative.com/post/facebook-messenger-gradient-conversation

**Stacked blurs.** A blur view per blob underneath one full-screen blur adds blur passes the full-screen one already covers. One full-screen blur over the whole layer is the look. https://www.animatereactnative.com/post/lava-lamp-background-animation

### Identity and closures

**Worklet closures** serialize their captured scope. Capturing a theme object, a nav object, or a whole item captures everything reachable from it:

```tsx
// WRONG: captures the entire theme
const theme = useTheme();
const style = useAnimatedStyle(() => ({ backgroundColor: theme.colors.primary }));

// RIGHT: capture one primitive
const primary = useTheme().colors.primary;
const style = useAnimatedStyle(() => ({ backgroundColor: primary }));
```

Also: `'worklet'` functions are not hoisted; define them above their first use in another worklet.

**Refs captured by worklets are copies.** A `useRef` object captured by a worklet is copied (and frozen in dev, which warns on later writes), so the UI thread never sees `.current` change. Anything both threads read or write is a shared value.

**Gestures rebuilt every render.** Seven of the thirteen gesture examples build their gesture objects inline; with Gesture Handler v2, wrap them in `useMemo` (`references/gestures.md`).

**Inline `CellRendererComponent`.** Defined inside render, it is a new component type every render, so every cell remounts. Define it once outside; if you override it, forward `props.onLayout?.(e)` (the list's own bookkeeping depends on it), and compute offsets from the index (prefix sums in `useMemo`) rather than measuring inside the cell.

**Hooks in loops.** `useAnimatedRef` inside a `.map` or `.reduce` breaks the rules of hooks when the count changes. One component per slot owns its ref.

**Hoist builders, but don't share a delayed one.** A layout animation built at module scope is not rebuilt per render. But `.delay()` on an instance mutates and returns that same instance, so children sharing one instance all get the last delay set. Per-index delays come from a factory:

```tsx
const _layout = LinearTransition.springify();                         // shared: no per-child config
const enterAt = (i: number) => FadeInDown.delay(i * 50).springify(); // static .delay() makes a new instance
```

Same reason to memoize a debounced search handler instead of rebuilding it every render. https://www.animatereactnative.com/post/reanimated-custom-stagger-component, https://www.animatereactnative.com/post/contacts-filter-layout-animation-+-dialpad

### Clocks, sizes, and endings

**Speed tied to frame rate.** Adding 0.2px per frame is 12px/s at 60Hz and 24px/s on a 120Hz screen. Integrate elapsed time:

```tsx
useFrameCallback((frame) => {
  const dt = frame.timeSincePreviousFrame ?? 0; // ms, null on the first frame
  offset.value += (SPEED_PX_PER_S * dt) / 1000;
});
```

https://www.animatereactnative.com/post/ai-image-playground-react-native-marquee

**Animation callbacks that ignore `finished`.** A follow-up that runs on cancellation fights the next gesture:

```tsx
x.value = withDecay({ velocity: e.velocityX }, (finished) => {
  if (!finished) return; // a new touch cancelled the coast: don't snap under the finger
  x.value = withSpring(nearestSnap(x.value));
});
```

**`Dimensions.get('screen')` for layout.** On Android the screen includes the system bars, so full-page items come out too tall and scroll math drifts. Use `useWindowDimensions()` or `onLayout`.

**`scrollEventThrottle`.** Use `16`: React Native treats any value of 16 or less as "no throttling", at any refresh rate. `32` delivers about 30 events a second, and the animation visibly steps.

## Reanimated 4 API hygiene

Checked against Reanimated 4.5.1 and react-native-worklets 0.10.1.

- **`runOnJS(fn)(args)` → `scheduleOnRN(fn, args)`** from `react-native-worklets`; `runOnUI(fn)(args)` → `scheduleOnUI(fn, args)`. The old names are still exported, deprecated, as shims that call the new ones, so migrating changes nothing about cost: what matters is how often you cross.
- **`addWhitelistedNativeProps` / `addWhitelistedUIProps`** are deprecated no-ops in v4 (still reachable on the default `Animated` export). Delete them.
- **`useAnimatedKeyboard`** is deprecated in favor of `react-native-keyboard-controller`.
- **Bare `withSpring(x)` and `.springify()` don't bounce in v4.** The default is `GentleSpringConfig` (damping 120, mass 4, stiffness 900: damping ratio 1, no overshoot), so a squish ported from v3 goes flat. Pass a config: `WigglySpringConfig` (ratio 0.75), `SnappySpringConfig` (overshoot clamped), `Reanimated3DefaultSpringConfig` (damping 10, stiffness 100, mass 1: ratio 0.5, the old bounce), or your own. Ratio = damping / (2·√(stiffness·mass)).
- **`useSharedValue` cancels its own animation on unmount.** Call `cancelAnimation` in a cleanup only for values the component doesn't own: a parent's or a context's shared value, or a module-level `makeMutable`.
- **`useScrollViewOffset`** is a deprecated alias of `useScrollOffset`.

## Lists

The pattern that survives production: **the list renders once, items animate themselves.**

- The scroll offset is a shared value (`useScrollOffset` or `useAnimatedScrollHandler`), passed to items via props or context. Shared values are stable references; passing one does not re-render.
- Each item computes its own style from `(offset, index)` inside `useAnimatedStyle`. No `onViewableItemsChanged` + state, no per-frame props.
- `renderItem` returns a real component (so hooks live per item), memoized, with a stable `keyExtractor`.
- Entering animations on list items: keep them short and few. Hundreds of springs mounting during a fast scroll is a self-inflicted stress test; the good examples animate only the first visible title, or cap animated filter results at 10.

## Flags and setup worth knowing

- **120fps on iOS**: add `CADisableMinimumFrameDurationOnPhone: true` to `Info.plist`, or ProMotion devices cap at 60.
- **Flickering sticky headers / animated views while scrolling** (New Architecture): RN 0.81+, enable `preventShadowTreeCommitExhaustion` (RN) and `DISABLE_COMMIT_PAUSING_MECHANISM` (Reanimated).
- **FPS drops with many animated components during scroll**: `USE_COMMIT_HOOK_ONLY_FOR_REACT_COMMITS` (RN 0.80+, Reanimated 4.2+). It is already on in 4.5.1's defaults; on earlier 4.x, enable it.
- **Many simultaneous animations**: `ANDROID_SYNCHRONOUSLY_UPDATE_UI_PROPS` / `IOS_SYNCHRONOUSLY_UPDATE_UI_PROPS` (may affect touch on transformed elements; prefer Gesture Handler's `Pressable` then).
- **Android profiling**: use the `debugOptimized` variant during development for closer-to-real numbers.

## Accessibility is a performance feature

`useReducedMotion()` (read at app start) and `ReducedMotionConfig mode={ReduceMotion.System}` near the root. Decorative motion (parallax, staggers, infinite loops) should disappear; state changes should become instant, not absent. On low-end devices this doubles as a graceful degradation path.
