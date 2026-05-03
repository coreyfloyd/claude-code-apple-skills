# SwiftUI Gesture & Animation Interaction Pitfalls

Interaction bugs that look like performance problems but are actually gesture/animation correctness issues. Symptoms: jitter, snap-back glitches, resistance feel, gestures that fire on a child instead of the intended view. Fixes are usually one-line.

## 1. Coordinate-Space Feedback Loop

### The Problem

When a `DragGesture` is attached to a *child* view but `.offset()` is applied to an *ancestor* (any parent that wraps the gesture-receiving child), you get a coordinate-space feedback loop:

1. User drags down 10pt
2. Gesture fires, you set `parentOffset = 10`
3. The parent view moves down 10pt
4. The child moves with it (it's inside the parent)
5. The DragGesture's `value.translation.height` is computed in the child's coordinate space — but the child has moved with the parent, so the finger's relative position has shifted
6. Translation is reported as a smaller delta than the finger actually traveled
7. `parentOffset` corrects to a smaller value
8. Now the finger is "ahead" of the view again → translation reports larger
9. Oscillation

### Symptom signature

- Sheet/view "follows" the finger but jitters around it (1-3pt high-frequency oscillation)
- Feels like the view is *resisting* being pushed
- **Slow drags are noticeably worse than fast flicks** — fast flicks resolve before the loop converges; slow drags give the loop time to oscillate visibly
- No `withAnimation` is involved — defeating implicit animation context with `Transaction(animation: nil)` doesn't fix it (this is the diagnostic that confirms it's the feedback loop, not implicit animation)

### Before (jittery)

```swift
SheetContainer {
    ShowDetail()
        .gesture(  // ❌ Gesture on child
            DragGesture()
                .onChanged { value in
                    sheetOffset = value.translation.height
                }
        )
}
.offset(y: sheetOffset)  // ❌ Offset on parent
```

### After (smooth)

```swift
SheetContainer {
    ShowDetail()
}
.offset(y: sheetOffset)
.gesture(  // ✅ Gesture on the same view that owns the offset
    DragGesture()
        .onChanged { value in
            sheetOffset = value.translation.height
        }
)
```

### When you need source-aware behavior

If different drag regions must trigger different actions (e.g. drag on the sheet body dismisses the sheet; drag on an inner card dismisses the card only), put the *primary* gesture on the offset-owning view and use `.highPriorityGesture` on inner children to override for their region:

```swift
SheetContainer {
    ShowDetail()  // Drags here fall through to the parent's gesture

    Card()
        .offset(y: cardOffset)
        .highPriorityGesture(  // ✅ Wins for card region; offset on same view
            DragGesture()
                .onChanged { value in cardOffset = value.translation.height }
        )
}
.offset(y: sheetOffset)
.gesture(
    DragGesture()
        .onChanged { value in sheetOffset = value.translation.height }
)
```

The card's `.highPriorityGesture` wins over the parent's `.gesture` for touches in the card region (SwiftUI evaluates the innermost gesture first when both have explicit priority). Card-region drags drive `cardOffset`; everything else drives `sheetOffset`. Each gesture is on the view it offsets — no feedback loop in either path.

### Rule

**Always attach a `DragGesture` to the same view that owns the corresponding `.offset()`.** If you need to trigger different behavior from different regions, use nested `.highPriorityGesture` on inner views — never `.gesture` on a child of an offset parent.

---

## 2. @GestureState Resets Before Transition Completes

### The Problem

`@GestureState` automatically resets to its default value the *instant* the gesture ends. If you use the value to position a view via `.offset()` and then trigger a `.transition(...)` to remove the view on gesture end, the offset snaps back to 0 *before* the transition runs. The user sees the view jump to its rest position, then animate out — looks like a glitch.

### Symptom signature

- On dismiss-by-drag: view jumps back up briefly, then slides out
- The further the user dragged before releasing, the more visible the snap-back
- The transition itself looks correct in isolation; only the start position is wrong

### Before (snap-back glitch)

