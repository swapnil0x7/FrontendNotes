# Subzero Design System — Interview Guide

**How to use this:** each step has four parts — the code, what it actually does in plain words, the exact words to say out loud, and the follow-up question you'll get. Read the "Say this" blocks aloud a few times. You don't need to memorise them word for word; you need the *shape* of the answer to be automatic so your brain is free to handle the follow-up.

---

## Part 0 — The opening answer

You will get asked some version of "tell me about the design system you built." This is your 45-second answer. Everything else in this guide is you defending it under questioning.

> "We built Subzero, a design system layered on top of MUI v7. The core of it is a token pipeline: raw colour values live in one palette file, they get mapped to semantic names per colour scheme — light, dark and high contrast — and MUI compiles those into CSS custom properties. Every component override then references the CSS variable rather than a hardcoded value.
>
> The payoff is that changes stay contained. A brand colour change is a one-line edit in the palette. A dark mode toggle is a CSS repaint with no React re-render. Adding high contrast mode meant adding one mapping file and touching zero components. On top of that we used MUI's `defaultProps` to lock down the defaults, so teams got the correct-looking component without having to make styling decisions.
>
> I also led migrating a high-traffic customer-facing mobile web app off its legacy styling onto Subzero, which is where the consistency and accessibility wins actually showed up."

**Why this works:** it names the architecture, states the benefit in terms of *contained change*, and ends with evidence that it was adopted, not just built. Adoption is what separates a design system from a component folder.

---

## Part 1 — The five-step chain

Before the detail, hold this in your head:

**Define → Name → Variable → Reference → Render**

| Step | File | What it holds |
|---|---|---|
| Define | `PALETTE.ts` | `#97144D` |
| Name | `getColorScheme/light.ts` | `actionPrimary → PALETTE.primary` |
| Variable | `Theme/index.ts` | `--ds-colour-actionPrimary: #97144D` |
| Reference | `DsButton.Overrides.ts` | `background: var(--ds-colour-actionPrimary)` |
| Render | browser | a magenta button |

Each arrow is a layer of **indirection**. Indirection sounds like unnecessary complexity until you understand that each layer absorbs a different *kind* of change. That's the whole thesis and we'll come back to it in Part 3.

---

## Step 1 — Define: the raw value

### The code

```ts
// PALETTE.ts
export const PALETTE = {
  // brand
  primary:    '#97144D',
  primary100: '#F14687',

  // neutrals
  neutral0:   '#FFFFFF',
  neutral100: '#F5F5F5',
  neutral900: '#0A0A0A',

  // feedback
  success:    '#0C8B51',
  error:      '#D32F2F',
} as const;
```

### In plain words

This is a flat dictionary of colours. Nothing more. `primary` does **not** mean "the button colour" — it's just a magenta that happens to be our brand colour. These are called **primitive tokens** or **global tokens**.

The key rule: **nothing outside the design system ever imports from this file.** Not consumer apps, not even most of your own components. Only the colour scheme files (Step 2) are allowed to read it.

The `as const` matters — it makes TypeScript infer the literal string types instead of widening to `string`, which is what lets you derive union types for autocomplete later.

*Mental picture:* paint cans on a shelf in a hardware store, labelled by colour code. Nobody has decided what gets painted yet.

### Say this in the interview

> "The bottom layer is `PALETTE.ts` — a flat map of primitive tokens. These are raw values with no semantic meaning attached; `primary` is just a hex code, it doesn't know it's going to end up as a button background. It's deliberately dumb, and it's deliberately private — only the colour scheme layer reads from it. Product code never touches it."

### Follow-up you'll get

**"How do you stop a product engineer importing `PALETTE.primary` directly?"**

> "Partly by not exporting it from the package's public entry point. Partly through types — our spacing and colour props were TypeScript union types, so an arbitrary value fails at compile time. Honestly though, the strongest guardrail is a lint rule, and that's something we didn't have and I'd add. A custom ESLint rule that flags raw hex literals and deep imports into internal paths. Documentation asks people to do the right thing; tooling makes it the only option."

That last sentence is worth landing. Admitting a gap *and* naming the fix reads as senior. Pretending you had everything reads as junior.

---

## Step 2 — Name: the value gets a job

### The code

