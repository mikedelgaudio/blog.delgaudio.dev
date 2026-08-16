---
title: "When CSS Can't Express Priority: Notes on a Layout Engine"
date: 2026-08-02T15:31:42-07:00
draft: false
tags: ["typescript", "css", "layout", "architecture"]
categories: "web development"
cover:
  image: "/images/covers/responsive-conductor-vs-pretext.svg"
  alt: "Diagram of a header bar dropping lower-priority controls as it narrows"
---

Back in 2024 I built a small library called **ResponsiveConductor** to solve a layout problem CSS has no vocabulary for. I recently revisited it, and this post is about the shape of that design: what the problem actually is, what architecture it pushes you toward, and how that compares to another project that arrived later from a similar starting complaint.

## The problem CSS won't solve

Every app I've worked on eventually grows a header bar like this:

```
[ Control A ] [ Control B ................... ] [ Control C ] [ Control D ] [ Control E ]
```

A row of header control boxes, each with its own idea of how much room it deserves. It's fine at 1600px. At 900px the wide one starves its neighbors. At 600px it's a smear of ellipses. So you reach for CSS, which offers three tools:

- **Media queries** react to the *viewport*, not the box your bar actually lives in. Put the same bar in a side panel and every breakpoint lies.
- **Container queries** fix that, but they're still *thresholds*. You get "below 700px, hide Control D." You don't get "hide Control D only if Control B still can't reach 120px after Control C has already shrunk."
- **Flexbox** gets closest, with `flex-shrink`, `min-width`, and `max-width`. But flexbox shrinks everything *proportionally and simultaneously*. There's no way to say "exhaust Control B's slack before touching Control C." And flexbox will never hide anything for you.

What I wanted to express was an **ordering**, not a set of thresholds:

> Control E never shrinks. Control A is fixed at 50px. Shrink Control B before Control C. If we're still over budget, drop Control D entirely. If we then have slack, give it all to Control B.

That's a priority list, and CSS has no syntax for one. Notice that this isn't a measurement problem. It's a *policy* problem. Knowing exactly how wide everything wants to be still doesn't tell you which sibling should surrender first. That's product intent, and it has to be declared somewhere.

## The architecture

The central decision was to make that intent **data**, attached per child:

```ts
interface IResponsiveConductorSchema {
  key: string;
  minWidth: number;
  maxWidth: number;
  /** Higher numbers give up space first. */
  shrinkPriority: number;
  isAllowedToHide?: boolean;
  isAllowedToGrowBeyondMaxWidth?: boolean;
}
```

Everything else follows from that. The runtime is three stages:

1. **Measure** the container's available width.
2. **Allocate**: a pure function from that width plus the schemas to a list of pixel widths.
3. **Apply**: pipe the result into `grid-template-columns`. A width of `0` means "you don't exist this frame."

The allocation step is deliberately greedy and cheap: sort by priority, hand out `maxWidth` optimistically, then reclaim from the lowest-priority elements backwards until things fit, hiding droppable elements if that isn't enough. Walking backwards is the whole point, because it's what makes "Control C collapses before Control B" expressible at all.

### The part that held up

The design decision I'd defend without qualification is that **the layout decision is a pure function of one number and a plain array**:

```ts
determineColumnWidths(contentWidth: number, schemas: Schema[]): number[]
```

No DOM, no framework, no refs. It's bin-packing with a tiebreaker. Everything stateful and browser-coupled lives in a thin shell around it.

That boundary pays off in ways I didn't fully anticipate. The policy becomes inspectable, so you can reason about "what happens at 740px" without a browser. It stays framework-agnostic almost by accident. And when something looks wrong on screen, you immediately know which side of the line to look at: either the number going in is wrong, or the policy acting on it is.

### Where the architecture shows its seams

Three things, and notably none of them are in the allocator itself.

**The measurement layer observes the wrong thing.** It listens to `window` resize events. But the entire pitch of the component is "react to your *container*, not the viewport", so the measurement layer contradicts the premise. Open a side panel, collapse a nav rail, expand an accordion above it: the container changes width and nothing notices. `ResizeObserver` on the container is the right primitive. The lesson generalizes: **a container-aware component has to observe the container, or it's a media query with extra steps.**

**There's no separation between setup and steady state.** Every resize tick reruns the whole pipeline: validation, partitioning, two sorts, several array rebuilds. But almost all of that depends only on the schemas, which don't change when the window moves. The shape it wants is a split:

```ts
const prepared = prepareConductor(schemas);                // once
const widths = layoutConductor(prepared, containerWidth);  // every frame
```

Recognizing which inputs actually change, and at what rate, turns out to be one of the highest-leverage decisions in a layout library, and it's invisible until you look for it.

**Visibility and width are coupled decisions handled as one pass.** Elements are re-admitted greedily into whatever slack remains after everyone else has claimed their maximum. That means a *higher*-priority hidden element can lose to a lower-priority one that simply happens to fit:

```ts
determineColumnWidths(150, [
  { key: 'a', minWidth: 50,  maxWidth: 100, shrinkPriority: 1 },
  { key: 'b', minWidth: 100, maxWidth: 200, shrinkPriority: 2, isAllowedToHide: true },
  { key: 'c', minWidth: 50,  maxWidth: 150, shrinkPriority: 3, isAllowedToHide: true },
])
// => [100, 0, 50]   (c survives, b doesn't)
```

This is the interesting one, because it's not a slip in the code. It's the architecture being one phase short. **A greedy pass over a sorted list isn't the same as honoring the sort.** Once you have two decisions that feed each other, where *who is visible* changes the budget that *who gets width* works against, they need to be sequenced explicitly: settle the visible set first (the largest prefix of the priority order whose minimums fit), then distribute within it. Same complexity, one more phase.

There's also a smaller structural note worth generalizing: the input validation and the allocator disagree about what counts as a legal state, because they were written at different times. Validation rejects a configuration the allocator has a deliberate branch for. **Validators and implementations are two encodings of the same spec.** When they drift, you don't have one spec, you have two. Related: that validation runs inside the render path and throws, which means a sufficiently narrow window unmounts the component. Layout should clamp and degrade; assertions that fire on user input aren't assertions.

## A project that arrived later, from the same complaint

While writing this up I went looking for prior art and found [chenglou/pretext](https://github.com/chenglou/pretext). It's a pure JS/TS library for multiline text measurement and layout that computes a paragraph's height and line breaks *without ever touching the DOM*, with no `getBoundingClientRect` and no reflow, using the canvas font engine as ground truth, with real grapheme segmentation, bidi text, and CSS-matching `white-space` / `word-break` / `letter-spacing` semantics.

It showed up well after mine and I hadn't read a line of it while building ResponsiveConductor. That's what makes it a useful mirror rather than a missed reference: two people arriving independently at the same complaint about CSS.

### How similar are they, really?

Shallower than it first looks. They rhyme in thesis and shape. Both reject CSS as the layout authority, both are pure functions of `(content descriptor, availableWidth) → numbers you feed back into styles`, both solve 1-D horizontal fitting. But they sit on different axes:

- **Pretext works inside one element.** How does this text flow?
- **ResponsiveConductor works between siblings.** Who gets the budget?

Measurement versus allocation policy. They're complementary layers, not competitors, which is why they'd compose neatly. Pretext's `measureNaturalWidth()` returns exactly the kind of number my schemas ask you to hardcode.

The deepest divergence is epistemic: **pretext has an oracle and mine doesn't.** Correct text height is verifiable against the browser's own font engine, which is why that project can carry accuracy snapshots across three browsers, corpus sweeps, and a documented list of platform bugs. "Should Control B shrink before Control C?" has no ground truth. It's product taste. That makes the spec *more* important to write down explicitly, not less, and it's the thing my design leaves implicit.

|  | ResponsiveConductor | pretext |
|---|---|---|
| Layer | allocation policy between siblings | measurement & flow within content |
| Scope | ~250 lines, one function | ~7,700 lines, ~20 exports |
| Correctness | product judgment, no oracle | verifiable against the font engine |
| Runtime deps | Preact (unnecessarily) | zero |
| Maturity | internal experiment | published, versioned, documented |

### What each does better

**ResponsiveConductor** solves something pretext structurally cannot. No amount of accurate measurement tells you which of five header controls to drop. Inter-sibling arbitration requires declared intent. It also handles *discrete* visibility, where pretext is continuous: text reflows, it never disappears. And priority-as-data beats a pile of media queries as an authoring model for progressive collapse. That's the instinct `flex-shrink` should have grown into.

**Pretext** does better on nearly every architectural axis worth stealing from:

- **The `prepare()` / `layout()` split**: expensive input-dependent work once, pure arithmetic on the hot path. The single idea I'd retrofit first.
- **Zero framework coupling.** My allocator is already framework-independent; it just never got extracted.
- **Measured inputs instead of guessed ones.** `minWidth: 40, maxWidth: 500` on a text-bearing control is a guess that goes stale the moment the font loads, the user zooms, or the string is localized. Hand-authored constants are a cache with no invalidation.
- **No DOM reflow**, where my measurement layer forces a synchronous layout flush at frame rate.
- **Documented non-goals.** Its README states outright what it won't do. Enumerated boundaries are how a small library stays small.

## Takeaways

1. **Some layout problems are policy, not measurement.** Knowing every intrinsic width still doesn't tell you who yields first. Make that intent explicit data.
2. **Keep the decision pure and the I/O at the edges.** A layout policy that's a pure function of numbers is inspectable, portable, and easy to reason about.
3. **Separate what changes from what doesn't.** Setup versus steady state is the highest-leverage split in anything that reruns on resize.
4. **Container-aware means observing the container.** Otherwise you've rebuilt media queries.
5. **Coupled decisions need explicit sequencing.** Greedy passes silently discard the ordering you sorted for.
6. **Guessed constants are a cache with no invalidation.** If a number is measurable, measure it.

The core idea holds up: a priority list is the right way to express progressive collapse, and CSS still can't say it. What the revisit taught me is that the interesting part of a system attracts all the design attention, while the plumbing around it quietly decides whether the thing actually works.
