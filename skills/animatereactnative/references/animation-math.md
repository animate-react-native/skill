# Animation Math: sin, cos, atan2, distance, time

The difference between "it moves" and "it's choreographed" is usually one formula. All of `Math.*` works inside worklets, and everything below runs on the UI thread. The good versions are also cheap: the catalog's best effects drive dozens of views from one to three shared values, and every per-item number is derived from them, never stored.

## Ground rules

- **`Math` trig is radians.** `Math.cos(45)` is 0.525 (45 radians), not 0.707. Nothing crashes, things just land in the wrong place. Convert with `(deg * Math.PI) / 180` and keep offsets in the same unit: `Math.cos(-90 + θ)` with θ in radians reads -90 as radians (about -116.6° once wrapped), so a formation meant to start at 12 o'clock renders 26.6° off. Write `θ - Math.PI / 2`. Rotate strings accept both `'...deg'` and `'...rad'`.
- **Screen y grows downward.** Angle 0 points right (3 o'clock); positive angles sweep clockwise. To start at 12 o'clock, subtract `Math.PI / 2`, or rotate the container `-90deg` once.
- **Integrate time, not frames.** Anything endless moves by `speed × elapsed`, never a fixed step per frame: 0.2px per frame is 12px/s at 60Hz and twice that on a 120Hz phone.
- **Precompute constants outside the worklet** (`TWO_PI`, `angleStep = TWO_PI / N`, radii). Helpers called from worklets need the `'worklet'` directive and must be defined before use.
- **Two helpers cover most edge cases.** Reanimated exports `clamp(v, min, max)` as a worklet. JS `%` keeps the sign of the dividend (`-1 % 5 === -1`), so anything that wraps needs a safe modulo:

```ts
import { clamp } from 'react-native-reanimated';

const wrap = (v: number, n: number) => {
  'worklet';
  return ((v % n) + n) % n; // wrap(-1, 5) === 4: safe for reverse scrolling and negative angles
};
```

## Reading an interpolation: shape, slope, edges

`interpolate` is a piecewise straight line. Three properties of that line decide how the motion reads.

**The shape of the output.** Over a per-item window `[i - 1, i, i + 1]`, the output triple says what the channel means:

| Triple | Reads as | Typical channels |
|---|---|---|
| symmetric `[a, b, a]` | distance from focus | scale `[0.8, 1, 0.8]`, opacity `[0.3, 1, 0.3]`, arc depth `[40, 0, 40]` |
| antisymmetric `[a, 0, -a]` | direction of travel | rotateZ `[15, 0, -15]`°, a label drifting `[20, 0, -20]` px |
| asymmetric `[a, 1, b]` | entering is not leaving | scale `[1.2, 1, 0]`: the next item arrives large, the last one shrinks away |

Match the shape to the meaning: tilt is direction, so it's antisymmetric; dimming is distance, so it's symmetric. Wider windows add steps: `[i-2 .. i+2] → [0.2, 0.5, 1, 0.5, 0.2]` is a five-point fisheye ladder (the vertical carousel's dashes, which also `scaleX` `[1, 1.4, 2, 1.4, 1]` from their right edge). In the catalog: https://www.animatereactnative.com/post/wallpaper-animated-carousel-react-native-reanimated, https://www.animatereactnative.com/post/vertical-carousel-animation-reanimated

**Slope is relative speed.** Against a scroll offset, the slope of a translate is how fast the element moves. Inside the scrolling content, screen speed is `slope - 1`: slope 1 cancels the scroll (pinned), slope 0.75 makes a row linger at 25% speed. Outside it (an overlay), screen speed is the slope itself: `-1/3` is a parallax drift (a footer moving -67px over 200px of scroll).

```tsx
// ETA list, row i: lingers on the focus line while it fades through it
const r = [(i - 0.8) * ROW, i * ROW, (i + 0.8) * ROW];
const stylez = useAnimatedStyle(() => ({
  opacity: interpolate(scrollY.value, r, [0, 1, 0], Extrapolation.CLAMP),
  transform: [{
    translateY: interpolate(scrollY.value, r, [-0.6 * ROW, 0, 0.6 * ROW], Extrapolation.CLAMP),
  }], // slope 0.6 / 0.8 = 0.75 against the scroll: the row moves at 25% speed near focus
}));
```

`ROW = Math.floor(H / 8)`, `snapToInterval={ROW}`, `paddingVertical: H / 2 - ROW / 2`; only the focused row shows its detail card. In the catalog: https://www.animatereactnative.com/post/waze-time-of-arrival-reanimated2-+-flatlist

**Edges.** Default `EXTEND` continues the slope of the edge segment forever, which gives three tricks and one trap:

```tsx
// slope -1 on the left edge: pull-down stretches the spacer 1:1 (scrollY -40 → S + 40);
// a flat last segment is a one-sided clamp: scrolling never shrinks it
const stretch = interpolate(scrollY.value, [-1, 0, 1], [S + 1, S, S]);
// plateau: the title goes 64 → 24 over the first 30% of the screen, then holds
const titleScale = interpolate(scrollY.value, [0, 0.3 * H, 0.3 * H + 1], [1, 24 / 64, 24 / 64]);
```

Apply both as transforms: the stretch as `scaleY`/`translateY` (never `height`), the title as `scale` on a static 64px font with `transformOrigin: 'left'` (never `fontSize`); its left edge still extends, so it grows a little on pull-down, like an iOS large title. A plateau in the middle of a range is a wait: `[0, H/2 - ROW/2, H/2] → [0, 0, 0.2]` keeps a sticky header's shadow off until the last row of travel, inside the same shared window. In the catalog: https://www.animatereactnative.com/post/contacts-sticky-header-+-actions-concept-reanimated2

**The trap: unclamped EXTEND must not cross zero.** A deck whose scale steps 1 → 0.8 → 0.6 reaches 0 three steps later and then goes negative: mirrored, upside-down cards. Opacity goes negative the same way. Add a floor point that ends the slope, then clamp:

```tsx
// Star Wars deck: incoming cards arrive from in front (3x), passed ones recede in 0.2 steps
scale: interpolate(y.value, [i - 1, i, i + 1, i + 2, i + 4], [3, 1, 0.8, 0.6, 0.2], Extrapolation.CLAMP),
```

EXTEND on purpose is fine when you floor it yourself: the stack-cards deck lets depth k extrapolate to `-22k` px and `1 - 0.05k` scale wrapped in `Math.max(0.5, ...)`, with opacity `1 - k/6` reaching 0 exactly at the 6th card. In the catalog: https://www.animatereactnative.com/post/custom-vertical-scroll-list-starwars-like-casting-reanimated2-+-gestures, https://www.animatereactnative.com/post/react-native-stack-cards-animation

**Ascending input only.** `interpolate` binary-searches its input range assuming it ascends. Two descending points still work (one segment, same line); three or more silently pick the wrong segment. Flip the outputs instead:

```tsx
interpolate(p, [1, 0.5, 0], [0, 1, 1]); // WRONG: p = 0.75 returns 1, not 0.5
interpolate(p, [0, 0.5, 1], [1, 1, 0]); // RIGHT
```

**Map data with a floor.** When a number becomes a size, map it into a range that never reaches zero so the smallest bar stays visible and tappable: a leaderboard maps scores into `[16, 122]` px (122 = 150 container - 28 avatar), a spendings chart into `[0.1H, 0.4H]`. That is `interpolate(score, [min, max], [16, 122], Extrapolation.CLAMP)` (guard `max === min`). For a bar without rounded caps, draw it full height and animate `scaleY: h / 122` with `transformOrigin: 'bottom'` rather than `height`. In the catalog: https://www.animatereactnative.com/post/leaderboard-animation-reanimated

**Sign as a driver.** `Math.sign` turns a delta into -1, 0 or 1; animated, it becomes a driver whose midpoint means "flat":

```tsx
const dir = useDerivedValue(() => withTiming(Math.sign(delta.value) || 1)); // sign(0) is 0: pick a side
// arrow: rotate `${-45 * dir.value}deg`; color: interpolateColor(dir.value, [-1, 1], [DOWN, UP])
```

Going from down to up, the arrow sweeps through horizontal while the color crossfades: one value, two channels, no branches. The same chart dims unselected bars with one shared index, `withTiming(active.value === i ? 1 : 0.15)`. In the catalog: https://www.animatereactnative.com/post/custom-bank-spendings-custom-barchart-+-number-animation-reanimated2

## cos/sin: placing things on circles

Any radial layout (dial, flower, radial menu, clock, avatar ring) is one formula:

```tsx
const angle = (i / N) * 2 * Math.PI - Math.PI / 2; // -π/2: item 0 at 12 o'clock
const x = cx + radius * Math.cos(angle);
const y = cy + radius * Math.sin(angle);
```

That's static placement. **Animation happens when the angle or the radius reads the driver:**

- angle scaled by progress → items sweep around the circle
- radius scaled by progress → items bloom outward
- **both** scaled by progress → a spiral (the flower open/close from the ARN catalog):

```tsx
// per petal: angle = (2π / petals) * (petals - index), reversed so petal 0 sweeps the full circumference
const radius = useDerivedValue(() => interpolate(progress.value, [0, 1], [0, petalSize]));
const stylez = useAnimatedStyle(() => ({
  transform: [
    { translateX: Math.cos(progress.value * angle) * radius.value },
    { translateY: Math.sin(progress.value * angle) * radius.value },
    { rotate: `${progress.value * angle}rad` }, // petals also turn as they travel
  ],
}));
```

One progress driver, and the whole formation blooms, spirals, and collapses in reverse for free. Craft numbers from the 13-petal theme picker: the driver is `withTiming(target, { duration: 1000, easing: Easing.elastic(0.9) })` rather than a spring, because an elastic timing overshoots and still ends at exactly 1000ms, so the 1000ms background crossfade lands with it. Petals pop in during the first 5% (`[0, 0.05] → scale [0.5, 1]`), depth grows with radius (`shadowOpacity` 0 → 0.5, `shadowRadius` 0 → petal / 6), and the selected color sits on top as a cover petal fading over `[0.2, 1]`. In the catalog: https://www.animatereactnative.com/post/theme-picker-micro-interaction

## Rotation instead of trig

Often you don't need cos/sin at all: rotate a view around the right pivot and the geometry comes free.

**Radial ticks** (the circular "100 lines" progress indicator): render N thin views, each statically rotated into place, and let the per-index window pattern animate them off one driver:

```tsx
// container of line i: static placement
transform: [{ rotateZ: `${(360 / N) * i}deg` }]   // a full-height view, bar in its top half

// inside: the animation is an index window over progress (0-100)
const inputRange = [((i - 2) * 100) / N, (i * 100) / N, ((i + 2) * 100) / N];
translateY: interpolate(progress.value, inputRange, [size * 0.07, size * 0.07, 0], Extrapolation.CLAMP),
backgroundColor: interpolateColor(progress.value, inputRange, [inactive, inactive, active]),
```

Static rotation does the geometry; the driver does the motion. `memo` each line: they mount once and never re-render.

**Cards on a ring: rotate a tall wrapper.** Give each card a wrapper `2R` tall, centered on the ring's center, with the card at its top edge. Rotating the wrapper puts the card on the ring already tangent, and spinning the ring is one shared angle added to every wrapper:

```tsx
const THETA = (2 * Math.PI) / N;                  // 30 cards → 12° apart
const stylez = useAnimatedStyle(() => ({          // default origin = wrapper center = ring center
  transform: [{ rotate: `${fan.value * i * THETA + angle.value}rad` }],
}));
<Animated.View style={[{ position: 'absolute', width: CARD_W, height: 2 * R }, stylez]}>
  <Card />{/* at the wrapper's top edge */}
</Animated.View>
```

`fan` (0 → 1 over 500ms on mount) deals the stacked deck out into the ring. Size the ring so its circumference holds N cards at 90% of their width, never smaller than the screen: `R = Math.max((0.9 * CARD_W * N) / (2 * Math.PI), SCREEN_W)`. Two shared values (`angle`, `fan`) drive all 30 cards. Spinning and snapping it is under Rings below. In the catalog: https://www.animatereactnative.com/post/tarot-cards-animation-reanimated-3-+-gestures-+-math

**Orbits: rotate an offset pivot.** A blob circling a point is a full-screen layer rotating about that point, with the blob drawn off-center inside it; the orbit radius is the blob's distance from the pivot. As a Reanimated 4 CSS animation it costs zero shared values and zero JS:

```tsx
// blob drawn at (blobX, blobY) inside a full-screen layer; orbit radius = |blobX - W / 2|
<Animated.View
  style={[StyleSheet.absoluteFill, {
    transformOrigin: ['50%', blobY, 0],              // pivot: screen center-x, at the blob's height
    animationName: {
      from: { transform: [{ rotate: `${phase}deg` }] },
      to: { transform: [{ rotate: `${phase + 360}deg` }] }, // +360: seamless loop
    },
    animationDuration: '20s', animationIterationCount: 'infinite', animationTimingFunction: 'linear',
  }]}
/>
```

`phase` is random per blob but created once (`useState(() => Math.random() * 360)`): randomness read during render restarts the loop on every re-render. Numbers from the lava lamp: 4 blobs at 50-120% of screen width, light tints at alpha 0.3 over a dark base of the same hue, 18°/s (20s a turn), and one full-screen blur on top instead of one per blob. In the catalog: https://www.animatereactnative.com/post/lava-lamp-background-animation

## Rotation geometry: cover and gaps

**Rotate and zoom inside a still frame.** When an image tilts while its card only slides, rotate the image inside an `overflow: 'hidden'` frame and scale it just enough to never show a corner. A W×H image rotated θ covers its own W×H frame when `s ≥ cosθ + (long / short) · sinθ`:

```tsx
// image inside an overflow:'hidden' card, per-item window in index space
transform: [
  { rotateZ: `${interpolate(x.value, [i - 1, i, i + 1], [15, 0, -15], Extrapolation.CLAMP)}deg` },
  { scale: interpolate(x.value, [i - 1, i, i + 1], [1.6, 1, 1.6], Extrapolation.CLAMP) },
],
// 15° on a 1:1.76 card: cos 15° + 1.76 × sin 15° = 1.42 needed; 1.6 leaves margin
```

The card around it only rides a 40px arc (`[40, 0, 40]`) inside a 2px padded hairline frame, radius 12: all the drama is inside the frame. In the catalog: https://www.animatereactnative.com/post/wallpaper-animated-carousel-react-native-reanimated

**Close the gap after rotateY.** A card turned θ around Y is `w · cosθ` wide, so each side pulls in by `(w / 2)(1 - cosθ)`: 0.146w at 45°, a visible hole between neighbors. Pull the neighbors in by that much:

```tsx
const deg = interpolate(x.value, [i - 1, i, i + 1], [45, 0, -45], Extrapolation.CLAMP);
const lost = (CARD_W / 2) * (1 - Math.cos((Math.abs(deg) * Math.PI) / 180)); // degrees → radians
transform: [
  { perspective: 1000 },
  { translateX: clamp(x.value - i, -1, 1) * lost }, // toward the focused card
  { rotateY: `${deg}deg` },
],
```

The formula is the flat part; perspective then widens the near edge and narrows the far one (a perspective equal to the card width, as in the X/Y rotation carousel, is very strong), so start from it and tune by eye. Cards k slots out that are still visible need `(2k - 1)` times the per-side loss. In the catalog: https://www.animatereactnative.com/post/carousel-animation-x-and-y-rotation

## atan2: pointing at things

`Math.atan2(dy, dx)` is the angle of a vector, in radians, correct in every quadrant (unlike `atan(dy/dx)`, which divides by zero and loses sign). Use it whenever something should **aim**: at a target, along a curve, into its direction of travel.

**The wheel picker** (vertical list curved like a rolodex): each item's index-space distance from center drives both recession and tilt:

```tsx
const dist = useDerivedValue(() =>
  interpolate(scrollY.value, [index - 1, index, index + 1], [30, 0, -30]),
);
const stylez = useAnimatedStyle(() => ({
  transformOrigin: 'left center',           // rotate around the wheel's hub side
  transform: [
    { perspective: 1000 },                  // perspective FIRST, always
    { translateX: Math.cos((dist.value / 180) * Math.PI) * RADIUS }, // recede on the arc
    { rotateZ: `${Math.atan2(dist.value, screenWidth / 4)}rad` },    // tilt tangent to the arc
  ],
}));
```

`cos` pushes off-center items back along the rim; `atan2(dist, radius)` tilts them as if mounted on it. The driver is still index-space scroll and `dist` is still a per-item window; the math is just a smarter output stage. Numbers from the arc carousel: 30° per row, unclamped on purpose so three rows out reaches 90° (the rim, where cos is 0); radius 100; snap interval = row 100 + gap 20; `paddingVertical: (H - ITEM) / 2` so the end rows can reach the center. Past ±3 rows cos turns negative and rows swing back across: if more than 7 can be on screen, write the same 30°/row as `[i - 3, i, i + 3] → [90, 0, -90]` with CLAMP. In the catalog: https://www.animatereactnative.com/post/arc-carousel-reanimated

**The follower-lag lean** (from the ARN balloon slider): derive a lazy copy of a value with a spring, and the gap between the copy and the original is a free velocity signal. Aim something along that gap with atan2 and it leans into motion like it has mass:

```tsx
const springyX = useDerivedValue(() => withSpring(x.value)); // trails the finger
const balloonAngle = useDerivedValue(
  () => 90 + (Math.atan2(-LEVER, springyX.value - x.value) * 180) / Math.PI,
);
// balloon: translateX by springyX (the lag), rotate by balloonAngle (the lean)
```

Drag fast and the balloon tips into the drag; stop and it settles upright, with zero explicit velocity tracking. `LEVER` is the knob-to-balloon distance (96px = 2 × 48 in the original): a longer lever leans less for the same lag. In the catalog: https://www.animatereactnative.com/post/ballon-slider-reanimated-gestures-sensors-and-math

**Velocity skew** is the same follower on a scroll offset: one spring copy for the whole list, and every item skews by the gap:

```tsx
// once, at list level
const lazyY = useDerivedValue(() => withSpring(scrollY.value));
const skew = useDerivedValue(() =>
  interpolate(scrollY.value - lazyY.value, [-LAG, 0, LAG], [8, 0, -8], Extrapolation.CLAMP),
);
// each item: transform: [{ skewY: `${skew.value}deg` }]
```

8° is the catalog's ceiling; tune `LAG` (the gap in px that maps to full skew) on a device and fling both ways to check the sign. When the finger stops, no scroll events arrive, but the follower still converges, so the skew settles to 0 by itself (a hand-rolled `dy/dt` from scroll events gets stuck at its last value). Never put the spring inside the per-item style: N items would restart N springs every frame. In the catalog: https://www.animatereactnative.com/post/jelly-scrolling-reanimated2-+-flatlist

## Rings: snap unwrapped, display wrapped

A spinnable ring (tarot deck, dial, circular picker) keeps one **unwrapped** angle: it grows past 2π freely and is never reduced, so springs never take the long way around. Wrapping happens only where a card asks "how far am I from the front?":

```tsx
// drag: one screen width ≈ 1 radian (angle = start + e.translationX / SCREEN_W)
.onEnd((e) => {
  const v = e.velocityX / SCREEN_W;                           // rad/s
  const projected = angle.value + v * 0.5;                    // where a free spin would stop
  angle.value = withSpring(Math.round(projected / THETA) * THETA, { velocity: v });
})

// card i: shortest signed distance to the front slot, in [-N/2, N/2)
const front = -angle.value / THETA;
const d = wrap(i - front + N / 2, N) - N / 2;
const lift = interpolate(d, [-1, 0, 1], [0, -CARD_H / 2, 0], Extrapolation.CLAMP);
```

`d` makes card N - 1 the neighbor of card 0, so the ordinary `[-1, 0, 1]` window works across the seam. The projection constant is the feel: the tarot deck projects half a second of velocity (`d / (1 - d) / 1000` for `d = 0.998`, the classic scroll-view coast), a free spin. Reanimated's own `withDecay` at its default 0.998 coasts about 0.35·v at 60Hz (0.25·v at 120Hz, its decay compounds per frame), and the 0.2 in the gestures reference snaps stickier. Seed the spring with the same `v` so the snap continues the throw. In the catalog: https://www.animatereactnative.com/post/tarot-cards-animation-reanimated-3-+-gestures-+-math

## Endless motion: integrate time, wrap in screen space

A marquee is a position that grows forever plus copies that wrap. Advance the position by elapsed time, and wrap each copy in **screen space** with the safe modulo: index-space windows stop working once the offset is unbounded.

```tsx
const pos = useSharedValue(0);
const running = useSharedValue(true);
useFrameCallback(({ timeSincePreviousFrame: dt }) => {
  if (running.value) pos.value += (SPEED * (dt ?? 0)) / 1000; // px/s; dt is null on the first frame
});

// copy i of N, P = tile + gap, (N - 1) * P ≥ screen width
const x = wrap(i * P - pos.value + P, N * P) - P;             // always in [-P, (N - 1) * P)
transform: [{ translateX: x }]
```

The same value takes a drag: pause the clock, scrub, coast, resume.

```tsx
Gesture.Pan()
  .onBegin(() => { running.value = false; cancelAnimation(pos); })  // catch it mid-coast
  .onChange((e) => { pos.value -= e.changeX; })                       // the finger scrubs pos
  .onFinalize((e) => {
    pos.value = withDecay({ velocity: -e.velocityX }, (finished) => {
      if (finished) running.value = true;                             // clock resumes after the coast
    });
  });
```

Numbers from the image-playground wall: 3 rows of 12 tiles (0.45W wide, 8px gaps, radius 8), alternating direction, 12px/s, the whole wall rotated -4°, edge fades 15% top and 30% bottom. Because `x` is a real screen position, screen-based effects are one interpolation: Apple Invites tilts each card by where it is on screen, `interpolate(x, [-ITEM, (W - CARD) / 2, W], [-3, 0, 3])` degrees, and the active card is `Math.floor(wrap(offset + W / 2, TOTAL) / ITEM)`. In the catalog: https://www.animatereactnative.com/post/ai-image-playground-react-native-marquee, https://www.animatereactnative.com/post/apple-invites-animated-onboarding

## Loops: one clock, phase per index

N looping items should read one clock at different phases, not run N loops with staggered delays: one clock can't drift, pauses with one `cancelAnimation`, and the phase math guarantees the relationship between items. The processing loop's three bars relay: each holds, grows, then shrinks (1000ms each, 3000ms period), bar i runs period / N behind the previous one, so at any moment one grows by `f(t)` while another shrinks by `1 - f(t)` with the same easing and the total width never changes.

```tsx
const ease = Easing.inOut(Easing.ease);  // easing functions are worklets
const t = useSharedValue(0);
useEffect(() => {
  t.value = withRepeat(withTiming(1, { duration: 3000, easing: Easing.linear }), -1, false);
}, []);                                  // a shared value cancels its own animation on unmount

// bar i of N: same clock, shifted by period / N; stages: hold, grow, shrink
const k = useDerivedValue(() => {
  const p = wrap(t.value - i / N, 1) * 3;
  return p < 1 ? 0 : p < 2 ? ease(p - 1) : 1 - ease(p - 2);
});
```

Three 1px outlined bars, each at most 0.49 of a 0.7W container, one shared value. Drive `scaleX: k` with `transformOrigin: 'left'`; the original animates `width`, which is defensible only because it's three shallow views and the siblings' reflow is the effect. In the catalog: https://www.animatereactnative.com/post/processing-loop-react-native-reanimated2

`withRepeat(withSequence(a, b), -1, true)` does **not** play the sequence backwards: reverse swaps a single animation's start and target, a sequence has neither, so it restarts forward every time. For a ping-pong, reverse a single `withTiming`, or write both legs into the sequence.

## Distance fields: staggering in 2D

A per-index delay staggers a list. A grid (or any 2D layout) staggers by **distance from an origin**, and which distance function you pick is the shape of the wave:

```ts
function euclidean(x1: number, y1: number, x2: number, y2: number) {
  'worklet';
  const dx = x2 - x1, dy = y2 - y1;
  return Math.sqrt(dx * dx + dy * dy);     // circular ripple
}
// Manhattan:  Math.abs(x1 - x2) + Math.abs(y1 - y2)          → diamond wavefront
// Chebyshev:  Math.max(Math.abs(x1 - x2), Math.abs(y1 - y2)) → square wavefront
```

Manhattan reads as a diamond because diagonal neighbors don't count:

```
4 3 2 3 4
3 2 1 2 3
2 1 0 1 2
3 2 1 2 3
```

The wiring (the ARN tap-ripple dot grid): the origin is a **shared value**, so re-tapping re-ripples with zero re-renders; each cell derives its own delay and plays a small local sequence:

```tsx
const origin = useSharedValue({ row: CENTER_R, col: CENTER_C }); // tap writes this

// inside each Dot
const delay = useDerivedValue(
  () => euclidean(origin.value.col, origin.value.row, dot.col, dot.row) * STAGGER,
);
const stylez = useAnimatedStyle(() => ({
  transform: [{
    scale: withDelay(delay.value, withSequence(
      withTiming(1, { duration: STAGGER * 5 }),
      withTiming(0.3, { duration: STAGGER * 3 }),
    )),
  }],
}));
```

`STAGGER` around 60ms per unit reads as a wave. What makes it a ring rather than a flash: **ring thickness = pulse length / stagger**, here (300 + 180) / 60 = 8 cells mid-pulse at once. Keep the durations as multiples of the stagger (5×, 3×) so retuning `STAGGER` rescales the wave without changing its shape. Dots rest at scale 0.3, opacity 0.2; from the center, the wave crosses ~14 cells in ~860ms. One tap gesture on the whole grid finds the cell, instead of a Pressable per dot:

```tsx
const tap = Gesture.Tap().onEnd((e) => {
  origin.value = { col: Math.floor(e.x / CELL), row: Math.floor(e.y / CELL) };
});
```

Grids are where the device limits bite (a 12-column full-screen grid is ~250 views): each cell is one flat view with no shared values of its own beyond the derived delay; see `references/performance.md` before going denser. In the catalog: https://www.animatereactnative.com/post/dots-ripple-reanimated

**Live press fields** are the continuous variant: instead of distance → *delay*, distance → *magnitude*, recomputed every frame from the finger position (the isometric press-grid, where tiles near the finger rise like keycaps):

```tsx
const lift = useDerivedValue(() => {
  const d = Math.sqrt((fingerX.value - cell.cx) ** 2 + (fingerY.value - cell.cy) ** 2);
  const falloff = Math.max(0, 1 - d / FIELD);
  return MAX_LIFT * falloff * falloff * pressed.value; // falloff²: soft shoulders, sharp center
});
```

`pressed` is a spring gate (`withSpring(1)` on touch, `withSpring(0)` on release) multiplying the whole field, so press-in and lift-off feel physical while the per-frame motion stays a pure derivation. For the field's shape, distance can be measured in screen space (circular wave) or in grid space (waves follow adjacency): with a squashed projection, un-squash one axis before measuring, `dy = (fingerY - cy) / SQUASH`. Numbers: 3 shared values (finger x, finger y, pressed) plus one derived lift per tile drive 100 tiles (300 views); the finger starts at a -9999 sentinel so nothing lifts before the first touch; `Gesture.Pan().minDistance(0)` registers the press immediately; field radius 1.9 tiles; max lift 26px, with 26px of top padding so the back row never clips. In the catalog: https://www.animatereactnative.com/post/isometric-press-grid

## Projected grids: isometric without 3D

The keycap/press-grid family (tiles rise under the finger on an isometric board) needs no 3D engine. The recipe:

**1. Draw flat, project once.** Lay the grid out normally, then put one transform on the container. Transform order matters and reads outermost-first: rotate the grid, *then* squash in screen space:

```tsx
transform: [{ scaleY: 0.5 }, { rotateZ: '45deg' }]  // 2:1 isometric; 0.577 for true 30° iso
```

**2. Gesture on the untransformed wrapper, centers precomputed.** Never inverse-transform the finger per frame. Attach `GestureDetector` to the plain wrapper *around* the projected container, and compute each cell's projected screen center once in plain JS with the same rotate-then-squash the transform applies:

```ts
// lx, ly: cell center offset from the grid's center, in flat grid space
const COS45 = Math.SQRT1_2;
cx = wrapperCenterX + (lx - ly) * COS45;
cy = wrapperCenterY + (lx + ly) * COS45 * SQUASH;
```

Finger `e.x/e.y` and cell centers now share one coordinate space; the per-frame work is just the distance field above.

**3. Lift = the inverse projection of screen-up.** A tile inside the projected container can't just `translateY: -lift`; that vector gets rotated and squashed too. Push screen-up (0, -1) backwards through the transform. For 45° + 2:1 squash the inverse is beautifully simple, equal negative x and y:

```tsx
transform: [
  { translateX: -lift.value * Math.SQRT2 },
  { translateY: -lift.value * Math.SQRT2 },
] // exactly `lift` screen pixels, straight up
```

**4. Extrusion is three stacked layers, not geometry.** Per tile: a dark pit that never moves (shadow + hole), a side-colored body at ~half lift, the bordered face at full lift, both fed by the same derived `lift`. The visible sliver of body *is* the side face, and thickness grows with lift for free. Render cells in row-major order: later rows paint on top, so a lifted tile correctly covers the tiles behind it.

## Deterministic noise: hash the index, don't roll dice

`Math.random()` per frame in a worklet flickers, can't be replayed or scrubbed, and changes on every re-render. Noise that should look random is a hash of (time step, index). The text scramble: one driver 0 → 1 over 1200ms linear, a lock time per character rolled once, and an integer hash for the glyphs in between:

```tsx
const GLYPHS = 'ABCDEFGHIJKLMNOPQRSTUVWXYZ0123456789#%&*'; // 40, coprime with 7 and 13
const lockAt = useMemo(() => TEXT.split('').map(() => 0.15 + Math.random() * 0.85), []);
const animatedProps = useAnimatedProps(() => {
  const tick = Math.floor(p.value * 35);                   // 35 glyph changes over the run
  let out = '';
  for (let i = 0; i < TEXT.length; i++) {
    const locked = p.value >= lockAt[i] || TEXT[i] === ' ';
    out += locked ? TEXT[i] : GLYPHS[(tick * 7 + i * 13) % GLYPHS.length];
  }
  return { text: out, defaultValue: out } as TextInputProps;
});
```

Coprime steps mean each character cycles through every glyph before repeating, and neighbors never show the same one. Render it in a non-editable `TextInput` with a **monospace** font (Menlo): proportional glyphs change width every tick and the line jitters. The same rule covers random layouts: roll once (`useMemo`, a `useState` initializer, or `useSharedValue(() => ...)`), never during render or inside a worklet. In the catalog: https://www.animatereactnative.com/post/hacker-text-greater-react-native-reanimated

## Geometry helpers live outside the worklet

Vertex math (triangle corners, polygon points, arc endpoints) is plain JS computed once in `useMemo`, not per frame. Regular polygon, the only formula you need:

```ts
const points = Array.from({ length: n }, (_, i) => {
  const angle = i * ((2 * Math.PI) / n) - Math.PI / 2 + rotationRad;
  return { x: cx + r * Math.cos(angle), y: cy + r * Math.sin(angle) };
});
```

Position elements at the computed points (absolute `left`/`top`), then let the animated layer do only transforms: the split keeps the per-frame work purely on the fast path. Placing an element's *center* on a point is `translateX: '-50%'`, `translateY: '-50%'` first in the transform array.