```ts
// getColorScheme/light.ts
import { PALETTE } from '../PALETTE';

export const light = {
  actionPrimary:      PALETTE.primary,     // #97144D
  actionPrimaryHover: PALETTE.primary100,
  surfacePrimary:     PALETTE.neutral0,
  typoPrimary:        PALETTE.neutral900,
  typoOnSurface:      PALETTE.neutral0,    // text sitting on a filled surface
};
```

```ts
// getColorScheme/dark.ts
import { PALETTE } from '../PALETTE';

export const dark = {
  actionPrimary:      PALETTE.primary100,  // #F14687 — brighter, readable on dark
  actionPrimaryHover: PALETTE.primary,
  surfacePrimary:     PALETTE.neutral900,
  typoPrimary:        PALETTE.neutral0,
  typoOnSurface:      PALETTE.neutral900,
};
```

```ts
// getColorScheme/highContrast.ts — same keys again, maximum-contrast values
```

### In plain words

This is the most important layer in the entire system, and the one candidates usually skip past.

Every file here exports the **same set of keys** with **different values**. The key name describes a *role* — what the colour is *for* — not what it looks like. `actionPrimary` means "the colour a primary clickable thing should be." `typoOnSurface` means "text colour when sitting on top of a filled surface."

These are called **semantic tokens** or **alias tokens**.

Two things fall out of this immediately:

1. **Theming becomes free.** Three files, identical keys, different values. Nothing downstream needs to know which one is active.
2. **Design intent becomes editable independently of the palette.** If a designer decides primary actions should use a different colour, you change the *mapping*, not the palette and not the components.

*Mental picture:* a job sheet for a painter. "Front doors get the magenta can." A separate night-time job sheet says "front doors get the bright pink can." The paint shelf didn't change; the assignment did.

### Say this in the interview

> "The middle layer is the semantic layer. Each colour scheme file exports the same set of keys — `actionPrimary`, `typoOnSurface`, `surfacePrimary` — but points them at different primitives. The names describe purpose, not appearance.
>
> This layer is the whole reason the system works. If components read the palette directly, then every kind of change becomes a find-and-replace across the codebase. With the semantic layer in the middle, a brand change is a palette edit, a design-intent change is a mapping edit, and a mode change is just a different file. None of those touch component code."

### Follow-up you'll get

**"Why not just have components read the palette directly? It's one less layer."**

This is *the* question on tokens. Have the concrete example ready.

> "Because the two things change for different reasons and at different rates. Say the brand colour changes from magenta to blue — that's a palette edit, one line, and it's genuinely the same colour everywhere. But now say the designer decides primary buttons should use our secondary colour instead, while the brand colour stays as it is. If components read `PALETTE.primary` directly, that's a hunt through every file that references it, deciding case by case whether this particular usage is 'the brand colour' or 'the primary action colour' — and those had been the same value, so nothing in the code distinguishes them. With the semantic layer, it's one line: `actionPrimary` now points somewhere else. The indirection is what preserves the *distinction between two things that happen to be the same colour today*."

**"How did the values get into these files? Was Figma the source of truth?"**

Be honest here — this comes up because the JD mentions tokenization and Figma collaboration.

> "In practice designers gave us values and we transcribed them into the palette manually. That works at our scale but it's a drift risk — if a designer updates Figma and nobody tells us, the two diverge silently. The proper setup is Figma Variables or Tokens Studio syncing tokens to a Git repo as JSON, then Style Dictionary as a build step generating the palette and the CSS output. That also gives you multi-platform output from one source, which matters if you ever need native alongside web."

---

## Step 3 — Variable: MUI compiles it to CSS

### The code

```ts
// Theme/index.ts
import { extendTheme } from '@mui/material/styles';
import { light } from './getColorScheme/light';
import { dark } from './getColorScheme/dark';
import { componentOverrides } from './componentOverrides';

export const dsTheme = extendTheme({
  cssVarPrefix: 'ds',
  colorSchemes: {
    light: { palette: { colour: light } },
    dark:  { palette: { colour: dark  } },
  },
  spacing: 4,
  shape: { borderRadius: 8 },
  components: componentOverrides,
});
```

### In plain words

MUI walks that nested object and builds a CSS variable name out of the **path** you nested it under:

```
cssVarPrefix  +  palette key  +  token key
    ds        +    colour     +  actionPrimary
                     ↓
        --ds-colour-actionPrimary
```

The generated stylesheet looks like this:

```css
:root {
  --ds-colour-actionPrimary: #97144D;
  --ds-colour-typoOnSurface: #FFFFFF;
}

[data-mui-color-scheme="dark"] {
  --ds-colour-actionPrimary: #F14687;
  --ds-colour-typoOnSurface: #0A0A0A;
}
```

**Read those two blocks again, because this is the single most important mechanic in the system.** Both themes are present in the CSS *at the same time*. Nothing is computed at toggle time. Switching mode just changes an attribute on the `<html>` element, which changes which CSS rule wins, and the browser re-resolves the variable and repaints.

Compare that with the normal MUI approach, where `backgroundColor: theme.palette.primary.main` is a JavaScript value. Toggling the theme changes the theme object, which changes context, which re-renders every component that consumes it, which makes Emotion regenerate and re-inject style tags. On a large app that's visible jank.

*Mental picture:* a notice posted at the building entrance. "In this building, `actionPrimary` means the magenta can — unless it's night, in which case bright pink." The painters don't need new instructions when night falls; they just read the notice again.

### Say this in the interview

> "The theme layer is where the semantic tokens become CSS custom properties. MUI derives the variable name from the nesting path plus our `ds` prefix, so `palette.colour.actionPrimary` becomes `--ds-colour-actionPrimary`.
>
> The important part is what the output looks like: both colour schemes are emitted into the stylesheet simultaneously — light under `:root`, dark under a colour-scheme attribute selector. So switching theme doesn't recompute anything. The attribute on the root element changes, a different CSS rule wins, and the browser repaints. Zero React re-renders. With a JavaScript-valued theme you'd be re-rendering the whole tree and having Emotion re-serialise and re-inject styles on every toggle."

### Follow-up you'll get

**"So there's no runtime CSS-in-JS cost at all?"**

Careful — this is a trap, and getting it right is a strong signal.

> "Not quite, and it's worth separating the two. MUI's style engine is Emotion, which is runtime CSS-in-JS, so on initial render Emotion still serialises our `styleOverrides`, hashes them and injects style tags. That cost is still there. What CSS variables specifically eliminated is the cost of *theme switching* — that's the operation we decoupled from the JS pipeline.
>
> If I wanted to remove the runtime cost entirely, the answer is a zero-runtime solution — MUI's own Pigment CSS, or vanilla-extract or Panda if starting from scratch. You author in TypeScript, styles get extracted to static CSS at build time, and you keep CSS variables for theming. That also makes it SSR-safe and compatible with React Server Components, which runtime CSS-in-JS isn't."

**"You're on MUI v7 — is `extendTheme` still the recommended API?"**

Check your actual repo before the interview. MUI shifted this during the v6 cycle: the current path is `createTheme({ cssVariables: true })` with the regular `ThemeProvider`, and the CSS-vars-specific APIs were deprecated. You want to raise this yourself.

> "We're on `extendTheme` and `CssVarsProvider`, which was the CSS-variables API at the time we built it. MUI has since folded that into the main API — it's `createTheme({ cssVariables: true })` with the standard `ThemeProvider` now, and the old entry points are deprecated. Migrating is on my list; the underlying variable generation is the same, it's mostly an import and call-site change."

**"What if you need to theme just one part of the page differently?"**

> "Two options. Nested providers, which gives you a fully separate theme scope. Or — cheaper and usually sufficient — set the colour-scheme attribute on a wrapper element rather than the root. Since the variables are defined by an attribute selector, any subtree with that attribute picks up the other scheme automatically, with no extra JavaScript. That's a genuine advantage of the CSS-variable approach over context-based theming, where you'd need another provider and another re-render boundary."

---

## Step 4 — Reference: components point at the variable

### The code

```ts
// DsButton.Overrides.ts
export const MuiButton = {
  defaultProps: {
    variant: 'contained',
    size: 'small',
    disableElevation: true,
    disableRipple: false,
  },
  styleOverrides: {
    root: {
      borderRadius: 8,
      padding: '8px 16px',
      textTransform: 'none',
      fontWeight: 500,
      transition: 'background-color 150ms ease-out',
    },
    containedPrimary: {
      backgroundColor: 'var(--ds-colour-actionPrimary)',
      color: 'var(--ds-colour-typoOnSurface)',
      '&:hover': {
        backgroundColor: 'var(--ds-colour-actionPrimaryHover)',
      },
    },
  },
};
```

