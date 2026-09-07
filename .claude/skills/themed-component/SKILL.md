---
name: themed-component
description: Add or change a component in packages/ui, or a screen in apps/mobile/src/features. Use when writing JSX that reads theme tokens, when adding a Pressable, when adding accessibility props, and when a component needs different behaviour on web and native.
---

# Writing a themed component

Every screen is assembled from these, and the promise is that changing a tenant's seed colour
re-skins the whole app with no per-screen work (D2). Everything below protects that or protects
somebody using a screen reader.

## Tokens, never values

Read `t.color.*`, `t.space(n)`, `t.radius.*`, `t.type.*`. A literal in a stylesheet is a lint
error (D17) — but the rule cannot see a token smuggled through a variable or a computed string,
so the `style` prop is narrowed to `LayoutViewStyle`: position and size, never colour, padding,
radius or font.

**When you add a component with a `style` prop, pin that narrowing in a
`typeContracts.assertions.ts` file.** `LayoutViewStyle` being correct says nothing about whether
your component uses it — that gap left the boundary open at one component for four PRs.

Build the contract from a *complete* call. Written from `{ style: … }` alone it "rejects"
everything because a required prop is missing, and three green rejections prove nothing. Always
include an `Accepts` case: a contract that rejects everything is a broken prop, not a contract.

## Accessibility uses the ARIA spellings

`role`, `aria-label`, `aria-checked`, `aria-hidden`, `aria-disabled` — not `accessibilityRole`,
`accessibilityLabel`, `accessibilityElementsHidden`.

react-native-web 0.21's forwarded-prop table does not contain `accessibilityState` at all and
marks the others deprecated, so a chip's selected state written the `accessibility*` way is
dropped silently on web. React Native maps the ARIA names back to the native equivalents, so
nothing is lost on device — and web ships first (D30).

Two traps:

- **`accessible` on a container merges it and every descendant into one element.** It makes
  `onAccessibilityEscape` reachable and hides the fields inside a sheet from VoiceOver. UIKit
  passes the escape up the view hierarchy; you do not need it.
- **A role is a promise about the keyboard.** `radiogroup` means one Tab stop and arrow keys
  within. react-native-web gives every enabled `Pressable` `tabIndex={0}`, so a group of them is
  three Tab stops until you add a roving `tabIndex`. A control that claims a role and ignores its
  keys is worse than one that never claimed it.

## Pressables show four states

Hover, pressed, focus and disabled, or the web build feels dead. `InteractiveBox` is the one
place a box becomes pressable — add a variant there rather than a fifth hover implementation.

The one documented exception is the sheet backdrop: it is a scrim the size of the screen, and a
hover tint on it is the background misbehaving rather than feedback. If you add another
exception, say why at the prop.

## Platform splits

`Thing.tsx` is native and the default; `Thing.web.tsx` is web. Metro and Jest both pick the web
file first. Put the shared *types* in a third file — on web, `./Thing` resolves to
`Thing.web.tsx`, so importing a type from "the other one" imports itself.

Prefer a split over `Platform.OS` when the two implementations share no code: keyboard handling,
CSS variables, storage. A `display: contents` wrapper in the web file carries DOM handlers
without entering layout.

## Screens are pure

Props in, JSX out. No router hooks, no data fetching, no storage — the route above supplies them
and the feature controller shapes them. That is what lets a screen render in a test with four
lines of props, and under the theme editor's preview later.

## Before opening the PR

- `npm run verify`
- Run `mutation-check` on every new assertion.
- Component tests run through react-native-web, which is blind to native grouping and gestures.
  If a behaviour is native-only, say so at the line rather than implying it is covered.
