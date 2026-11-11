# Core Concepts: One Number, Fanned Out

## The anti-pattern this replaces

```tsx
// WRONG: one animation per property, fired independently
const radius = useSharedValue(8);
const scale = useSharedValue(1);
const onPress = () => {
  radius.value = withTiming(40, { duration: 300 });
  scale.value = withTiming(1.2, { duration: 350 }); // already drifting: different duration
  setExpanded(true);                                 // and a React re-render on top
};
```

Symptoms of this shape: properties finish at different times, reversing mid-flight needs bookkeeping, adding easing means editing N calls, and interrupting leaves the UI in a half state.

```tsx
// RIGHT: one driver, properties are pure functions of it
const progress = useSharedValue(0);        // 0 = collapsed, 1 = expanded
const [open, setOpen] = useState(false);   // the TARGET lives in JS, not in progress.value

const stylez = useAnimatedStyle(() => ({
  borderRadius: interpolate(progress.value, [0, 1], [8, 40]),
  transform: [{ scale: interpolate(progress.value, [0, 1], [1, 1.2]) }],
  opacity: interpolate(progress.value, [0, 0.5], [0.6, 1], Extrapolation.CLAMP),
}));

const toggle = () => {
  setOpen(!open);
  progress.value = withTiming(open ? 0 : 1, { duration: 350 }); // retarget from intent
};
```

The driver is semantic (0 → 1 means collapsed → expanded), not a pixel value. Pixels live in the output ranges, so the same driver can power a gesture later without changes.

**Toggle a target, never the live value.** `progress.value === 0 ? 1 : 0` (or `> 0.5`) reads `.value` on the JS thread and decides from wherever the animation happens to be: tap while it's closing and the test sends it closed again, so the tap is swallowed. Toggle on intent (a state boolean) and retarget; `withTiming` starts from the current value, so a mid-flight tap reverses smoothly.

### Discrete state in, continuous driver out

When the input is a choice (an index, a tab, the sign of a change), keep it in a shared value and let each consumer derive its own animated copy:

```tsx
// bar i: one shared selection index, every bar derives its own emphasis
const on = useDerivedValue(() => withTiming(active.value === i ? 1 : 0.15));
// the trend arrow: the SIGN of the change is the driver
const dir = useDerivedValue(() => withTiming(Math.sign(delta.value) || 1));
const arrowStylez = useAnimatedStyle(() => ({
  transform: [{ rotate: `${-45 * dir.value}deg` }],                               // sweeps through flat
  backgroundColor: interpolateColor(dir.value, [-1, 1], ['#E7B824', '#329F82']), // recolors on the way
}));
```