```ts
// componentOverrides.ts
import { MuiButton } from './DsButton.Overrides';
import { MuiChip } from './DsChip.Overrides';
import { MuiDialog } from './DsDialog.Overrides';

export const componentOverrides = { MuiButton, MuiChip, MuiDialog };
```

### In plain words

Notice what is **not** in that file: any hex code. The button does not know it's magenta. It knows it wants "the primary action colour," and it asks CSS what that currently is.

There are two genuinely different things happening in this one file, and you should name them separately because they solve different problems:

**`styleOverrides` is styling.** Visual rules, all colour values expressed as `var(--ds-colour-*)`.

**`defaultProps` is governance.** Every button across every consumer app is `contained`, `small`, no elevation — unless someone deliberately opts out. This is the "pit of success" idea: the laziest thing a developer can do is also the correct thing. They type `<DsButton>Save</DsButton>` and get a correct, on-brand, accessible button without making a single styling decision.

This is your answer to "raw MUI gives developers too much freedom." Restricting choice is a *feature* of a design system, not a limitation.

*Mental picture:* a blueprint that says "paint the front door `actionPrimary`" rather than "paint the front door magenta."

### Say this in the interview

> "Component overrides are where the tokens get consumed, and the notable thing is there are no colour values in these files at all — everything is `var(--ds-colour-…)`. The component is completely ignorant of what colour it renders as, which is exactly why a theme change or a brand change never requires touching it.
>
> The other half of this layer is `defaultProps`, which is really governance rather than styling. We set sensible locked defaults — contained variant, small size, no elevation — so a developer writing `<DsButton>Save</DsButton>` gets the correct component without deciding anything. Raw MUI lets you pass any colour, any size, any variant, and that's how you end up with fourteen slightly different buttons across an app. Narrowing the API was one of the main reasons we wrapped MUI rather than using it directly."

### Follow-up you'll get

**"MUI has an `sx` prop. Doesn't that let anyone bypass all of this?"**

Yes, and this is the sharpest question in this whole area. Have a real answer.

> "It does, and it's the biggest hole in any lockdown strategy — someone can write `sx={{ backgroundColor: '#ff0000', padding: '13px' }}` and every token boundary is gone.
>
> The way I'd think about it is tiered rather than absolute. The default path is sanctioned, token-based props that cover the large majority of cases. Then there's a deliberate, documented escape hatch, because if there's no escape hatch teams fork your component and you lose visibility completely. The critical part is that the escape hatch is *visible* — you can lint for it, you can grep for it, and when the same override keeps appearing in three different apps, that's a signal the pattern should be promoted into the system properly. A design system that says no to everything gets abandoned; one that says yes to everything stops being a system. Making exceptions expensive but possible is the balance."

**"Did you build React wrapper components, or just theme configuration?"**

Know the answer for your own codebase. If you wrote wrappers, be ready to talk about:

- **Prop narrowing** — `Omit<ButtonProps, 'color' | 'variant'>` and re-exposing a constrained union type
- **Ref forwarding** — a wrapper *must* forward refs or Tooltip, Popover, and focus management break for consumers
- **Module augmentation** — `declare module '@mui/material/Button'` to add your own variants to `ButtonPropsVariantOverrides`, plus augmenting `Theme` and `Palette` for your custom token keys

That last one is worth volunteering unprompted. It's specific, non-obvious, and proves you were hands-on rather than copying a tutorial.

---

## Step 5 — The consumer app wires it up

### The code

```tsx
// consumer app root
import { CssVarsProvider } from '@mui/material/styles';
import { dsTheme } from '@subzero/theme';

export default function App() {
  return (
    <CssVarsProvider theme={dsTheme} defaultMode="light">
      <Routes />
    </CssVarsProvider>
  );
}
```

```tsx
// anywhere in a feature
import { DsButton } from '@subzero/components';

<DsButton onClick={handleTrack}>Track order</DsButton>
```

### In plain words

