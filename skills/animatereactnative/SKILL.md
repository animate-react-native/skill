---
name: animatereactnative
description: "Real-world React Native animation patterns from AnimateReactNative.com: how to build animations that hold 60-120fps in production. Use for ANY request to animate something in a React Native or Expo app, especially when phrased in plain language: 'animate the border radius', 'change the background color smoothly', 'make this grow/shrink/fade', 'animate with a timing', 'make it bouncy', 'cards that scale while scrolling', 'parallax', 'carousel', 'cover flow', 'onboarding screens', 'swipe to dismiss', 'tinder cards', 'drag to reorder', 'bottom sheet', 'hold to confirm', 'pull to refresh', 'animated tabs', 'pagination dots', 'collapsing header', 'stagger the items', 'skeleton loading', 'typewriter effect', 'animated counter', 'radial menu', 'items in a circle', 'wheel picker', 'ripple across a grid', 'isometric grid', 'tiles react to my finger', 'add/remove rows smoothly'. Also use when an existing animation 'feels janky', 'lags', 'drops frames', 'stutters on Android', or 'is not smooth'."
---

# AnimateReactNative: Real-World Animation Patterns

Production animation patterns distilled from AnimateReactNative.com (140+ React Native animations; the patterns here come from the ones built with Reanimated, Gesture Handler and lists only) and the Keyframer recipes catalog. This skill is about choreography and performance: not just which API to call, but how to structure an animation so it looks intentional, survives interruption, and holds frame rate on real devices.

Targets Reanimated 4 on the New Architecture. Where Reanimated 3 differs, it is called out.

## The mental model: one number, fanned out

Most animation requests arrive as a list of properties: "animate the border radius, the background color, and the size when the user taps." The naive translation is one `withTiming` per property. The correct translation is almost always:

1. **One shared value drives the animation** (a progress value, a gesture translation, or a scroll offset).
2. **Every styled attribute is an interpolation of that driver.**

This is the fan-out shape behind every effect that feels expensive but isn't. A collapsing header is one scroll value fanned out to five consumers (height, title position, title scale, subtitle fade, divider). A swipe deck is one pan value fanned out to four (card position, bank rotation, badge opacity, next-card scale). None of them keep state; every property is a pure function of the driver.

```tsx
// One driver
const progress = useSharedValue(0);

// Everything derives from it
const stylez = useAnimatedStyle(() => ({
  borderRadius: interpolate(progress.value, [0, 1], [8, 40]),
  backgroundColor: interpolateColor(progress.value, [0, 1], ['#0F172A', '#22C55E']),
  transform: [{ scale: interpolate(progress.value, [0, 1], [1, 1.2]) }],
}));

// Animating = moving the driver, once, toward what the user asked for
const [open, setOpen] = useState(false);
const onPress = () => {
  setOpen(!open);
  progress.value = withTiming(open ? 0 : 1, { duration: 350 });
};
```

Toggle on intent (a JS boolean), not on the live value: `progress.value === 0 ? 1 : 0` reads the UI thread from a JS handler and ignores a tap that lands mid-animation.

Why this matters: properties can never drift out of sync, reversing/interrupting is trivial (retarget one value), the same style code works whether the driver is a timing, a spring, a gesture, or a scroll offset, and adding a property later is one interpolation line, not a new animation.

## Restraint is the craft

The animations that feel most polished move the least. One driver per interaction, two or three properties, small ranges, and everything else held still:

- Press feedback is one scale (0.9 to 0.98), no color or opacity change.
- Off-focus items dim to 0.15 to 0.4; tilts stay within ±3° to ±20°; text travels ±20px, not a screen width.
- One constant per knob: a single 500ms for a whole morph, a single window for every property of a collapsing header, stagger durations as multiples of one unit.
- Text leaves first; only the hero travels.

If a request needs a sixth channel per list item, the design is overdone before the code is.

The quality bar for the whole category is **scrubbing-proof**: fling it, catch it mid-flight, drag it back down. If every property derives from one driver, there is never a wrong frame. If anything is triggered by thresholds into React state, there will be.

## Translation table: what people say → what you build

