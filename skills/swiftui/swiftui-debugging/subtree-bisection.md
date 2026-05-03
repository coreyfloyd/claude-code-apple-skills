# Subtree Bisection (SwiftUI)

Empirical isolation for SwiftUI animation, layout, and rendering bugs that resist static analysis. See [debugging/subtree-bisection.md](../debugging/subtree-bisection.md) for the generic principle. This page covers the SwiftUI-specific recipe.

## Why SwiftUI Needs This

SwiftUI's behavior is a function of layered, scoped, context-dependent semantics:

- **Modifier ordering** — `.frame().background().clipShape().offset()` ≠ `.offset().frame().background().clipShape()`. The same modifiers in different order produce different layout, clipping, and animation behavior.
- **Animation context scoping** — `.animation(_:value:)`, `.transaction { }`, `withAnimation { }`, and `.transition(_:)` all create animation contexts that can re-scope or block parent-side animations from propagating to descendants.
- **View identity rules** — `.id()`, `if/else` branches, and `ForEach` keys decide whether a state change is an update or a fresh mount with a transition.
- **UIKit hosting boundaries** — `UIViewRepresentable` and `UIViewControllerRepresentable` cross into UIKit's animation/layout system. SwiftUI parent animations may or may not propagate cleanly to the hosted view.
- **Environment values** — `\.scenePhase`, `\.dismiss`, custom keys, etc. flow down implicitly. A modifier far up the tree can change behavior far down.

When a bug arises from the *interaction* of two of these systems, reading code in isolation often won't surface the cause. Multiple modifications look like reasonable hypotheses, but each fix attempt fails because the actual cause is somewhere none of them touched.

## When to Reach for This

- Two `.animation` / `.transition` / `.offset` / `.matchedGeometryEffect` edits have failed to fix an animation bug.
- A view "doesn't animate" but you can't tell which modifier in a deep stack is responsible.
- A layout bug where you can't tell whether the cause is the parent, the child, or a wrapping container.
- `_printChanges()` shows the body re-evaluates but you can't tell which subtree triggered the visible change.
- A `UIViewRepresentable` is in scope and may or may not be the breakage point.

## The Recipe

### Step 1 — Replace the Suspect Subtree With `Color.red`

```swift
@ViewBuilder
private var currentPage: some View {
    Color.red
        .frame(maxWidth: .infinity, maxHeight: .infinity)
        .overlay(Text("PAGE: \(page.title)").foregroundStyle(.white))
}
```

`Color` is a leaf SwiftUI primitive with no internal state, no animation modifiers, and no UIKit hosting. The `.frame(maxWidth: .infinity, maxHeight: .infinity)` anchors it in the parent's available space so layout assumptions don't change. The overlay text confirms the right path executed.

### Step 2 — Confirm the Primitive Animates / Lays Out Correctly

Build and run. Reproduce the bug.

- **Red animates correctly** → bug is *inside* the subtree you replaced. Continue to Step 3.
- **Red also breaks** → bug is *outside* the subtree (parent container, modifier on the parent, environment value, presentation boundary). Bisect outward — replace the parent's content the same way.

### Step 3 — Add Halves Back

Stop replacing the entire subtree. Replace it with the *first half* of the original.

For a `ShowDetailView` that's `VStack { staticHeader; scrollableEpisodeList }`, the bisect is:
1. Just `staticHeader` — does it animate?
2. Just `scrollableEpisodeList` — does it animate?

For a deep modifier chain `view.A().B().C().D()`, bisect by removing modifiers:
1. `view` (no modifiers) — works?
2. `view.A().B()` — works?
3. `view.A().B().C()` — works?

### Step 4 — Stop at the Smallest Unit That Breaks

When adding a single modifier, wrapper, or subview makes the bug return, that's the cause. Now read THAT specific code with full attention — its docs, its source, its interaction with the parent context.

## Common Bisect Axes (in order of likelihood)

| Axis | Why it bites | Bisect tactic |
|---|---|---|
| **`UIViewRepresentable` / `UIViewControllerRepresentable`** | UIKit hosting boundary — SwiftUI animation context may not propagate to hosted layer's transform. | Replace the representable with a SwiftUI equivalent (`ScrollView` for `UIScrollView`). |
| **`.animation(_:value:)`** | Scopes implicit animations; can block parent animations on descendants. | Remove the modifier. Move animation to `.transition(.opacity.animation(...))` on conditional content, or use `withAnimation { }` at the state-change site. |
| **`.transition(_:)`** | Only fires on identity changes (insertion/removal). If you expected continuous animation, this is wrong. | Replace with `.offset(_:)` + animatable state. |
| **`.matchedGeometryEffect`** | Cannot cross presentation boundaries (`.sheet`, `.fullScreenCover`). Animates from wrong position or flickers. | Replace with `navigationTransition(.zoom(sourceID:in:))` for cross-presentation, or remove and verify base behavior first. |
| **`.id()`** | Forces fresh mount on identity change → fade/transition instead of animation. | Remove `.id()` or stabilize the value. |
| **`if/else` view branches** | Each branch has independent identity; switching = unmount + mount. | Replace with `ZStack` + opacity, or unify to a single view with parametric content. |
| **`GeometryReader`** | Reports `.local` coordinates by default; does not cross presentation boundaries. | Replace with a fixed frame to test, then add reads back one at a time. |
| **`@State` / `@StateObject` placement** | View recreation resets state. | Move state up; use `@Bindable` and inject. |