The provider does exactly two jobs. Know both, because "what does the provider actually do?" is a fair question and a lot of people can't answer it.

1. **Injects the CSS variables** into the document, so `--ds-colour-actionPrimary` actually resolves to something.
2. **Puts the theme object into React context**, so when MUI renders a `Button` it finds your `styleOverrides` and `defaultProps`.

If you forget the provider, you get an unstyled MUI default button, because the variables don't exist and the overrides were never registered.

**The SSR flash:** if the app is server-rendered, the server doesn't know the user's preferred mode, so the first paint can be light and then snap to dark. MUI ships `InitColorSchemeScript` — a small blocking script that sets the attribute before first paint. Know whether you handled this or not; "we had a known flash on first load and the fix is the init script" is a perfectly good answer.

### Say this in the interview

> "The consumer wraps their root in our provider, which does two things: injects the CSS variables into the document so the custom properties resolve, and puts the theme into React context so MUI picks up our overrides and defaults. From there teams just import `DsButton` and use it — no styling decisions, no theme knowledge required on their side.
>
> One thing worth flagging is the server-rendering case: the server can't know the user's mode preference, so you can get a flash of the wrong theme on first paint. MUI's init script sets the colour-scheme attribute before paint to avoid it."

---

## Step 6 — Render: the browser resolves it

### What actually happens

Emotion generates a class from your overrides:

```css
.css-1x4y7z {
  border-radius: 8px;
  padding: 8px 16px;
  text-transform: none;
  font-weight: 500;
  background-color: var(--ds-colour-actionPrimary);
  color: var(--ds-colour-typoOnSurface);
}
```

The browser resolves `var(--ds-colour-actionPrimary)` by walking up the DOM until it finds a definition — `:root` in light mode, or the colour-scheme block if that attribute is set on an ancestor. It finds `#97144D` and paints.

**Result:** a magenta button, white label, 8px radius, 8px vertical and 16px horizontal padding.

### Why this is a genuinely nice property

Open DevTools and you can *see* `--ds-colour-actionPrimary: #97144D` in the Styles panel, edit it live, and watch every button on the page change at once. With a JavaScript-valued theme, DevTools shows you the computed `#97144D` and no trail back to where it came from. Your design system becomes inspectable and debuggable by anyone.

### Say this in the interview

> "At render time Emotion generates the class, and the browser resolves the custom property by walking up the DOM for the nearest definition. The nice side effect is that the whole system is visible in DevTools — you can see the variable name, trace it, and live-edit it to test a value across the entire app instantly. With JS-based theming you only see the final computed value with no link back to the token, so debugging a colour means reading source."

---

## Part 3 — The part that proves you understand it

Everything above is *description*. This section is *understanding*. If you only have time to nail one section of this guide, make it this one — because "why did you build it this way" is worth more than "what did you build."

The claim: **each layer absorbs a different kind of change, and a change never travels further than it has to.**

### Scenario A — brand rebrands to blue

```diff
  // PALETTE.ts
- primary: '#97144D',
+ primary: '#0055FF',
```

Flows down through all three colour schemes, all CSS variables, every component, in light and dark and high contrast.

**Files changed: 1. Components touched: 0.**

### Scenario B — user toggles dark mode

Nothing recomputes. The colour-scheme attribute on the root element flips, a different CSS rule wins, the browser repaints affected pixels. React doesn't know a colour changed — no re-render, no virtual DOM diff, no Emotion re-injection.

**This is why Step 3 emits both schemes up front** instead of computing the active one.

### Scenario C — designer remaps primary actions

```diff
  // getColorScheme/light.ts
- actionPrimary: PALETTE.primary,
+ actionPrimary: PALETTE.secondary100,
```

The palette is untouched. Every component is untouched. Only the *assignment* changed.

**This is the scenario that justifies the semantic layer existing.** Without it, you'd be searching for `PALETTE.primary` and deciding case by case whether each usage meant "the brand colour" or "the primary action colour" — two ideas that had the same value, so nothing in the code told them apart.

### Scenario D — add high contrast mode for accessibility

Add one file exporting the same keys with maximum-contrast values, register it as a third colour scheme.

**Components touched: 0.**

Without tokens this would be conditional logic inside every single component.

### Say this in the interview

