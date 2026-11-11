# Gesture-Driven Animation

Every gesture animation has two phases, and they obey different laws:

- **The follow phase**: while the finger is down, the gesture writes the driver directly, frame by frame. The finger must never lag, so nothing time-based runs here. Everything else (rotation, badges, the next card waking up) is coupling: interpolations and multiplies of the live gesture value.
- **The release phase**: when the finger lifts, physics takes over, seeded with the finger's velocity, and any *consequence* (dismiss, snap, promote) is a **guarded commit**: a decision, not a reflex.

Any timing-based animation while the finger is down feels laggy; any release without velocity feels dead; any commit that ignores how the finger was moving fires when the user had already changed their mind.

Examples use the Gesture Handler 2 builder API (`Gesture.Pan()`, always inside `useMemo`) with Reanimated 4 and `react-native-worklets`. `GestureHandlerRootView` must wrap the app root or `GestureDetector` crashes. On a newer Gesture Handler major, check its docs for the equivalents.

## The callback lifecycle: where each job goes

| Callback | Runs | Put here |
|---|---|---|
| `onBegin` | finger down, before activation, **also for taps** | catch (`cancelAnimation`), capture the start value, press feedback |
| `onStart` | on activation (past `activeOffset` / `minDistance`) | "dragging" flags |
| `onUpdate` / `onChange` | every move | the follow: write the driver (`onChange` adds `changeX/Y`, the delta since the last event) |
| `onEnd` | release, **only if it activated** | rarely anything |
| `onFinalize` | always, last, with a `success` flag | **settle**: snap, commit, spring home |

Two rules fall out of this table:

- **Settle in `onFinalize`, and only there.** A tap that catches a moving card in `onBegin` never activates, so it never reaches `onEnd`: a settle there leaves the card frozen mid-flight. Settling in both makes two springs fight over one value.
- **`onBegin` with `e.x` makes a tap jump.** `e.x` is the finger's position in the view, so writing it on begin moves the value under the finger on touch: exactly right for a scrubber or a ruler, a bug for a card, which should only ever move by `translation`.

## The offset accumulation pattern (follow phase)

Every draggable thing (cards, sheets, sliders, reorder handles) is this pattern:

```tsx
const offset = useSharedValue(0);
const start = useSharedValue(0);

const pan = useMemo(() =>
  Gesture.Pan()
    .onBegin(() => {
      cancelAnimation(offset);    // catch it mid-flight
      start.value = offset.value; // and pick it up exactly there
    })
    .onUpdate((e) => {
      offset.value = start.value + e.translationY; // the finger owns the value
    })
    .onFinalize((e) => {
      offset.value = withSpring(0, { velocity: e.velocityY, damping: 20, stiffness: 200, mass: 1 });
    }),
[]);
```

Capturing in `onBegin` is what makes the gesture **interruptible**: grabbing a card mid-spring picks it up exactly where it is, no jump. This detail separates native-feeling from janky more than anything else.

## Release springs in Reanimated 4

A bare `withSpring(x)` in Reanimated 4 uses `GentleSpringConfig` (mass 4, damping 120, stiffness 900): damping ratio 1, **no overshoot**. Code ported from v3 that relied on the old bouncy default (`Reanimated3DefaultSpringConfig`, ratio 0.5) goes flat. Pass a config whenever you want the release to overshoot.

```tsx
// damping ratio = damping / (2 * sqrt(stiffness * mass))
withSpring(dest);                                                        // 1.0: settles, never passes the target
withSpring(dest, { velocity: v, damping: 20, stiffness: 200, mass: 1 }); // ~0.7: one small, honest overshoot
withSpring(dest, { damping: 45, stiffness: 200, mass: 1 });              // ~1.6: overdamped glide (deck promotion)
```

