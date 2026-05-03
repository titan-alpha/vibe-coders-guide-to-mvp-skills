# 02 · Design

Goal: lock in a visual direction *before* writing UI components. Re-skinning later is the most expensive thing a vibe coder can do.

## DIALOGUE — ask the user

1. **Mood.** "Pick two: minimal · playful · serious · luxurious · technical · friendly · brutalist · soft." Then ask why, in one sentence.
2. **Reference.** "Name one site or app whose aesthetic you'd be proud to be compared to." If they can't name one, suggest 3 from different ends of the spectrum (e.g., Linear, Stripe, Notion, Are.na, Vercel) and ask which feels right.
3. **Density.** "Spacious and breathable, or dense and informational?"
4. **Color.** "Do you want a single accent color, or a gradient identity? Any colors you absolutely want or want to avoid?"
5. **Light/dark.** "Light only, dark only, or both with system default?"

## Order of decisions (do these in sequence)

*Color and font set the platform's voice subconsciously; the logo references both. Doing them in this order means the logo doesn't fight the surrounding language.*

1. Re-read `PROJECT.md`'s `# Idea` and `# Audience` and pick a tone label from this list: **editorial · tech · friendly · luxurious · playful · brutalist**.
2. Pick the color scheme &mdash; informed by color-theory analysis (six dimensions, see Item #2 below), then propose 2 palettes, user picks.
3. Pick the display font &mdash; from the curated list, matched to tone.
4. Pick the logo (see item #9 below).
5. Build the header with the 4-element rule (see item #10 below).
6. Then everything else &mdash; type scale, spacing, components.

Items 2&ndash;4 are creative decisions; the user always sees and picks. Item 5 is mechanical; the agent just builds it.

## AUTONOMOUS — set up the design system

1. **Stack:** Next.js 15 (App Router) + Tailwind v4 + DaisyUI. If the project is not yet scaffolded, run `npx create-next-app@latest . --typescript --tailwind --app --eslint --no-src-dir --turbopack --import-alias "@/*"`, then install DaisyUI: `npm install -D daisyui@latest` and add `@plugin "daisyui";` to `app/globals.css`.
2. **Theme:** generate a custom DaisyUI theme that reflects the mood and color answers, **anchored to the tone label picked in the Order of decisions step**. Provide both light and dark unless the user opted out. Put the theme in `globals.css` via `@plugin "daisyui/theme"`. Use **OKLCH** values throughout.

   Color rubric by tone &mdash; agent proposes 1&ndash;2 palettes, user picks:
   - **Editorial** &mdash; muted neutrals + one strong accent (often a deep blue, oxblood, or forest green).
   - **Tech** &mdash; cool/neutral base + electric accent (purple, cyan, lime).
   - **Friendly** &mdash; warm base + saturated accent (coral, sage, butter yellow).
   - **Luxurious** &mdash; near-black + gold, or off-white + deep brown.
   - **Playful** &mdash; high-saturation palette, **3 colors not 1**.
   - **Brutalist** &mdash; black + white + one neon (yellow, red, magenta).

   State the proposed palette(s) and reasoning before applying. The user can override.

   ### Color theory analysis (do this before proposing the palette)

   The per-tone rubric above is the *quick fallback*. Before proposing palettes, walk through these six dimensions of established color theory and write a short note on each. This makes the palette a reasoned choice, not a vibe.

   1. **Color psychology by audience.** Warm hues (red, orange, yellow) bias the viewer toward arousal, urgency, appetite. Cool hues (blue, green, purple) bias toward calm, trust, focus. Neutrals (gray, beige, off-white) bias toward neutrality and let other elements speak. Pick a base orientation that matches the *intended emotional state* of the user when they're using the product. *(Research basis: Mehta &amp; Zhu 2009; Elliot &amp; Maier 2014; Labrecque &amp; Milne 2012.)*

   2. **Cultural color associations for the target audience.** Re-read `PROJECT.md`'s `# Audience` section. If the audience is global or non-Western, check the dominant region's color associations:
      - **White** &mdash; purity (US/EU); mourning (parts of East Asia).
      - **Red** &mdash; passion / danger (US/EU); luck / celebration (China); mourning (parts of Africa).
      - **Yellow** &mdash; caution / cheer (US/EU); royalty (Southeast Asia); mourning (Egypt).
      - **Green** &mdash; nature / money / "go" (US); Islam / luck (Middle East / Ireland).
      - **Purple** &mdash; luxury / royalty (most cultures); mourning (Latin America / Thailand).
      - **Blue** &mdash; trust / corporate (most cultures, the most-universally-liked hue).

      When in doubt, lean blue &mdash; it's the most cross-culturally safe.

   3. **Color harmony scheme.** Pick one of the five canonical harmonies (Itten / Munsell color theory):
      - **Monochromatic** &mdash; single hue, multiple lightness/saturation values. Calm, focused, easy to balance. Default for editorial / luxurious / brutalist.
      - **Analogous** &mdash; 3 adjacent hues on the color wheel (e.g., blue, blue-green, green). Harmonious and natural. Good for friendly / wellness / lifestyle.
      - **Complementary** &mdash; two hues opposite on the wheel (e.g., blue + orange). High contrast, bold, energetic. Use sparingly &mdash; easily becomes loud.
      - **Triadic** &mdash; three hues evenly spaced (e.g., red, yellow, blue). Vibrant. Default for playful.
      - **Split-complementary** &mdash; base hue + the two hues adjacent to its complement. Like complementary but softer. Versatile, modern. Good default for tech / SaaS.

   4. **Contrast budget.** WCAG 2.2 AA requires &ge; 4.5:1 contrast for body text and &ge; 3:1 for large text and UI components. Pick foreground/background pairs that *start* above 7:1 (AAA) so the design has headroom for nuance without breaking accessibility. Use OKLCH lightness deltas &mdash; pairs whose `L` values differ by at least 0.50 reliably hit 4.5:1.

   5. **60-30-10 distribution rule.** Healthy palettes follow ~60% neutral / dominant base, ~30% secondary, ~10% accent. Map this to DaisyUI's tokens explicitly:
      - **60%**: `base-100` (background) and `base-200` (cards / surfaces).
      - **30%**: `base-content` (text), `neutral` (borders, secondary buttons).
      - **10%**: `primary` (CTAs, brand moments). Plus tiny doses of `accent` for state highlights (success / warning / error already covered by DaisyUI's `success` / `warning` / `error` semantic tokens &mdash; keep them).

   6. **Vertical / brand-color research.** Quickly think about the dominant color in the product's vertical:
      - Fintech / banking &rarr; blues, deep greens (trust + money associations).
      - Healthcare / wellness &rarr; calming greens, soft blues, warm neutrals.
      - Developer tools &rarr; cool dark themes, electric accents (purple, cyan, lime).
      - Food / hospitality &rarr; warm earth tones, appetite-stimulating reds/oranges (avoid blue &mdash; research shows blue suppresses appetite).
      - Fashion / luxury &rarr; near-black + metallic accent.
      - Children / education &rarr; bright primaries; high saturation.

      *Don't blindly conform &mdash; sometimes contrarian color (e.g., a fintech that goes warm) is the brand differentiator. Note the convention, then decide intentionally.*

   After working through these six points, **propose 2 candidate palettes** in chat as small inline color swatches:

   ```
   Palette A — "Trust-forward fintech"
     Primary  oklch(0.55 0.18 240)   #2563eb   ▮ deep electric blue
     Accent   oklch(0.78 0.15 50)    #f59e0b   ▮ amber
     Neutral  oklch(0.20 0.01 240)   #0f172a   ▮ near-black slate
     Surface  oklch(0.98 0.005 240)  #f8fafc   ▮ off-white

   Palette B — "Warmer, optimistic"
     Primary  oklch(0.55 0.18 240)   ▮ same blue
     Accent   oklch(0.72 0.20 25)    ▮ coral
     Neutral  oklch(0.30 0.02 60)    ▮ warm dark brown
     Surface  oklch(0.97 0.01 90)    ▮ ivory
   ```

   Then ask:

   > *"Both pass AA contrast and follow the 60-30-10 rule. **A** leans cooler / more authoritative; **B** leans warmer / more approachable. Which fits the audience better &mdash; or want me to iterate on either?"*

   User picks one. Apply it. Note the choice and the reasoning in `STATE.md # Decisions`.
3. **Icons:** install **Lucide React** (`npm install lucide-react`) and use it for every icon. It has 1400+ icons, perfect stroke consistency, and tree-shakes per-import so bundle stays tiny. Don't mix icon libraries; don't hand-roll SVGs for icons that Lucide already has.
   ```tsx
   import { ArrowRight, Check, Copy } from 'lucide-react';
   <ArrowRight className="w-4 h-4" />
   ```
4. **Type scale:** one **deliberately picked** display font + system-stack body font, at 56/40/28/20/**16**. **16px is the floor.** No body text, caption, label, or nav item smaller than 16px (`text-base`). The only allowed exceptions are technical metadata like timestamps, version numbers, or inline code badges &mdash; and even those should prefer 14px (`text-sm`) over smaller. Never ship `text-xs` for readable content. No more than two font weights.

   **Display font is a tone decision, not a default.** The agent picks based on the platform's emotional tone (audience, mood, vertical from sub-skill 01). Body font stays as the system stack. The picked display font goes in `globals.css` via `@font-face` from Google Fonts (weight subset only).

   Curated display fonts by tone:
   - **Editorial / serious / publication-like** &mdash; Fraunces (variable serif) or Spectral. Trustworthy, considered.
   - **Tech / dev tool / SaaS** &mdash; Inter Tight, Geist, or Space Grotesk. Crisp, modern.
   - **Friendly / consumer / lifestyle** &mdash; General Sans, Plus Jakarta Sans, or DM Sans. Warm, rounded.
   - **Luxurious / premium** &mdash; Cormorant Garamond or Playfair Display. Elegant, high-contrast.
   - **Playful / creative** &mdash; Recoleta, S&ouml;hne, or Bricolage Grotesque. Distinctive, expressive.
   - **Brutalist / technical / raw** &mdash; JetBrains Mono or IBM Plex Mono used as display. Industrial.

   State the choice and reasoning to the user before applying. The user can override.
5. **Text content is minimal wherever possible.** Cut every word that isn't earning its place. One-sentence descriptions beat paragraphs. Button labels are verbs (`Save`, `Send`, `Create`), not phrases. Headings are the shortest fragment that names the section. Use whitespace and hierarchy instead of prose to convey structure.
6. **No gradients on buttons or titles.** Solid theme colors only. Gradient backgrounds on interactive elements or headings read as dated and reduce contrast predictability across themes. Gradients are fine for purely decorative background layers (hero glow, illustration accents) &mdash; not for anything the user reads or clicks.
7. **Spacing:** stick to Tailwind's default scale &mdash; do not invent custom spacing values.
8. **Component primitives:** build only what you need from DaisyUI (`btn`, `card`, `badge`, `input`, `alert`, `navbar`). Do not pre-build a component library.
9. **Logo + favicon (same file).** Every product needs a simple mark. The logo you design *is* the favicon &mdash; one SVG, no variants.

   **Design process:**
   1. **DIALOGUE:** ask the user one question: *"If I had to express what this project does in a single visual idea, what would it be? A letter, a shape, an object, or a metaphor?"* Re-read `PROJECT.md` for the audience and MVP slice before proposing.
   2. **Propose 2&ndash;3 concepts** as inline SVG sketches in chat. Pick from these approaches (don't overthink):
      - **Monogram** &mdash; the first letter (or two) of the project name, styled distinctively (weight, cutout, angle).
      - **Abstract symbol** &mdash; a geometric mark evocative of the domain (e.g., sprout for a garden app, waveform for audio, spiral for knowledge).
      - **Object icon** &mdash; a Lucide-style line icon of the most obvious object in the domain, simplified.
   3. **User picks one.** Iterate max twice. Ship.

   **Constraints (non-negotiable):**
   - `viewBox="0 0 64 64"`, centered composition.
   - **Works at 16&times;16 pixels.** Test by opening at that size; if details vanish, simplify until they don't.
   - **1&ndash;2 solid theme colors.** No gradients on the logo if it will ever appear as a favicon on a white browser-tab bar (gradient tints disappear at 16px). A single primary color almost always reads best.
   - Transparent background. No outer frame rectangle, no pill, no rounded-square backdrop &mdash; the browser / OS composites the logo onto whatever surface it sits on. A backdrop fights the host context.
   - No text inside the logo at favicon sizes (text becomes unreadable). Save wordmarks for the header next to the logo.

   **Files to produce:**
   - `public/favicon.svg` &mdash; the exact logo SVG. This is also what the site renders inline (imported or via `<img src="/favicon.svg">`) for the header and OG image.
   - `app/layout.tsx` metadata:
     ```ts
     export const metadata: Metadata = {
       icons: { icon: '/favicon.svg' },
       title: '…',
       description: '…',
     };
     ```
   - Header: render the same SVG inline at ~28&ndash;40px next to the wordmark. Same file, same paths, same colors.

   **Non-goals at v1:** multiple sizes, animated logos, dark/light variants. One file. Ship.

10. **Header and footer content &mdash; know what belongs where.** Modern design theory separates the *working surface* (header) from the *reference surface* (footer). Keep the header clean; put everything informational in the footer.

    **Header &mdash; exactly 4 elements, always. This is non-negotiable.**

    1. **Platform logo** &mdash; left, links to `/`.
    2. **Platform title** &mdash; next to the logo, semibold, in the curated display font.
    3. **Bell icon** &mdash; right-aligned. **ONLY** rendered if the project has a notification center (sub-skill 07's Notifications tab). Shows unread count as a small badge. Click &rarr; dropdown with the 10 most recent notifications + a "View all" link.
    4. **Hamburger menu** &mdash; rightmost. Opens a dropdown containing every top-level page, plus a **"Settings"** link at the bottom (which goes to `/settings` &mdash; a per-user page where signed-in users edit name, password, notification preferences). For projects without auth, omit Settings.

    **Don't add nav links to the header itself &mdash; all of those go in the hamburger.** The header stays clean: identity (logo + title) on the left, alerts (bell) and access (hamburger) on the right. This rule supersedes any older "2&ndash;5 primary product surfaces in the header" guidance.

    **Footer (everything discoverable but non-essential):**
    - About, Contact (or a `/contact` form, or just `hello@<domain>`).
    - Terms, Privacy, "Do Not Sell or Share" (from sub-skill 08 if applicable).
    - Social links (optional, and only if the brand has a real social presence).
    - Docs, changelog, press (only if they exist).
    - Copyright line.

    The rule: if the user is more likely to *need it once* than to *use it daily*, it belongs in the footer.

11. **One landing/home page first.** Ship a minimal, on-brand landing route before building any feature surface. This anchors the visual direction so everything after it inherits the same language.

## AUTONOMOUS — critical review of user flow

Once the landing page and basic component library exist, **stop and audit before building feature surfaces.** Rushed information architecture is the most expensive mistake a vibe coder can make. Take 10 minutes to reason through the experience as the target user would, then talk it through with the user.

### 1. Classify the platform

Name the product type out loud. Each type has conventions &mdash; leaning into them cuts friction; fighting them adds it.

| Type | Convention to respect |
| --- | --- |
| Single-purpose consumer tool | One big CTA on landing, instant value; no upfront signup if you can avoid it |
| Content / media platform | Grid or feed on the landing; search prominent; auth optional until an action needs it |
| Marketplace (two-sided) | Two clear entry paths ("I'm buying" / "I'm selling") on landing; split shells post-auth |
| SaaS tool (daily use) | Sidebar nav with persistent workspace; search + command palette (⌘K); app shell, not marketing layout |
| Community / social | Feed-centric; post composer always reachable; profile top-right |
| Admin / dashboard | Sidebar sections; dense tables; filters prominent; charts above tables |

### 2. Map the core user journey

Write it as one sentence: *"A user wants to \_\_\_, starting from \_\_\_, and leaves with \_\_\_."* This is the MVP slice from `PROJECT.md`. Everything else is noise for now.

Sequence the minimum user actions from landing to value. Example (note-taking app): **land &rarr; sign up &rarr; create first note &rarr; share**. That's four steps. If you have seven, three of them are probably removable or mergeable.

### 3. For each step, reason through it

Ask these questions out loud for every step. Write the answers in a short scratch note. If an answer is weak, that step needs redesign.

- **Intent.** What is the user trying to do at this moment? State it as a verb.
- **Cognitive load.** How many fields, choices, or pieces of information are on screen? The answer for an MVP step should be small.
- **Mental model fit.** Does the UI match what this user &mdash; given their background &mdash; expects? If the audience is used to "Save" buttons, don't invent autosave without a visible indicator. If they're used to autosave (most modern apps), don't make them click Save.
- **Removability.** Can this step be collapsed into the previous one? Can we default to a reasonable value instead of asking? Can we defer it to after they've seen value?
- **Reversibility.** If they make a wrong choice here, how do they undo it? If they can't, the step is higher-stakes and needs more affordance.

### 4. Audit the information architecture of each page

Walk each page the user will see and answer:

- **Where does the eye land first?** Hero copy, primary CTA, the form, a chart? Is that what you *want* them to see first?
- **Is the primary action the most visually prominent thing?** (`btn-primary`, larger, above-the-fold.)
- **Are secondary actions visually secondary?** (Ghost buttons, smaller, footer position, or hidden in a menu.)
- **Does the page shape match the content?** Marketing pages: centered, single column, generous whitespace. App pages: full-bleed, two-column or sidebar+main, dense.
- **Is the right surface doing the work?** Header for doing, footer for learning, sidebar for navigating, modal for focused decisions. Don't put information-architecture items in the header or primary-action items in the footer.

### 5. Platform-specific checks

- **Mobile users are the majority for most MVPs.** If the audience skews mobile, the primary nav often belongs in a bottom tab bar, not a top hamburger. Thumb reach matters.
- **Desktop-first SaaS:** sidebar nav, keyboard shortcuts (⌘K for command palette), multi-pane layouts for persistent workspaces.
- **Forms:** a single form for &le; 4 fields; multi-step wizard for 5+. Never a 30-field form on one page.
- **Empty states, error states, loading states** &mdash; sketch them for the core flow. Empty states especially: they're the first thing new users see and they're usually an afterthought.

### 6. Write the critique

Produce a short doc &mdash; 5&ndash;10 bullets max &mdash; structured as:

- **What's working and why** (so we don't break it).
- **What feels wrong and why**, each with a proposed fix: *"Signup asks for first + last name before showing value &mdash; defer to account settings, make signup one field (email)."*
- **The single biggest opportunity to simplify.** Pick one.

### 7. DIALOGUE with the user

Share the critique. Ask: *"Want me to apply these now, or keep what's there and iterate after first users try it?"* Let them choose. An opinionated suggestion the user rejects is better than a silent compromise.

Apply what's agreed. Keep rejected suggestions under `# Open questions` in `PROJECT.md` &mdash; they often come back after user testing.

## Anti-patterns to avoid

- Inventing custom color tokens before the theme is locked.
- Adding a UI component library *and* DaisyUI ("just in case").
- **Gradient buttons or gradient headings.** Solid colors only.
- **Text under 16px** for anything the user is expected to read.
- Verbose copy. Cut every word that isn't earning its place.
- Animations on every element. Reserve motion for state changes (reveal, success, error).
- Stock illustrations. Use type, color, and whitespace instead.
- Mixing icon libraries &mdash; one of anything is fine, two of anything is a smell.
- Putting nav links directly in the header. Top-level pages live inside the hamburger dropdown; the header has 4 elements only.
- Defaulting to the system stack as the display font, or picking a color palette before the tone label is named. Tone first, then palette and font.
- Picking colors by feel without checking contrast ratios or cultural connotations. The agent always proposes palettes that pre-pass WCAG AA &mdash; accessibility is a *constraint* on color choice, not a follow-up audit.

## Exit criteria

- The user has approved the landing page screenshot or has clicked through it on `localhost`.
- `globals.css` contains the locked theme (OKLCH values) and the chosen display font loaded via `@font-face` from Google Fonts.
- `public/favicon.svg` exists and renders correctly as both the favicon (browser tab) and the inline header logo.
- A tone label has been picked from the curated list, and color palette + display font were chosen against that tone with the user's approval.
- Color palette was selected via the color-theory analysis (six dimensions), passes WCAG AA contrast, and is recorded in `STATE.md # Decisions` with the reasoning.
- Header has the 4-element layout (logo, title, bell if notifications, hamburger). The hamburger contains every top-level page plus Settings (when auth exists). Footer carries About / Contact / legal.
- The user-flow critique has been written, discussed with the user, and applied where agreed.
- A `# Design` section in `PROJECT.md` captures the chosen tone label, color decisions, display font, logo concept, the core user journey sentence, and any flow-critique items deferred to post-MVP.

Move on to `03-compliance.md`.