> "The way I'd summarise the architecture is that each layer absorbs a different class of change. Primitives absorb brand changes. The semantic layer absorbs design-intent changes. The scheme files absorb mode changes. Component overrides absorb styling changes. The point of the indirection is that a change never propagates further than it has to — a rebrand is one line, adding high contrast mode was one file and zero component edits, and a mode toggle doesn't reach JavaScript at all."

---

## Part 4 — Shipping it

The runtime story is Steps 1–6. There's a packaging story too, and the JD explicitly asks for "own the maintenance and versioning of the design system packages," so expect questions here.

```json
{
  "name": "@subzero/components",
  "type": "module",
  "sideEffects": false,
  "exports": {
    ".":         { "types": "./dist/index.d.ts", "import": "./dist/index.js" },
    "./button":  { "types": "./dist/button.d.ts", "import": "./dist/button.js" }
  },
  "peerDependencies": {
    "react": "^18 || ^19",
    "@mui/material": "^7"
  }
}
```

Points to be able to defend:

- **`peerDependencies`, never `dependencies`,** for React and MUI. Two copies of React breaks hooks. Two copies of Emotion breaks style injection order, so your overrides start losing to MUI's defaults non-deterministically. This is a concrete, memorable reason — use it.
- **`sideEffects: false`** lets bundlers drop components a consumer never imports.
- **Subpath `exports`** so consumers can import `@subzero/components/button` directly. Note honestly that a single root barrel file re-exporting a hundred components can *hurt* tree-shaking rather than help it — it forces the bundler to pull in and analyse the whole module graph, and any module-level side effect anywhere in that graph pins everything.
- **Build output** — ESM plus type declarations via `tsup` or Vite library mode.

### One correction to make to your own notes

Your notes claim 70–90% bundle reduction from tree-shaking. Be careful stating that, for two reasons: barrel exports can defeat tree-shaking rather than enable it, and `@mui/material` plus Emotion dominates the bundle anyway, so saving your own component code moves the needle less than it sounds.

The stronger version of the claim is about the *consumer* app:

> "The number I'd actually want to quote is what migrating the mobile web app did to its total JavaScript — that's the number that matters to a user. I don't have it precisely to hand, which is a gap; a bundle-size budget enforced in CI is the first thing I'd add so that claim is measured rather than asserted."

That is a better answer than a confident wrong number.

---

## Part 5 — Accessibility: separate what you inherited from what you owned

Your resume says the migration improved accessibility. Expect "how, specifically?" Split the answer, because the second half is the part that's actually yours.

**Inherited from MUI:** ARIA roles, keyboard navigation, focus trapping in dialogs, `aria-*` wiring on form controls. Real value — say so plainly, don't claim credit for it.

**Genuinely yours:**

- **Contrast ratios on your own palette.** MUI cannot help here. `#97144D` on white, and every semantic pairing like `typoOnSurface` over a filled surface, needs 4.5:1 for body text and 3:1 for large text and UI boundaries. If you checked these, that's a real contribution. If it was informal, say so and note that automated contrast validation belongs in the token build step.
- **The high contrast colour scheme.** This is your best accessibility story, because it's the token architecture paying off concretely — a whole accessibility mode delivered by adding one mapping file and changing zero components. Lead with it.
- **What the legacy code was doing wrong.** Hardcoded colours failing contrast, `<div onClick>` instead of `<button>`, missing focus states. If you can name one specific thing you found and fixed during the migration, the claim becomes credible instead of generic.

> "Two different things. MUI gave us keyboard navigation, focus trapping and ARIA wiring for free, and I wouldn't claim credit for that. What was genuinely ours was the colour layer — MUI can't validate that our brand magenta clears 4.5:1 against white, or that our on-surface text pairings work. And the clearest win was high contrast mode: because every component read semantic tokens rather than values, we delivered a full accessibility mode by adding one mapping file and touching zero components. On the migration side, the legacy screens had hardcoded colours that failed contrast and clickable divs without focus states, and moving them onto Subzero fixed those by construction rather than by audit."

---

## Part 6 — Where you're thin, and how to answer honestly

Don't bluff these. "Here's what we did, here's the tradeoff we accepted, here's what I'd do differently" scores higher than a confident wrong answer, every time.

### Headless architecture