Unselected bars sit at 0.15, not 0, so they stay readable and tappable; bar heights map into 10-40% of the chart so the smallest never vanishes. A tappable pagination row works the same: `useDerivedValue(() => withSpring(selectedIndex))` turns the tapped index into a float that reuses the scroll windows. (https://www.animatereactnative.com/post/custom-bank-spendings-custom-barchart-+-number-animation-reanimated2, https://www.animatereactnative.com/post/onboarding-pagination-indicator-reanimated)

## Restraint: small numbers read as polish

The animations that feel expensive move few properties, by small amounts, from one driver. Numbers from the catalog worth copying:

- **Press**: one property. Scale 0.9 on a button; no color or opacity change on press.
- **Selection**: the chosen item stays at 1, the rest drop to 0.15 opacity (bar chart) or dim to 0.4 (2D grid). Don't also scale or move the losers.
- **Pop**: scale 1.1 plus a 2px label rise over 200ms marks a picked swatch.
- **Neighbors**: 0.98 scale (2% on a ruler), 0.92 scale + 0.4 opacity (a vertical feed), 0.3 opacity (a full-screen slider); chat recency in opacity steps of 1/6.
- **Travel**: 25px for a fade-in (the Fade presets' default), ±20px for text riding a carousel, ±3° tilt across the whole screen width, 8° cap on velocity skew.
- **One change at a time**: a folding header keeps full opacity until the fold is half done; text leaves first (labels gone by 10% of a sheet's travel). On mount, animate the first visible title only; cap animated filter results at 10.

Before adding a channel, ask what it says that the others don't: a fifth property on the same window is usually noise. (https://www.animatereactnative.com/post/squishy-button-reanimated-layout-animation, https://www.animatereactnative.com/post/telegram-stories-scroll-animation)

## Shared input windows

When several properties belong to one motion, they interpolate over the **same window**. In the collapsing-header recipe, header height, title slide, title scale, subtitle fade, and divider all read `[0, 120]` of scroll: change 120 to 160 in one place and everything still finishes together. The credit-card screen does the same over `[0, 200]`: the card collapses while the footer fades 1 → 0 and drifts -67px (a third of the scroll distance). Properties with private ranges drift apart the first time someone retunes.

Sub-windows inside the shared window are the choreography:

```tsx
// fade starts late, ends with everything else
opacity: interpolate(progress.value, [0.5, 1], [0, 1], Extrapolation.CLAMP)
// pop: overshoot built into the range
transform: [{ scale: interpolate(progress.value, [0, 0.8, 1], [0, 1.08, 1]) }]
```

The Telegram stories sheet nests them on one sheet index: labels gone by 0.1, non-header avatars by 0.3, the header stack appearing over `[0.95, 1]`. (https://www.animatereactnative.com/post/credit-card-scroll-animation, https://www.animatereactnative.com/post/telegram-stories-scroll-animation)

## Interpolation shapes: read the output triple

Over a `[before, focus, after]` window (usually `[i - 1, i, i + 1]`), the output triple says what the motion means:

| Shape | Example | Reads as |
|---|---|---|
| Symmetric `[a, b, a]` | opacity `[0.3, 1, 0.3]`, arc drop `[40, 0, 40]` | distance from focus |
| Antisymmetric `[a, 0, -a]` | rotateZ `[15, 0, -15]`°, text `[20, 0, -20]`px | direction of travel |
| Asymmetric `[a, 1, b]` | scale `[1.2, 1, 0]`, `[0.2, 1, 0]` | arriving differs from leaving |
| Fill-up `[off, on, on]` | dot color via `interpolateColor` | stays lit once passed |
| Plateau `[0, t, max] → [a, b, b]` | `[64, 24, 24]` | finishes early, then rests |

More points shape the falloff. A vertical pagination rail uses five:

```tsx
// dash i (10×2): the active dash is twice as long, neighbors halfway
const r = [i - 2, i - 1, i, i + 1, i + 2];
const stylez = useAnimatedStyle(() => ({
  opacity: interpolate(index.value, r, [0.2, 0.5, 1, 0.5, 0.2], Extrapolation.CLAMP),
  transform: [{ scaleX: interpolate(index.value, r, [1, 1.4, 2, 1.4, 1], Extrapolation.CLAMP) }],
}));
// static style: transformOrigin: ['100%', '50%', 0] so dashes grow away from the right rail
```

- **Input ranges must ascend.** `interpolate` binary-searches the input; with 3+ descending points it silently picks the wrong segment. Flip the outputs instead.
- **The default extrapolation is EXTEND.** A few steps out, unclamped scale goes negative (mirrored cards) and opacity below 0. Clamp, or add a floor point. Extending on purpose is fine when floored: a deck where each deeper card sits 22px higher at `1 - 0.05k` scale (floored at 0.5) and `1 - k/6` opacity.
- **Hand-off windows are narrower than move windows.** Page labels move over `[i - 1, i, i + 1]` but fade over `[i - 0.5, i, i + 0.5]`, so both are invisible at mid-swipe and never overlap. (https://www.animatereactnative.com/post/vertical-carousel-animation-reanimated, https://www.animatereactnative.com/post/wallpaper-animated-carousel-react-native-reanimated)

## Coupling: secondary motion is a multiply

"The card follows my finger" becomes "the card feels physical" through coupling: secondary channels are the primary value times a small named constant. In the swipe-deck recipe (tinder-swipe-cards-react-native-reanimated on keyframer.dev/recipes) one pan drives `translateX: panX`, a damped `translateY: panY * 0.25`, and the bank `rotate: panX * 0.06deg`. Those constants are **personality knobs**: 0.06°/px is a lazy bank, 0.12 is cartoonish. Coupled elements **ride** the driver: the LIKE badge takes the card's `translateX` plus its own opacity window (`[30, 130] → [0, 1]`), so confidence builds with distance while the badge stays glued to the card. The full follow-phase treatment is in `references/gestures.md`.

## Computed, not positioned

If element B's position depends on element A's animated property, derive it; never hand-place it. The collapsing header's divider rides the header's **live** bottom edge:

```tsx
const headerHeight = useDerivedValue(
  () => interpolate(scrollY.value, [0, 120], [150, 64], Extrapolation.CLAMP),
);
const dividerStylez = useAnimatedStyle(() => ({
  transform: [{ translateY: headerHeight.value + HEADER_TOP }], // computed, not positioned
}));
```

It cannot drift when constants change, and a value with two consumers is hoisted into one `useDerivedValue` instead of computed twice per frame.

## Color

`interpolateColor(progress.value, [0, 1], ['#0F172A', '#22C55E'])`, driven by a timing, not a spring. Between many colors, grow both ranges together (`[0, 0.5, 1]`, three colors). Never fade to `'transparent'`: it is black at alpha 0, so white → transparent passes through grey. Fade to the same color at alpha 0 (`'rgba(255,255,255,0)'`).

## Spring or timing

| Use `withSpring` | Use `withTiming` |
|---|---|
| Anything that moves through space (translate, scale, rotate) | Opacity, color |
| Release of a gesture (pass the velocity) | Choreography with exact durations |
| Snapping, retargeting from live data | Progress bars, loaders, loops |

**Reanimated 4 springs don't bounce by default.** Bare `withSpring(x)` and `.springify()` use `GentleSpringConfig`, so any code that expects bounce (including v3 code like a bare `withSpring(0.9)` squish) must pass a config. Damping ratio is `damping / (2·√(stiffness·mass))`: below 1 overshoots, 1 settles without overshoot, above 1 glides in.

| Exported preset | Config | Ratio | Feel |
|---|---|---|---|
| `GentleSpringConfig` (v4 default) | mass 4, damping 120, stiffness 900 | 1.0 | no overshoot |
| `SnappySpringConfig` | mass 4, damping 110, stiffness 900, overshootClamping | 0.92 | quick, clamped |
| `WigglySpringConfig` | mass 4, damping 90, stiffness 900 | 0.75 | visible squish |
| `Reanimated3DefaultSpringConfig` | mass 1, damping 10, stiffness 100 | 0.5 | the old v3 bounce |

```tsx
scale.value = withSpring(1);                                        // v4: no overshoot
scale.value = withSpring(1, WigglySpringConfig);                    // squish
scale.value = withSpring(1, { duration: 300, dampingRatio: 0.7 });  // design-spec spring
offset.value = withSpring(snapTo, { velocity: e.velocityY, mass: 1, damping: 18, stiffness: 180 });
```

- **Partial physics configs inherit mass 4.** `{ damping: 18, stiffness: 180 }` without `mass` is ratio ≈ 0.34 on v4 (0.67 on v3's mass 1): slower and much bouncier. Pass `mass: 1` when porting.
- **Duration mode is perceptual**: `{ duration, dampingRatio }` settles in about 1.5× the duration you pass; `dampingRatio` alone defaults to 550ms.
- **Seed releases with gesture velocity** so the handoff from finger to physics is seamless.
- Ratios that work: 1 for chrome and layout, ~0.7 for squish and pop, above 1 for motion that shouldn't draw the eye (a deck's next card promotes with `{ mass: 1, damping: 45, stiffness: 200 }`, ratio ≈ 1.6, while the thrown card flies).
- **A spring is a low-pass filter.** Retargeting it on every data point keeps velocity, so jittery input renders as smooth motion: a QR scanner's box retargets x, y, width, height per detection on the gentle default. (https://www.animatereactnative.com/post/squishy-button-reanimated-layout-animation, https://www.animatereactnative.com/post/qrcode-scanner-bounding-box)

## Press feedback

Feedback belongs to the finger: start it on press-in, release it on press-out, run the action in `onPress`. One spring gate (0 → 1) drives it, so a press released mid-spring reverses from wherever it is.

```tsx
const pressed = useSharedValue(0);
const stylez = useAnimatedStyle(() => ({
  // unclamped on purpose: the spring's overshoot past 1 IS the squish
  transform: [{ scale: interpolate(pressed.value, [0, 1], [1, 0.9]) }],
}));

<Pressable
  onPressIn={() => { pressed.value = withSpring(1, WigglySpringConfig); }}
  onPressOut={() => { pressed.value = withSpring(0, WigglySpringConfig); }}
  onPress={onPress}
>
  <Animated.View style={stylez}>{children}</Animated.View>
</Pressable>
```

- **0.9 with a ratio-0.75 spring** on a button. On v4 a bare `withSpring` has no overshoot, so the squish reads as a plain shrink.
- **One gate can feed a whole gesture's cast**: the balloon slider's knob gate scales the knob 1 → 2 while the balloon fades in, grows and rises 48px.
- **Hide fast, return slow**: a selection ring steps aside in 100ms when scrolling starts and comes back in 300ms when it settles.
- **Long press is its own intent** (`delayLongPress={1000}` in the squishy button) with its own feedback.
- **Haptics on discrete changes only**: `useAnimatedReaction` on `Math.round(index.value)`, and `scheduleOnRN` the haptic when it changes, never per frame.
- Use `Pressable` or `Gesture.Tap()`, not `onTouchStart`: raw touch handlers don't cancel when the user starts scrolling. (https://www.animatereactnative.com/post/squishy-button-reanimated-layout-animation, https://www.animatereactnative.com/post/ballon-slider-reanimated-gestures-sensors-and-math)

## Staggering

Two ways, pick by driver type. **Mount stagger**: every item runs the identical entering animation; only the start time differs. The step scales with the size of what moves:

| Step | For | Catalog example |
|---|---|---|
| ~20ms | tiny marks (2px pagination dashes) | vertical carousel |
| ~50ms | small items: tiles, chips, bars | stagger grid (`ZoomInEasyDown.springify()`), leaderboard bars |
| ~70ms | text lines | stagger text (`FadeInDown.springify()`) |
| ~140ms | full rows | staggered-list recipe (fade + ~70px ease-out slide) |

140ms across 30 tiles is a four-second wait; 50ms across five rows reads as one blob. A row's icon, label and background share identical times so the row moves as a unit.

**Builders are mutable.** Calling `.delay()` on an instance changes that instance and returns it, so one hoisted builder shared by every child gives them all the last delay. Static calls (`FadeInDown.delay(...)`) create a fresh instance: build one per child.

```tsx
const STEP = 70;
const enter = (i: number) => FadeInDown.delay(i * STEP).springify();
// for a group that leaves as a whole: last in, first out
const exit = (i: number, n: number) => FadeOutUp.delay((n - 1 - i) * STEP).springify();

{lines.map((line, i) => (
  <Animated.Text key={line.id} entering={enter(i)} exiting={exit(i, lines.length)} layout={LinearTransition}>
    {line.text}
  </Animated.Text>
))}
```

Stagger the exit only when the whole group unmounts. A single row removed from a live list with a reversed delay hangs on screen after its siblings have closed the gap.

**Driver stagger** (one progress drives all items, offset per index): shift each item's input window. Scrubbing-friendly and interruption-proof, because it is still one driver; animate the driver back to 0 and the stagger reverses, last in first out.

```tsx
// inside item i of N
const start = (index / N) * 0.5; // each item starts later
opacity: interpolate(progress.value, [start, start + 0.5], [0, 1], Extrapolation.CLAMP)
```

**Two-phase entrance**: a layout stagger hands off to a driver. Leaderboard rows enter with `FadeInRight.delay(i * 50).springify()`; the last row's callback starts the bars on the UI thread, and each bar adds its own 50ms offset:

```tsx
const grow = useSharedValue(0);
const rowEnter = (i: number, isLast: boolean) =>
  FadeInRight.delay(i * 50).springify().withCallback((finished) => {
    if (finished && isLast) grow.value = 1; // runs as a worklet: no React round trip
  });
// inside bar i
const g = useDerivedValue(() => withDelay(i * 50, withSpring(grow.value)));
```

Scores map into 16-122px (the 150px row minus a 28px avatar; the 16px floor keeps the lowest visible), only the winner gets color, and each label waits for `[0.2, 1]` of its bar's growth. (https://www.animatereactnative.com/post/reanimated-custom-stagger-component, https://www.animatereactnative.com/post/leaderboard-animation-reanimated)

## Sequencing

For short chains, compose on the driver: `withSequence(withTiming(1.1, { duration: 100 }), withSpring(1))`. For anything longer, one driver 0 → 1 with the sequence as input windows: long `withSequence` chains are hard to interrupt and impossible to scrub.

A pause between loops is a `withDelay` *inside* the repeat:

```tsx
// shimmer: a 90px soft band crosses the label, rests 300ms, crosses again
sweep.value = withRepeat(
  withDelay(300, withTiming(1, { duration: 1300, easing: Easing.inOut(Easing.ease) })),
  -1,
  false,
);
// band: translateX = interpolate(sweep.value, [0, 1], [-BAND, labelWidth + BAND])
```

- **Constant speed, not constant duration**: derive it as `(labelWidth + 2 * BAND) / SPEED`, or long labels sweep faster than short ones.
- **`withRepeat(anim, -1, true)` reverses a single timing or spring, not a `withSequence`**: a repeated sequence restarts from its first step. Write the return leg into the sequence.
- **Keep-alive**: restart a short sequence on every event, so the element stays while events keep coming and leaves soon after the last. The QR box runs `withSequence(withSpring(1, { duration: 300 }), withDelay(200, withTiming(0, { duration: 300 })))` per detection: gone ~500ms after the code leaves the frame.
- **Let a move land before the reveal**: when a 2D grid pans to a tile, the others dim to 0.4 at once, and the target brightens with `withDelay(200, withTiming(1, { duration: 300 }))` as the 300ms pan settles.
- **Cleanup**: `useSharedValue` cancels its own animation on unmount. Call `cancelAnimation` for values owned elsewhere (a parent, a context) or to stop a loop while mounted. Start loops from `useEffect`, never inside `useDerivedValue` or `useAnimatedStyle` (they re-run), and memoize any randomness. (https://www.animatereactnative.com/post/shimmer-text, https://www.animatereactnative.com/post/qrcode-scanner-bounding-box)

## Layout animations: mount, unmount, things that shift

Use `entering`/`exiting`, not `useEffect` + timing: effect-driven mounts run a render late and can't animate removal at all.

**Know how far the presets travel:**

| Preset | Starts from | Good for |
|---|---|---|
| `FadeIn` / `FadeOut` | opacity only | swaps in place, backdrops |
| `FadeInDown` / `Up` / `Left` / `Right` | 25px away + fade (`FadeInDown` starts below and rises) | rows, labels, cards |
| `ZoomIn` / `ZoomOut` | scale 0 | icons, badges, checkmarks |
| `SlideInDown` / `Up` / `Left` / `Right` | a full window height (or width) away | sheets and full-screen panels only |

Change the distance with `withInitialValues`: a chat bubble pushed up from 100px is `FadeInDown.withInitialValues({ transform: [{ translateY: 100 }] })`; a digit rolling into a clipped row is `SlideInDown.withInitialValues({ originY: fontSize / 2 })` (originY is its y inside the parent). A bare Slide preset inside a card flies in from off-screen.

**Hoist configured instances** (module scope): the screen shares one motion character and nothing is rebuilt per render. Only hoist builders you never modify per child (see Staggering).

```tsx
const _layout = LinearTransition.springify();
const _entering = FadeInDown.springify();
const _rowExit = FadeOut.springify();       // a row leaves in place: layout closes the gap
const _blockExit = FadeOutDown.springify(); // the whole block drops away as one

<Animated.View layout={_layout} entering={_entering} exiting={_rowExit} key={`hour-${hour}`}>
```

For screens where rows are added and removed (the schedule builder, settings forms, tag editors), the feel comes from four habits:

1. **`layout` goes on every view that moves when siblings change**: the card, its header row, every hour row, and the "Add more" button (layout only, no entering). A view without `layout` teleports while its siblings glide.
2. **Keys are identity, not index.** `key={`hour-${hour}`}` lets Reanimated track which row left; index keys make one removal look like every row changed.
3. **Different exits for different scopes**: a removed row exits with a plain `FadeOut.springify()` (its siblings slide into the gap, so travel would double the motion); the whole Hours container exits with `FadeOutDown.springify()`, one movement for the group.
4. **Expanding sections are conditional renders**, `{isActive && <Hours />}` inside a parent with `layout` and `overflow: 'hidden'`. No measured heights, no `maxHeight` hacks.

`.springify()` is the v4 gentle spring (no overshoot), which is the calm feel. For a little life: `LinearTransition.springify().dampingRatio(0.7)`. Note `springify()` resets a duration set before it: write `.springify().duration(400)`.

**Morph by toggling a static size.** To turn a round button into a panel, don't interpolate width and height: switch plain styles and let one hoisted transition animate the frame.

```tsx
const MORPH = LinearTransition.duration(500); // one constant for every piece of the morph

<Animated.View layout={MORPH} style={{
  width: open ? width * 0.85 : 64, height: open ? 'auto' : 64, overflow: 'hidden',
  borderRadius: 32, // closed size / 2: circle → rounded panel, the radius never animates
}}>
  {open && <Animated.View entering={FadeInDown.duration(500)} exiting={FadeOutDown.duration(500)}>{menu}</Animated.View>}
</Animated.View>
```

The FAB uses the same 500ms for the frame, the content, the icon crossfade and the blur backdrop. The squishy button grows its pill to fit a waveform the same way: everything inside carries the one hoisted `LinearTransition.springify()`.

**Filtering and reordering lists:**

- Swap the data with stable ids as keys; `itemLayoutAnimation={LinearTransition}` on `Animated.FlatList` glides the rows that stay (an All/Missed calls toggle is just that plus `keyExtractor`).
- Entering and exiting follow the list's axis (`FadeInRight` / `FadeOutLeft` for a horizontal strip).
- Cap the results that animate (10), debounce the query (500ms), and scroll to offset 0 before swapping.
- `itemLayoutAnimation` works only with a single column. `skipEnteringExitingAnimations` skips item animations when the list itself mounts or unmounts. Use `itemLayoutAnimation` or a `layout` prop on the item, not both. (https://www.animatereactnative.com/post/schedule-reanimated-layout-animation, https://www.animatereactnative.com/post/fab-button-animation-reanimated-3, https://www.animatereactnative.com/post/contacts-filter-layout-animation-+-dialpad)

## Swapping content with keys

Changing a view's `key` unmounts the old one and mounts a new one, so `exiting` and `entering` crossfade them for free: play → stop icons, per-page backgrounds, a label's value. The old view stays mounted until its exit finishes, so give both the same box (absolute fill) or the layout jumps for the length of the exit.

```tsx
<View style={{ width: 24, height: 24 }}>
  <Animated.View key={playing ? 'stop' : 'play'} entering={ZoomIn} exiting={ZoomOut} style={StyleSheet.absoluteFill}>
    <Icon name={playing ? 'stop' : 'play'} />
  </Animated.View>
</View>
```

Icons swap with `ZoomIn`/`ZoomOut` at the 300ms default, or a crossfade matching the surrounding morph (500ms in the FAB). Backdrops swap slower and land with the motion they accompany: 1000ms, the same as the theme picker's petal bloom; a per-page `bg-${index}` image crossfades at 1000ms behind an onboarding carousel (blurRadius 50, container at 0.5 opacity). (https://www.animatereactnative.com/post/theme-picker-micro-interaction, https://www.animatereactnative.com/post/apple-invites-animated-onboarding)

## Custom entering animations

When no preset fits, an entering or exiting animation is a worklet that returns start values and animations. `values` holds the target frame and the window size, so travel can be relative to where the view will rest:

```tsx
const riseIn: EntryExitAnimationFunction = (values: EntryAnimationsValues) => {
  'worklet';
  const t = { duration: 300, easing: Easing.out(Easing.cubic) };
  return {
    initialValues: { opacity: 0, originY: values.targetOriginY + 20 }, // 20px below its slot
    animations: { opacity: withTiming(1, t), originY: withTiming(values.targetOriginY, t) },
  };
};
<Animated.View entering={riseIn} />
```

Type it as `EntryExitAnimationFunction` (no `@ts-ignore`). Exits receive `ExitAnimationsValues` (`currentOriginY`, `currentHeight`, ...): a character rolling out moves up half its height while fading and shrinking over 200ms.

A custom entering can own a whole lifecycle. The announcements toast enters in 300ms (`Easing.out(Easing.ease)`) from off the right edge, 40px low, rotated 45°, scale 0; cruises linearly for a random 2000-4000ms; exits in 300ms; and the last timing's callback calls `scheduleOnRN(remove, id)` so React drops it only once it's gone. Variety comes from three random inputs (vertical offset 50-100px, scale 0.8-1.4, speed); everything else is shared. (https://www.animatereactnative.com/post/announcements-react-native-reanimated-custom-layout-animation)

## Animated text and counters

Text content isn't a style, so `useAnimatedStyle` can't set it, and `setState` per frame defeats the point. Pick by what the text does.

**A number that counts**: `useAnimatedProps` on a non-editable `TextInput`. Return `defaultValue` alongside `text` so a re-render shows the current value (Reanimated's own FPS monitor does exactly this):

```tsx
const AnimatedTextInput = Animated.createAnimatedComponent(TextInput);

const animatedProps = useAnimatedProps(() => {
  const text = `${Math.round(count.value)}`;
  return { text, defaultValue: text };
});

// tabular-nums + fixed width: digits don't jitter, neighbors don't shift
<AnimatedTextInput editable={false} animatedProps={animatedProps} style={{ fontVariant: ['tabular-nums'], width: 90 }} />
```

Stat changes count with the default `withTiming` (300ms, `Easing.inOut(Easing.quad)`). Never `value={sv.value}`: that reads the shared value during render.

**Characters that roll**: one `Animated.Text` per character, and the key decides what animates.

- Key by position (`digit-${i}`, separators `comma-${i}`) when characters are typed: new ones enter, existing ones glide right via `layout`. The typed-number odometer uses 300ms for entering, exiting and layout, clips the row with `overflow: 'hidden'` and fades its top and bottom edges over half a font size.
- Key by position and character (`${i}-${ch}`) when a value changes: only the changed characters swap. The fancy slider label enters with `FadeInDown.withInitialValues({ transform: [{ translateY: 10 }] }).duration(200)` and `LinearTransition.duration(200)` on the characters and the row; the value reaches React once per detent via `useAnimatedReaction` on `Math.round(x.value)`.

**Text that scrambles or types**: one driver, not timers. A `withTiming(1, { duration: 1200, easing: Easing.linear })` maps to a per-character lock time and a deterministic glyph per tick:

```tsx
// LOCK[i] in 0..1, precomputed once in useMemo (random order)
const animatedProps = useAnimatedProps(() => {
  const tick = Math.floor(p.value * 35);
  let text = '';
  for (let i = 0; i < TARGET.length; i++) {
    text += p.value >= LOCK[i] ? TARGET[i] : GLYPHS[(tick * 7 + i * 13) % GLYPHS.length];
  }
  return { text, defaultValue: text };
});
```

A monospace font is required, or the line width jitters every tick. A typewriter is the same driver with `TARGET.slice(0, Math.floor(p.value * TARGET.length))`. (https://www.animatereactnative.com/post/animated-textinput-reanimated-3, https://www.animatereactnative.com/post/hacker-text-greater-react-native-reanimated, https://www.animatereactnative.com/post/slider-with-fancy-animated-text)

## Shared element without navigation

- **Layout-driven**: the tapped card toggles its static size (`width: '50%' → '100%'`, `height: 'auto' → 400`) and one hoisted `LinearTransition.springify()` animates the frame. Raise its `zIndex` shared value to 1000 *before* the state change so it grows over its neighbors, and drop it in the transition's `withCallback` once a shrinking card has landed. A fixed 400px spacer holds the parent's height while the active card goes absolute; the backdrop's `FadeOut.withCallback` resets the selection (via `scheduleOnRN`) only after the exit finishes.
- **Measured flight**: `measure()` the source and target on the UI thread (it can return null: guard it), then fly a proxy center to center. Stagger by finishing earlier, not starting later: each index lands 2% sooner. The real target fades in over `[0.95, 1]` under the proxy.

```tsx
const flightStylez = useAnimatedStyle(() => {
  const { dx, dy, ratio } = geometry.value; // from measure(): center deltas, target/source width
  const t = interpolate(p.value, [0, 1 - index / 50], [0, 1], Extrapolation.CLAMP);
  return {
    transform: [{ translateX: dx * t }, { translateY: dy * t }, { scale: 1 + (ratio - 1) * t }],
  };
});
```

Measure once unless the source can scroll; use a `ScrollView` (virtualized cells can measure null) and `collapsable={false}` on Android. (https://www.animatereactnative.com/post/fake-shared-element-transition-reanimated, https://www.animatereactnative.com/post/telegram-stories-scroll-animation)

## Reanimated 3 vs 4 notes

- `runOnJS` is deprecated in v4 (still exported, now a thin wrapper): write `scheduleOnRN(fn, ...args)` from `react-native-worklets`. On v3, `runOnJS(fn)(args)` is the API.
- Spring defaults changed: v4 uses `GentleSpringConfig` (no overshoot); v3's `{ mass: 1, damping: 10, stiffness: 100 }` is exported as `Reanimated3DefaultSpringConfig`. Partial physics configs inherit mass 4.
- `addWhitelistedNativeProps` is a no-op: delete it. `useAnimatedKeyboard` is deprecated in favor of react-native-keyboard-controller. `Layout` is a deprecated alias of `LinearTransition`. v4 requires the New Architecture.
- On react-native-web, pass explicit dependency arrays to `useAnimatedStyle`/`useDerivedValue`/`useAnimatedReaction`; auto-tracking is unreliable there.

## Real-world examples of these concepts

Interactive versions with live previews and full generated code, at keyframer.dev/recipes/:

- Shared windows + computed geometry: `collapsing-sticky-header-react-native-reanimated`
- Coupling + riding badges: `tinder-swipe-cards-react-native-reanimated`
- Mount stagger craft: `react-native-staggered-list-animation`
- Easing choices side by side: `easing-functions-compared-react-native`
- Animated text: `typewriter-text-reveal-react-native-reanimated`