**Always pass `mass` with `damping`/`stiffness`.** Your config merges over the gentle default, mass 4 included: `{ damping: 20, stiffness: 200 }` alone is ratio 0.35 in v4 (it was 0.71 in v3, where mass defaulted to 1), twice as slow and visibly wobbly. Presets are exported too: `WigglySpringConfig` (0.75), `SnappySpringConfig` (0.92 with `overshootClamping`). The gentle default is right for tracking and snapping to ticks; a sheet or a thrown card wants a little life.

## Coupling in the follow phase

One pan value, many consumers (the snippet lives in `references/core-concepts.md`). The swipe-deck recipe (tinder-swipe-cards-react-native-reanimated) fans a single `translationX` out to four: the card follows 1:1 with a 0.25 damped vertical follow and a 0.06°/px bank; the LIKE badge rides the card's `translateX` and fades in over `[30, 130]` px; the next card scales 0.94 → 1 and rises 14px over the first 170px of `|x|`, so the stack breathes *before* release. Nothing here is stateful: the whole follow phase is pure functions of the live gesture, which is why dragging it back and forth feels right at any speed.

## The guarded commit (release phase)

How strict the commit is depends on what it costs to be wrong:

- **Low stakes (decks, pagers, sheets you can reopen): distance OR velocity.** The Tinder deck commits on `|vx| > 500 || |tx| > W / 4`: a flick commits, and so does a slow, deliberate drag past a quarter of the screen.
- **Destructive (delete, discard, a dismissal that loses input): distance AND velocity.** Far enough, *and* still moving outward at release. Drag a row two thirds across, hold while you think, release without moving: that is a changed mind, not a delete. `Math.abs(tx) > W / 2 && vx * Math.sign(tx) > 100` (100 px/s is a starting point: barely moving, but not a resting finger's jitter).

Direction comes from where the finger went, not from velocity alone: a slow committed drag released with a tiny backward twitch would otherwise fly the wrong way, or spring home while the index advances.

```tsx
.onFinalize((e) => {
  const tx = e.translationX, vx = e.velocityX;
  const commit = Math.abs(vx) > 500 || Math.abs(tx) > W / 4;
  if (!commit) {
    x.value = withSpring(0, { velocity: vx });
    y.value = withSpring(0, { velocity: e.velocityY });
    return;
  }
  const dir = Math.abs(vx) > 500 ? Math.sign(vx) : Math.sign(tx); // a hard flick wins, else the drag
  x.value = withSpring(dir * W * 2, { velocity: vx });           // seeded: no speed change at handoff
  y.value = withDecay({ velocity: e.velocityY });                // the throw keeps its vertical momentum
  scheduleOnRN(onSwiped, dir);                                   // discrete event: cross threads once
})
```

In 2D (a grid you swipe one cell at a time), **lock the dominant axis first** (`Math.abs(tx) > Math.abs(ty)`), then apply the same rule on that axis only.

Related rule from the swipe-to-dismiss-sheet-state-machine-react-native recipe: **the drag and the dismissal are different properties.** The pan drives the live `translateY`, back to zero on release; the committed dismissal is its own motion. Making them the same value makes them fight.

Seen in: https://www.animatereactnative.com/post/tinder-swiper-animation-reanimated-+-gesture-handler · https://www.animatereactnative.com/post/2d-scrollable-grid-reanimated2-+-gesture-handler

## Snap points (sheets, wheels, sliders with detents)

Project the release velocity forward, then snap to the nearest point, in one spring:

```tsx
function snapTo(value: number, velocity: number, points: number[], factor = 0.2) {
  'worklet';
  const projected = value + velocity * factor; // where the fling is heading
  return points.reduce((a, b) =>
    Math.abs(b - projected) < Math.abs(a - projected) ? b : a);
}

.onFinalize((e) => {
  const dest = snapTo(offset.value, e.velocityY, [0, MID, OPEN]);
  offset.value = withSpring(dest, { velocity: e.velocityY, damping: 20, stiffness: 200, mass: 1 });
})
```

The projection factor is a feel choice, from **0.2 (stickier)** to **0.5 (looser)**:

- **0.2** keeps flicks from skipping a detent. Use it for sheets and pagers, where skipping a stop is a surprise.
- **0.5** is the classic scroll-view coast (the tarot ring uses about that), so a flick can travel several detents. Use it where travel should feel free: wheels, rings, free-scroll carousels. Ring and wheel snapping (unwrapped angle, wrapped display) is in `references/animation-math.md` (Rings).

Don't calibrate against `withDecay`: its slowdown compounds per frame, so at the default deceleration 0.998 it coasts about 0.35·v at 60 Hz and 0.25·v at 120 Hz, not one fixed distance.

Projecting with velocity (not just position) is why a flick from the top still closes the sheet. Prefer one spring to the projected point over a decay followed by a snap: two chained motions stutter at the handoff. Use `withDecay` alone for free coasting with no detents (maps, canvases); `clamp: [min, max]` with `rubberBandEffect: true` gives it soft edges (`rubberBandFactor` 0.6 by default).

Seen in: https://www.animatereactnative.com/post/tarot-cards-animation-reanimated-3-+-gestures-+-math

## Index space for gestures: tap-to-jump sliders and rulers

Scroll animations normalize to index space (`scrollX / ITEM`); gestures can too. A tick slider stores the finger position in px and every tick reads `pos / PITCH` as a float index:

```tsx
const PITCH = TICK_W + GAP;      // 2 + 4: one detent
const width = useSharedValue(0); // onLayout={(e) => { width.value = e.nativeEvent.layout.width; }}
const pos = useSharedValue(0);

const pan = useMemo(() => Gesture.Pan()
  .activeOffsetX([-1, 1])                                      // activate on the first pixel
  .onBegin((e) => { pos.value = clamp(e.x, 0, width.value); }) // a tap jumps (wanted here)
  .onChange((e) => { pos.value = clamp(e.x, 0, width.value); })
  .onFinalize(() => {                                          // taps land here too
    const i = Math.round(pos.value / PITCH);
    pos.value = withSpring(i * PITCH);                         // gentle default: no overshoot past a tick
    scheduleOnRN(onValueChange, i);                            // once per gesture
  }), []);
```

Width comes from `onLayout` into a shared value, read in the worklet: never `measure()` every frame. Each tick then magnifies itself off the same float, a five-point fisheye:

```tsx
const idx = pos.value / PITCH; // inside tick t's useAnimatedStyle
opacity: interpolate(idx, [t - 4, t, t + 4], [0.3, 1, 0.3], Extrapolation.CLAMP),
transform: [{ scaleY: interpolate(idx, [t - 3, t - 1, t, t + 1, t + 3], [1, 1.5, 2, 1.5, 1], Extrapolation.CLAMP) }],
// static tick style: height 12, transformOrigin: 'bottom center' (grows up from its baseline)
```

Peak 2x on a 12px tick is enough; the label above uses `fontVariant: ['tabular-nums']` so digits don't jitter in width. Report and display the same `Math.round`: a floored callback and a rounded label disagree at 2.9999.

The scroll version (animated ruler) is the same index space on a list: 8px pitch, 61 ticks, `snapToInterval={8}`, `paddingHorizontal: W / 2 - 4` so tick 0 starts under a fixed needle drawn outside the list, and neighbors scale only 0.98. Clamp the index to `[0, n - 1]`: iOS bounce reports offsets past both ends.

Seen in: https://www.animatereactnative.com/post/slider-with-fancy-animated-text · https://www.animatereactnative.com/post/react-native-animated-ruler-reanimated

## Decks: one float index, extrapolation is the stack

A card stack is one float `index` and N cards, each interpolating its own window over it. Commits move an **integer target**; the float chases it:

```tsx
// in the commit branch of the gesture (a worklet)
target.value = Math.min(target.value + 1, N - 1); // integer, never index.value + 1
index.value = withSpring(target.value, { damping: 45, stiffness: 200, mass: 1 });
```

Retargeting from the in-flight float (`withTiming(index.value + 1)` while it's at 0.6) rests at 1.6, and every card sits between poses forever. The overdamped spring (ratio ~1.6) lets the promotion glide in while the thrown card is still flying; the stack-cards version uses `withTiming(target, { duration: 300 })`.

```tsx
const stylez = useAnimatedStyle(() => {
  const depth = i - index.value; // 0 = front, 1 = next back, ...
  return {
    opacity: interpolate(depth, [0, 6], [1, 0], Extrapolation.CLAMP), // 1 - k/6: gone 6 deep
    transform: [
      // EXTEND on purpose: depth k sits at -22k, a passed card drops H - 44
      { translateY: interpolate(index.value, [i - 1, i, i + 1], [-22, 0, H - 44]) },
      { scale: interpolate(depth, [0, 10], [1, 0.5], Extrapolation.CLAMP) }, // 1 - 0.05k, floor 0.5
    ],
  };
});
```

This is the deliberate exception to "clamp by default": one three-point window, extended, *is* the whole stack, every depth. Every unclamped channel still needs a floor so it can't cross zero (an extrapolated scale past 0 mirrors the card).

The Tinder deck fans instead of stacking: a four-point window gives alternating peeks, so one float poses three cards:

```tsx
const r = [i - 3, i - 2, i - 1, i]; // three back (hidden), two back, one back, front
const stylez = useAnimatedStyle(() => ({
  opacity: interpolate(index.value, r, [0, 0.8, 0.8, 1], Extrapolation.CLAMP),
  transform: [
    { translateX: interpolate(index.value, r, [0, SIZE / 2, -SIZE / 2, 0], Extrapolation.CLAMP) },
    { rotate: `${interpolate(index.value, r, [0, 20, -20, 0], Extrapolation.CLAMP)}deg` },
  ],
}));
```

`SIZE` is 0.6 of the screen width; the front card adds the drag's `x` on top. Mount only four (`cards.slice(active, active + 4)`, keyed by id) and update React's `active` only when `Math.round(index)` flips (see Side effects below).

Seen in: https://www.animatereactnative.com/post/react-native-stack-cards-animation · https://www.animatereactnative.com/post/tinder-swiper-animation-reanimated-+-gesture-handler

## Counter-translated clip window (before/after compare)

Don't resize two images with `flex` or `width`: that's a layout pass per frame, and a `cover` image re-crops as its box changes, so it visibly shifts. Slide a clipping window and counter-slide its content by the same amount:

```tsx
const x = useSharedValue(W / 2); // handle position, 0..W
const pan = useMemo(() => Gesture.Pan()
  .onChange((e) => { x.value = clamp(x.value + e.changeX, 0, W); }), []);

const windowStylez = useAnimatedStyle(() => ({ transform: [{ translateX: x.value - W }] })); // W wide, overflow: 'hidden'
const imageStylez = useAnimatedStyle(() => ({ transform: [{ translateX: W - x.value }] }));  // net 0: the image never moves
const handleStylez = useAnimatedStyle(() => ({ transform: [{ translateX: x.value }] }));
```

Three transforms, zero layout. The handle is a 60px knob on a 6px column: attach the gesture to the column and widen it with `.hitSlop({ horizontal: 27 })` (6 + 2·27 = 60), because on Android a knob that overflows its parent's bounds doesn't receive touches.

Seen in: https://www.animatereactnative.com/post/slide-to-compare-image-reanimated2-and-gesturehandler

## Axis unlock and remaining runway (pull-down pickers)

A pull-down menu with a horizontal picker inside one gesture (drag tasks): the vertical pull opens it, and the horizontal axis stays **locked until the pull is 70% done**, so a slightly diagonal pull doesn't scrub the picker. Then map X to the runway the finger actually has, so the last option is reachable wherever the finger landed:

```tsx
.onBegin((e) => { startX.value = e.absoluteX; })
.onChange((e) => {
  y.value = clamp(e.translationY, 0, MAX_Y);              // MAX_Y = 220
  if (y.value < MAX_Y * 0.7) return;                      // X ignored until the pull is mostly done
  const runway = Math.max(W * 0.8 - startX.value, SLOT);  // room right of the finger
  x.value = interpolate(e.translationX, [0, runway], [0, (N - 1) * SLOT], Extrapolation.CLAMP);
})
.onFinalize(() => {
  if (y.value >= MAX_Y) scheduleOnRN(onPick, Math.round(x.value / SLOT));
  y.value = withSpring(0);
})
```

Position-only commit is fine here: 220px of deliberate pull is the intent, and picking is reversible. The menu reveals at half speed (`translateY` over `[-MAX_Y / 2, 0]`) and fades in over the first half of the pull. The header uses `interpolateColor(y, [0, MAX_Y], [colors[picked], colors[previous]])`: both are the same color until a pick, and at release `previous` is the old pick, so nothing jumps and the spring back to 0 *is* the color change. The closing animation and the confirmation are one motion.

Seen in: https://www.animatereactnative.com/post/drag-tasks-react-native-reanimated2-animation

## Rubber-banding (resistance past the edge)

Dragging past a boundary should move less the further you pull. Interpolate the overshoot instead of applying it raw:

```tsx
.onUpdate((e) => {
  const raw = start.value + e.translationY;
  offset.value = raw < 0
    ? interpolate(raw, [-300, 0], [-90, 0], Extrapolation.CLAMP) // 300px of pull → 90px of movement
    : raw;
})
```

The same shape gives pull-to-refresh its feel.

## Sensors are gestures too

`useAnimatedSensor` writes a shared value on the UI thread, so device tilt plugs into the same pipeline as a finger. Two shapes from the catalog:

**Gated tilt**: the sensor is the *peak* of a per-item window, so only the focused card listens and neighbors ignore the phone:

```tsx
const { sensor } = useAnimatedSensor(SensorType.ROTATION, { interval: 20 }); // 50 Hz, radians

const stylez = useAnimatedStyle(() => {
  const focus = interpolate(scrollX.value, [i - 1, i, i + 1], [0, 1, 0], Extrapolation.CLAMP);
  const roll = clamp(sensor.value.roll, -Math.PI / 4, Math.PI / 4) * focus; // cap at 45°
  return {
    transform: [{ perspective: 1000 }, { translateX: roll * 50 }, { rotateY: `${roll}rad` }],
  };
});
```

Pitch gets a 40° offset before the same treatment: that's the angle people hold a phone at rest, so it has to read as neutral. Neighbors stay calm at scale 0.9 with a ±15° `rotateZ`.

**Tilt as velocity** (balloon slider): the angle sets a *speed*, integrated per frame, with a 5° dead zone so a resting hand doesn't drift, and the finger outranks the phone:

```tsx
useFrameCallback((f) => {
  if (dragging.value) return;                  // set in the pan's onBegin, cleared in onFinalize
  const deg = (sensor.value.roll * 180) / Math.PI;
  if (Math.abs(deg) < 5) return;               // dead zone
  const dt = f.timeSincePreviousFrame ?? 0;
  x.value = clamp(x.value + deg * 0.2 * (dt / 100), 0, W); // 0.2 px per degree per 100ms
});
```

Integrating with `timeSincePreviousFrame` keeps the speed identical at 60 and 120 Hz; a fixed step per frame runs twice as fast on ProMotion.

Seen in: https://www.animatereactnative.com/post/parallax-carousel-reanimated-and-sensors · https://www.animatereactnative.com/post/ballon-slider-reanimated-gestures-sensors-and-math

## External data: springs as a low-pass filter

Camera detections (a QR bounding box) arrive at an irregular rate and jitter by a few pixels. Don't write them raw: **retarget a spring per edge on every detection**. A retargeted spring keeps its current velocity, so the box glides between readings and the noise is filtered out:

```tsx
function onDetected(b: Box) { // call from wherever detections arrive
  x.value = withSpring(b.x - b.width * 0.05);  // 10% padding, 5% per side
  y.value = withSpring(b.y - b.height * 0.05);
  w.value = withSpring(b.width * 1.1);
  h.value = withSpring(b.height * 1.1);
  alpha.value = withSequence(
    withSpring(1, { duration: 300 }),
    withDelay(200, withTiming(0, { duration: 300 })), // keep-alive: every scan restarts it
  );
}
```

The bare v4 spring is exactly right here: a tracking box that overshoots looks like it lost the target. The opacity sequence makes the box visible while codes keep arriving and gone 200ms + 300ms after the last one, with no timers. Draw the box as four fixed-size corner brackets that only translate (`x + (k === 1 || k === 2 ? w : 0)`, `y + (k >= 2 ? h : 0)`, static rotation `k · 90°`), not as one view animating `left/top/width/height/borderRadius`: corners move, nothing re-lays out. The decoded label is keyed by its data with `FadeInDown.springify().delay(200)`, so a new code swaps it.

Seen in: https://www.animatereactnative.com/post/qrcode-scanner-bounding-box

## Micro-interactions on gestures

**One press gate drives every press channel.** `pressed.value = withSpring(1)` in `onBegin`, `withSpring(0)` in `onFinalize`. On the balloon slider that one gate drives five properties: the knob opens from a dot into a ring and doubles (scale 1 → 2), and the balloon fades in, scales up, and rises 48px. The gentle default never overshoots, so the same gate is safe for opacity; with a bouncy config, clamp it. The knob is 24px on a 5px track with `hitSlop` 25.

**A haptic per detent**, not per frame: `Haptics.selectionAsync()` fired from a reaction on the rounded index (custom picker), so each tick of the wheel clicks exactly once.

**Celebrate on landing, not on release** (confetti drag): fire only past a real drag (`max(|x|, |y|) > 40`), return the two axes with different durations so the path home curves, and trigger the burst from the completion callback (its origin comes from one `measure()` in `onBegin`, not per frame):

```tsx
.onFinalize(() => {
  const fire = Math.max(Math.abs(x.value), Math.abs(y.value)) > 40;
  y.value = withTiming(0, { duration: 500, easing: Easing.inOut(Easing.exp) });
  x.value = withTiming(0, { duration: 400, easing: Easing.inOut(Easing.exp) }, (finished) => {
    if (finished && fire) scheduleOnRN(celebrate); // the confetti lands with the object
  });
})
```

**Focus after travel** (2D grid): when a swipe lands on a new cell (300ms), every other cell dims to 0.4 immediately and the target fades up with `withDelay(200, withTiming(1, { duration: 300 }))`; active index is `cols * row + col`. **Pop, don't bounce** (drag tasks): a picked swatch scales to 1.1 over 200ms while its label rises 2px. **Hold-to-confirm** (hold-to-confirm-button-react-native-reanimated recipe): the press drives a progress driver up over time and an early release springs it back, so the commit is the driver reaching 1, guarded by "still holding".

Seen in: https://www.animatereactnative.com/post/react-native-confetti-animation · https://www.animatereactnative.com/post/custom-react-native-picker-reanimated

## Side effects: reactions, callbacks, the JS thread

Gesture and animation callbacks are worklets on the UI thread. Mutating shared values is safe; anything else (setState, navigation, haptics, analytics) goes through `scheduleOnRN(fn, ...args)` from `react-native-worklets` (`runOnJS` still exists in Reanimated 4, deprecated), on discrete transitions only, never in `onUpdate`.

```tsx
useAnimatedReaction(
  () => Math.round(index.value),
  (i, prev) => {
    if (prev !== null && i !== prev) scheduleOnRN(setActive, i); // once per crossing
  },
);
```

- **Side effects go in `useAnimatedReaction`, not `useDerivedValue`.** A derived value is for computing values; the reaction hands you `(current, previous)` so you fire once per change (and `prev` is `null` on the first run).
- **Never compare against React state inside a worklet.** The worklet captured `active` when it was built; `if (i !== active) scheduleOnRN(setActive, i)` sees a stale copy and fires every frame.
- **Animation callbacks check `finished`.** They also run when the animation is cancelled, and a re-grab cancels it. The marquee stops its clock in `onBegin`, scrubs while dragging, coasts on release, and restarts only if the coast completed:

```tsx
pos.value = withDecay({ velocity: -e.velocityX }, (finished) => {
  if (!finished) return; // caught mid-coast: the finger owns it, don't snap or restart under it
  running.value = true;
});
```

- **One spring, not two chained.** A spring follower feeding another spring adds lag on lag; derive from one.
- Taps are `Gesture.Tap()` or Gesture Handler's `Pressable`, never `onTouchStart`: a tap must cancel when the user starts scrolling instead.

Seen in: https://www.animatereactnative.com/post/ai-image-playground-react-native-marquee

## Gestures inside scroll views

A horizontal pan inside a vertical `ScrollView` (or the reverse) must declare its axis, or the scroll and the pan fight:

```tsx
Gesture.Pan()
  .activeOffsetX([-10, 10]) // activate after 10px horizontally
  .failOffsetY([-10, 10])   // give up if the user is scrolling vertically
```

Import `ScrollView`/`FlatList` from `react-native-gesture-handler` when composing with gestures, and never mix core RN touch handlers (`TouchableOpacity`, `onPress` on plain views) with Gesture Handler in the same subtree; pick one system. When a pan and a scroll must both stay live (a sheet over its own list), wrap the scroll view in `Gesture.Native()`, pass it to `pan.simultaneousWithExternalGesture(native)`, and decide in the pan's `onChange` who moves, from the scroll's shared offset.

**Nested scroll locks.** A horizontal pager of vertical pages (the expandable carousel: scroll a card's content and it grows edge to edge) must not page sideways while a page is scrolled. Derive the lock from the inner offset on the UI thread:

```tsx
const pagerProps = useAnimatedProps(() => ({ scrollEnabled: innerY.value <= 0 })); // innerY: the open page's offset
<Animated.ScrollView horizontal pagingEnabled animatedProps={pagerProps}>
```

`<= 0`, not `=== 0`: iOS bounce takes the offset negative, and an exact-zero test stays locked after every overscroll. Never `scrollEnabled={innerY.value === 0}` in JSX: that reads `.value` during render, once, and never updates. The close button scrolls the page back to 0 (`scrollTo(innerRef, 0, 0, true)`), which unlocks the pager by the same rule. Seen in: https://www.animatereactnative.com/post/expandable-carousel

For "drag one item of a list" (reorder, swipe actions), prefer the built-ins first: `ReanimatedSwipeable` covers swipeable rows; hand-rolled pans are for behavior the built-ins cannot express.

## Composition (v2)

`Gesture.Simultaneous(pinch, rotate)` runs both at once (photo viewer); `Gesture.Race(pan, tap)` lets the first to activate win; `Gesture.Exclusive(double, single)` sets priority (double-tap beats single-tap). Always `useMemo` v2 gesture objects: recreating them per render resets recognizer state mid-gesture.

## Restraint, in numbers

The catalog's gesture pieces feel rich because each one moves very little:

- Ruler: neighbors scale 0.98. Tick magnifier: 2x peak, five ticks wide.
- Stack: one float, 22px per step, 0.05 scale, 1/6 opacity.
- Tinder: one float poses three cards; ±20° is the only rotation, peeks at 0.8 opacity.
- Balloon: one spring gate, five properties.
- Sensors: only the focused card listens, tilt capped at 45°, neighbors at 0.9.
- Swatch: 1.1 pop, 2px label rise.

## Real-world examples

Interactive versions with live previews and full generated code, at keyframer.dev/recipes/:

- Follow-phase coupling: `tinder-swipe-cards-react-native-reanimated`
- Guarded commit as a state machine: `swipe-to-dismiss-sheet-state-machine-react-native`
- Progress-as-commitment: `hold-to-confirm-button-react-native-reanimated`
- Drag with detents: `drag-slider-react-native`

The animatereactnative.com posts cited in each section above are the complete, polished versions.