Subzero is the *opposite* of headless — you consumed styled components and re-skinned them. The JD lists headless as preferred. Use MUI's own layering to show you understand the seam:

> "MUI is itself layered — Base UI is the headless layer with behaviour, state and accessibility and no styles; Material UI is the styled layer on top. We consumed the styled layer, which was right for our constraint: one brand, three modes, ship fast. The cost is coupling to Material's DOM structure and visual assumptions, so a component Material doesn't model means fighting the library instead of composing. For a system that has to serve genuinely divergent visual languages I'd invert it — build on Base UI or Radix, own styling completely, and keep behaviour and appearance in separate packages. The tradeoff there is that consistency erodes, because every team styles independently, so you end up needing a preset or recipe layer on top anyway."

Vocabulary to have: Radix (compound components, `asChild`), React Aria (hooks returning prop objects you spread), Base UI, CVA for variant mapping, shadcn/ui as a distribution model rather than a library.

### Release engineering

Likely a real gap. Even if the honest answer is "we versioned manually and coordinated over Slack," describe what *should* exist:

- **Semver for a component library** — what counts as breaking: removing a prop, renaming a CSS custom property (yes, `--ds-colour-actionPrimary` is public API the moment a consumer references it), changing a default, tightening a type. Purely visual changes are the hard case; have a stated policy.
- **Changesets** for versioning and changelogs.
- **Release channels** — canary off main, snapshot builds per PR so consumers can validate before merge.
- **Deprecation path** — `@deprecated` JSDoc, dev-only console warnings, keep for N minors, remove in a major, ship a **codemod** so consumers don't migrate by hand.

The scenario question is: *"You need to break the Button API. Many teams depend on it. Walk me through it."* Answer: add the new API alongside the old, deprecate with dev warnings, publish a codemod and migration guide, track adoption, remove in the next major after a long runway.

### Testing

Also likely thin. The one you must name is **visual regression** — Chromatic or Playwright screenshots — because for a design system, CSS changes are invisible to unit tests and a regression breaks every downstream app at once. Plus React Testing Library with role-based queries for behaviour, `jest-axe` in CI, and type-level tests since your public types *are* your API.

---

## Part 7 — Your migration story needs numbers

"Revamped the UI architecture of a high-traffic customer-facing mobile application" will get pulled on hard, and right now it has no scale, no obstacles and no numbers. Reconstruct before the interview:

- **Scale** — how many screens, components, engineers. What the app actually was.
- **Strategy** — almost certainly incremental, which raises the interesting question: how did legacy and Subzero coexist? Two providers? Specificity collisions between legacy CSS and Emotion's injected styles? Whatever the ugly reality was, *that's the good story* — interviewers respect collision handling because it's a real problem.
- **Resistance** — who pushed back and why. "Your button doesn't support X." How you handled it. This is the governance question in disguise.
- **Numbers** — lines of CSS deleted, component count before and after, a feature that took a week and now takes a day, count of hardcoded hex values eliminated. Rough figures with an honest "approximately" beat "significantly reduced."

---

## Part 8 — One-page revision sheet

**The chain:** Define → Name → Variable → Reference → Render

**The one-line thesis:** each layer absorbs a different kind of change, so a change never travels further than it has to.

**Three killer facts:**
1. Both colour schemes are in the CSS simultaneously, so switching theme is a repaint, not a re-render.
2. High contrast mode = one new mapping file, zero component changes.
3. `defaultProps` is governance — the lazy path is the correct path.

**Three things to admit before you're asked:**
1. No ESLint rule enforcing token usage — types were the only guardrail.
2. Emotion is still runtime CSS-in-JS; only *theme switching* was decoupled from JS.
3. Tokens came from designers manually, not synced from Figma.

**Three questions to be ready for:**
1. Why not read the palette directly? → the remap scenario
2. Doesn't `sx` bypass everything? → tiered escape hatch, made visible
3. Is this headless? → no, and here's the tradeoff we accepted

**One thing to check in your repo tonight:** `extendTheme` vs `createTheme({ cssVariables: true })`. You're on v7; raise the deprecation yourself rather than getting caught.

**One thing to ask your recruiter:** the invite says "Senior Manager," the JD body says "Senior Developer," you're expecting SDE-2. Those three calibrate very differently on strategy versus implementation. Find out which track this is.