```swift
@GestureState private var dragTranslation: CGFloat = 0
@State private var isPresented = true

if isPresented {
    Card()
        .offset(y: dragTranslation)
        .gesture(
            DragGesture()
                .updating($dragTranslation) { value, state, _ in
                    state = value.translation.height
                }
                .onEnded { value in
                    if value.translation.height > 120 {
                        // ❌ dragTranslation snaps to 0 NOW, before withAnimation runs
                        withAnimation { isPresented = false }
                    }
                }
        )
        .transition(.move(edge: .bottom))
}
```

### After (smooth dismiss)

```swift
@State private var dragTranslation: CGFloat = 0  // ✅ @State, not @GestureState
@State private var isPresented = true

if isPresented {
    Card()
        .offset(y: dragTranslation)
        .gesture(
            DragGesture()
                .onChanged { value in
                    dragTranslation = value.translation.height
                }
                .onEnded { value in
                    if value.translation.height > 120 {
                        // ✅ Hold the offset; transition continues from current position
                        withAnimation(.easeOut(duration: 0.28)) {
                            isPresented = false
                        }
                        // Reset after transition completes so next presentation
                        // starts at offset 0
                        DispatchQueue.main.asyncAfter(deadline: .now() + 0.32) {
                            dragTranslation = 0
                        }
                    } else {
                        // Drag didn't meet threshold — spring back to rest
                        withAnimation(.spring(response: 0.35, dampingFraction: 0.85)) {
                            dragTranslation = 0
                        }
                    }
                }
        )
        .transition(.move(edge: .bottom))
}
```

### Rule

If a `DragGesture`'s value drives a position that *continues into a dismiss/transition animation*, use `@State` (not `@GestureState`) so you control when the value resets. `@GestureState` is only the right choice when the value should snap back to zero on release (e.g. a button press scale effect, or a drag that always returns to origin).

---

## 3. ScrollView Eating Drag Gesture

### The Problem

A `DragGesture` attached to a view that contains a `ScrollView` (or has a `ScrollView` as a descendant) won't fire on `.gesture(...)` — the ScrollView's internal `UIPanGestureRecognizer` claims the touch first. The user drags and nothing happens.

### Symptom signature

- DragGesture's `.onChanged`/`.onEnded` never fire — easy to confirm with a `print(value)` inside `.onChanged`
- Tap gestures on the same view still work (they're not pans)
- Removing the inner ScrollView "fixes" the drag — confirms the conflict

### Before (drag never fires)

```swift
Card()
    .gesture(  // ❌ Loses to ScrollView's pan recognizer inside Card
        DragGesture().onEnded { /* dismiss */ }
    )

struct Card: View {
    var body: some View {
        ScrollView { /* content */ }
    }
}
```

### After (drag wins)

```swift
Card()
    .highPriorityGesture(  // ✅ Beats ScrollView's pan recognizer
        DragGesture().onEnded { /* dismiss */ }
    )
```

### Tradeoff

`.highPriorityGesture` on the outer view blocks scrolling inside the card whenever the gesture's minimum distance is met. Acceptable when:

- The card content is short and doesn't need to scroll
- The gesture is dismiss-only (you want any drag to dismiss, not scroll)

Not acceptable when:

- You want drag-to-dismiss only when scrolled to top (pull-down-to-dismiss), with normal scroll otherwise

For the second case, use `simultaneousGesture` and gate on the ScrollView's content offset (track via `GeometryReader` or `scrollPosition` API), or wrap a UIKit `UIScrollView` for finer recognizer control.

---

## Quick Reference

| Symptom | Likely pitfall | Fix |
|---------|---------------|-----|
| Sheet follows finger but jitters / feels resistant during drag | Coordinate-space feedback loop (#1) | Attach gesture to same view as offset |
| Card snaps back briefly before sliding out on dismiss | @GestureState resets too early (#2) | Use @State; reset manually after transition |
| DragGesture on a view containing ScrollView never fires | ScrollView pan claims touch (#3) | Use `.highPriorityGesture` (with the tradeoff) |
| Slow drags jitter, fast flicks work | Feedback loop (#1) — fast flicks resolve before the loop oscillates | Same as #1 |

## Diagnostic shortcut for the feedback loop

If you suspect #1 specifically — try wrapping the `.onChanged` body in `withTransaction(Transaction(animation: nil)) { ... }`. If the jitter persists, it's not implicit animation interference; it's the coordinate-space loop. If the jitter goes away, you had a different problem (rare — `.onChanged` doesn't normally inherit animation context).