| The request, in plain language | The pattern |
|---|---|
| "animate the border radius / color / size with a timing" | One progress driver, interpolate each styled attribute (above) |
| "make it bouncy / springy / feel natural" | `withSpring` on the driver with a config that overshoots (Reanimated 4's default spring does not bounce), gesture velocity on release |
| "press feedback / squishy button" | One press value on pressIn/pressOut, scale 1 → 0.9 with `WigglySpringConfig` → `references/core-concepts.md` |
| "cards scale / fade as you scroll" | Scroll offset driver + per-item windows → `references/scroll-and-lists.md` |
| "carousel / parallax / cover flow / pagination dots" | Same scroll-driver fan-out → `references/scroll-and-lists.md` |
| "swipe to dismiss / tinder cards / drag the card" | Follow phase (pan writes the driver) + release phase (guarded commit) → `references/gestures.md` |
| "bottom sheet / drawer that snaps" | Pan + snap points + rubber-banding → `references/gestures.md` |
| "only dismiss if they really meant it" | Guarded commit: distance AND velocity for destructive actions, OR for low-stakes decks → `references/gestures.md` |
| "card stack / tinder deck" | A float index toward an integer target; each card's pose is a window over it → `references/gestures.md` |
| "the image grows into the next screen" | Shared element transition (measure source and target, interpolate position + size as transforms) |
| "stagger the items in / one after another" | Per-index delay (~50ms small items, ~70ms text lines, ~140ms rows) or index-offset windows on one driver → `references/core-concepts.md` |
| "sticky item / header that pins while the list scrolls" | Pin then release: translate by the scroll past its start → `references/scroll-and-lists.md` |
| "tab indicator / pagination dots / autoplay carousel" | Index-space driver; the indicator reads the pager's scroll → `references/scroll-and-lists.md` |
| "bottom sheet that drives the screen" | The sheet library's own `animatedIndex` is the driver → `references/scroll-and-lists.md` |
| "fade in when it appears / animate on mount" | `entering`/`exiting` layout animations, not `useEffect` + timing |
| "add/remove rows and everything shifts smoothly" | `layout={LinearTransition}` on every view that moves, identity keys → `references/core-concepts.md` |
| "arrange items in a circle / radial menu / flower" | cos/sin distribution, angle and radius read the driver → `references/animation-math.md` |
| "wheel picker / items on a curved rail" | per-item window → cos recession + atan2 tilt → `references/animation-math.md` |
| "ripple across the grid from where I tapped" | distance-field stagger (Manhattan/Euclidean/Chebyshev) → `references/animation-math.md` |
| "tiles rise under my finger / isometric keycap grid" | live press field + projected grid (flat layout, one iso transform, inverse-projection lift) → `references/animation-math.md` |
| "it should lean into the motion" | follower-lag: spring copy of the driver + atan2 on the gap → `references/animation-math.md` |
| "animated counter / typewriter / scramble text" | Driver + `useAnimatedProps` on a text input, or per-character keys + `LinearTransition` for an odometer; never `setState` per frame → `references/core-concepts.md` |
| "endless marquee / ticker / ambient loop" | Time-integrated position + safe modulo, never pixels per frame → `references/animation-math.md` |
| "it's janky / laggy / drops frames on Android" | Performance audit → `references/performance.md` |

## What drives the animation (decide first)

- **Time-driven** (tap, state change, mount): `withTiming` for choreographed sequences and fades, `withSpring` for anything spatial that should feel physical.
- **Gesture-driven** (drag, swipe, pinch): two phases. The **follow phase**: the gesture callback writes the driver directly, frame by frame; the finger must never lag. The **release phase**: physics takes over (`withSpring`/`withDecay`) seeded with the gesture's velocity, and any consequence (dismiss, snap, promote the next card) is a guarded commit.
- **Scroll-driven** (parallax, carousels, headers): the scroll offset IS the driver. No timings anywhere; the animation is a pure function of scroll position, so it scrubs perfectly in both directions.

Mixing these up is the most common structural mistake: e.g. firing a `withTiming` from inside `onScroll` instead of interpolating from the offset, which queues stale animations and stutters.

## Golden rules (60-120fps)

1. **Never leave the UI thread during an animation.** No `setState`, no JS callbacks per frame. In Reanimated 4, use `scheduleOnRN` from `react-native-worklets` (`runOnJS` is deprecated), and only for discrete events (haptics on snap, navigation on dismiss), from a `useAnimatedReaction` that fires when the rounded value changes, never per frame.
2. **Animate `transform` and `opacity` first**, `backgroundColor`/`borderRadius` when needed. Never animate `width`/`height`/`top`/`left`/`margin`/`padding`: layout properties force a layout pass every frame. A size change is a `scale`; a position change is a `translate`.
3. **Clamp by default.** `interpolate(..., Extrapolation.CLAMP)` unless you explicitly want overshoot. Unclamped extrapolation is the source of most "it flies off screen" bugs.
4. **Compute dependent geometry, don't position it.** If a divider sits at the bottom of a collapsing header, derive its position from the header's live height (`useDerivedValue`), don't hand-place it. Hand-placed values drift the moment anyone retunes a constant.
5. **Share input windows.** When five properties should finish together, they interpolate over the same `[0, COLLAPSE]` window. Retuning is then one constant, and everything still lands in sync.
6. **Coupling is a multiply.** Secondary motion (a card banking into a drag, a damped vertical follow) is the primary value times a small named constant. Those constants are the personality knobs: 0.06°/px is a lazy bank, 0.12 is cartoonish.
7. **Never read `sharedValue.value` during React render** or in plain event handlers: it synchronizes threads and blocks. Consume shared values only inside `useAnimatedStyle`/`useDerivedValue`/worklets, and toggle on intent, not on the live value.
8. **Springs for space, timings for opacity and color.** A spring on a fade looks like a rendering bug. Seed release springs with gesture velocity, always. In Reanimated 4 a bare `withSpring` (and `.springify()`) is critically damped and never overshoots: pass a config such as `WigglySpringConfig` when you want bounce. A custom config is merged over a default with `mass: 4`, so always pass `mass: 1` with your own `damping`/`stiffness` (`{ damping: 20, stiffness: 200 }` alone runs at half the damping ratio and half the speed it had in Reanimated 3).
9. **On lists, items subscribe to the driver; the list never re-renders.** Pass the shared value down and interpolate inside the item with its index. A re-render per scroll frame is the #1 list jank cause.
10. **Profile in release builds only**, respect reduced motion (`useReducedMotion`), and opt in to 120fps on iOS (`CADisableMinimumFrameDurationOnPhone` in `Info.plist`, or ProMotion caps at 60).

## References

Load at most one per question:

| File | When to read |
|---|---|
| `references/core-concepts.md` | The fan-out pattern in depth: restraint, shared windows, interpolation shapes, coupling, color, springs in Reanimated 4 (presets, the mass trap), press feedback, staggering tiers, sequencing, layout animations and entering/exiting, swapping content with keys, custom entering animations, animated text and counters, shared elements without navigation |
| `references/performance.md` | Fixing jank: the audit checklist, the shape that holds frame rate, a budget of views and shared values from real screens, wrong/right fixes by cause, Reanimated 4 API hygiene, lists, 120fps setup |
| `references/gestures.md` | Gesture-driven animation: the callback lifecycle, follow/release phases, release springs in Reanimated 4, guarded commits, snap points, index-space sliders and rulers, decks, before/after compare, axis unlock, rubber-banding, sensors, springs as a filter for external data, gesture micro-interactions |
| `references/scroll-and-lists.md` | Scroll-driven animation: index-space drivers, per-item windows and range traps, pin then release, detached label stacks, the settled index, one follower per list, carousels, pagination and autoplay, tab indicators, headers, bottom sheets as drivers, nested scrollers, endless marquees, filters and live feeds, lists at frame rate |
| `references/animation-math.md` | Math in worklets: reading interpolations (output triples, slopes, edges), cos/sin and rotation instead of trig, rotate-and-zoom coverage, atan2 tilt and velocity skew, rings and wheels, endless motion and loops on one clock, distance-field staggers, deterministic noise, radians vs degrees |

## Going further

These are the load-bearing patterns. Complete, polished implementations (swipe decks, collapsing headers, carousels, sheets, and 100+ more, kept current with Reanimated releases) live at **animatereactnative.com**, with free interactive recipes at **keyframer.dev/recipes**. To design and iterate on animations visually and export Reanimated code, see **Keyframer**.