## Anchoring the Primitive Correctly

The primitive must occupy the same layout slot as the original. Common anchors:

```swift
// Fills available space (most subtrees)
Color.red.frame(maxWidth: .infinity, maxHeight: .infinity)

// Fixed-size leaf (artwork, icon)
Color.red.frame(width: 96, height: 96)

// Inside a List or ForEach row
Color.red.frame(height: 60)

// Inside a ScrollView (must declare some height)
Color.red.frame(maxWidth: .infinity, minHeight: 400)
```

If the original subtree had `.padding`, `.background`, or other geometry-affecting modifiers, attach the same modifiers to the primitive — the goal is for layout to be identical so any bug remaining is from the *content*, not from changed geometry.

## Real Example: Sheet Slide Animation

**Symptom:** Bottom sheet slides up; show-detail title and artwork appear "parked" at final positions while the sheet's background slides under them.

**Tree under suspicion:**
```
ContainerSheet  (has .offset animation on slide-up)
  └─ BrowserContainerContent
      ├─ BrowserNavBar
      ├─ Divider
      └─ ShowDetailView
          ├─ staticHeader (artwork + title + actions)
          └─ scrollableEpisodeList (DraggableScrollView)
```

**Bisect:**

1. **Replace `ShowDetailView` with `Color.red`** — slides cleanly. Bug is in `ShowDetailView`.
2. **Replace with header-only stub** (`RoundedRectangle` artwork + plain `Text` title + fake button) — slides cleanly. Bug is NOT in the static header.
3. **Replace with just `DraggableScrollView`** — bug returns. `DraggableScrollView` is the cause.

**Diagnosis:** `DraggableScrollView` is a `UIViewControllerRepresentable` wrapping `UIScrollView`. SwiftUI's parent-side `.offset` animation does not propagate cleanly through the UIKit hosting boundary. The hosted UIScrollView's contents render at their final positions while the parent's CALayer transform animates around them.

**What two failed fixes had assumed:**
- That `.appBannerOverlay`'s `.animation(_:value:)` was scoping context. Empirically false — removing it changed nothing.
- That `CachedAsyncImage`'s `.animation(_:value:)` was scoping context. Empirically false — removing it changed nothing.

Static analysis kept finding plausible-looking `.animation(_:value:)` modifiers. Bisection found the actual cause — a UIKit hosting boundary — that no amount of modifier-reading would have surfaced.

## Cost Discipline

Each step is one build cycle (~30s on a fresh DerivedData) plus one repro of the bug (~10s). For a 4-deep view tree, you'll bisect to a single layer in 3–4 steps — under five minutes.

Compare that to: write a fix → build → test → observe still broken → repeat. Bisection is faster than the third hypothesis after two have failed, and produces certainty instead of new guesswork.

## Anti-Patterns

| Anti-pattern | Why |
|---|---|
| Bisecting by editing the suspect modifier (e.g., changing `.animation(.spring)` to `.animation(.linear)`) | You're testing variants of your hypothesis, not bisecting. Replace, don't tweak. |
| Using a SwiftUI primitive that has its own animation behavior (`AsyncImage`, `LinearGradient` with animated stops) | The primitive must be inert. `Color`, `Rectangle`, plain `Text` are safe. |
| Forgetting to anchor the primitive's layout (omitting `.frame(...)`) | Layout collapses, parent re-flows, and your "control" experiment changes the very geometry you were testing. |
| Bisecting before the bug is reliably reproducible | Each step needs a clean signal. If the bug is intermittent, stabilize repro first (slow animations with `Animation.linear(duration: 3)` to make differences visible). |

## Slowing Animations to See the Bug

When the bug is "something doesn't animate" but the animation is fast, slow it down so you can SEE which elements move and which don't:

```swift
// In the parent's animation
withAnimation(.linear(duration: 3)) {  // was .spring(response: 0.32, ...)
    sheetVisible = true
}
```

Or, simulator-wide: **Debug menu → Slow Animations** (also `⌘T` while a simulator window is focused).

A 3-second animation makes "elements moving in lockstep" vs "some pinned at final position" trivially visible. After diagnosis, restore the original animation.
