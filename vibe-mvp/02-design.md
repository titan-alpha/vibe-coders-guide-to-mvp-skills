# 02 · Design

Goal: lock in a visual direction *before* writing UI components. Re-skinning later is the most expensive thing a vibe coder can do.

## DIALOGUE — ask the user

1. **Mood.** "Pick two: minimal · playful · serious · luxurious · technical · friendly · brutalist · soft." Then ask why, in one sentence.
2. **Reference.** "Name one site or app whose aesthetic you'd be proud to be compared to." If they can't name one, suggest 3 from different ends of the spectrum (e.g., Linear, Stripe, Notion, Are.na, Vercel) and ask which feels right.
3. **Density.** "Spacious and breathable, or dense and informational?"
4. **Color.** "Do you want a single accent color, or a gradient identity? Any colors you absolutely want or want to avoid?"
5. **Theme modes.** *Default is **both light and dark**, respecting the user's system preference, with a manual toggle in the header.* Confirm with the user — they can opt for light-only or dark-only if there's a strong product reason (e.g., a dev tool that's "always dark"). The default is not a question; it's a sensible default the user can override.

## Order of decisions (do these in sequence)

*Color and font set the platform's voice subconsciously; the logo references both. Doing them in this order means the logo doesn't fight the surrounding language.*

1. Re-read `PROJECT.md`'s `# Idea` and `# Audience` and pick a tone label from this list: **editorial · tech · friendly · luxurious · playful · brutalist**.
2. Pick the color scheme &mdash; informed by color-theory analysis (six dimensions, see Item #2 below), then propose 2 palettes, user picks.
3. Pick the display font &mdash; from the curated list, matched to tone.
4. Pick the logo (see item #10 below).
5. Build the header with the up-to-5-element rule (see item #11 below).
6. Then everything else &mdash; type scale, spacing, components.

Items 2&ndash;4 are creative decisions; the user always sees and picks. Item 5 is mechanical; the agent just builds it.

## AUTONOMOUS — set up the design system

1. **Stack:** Next.js 15 (App Router) + Tailwind v4 + DaisyUI. If the project is not yet scaffolded, run `npx create-next-app@latest . --typescript --tailwind --app --eslint --no-src-dir --turbopack --import-alias "@/*"`, then install DaisyUI: `npm install -D daisyui@latest` and add `@plugin "daisyui";` to `app/globals.css`.
2. **Theme:** generate a custom DaisyUI theme that reflects the mood and color answers, **anchored to the tone label picked in the Order of decisions step**. **Always produce both a light and a dark variant** (unless the user explicitly opted for one-only in DIALOGUE Q5). Put the theme in `globals.css` via `@plugin "daisyui/theme"`. Use **OKLCH** values throughout.

   The dark variant is not a recoloring of the light one — both must be palette-aware: pick swap-pairs so primary/accent stay readable against `base-100` in either mode (i.e., dark mode primary often needs +0.10 L; surface needs to drop from ~0.98 L to ~0.18 L). Re-run the WCAG AA contrast check on **both** variants — every text/background pair has to clear 4.5:1 in both.

   **System preference is the default**; users get whatever their OS is set to. Wire the manual toggle in the header (see Item #11) so users can override per-session, persisted to `localStorage`. The toggle pattern:

   ```tsx
   // components/ThemeToggle.tsx
   'use client';
   import { useEffect, useState } from 'react';
   import { Moon, Sun, Monitor } from 'lucide-react';

   type Mode = 'light' | 'dark' | 'system';

   export function ThemeToggle() {
     const [mode, setMode] = useState<Mode>('system');
     useEffect(() => {
       const saved = (localStorage.getItem('theme') as Mode | null) ?? 'system';
       apply(saved); setMode(saved);
     }, []);
     function apply(m: Mode) {
       const dark = m === 'dark' || (m === 'system' && matchMedia('(prefers-color-scheme: dark)').matches);
       document.documentElement.dataset.theme = dark ? 'vibedark' : 'vibelight';
       localStorage.setItem('theme', m);
     }
     function next() {
       const order: Mode[] = ['system', 'light', 'dark'];
       const m = order[(order.indexOf(mode) + 1) % order.length];
       setMode(m); apply(m);
     }
     const Icon = mode === 'dark' ? Moon : mode === 'light' ? Sun : Monitor;
     return (
       <button onClick={next} className="btn btn-ghost btn-sm btn-square" aria-label={`Theme: ${mode}`} title={`Theme: ${mode}`}>
         <Icon className="w-4 h-4" />
       </button>
     );
   }
   ```

   To prevent a flash-of-wrong-theme on first paint, set `data-theme` from a tiny inline `<script>` in `app/layout.tsx`'s `<head>` that reads `localStorage` synchronously (bog-standard pattern; the agent writes it).

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

   User picks one. Apply it. Note the choice and the reasoning in `STATE.yaml # Decisions`.
3. **Icons:** install **Lucide React** (`npm install lucide-react`) and use it for every icon. It has 1400+ icons, perfect stroke consistency, and tree-shakes per-import so bundle stays tiny. Don't mix icon libraries; don't hand-roll SVGs for icons that Lucide already has.
   ```tsx
   import { ArrowRight, Check, Copy } from 'lucide-react';
   <ArrowRight className="w-4 h-4" />
   ```
4. **Self-documenting via `<Tooltip>`.** Per the SKILL.md operating rule, every non-obvious interactive element gets a tooltip explaining what it does. Ship the component now in `components/Tooltip.tsx` so every later skill can wrap things with it.

   ```tsx
   'use client';
   import { useState } from 'react';

   export function Tooltip({ children, label, side = 'bottom' }: {
     children: React.ReactNode;
     label: string;
     side?: 'top' | 'bottom' | 'left' | 'right';
   }) {
     const [open, setOpen] = useState(false);
     // Respect the global "hide tooltips" setting (set in /settings).
     if (typeof window !== 'undefined' && localStorage.getItem('tooltips') === 'off') {
       return <span title={label}>{children}</span>;     // accessible fallback only
     }
     return (
       <span className="relative inline-block"
             onMouseEnter={() => setOpen(true)} onMouseLeave={() => setOpen(false)}
             onFocus={() => setOpen(true)} onBlur={() => setOpen(false)}>
         <span title={label}>{children}</span>
         {open && (
           <span role="tooltip" className={`absolute z-50 px-2 py-1 rounded-md bg-base-200 border border-base-300/60 text-xs shadow-lg whitespace-nowrap pointer-events-none ${
             side === 'top' ? 'bottom-full mb-1 left-1/2 -translate-x-1/2'
             : side === 'left' ? 'right-full mr-1 top-1/2 -translate-y-1/2'
             : side === 'right' ? 'left-full ml-1 top-1/2 -translate-y-1/2'
             : 'top-full mt-1 left-1/2 -translate-x-1/2'
           }`}>
             {label}
           </span>
         )}
       </span>
     );
   }
   ```

   Then add a "Hide tooltips" toggle in `/settings`:

   ```tsx
   // app/settings/page.tsx — extend with a tooltip-toggle
   <label className="label cursor-pointer justify-start gap-2">
     <input type="checkbox" defaultChecked={typeof window !== 'undefined' && localStorage.getItem('tooltips') !== 'off'}
       onChange={(e) => localStorage.setItem('tooltips', e.target.checked ? 'on' : 'off')}
       className="toggle toggle-primary" />
     <span className="label-text">Show hover tooltips on icons + status badges</span>
   </label>
   ```

   **Tooltips reinforce, never replace.** Labels and clear icons come first; tooltips add the *why* / *what happens*. If the agent finds itself wrapping a `<button>Save</button>` with a tooltip "Save your changes," cut the tooltip — the label is the explanation.
5. **Type scale:** one **deliberately picked** display font + system-stack body font, at 56/40/28/20/**16**. **16px is the floor.** No body text, caption, label, or nav item smaller than 16px (`text-base`). The only allowed exceptions are technical metadata like timestamps, version numbers, or inline code badges &mdash; and even those should prefer 14px (`text-sm`) over smaller. Never ship `text-xs` for readable content. No more than two font weights.

   **Display font is a tone decision, not a default.** The agent picks based on the platform's emotional tone (audience, mood, vertical from sub-skill 01). Body font stays as the system stack. The picked display font goes in `globals.css` via `@font-face` from Google Fonts (weight subset only).

   Curated display fonts by tone:
   - **Editorial / serious / publication-like** &mdash; Fraunces (variable serif) or Spectral. Trustworthy, considered.
   - **Tech / dev tool / SaaS** &mdash; Inter Tight, Geist, or Space Grotesk. Crisp, modern.
   - **Friendly / consumer / lifestyle** &mdash; General Sans, Plus Jakarta Sans, or DM Sans. Warm, rounded.
   - **Luxurious / premium** &mdash; Cormorant Garamond or Playfair Display. Elegant, high-contrast.
   - **Playful / creative** &mdash; Recoleta, S&ouml;hne, or Bricolage Grotesque. Distinctive, expressive.
   - **Brutalist / technical / raw** &mdash; JetBrains Mono or IBM Plex Mono used as display. Industrial.

   State the choice and reasoning to the user before applying. The user can override.
6. **Text content is minimal wherever possible.** Cut every word that isn't earning its place. One-sentence descriptions beat paragraphs. Button labels are verbs (`Save`, `Send`, `Create`), not phrases. Headings are the shortest fragment that names the section. Use whitespace and hierarchy instead of prose to convey structure.
7. **No gradients on buttons or titles.** Solid theme colors only. Gradient backgrounds on interactive elements or headings read as dated and reduce contrast predictability across themes. Gradients are fine for purely decorative background layers (hero glow, illustration accents) &mdash; not for anything the user reads or clicks.
8. **Spacing:** stick to Tailwind's default scale &mdash; do not invent custom spacing values.
9. **Component primitives:** build only what you need from DaisyUI (`btn`, `card`, `badge`, `input`, `alert`, `navbar`). Do not pre-build a component library.
10. **Logo + favicon (same file).** Every product needs a simple mark. The logo you design *is* the favicon &mdash; one SVG, no variants.

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

11. **Header and footer architecture &mdash; positioning, sizing, content rules.** Modern design theory separates the *working surface* (header) from the *reference surface* (footer). Each has different positioning and a different scope of content. Both are non-negotiable architectural decisions, not aesthetic ones &mdash; **the agent enforces them across every page**.

    ### Header

    **Position: sticky to the top. Never scrolls with content. Always visible. This is non-negotiable.**

    Implementation: `position: sticky; top: 0;` (or `position: fixed; top: 0;` with a body offset to compensate). The header lives in a single shared layout component (Next.js `app/layout.tsx`'s `<header>`); every page renders inside it; no page renders without it.

    **Height: 56px (Tailwind's `h-14`) on every page.** Same height across landing, app shell, admin, /settings, /404, modals. A user navigating between routes should not see the header jump.

    **Content: up to 5 elements, in this order. No exceptions.**

    1. **Platform logo** &mdash; left, links to `/`.
    2. **Platform title** &mdash; next to the logo, semibold, in the curated display font.
    3. **Theme toggle** &mdash; right-aligned, icon-only (sun / moon / monitor). Cycles light → dark → system. Skip only if the user opted for a single-mode product in DIALOGUE Q5.
    4. **Bell icon** &mdash; right of the theme toggle. **ONLY** rendered if the project has a notification center (sub-skill 07's Notifications tab). Shows unread count as a small badge. Click &rarr; dropdown with the 10 most recent notifications + a "View all" link.
    5. **Hamburger menu** &mdash; rightmost. Opens a dropdown containing every top-level page, plus a **"Settings"** link at the bottom (which goes to `/settings` &mdash; a per-user page where signed-in users edit name, password, notification preferences). For projects without auth, omit Settings. **Plus a "Feedback" item** (when sub-skill 07's feedback collection is enabled). Click opens the feedback modal &mdash; see sub-skill 07 for the form spec.

    **Don't add nav links to the header itself &mdash; all of those go in the hamburger.** The header stays clean: identity (logo + title) on the left, controls (theme + bell + hamburger) on the right. This rule supersedes any older "2&ndash;5 primary product surfaces in the header" guidance.

    ### Footer

    **Position: at the bottom of every page. Scrolls with content (NOT sticky, NOT fixed). This is non-negotiable.**

    Implementation: the footer is a normal block element at the end of `<main>` (or right after it) inside the shared layout. On short pages it sits below the content; on tall pages the user scrolls past content first then reaches the footer. Use a sticky-footer pattern (`min-h-dvh flex flex-col` on the body, `flex-1` on `<main>`) so on short pages the footer hugs the bottom of the viewport rather than floating mid-page &mdash; but it still scrolls when the page content is tall.

    **Height: uniform across every page** (e.g., `~96px` on desktop, `~140px` on mobile when items wrap). Same on landing, app, admin, errors, /settings. Like the header, it must not jump between routes.

    **Content scope &mdash; what belongs in the footer:**

    The footer holds the *reference surface*: things the user is more likely to need once (look up the policy, find the about page, contact someone) than to use daily. Per-class breakdown:

    | Class | Items | Notes |
    | --- | --- | --- |
    | **Identity / copyright** | `&copy; <YEAR> <Product Name>.` | Always present. Year is computed at build time, not hardcoded. Single line, left-aligned (or centered on mobile). |
    | **Informational** | About, FAQ | Always present whenever an `/about` and `/faq` route exists (sub-skills 02 and 17 typically produce these). Right-aligned (or below copyright on mobile). |
    | **Legal** | Terms, Privacy, "Do Not Sell or Share" | Always present whenever sub-skill 03 (compliance) ran. The CCPA "Do Not Sell or Share" link is required only if the comply path was chosen and CCPA applies; otherwise drop it. |
    | **Discoverable but optional** | Contact, Docs, Changelog, Press | Only if those surfaces actually exist. Don't add a "Contact" link that goes to a 404. |
    | **Social** | X / Bluesky / GitHub icons | Only if the brand has a real social presence. Do not add empty "Coming soon" social cards. |

    **What does NOT belong in the footer:**

    - Primary product nav (those go in the hamburger inside the header).
    - User-account controls (those live in the header's hamburger Settings item or in the sign-in/out flow).
    - Calls-to-action (signup, buy now). Footer-CTA fatigue is real and the conversion delta is negligible.
    - Long marketing-driven sitemaps. If your audience needs a 4-column footer with 30 links, you are not at MVP scope.

    **Order rule for footer items**: when there's a wrap, the order from left to right is identity → informational → legal → optional → social. On mobile the footer stacks; the order top-to-bottom matches.

    The principle &mdash; same as the header &mdash; is: **if the user is more likely to *need it once* than to *use it daily*, it belongs in the footer.** If the user *needs it daily*, it belongs in the hamburger menu inside the header. Items don't appear in both.

    ### Reference implementation

    ```tsx
    // app/layout.tsx — the shape every project ships
    export default function RootLayout({ children }: { children: React.ReactNode }) {
      return (
        <html lang="en" data-theme="vibelight">
          <body className="min-h-dvh flex flex-col">
            <header className="sticky top-0 z-30 h-14 border-b border-base-300/50 bg-base-100/70 backdrop-blur">
              {/* logo · title · theme toggle · bell · hamburger */}
            </header>
            <main className="flex-1">{children}</main>
            <footer className="border-t border-base-300/50 bg-base-100/70 px-6 py-6 text-sm">
              <div className="max-w-6xl mx-auto flex flex-wrap gap-4 items-center justify-between">
                <span className="opacity-70">&copy; {new Date().getFullYear()} {PRODUCT_NAME}.</span>
                <nav className="flex flex-wrap gap-x-4 gap-y-2 opacity-80">
                  <a href="/about">About</a>
                  <a href="/faq">FAQ</a>
                  <a href="/terms">Terms</a>
                  <a href="/privacy">Privacy</a>
                  {/* "Do Not Sell or Share" only if sub-skill 03 chose CCPA comply path */}
                </nav>
              </div>
            </footer>
          </body>
        </html>
      );
    }
    ```

12. **One landing/home page first.** Ship a minimal, on-brand landing route before building any feature surface. This anchors the visual direction so everything after it inherits the same language.

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

## AUTONOMOUS — brand voice (how the product speaks)

Tone (sub-skill 02 DIALOGUE Q1) is the visual + emotional register: friendly, editorial, brutalist. **Voice** is something different: it's the actual word choices the product uses across every surface — buttons, error messages, empty states, marketing copy, transactional emails. Every product has one whether it's intentional or not. The agent makes it intentional.

### Pick the voice axes

Four sliders, each scored 1–5:

- **Formal ↔ Casual** (1 = "We have received your application." | 5 = "Got it!")
- **Spare ↔ Generous** (1 = "Saved." | 5 = "Saved! Your changes are live and we've kicked off the email — you're all set 🎉")
- **Sincere ↔ Witty** (1 = "Account deactivated." | 5 = "Your account is sleeping. Wake it up anytime.")
- **Authoritative ↔ Conversational** (1 = "Authentication required." | 5 = "Sign in real quick?")

The agent reads the tone label + audience + product type and proposes scores. For example:

- **Recipe app for casual home cooks**: Casual 4 / Spare 3 / Sincere 4 / Conversational 5.
- **Dev tool for backend engineers**: Formal 2 / Spare 5 / Sincere 5 / Authoritative 4.
- **Fintech for small-business owners**: Formal 3 / Spare 4 / Sincere 5 / Authoritative 4.

Confirm with the user. The scores get committed to `STATE.yaml decisions.voice_axes` so every skill that writes user-facing text can reference them.

### Voice rules — concrete words

For each tone × voice combo, encode 3–5 specific rules as a writing guide. Examples:

- **Use** active voice ("We saved your post"), not passive ("Your post has been saved").
- **Use** "you" + "we"; avoid "users" (clinical) or "the system" (machine-like).
- **Avoid** these specific words for *this* voice (e.g., "platform", "leverage", "synergy" for casual voices; "awesome", "🎉" for formal voices).
- **Length cap** for body microcopy: 1 sentence in spare voices, 2 in generous.
- **Punctuation**: exclamation points used sparingly (one max per page) in spare/formal voices; allowed in casual/generous.

The agent writes these to `PROJECT.md # Voice` so every later copy decision can point at them.

### Apply — copywriting templates the agent generates

The agent uses the voice rules to write the user-facing strings the product needs. It does this **proactively**, presenting options for each, not waiting to be asked:

- **Landing page**:
  - Headline (8–12 words; states the value, not the feature).
  - Sub-headline (one sentence; states the audience and the differentiator).
  - Primary CTA button label (a verb phrase: "Save your first recipe", not "Get started").
- **Auth pages**: sign-in / sign-up button labels, success toasts, error messages (per validation case: "We've used that email already" not "Email already exists").
- **Empty states** for every list / dashboard / search-with-no-results — first impression for new users.
- **Error pages**: 404, 500, "you've been deactivated", "rate limited".
- **Transactional emails** (signup confirmation, password reset, invite, etc.) — voice-consistent with the product.
- **Loading / pending states** ("Working on it…" vs "Loading…" vs "Crunching the numbers…").

For each, propose 2–3 variants matching the voice axes and let the user pick. The configurator's `component_pick` and `multiple_choice` message types (sub-skill SKILL.md "Build channel protocol") are the right surface when in Path B.

**Anti-patterns**:
- **Using framework defaults.** "Welcome to Next.js" / generic 404 / "Loading…" are tells. Replace every default the user might see.
- **Voice that doesn't match audience.** A children's-education product written in the voice of a dev tool is jarring. Read the audience back through the voice rules to sanity-check.
- **Writing copy that the user can't pronounce.** If the founder reads the page out loud and stumbles, the voice is wrong. Read it out loud.

## AUTONOMOUS — engineered activation (the first 60 seconds)

Most MVPs lose users in the first 90 seconds, not at week 3. Activation is the path from "I just landed" or "I just signed up" to "oh, I get it." It's a separate concern from auth (which is just plumbing) — and the agent designs it explicitly.

### 1. Define the "aha" moment

Read `PROJECT.md`'s MVP slice + audience. The aha moment is the **single experience** that makes the user understand the value. Examples:

- Recipe app: the user sees their first recipe rendered cleanly without a life-story intro. *(Aha: "this is what I came for.")*
- Note-taking app: the user types and the note auto-saves with a visible indicator. *(Aha: "I don't have to think about saving.")*
- Marketplace: the user sees 3 relevant listings on the homepage without filtering. *(Aha: "there's actually supply here.")*
- AI tool: the user runs the AI on their own input and gets a useful result. *(Aha: "this works for my case.")*

The agent proposes the aha for the user's product, then asks: *"Is this the moment that makes them get it? Or is there a different one?"*

### 2. Time-to-value budget — under 60 seconds

From landing to aha must take **under 60 seconds for a first-time user, including signup**. Anything longer and most people leave. This constrains everything downstream:

- Signup is one screen, not three.
- The first authed page is the aha-producing page, not a generic dashboard with empty states.
- Optional fields (intended-use, profile photo, etc.) come AFTER the aha, not before.
- Onboarding tours that block content are forbidden — they push aha further away.

Test it: Playwright scripts a brand-new user from `/` to the aha screen and asserts wall-clock time < 60s on a slow-3G profile. The test lives in `tests/e2e/activation.spec.ts`.

### 3. The four activation states the agent designs

For every product:

| State | What the user sees | What the agent designs |
| --- | --- | --- |
| **Cold landing** (no account) | Public landing page | Single primary CTA leading toward aha; secondary "see an example" link bypasses signup for browse-able products |
| **Just signed up** (empty account) | Authed shell | First-action prompt, NOT empty dashboard. "Save your first recipe" / "Create your first note" / etc. — verb + concrete object |
| **First action complete** (one record exists) | Aha state | The thing that makes them get it. May include a small "what's next" hint but doesn't force one. |
| **Returning, day 2+** | Authed shell with content | Different from "just signed up" — usually shows the user's own most recent thing front-and-center. Reduces re-orientation cost. |

Each state gets its own visual baseline screenshot (sub-skill 02 test rig + sub-skill 16 visual regression).

### 4. Activation copy is the highest-leverage copy

The empty-state copy on the just-signed-up screen is the single most-important sentence in the product. The agent writes it explicitly using the voice rules from "Brand voice" above — and tests 2 variants if possible (sub-skill 08 analytics will tell you which converts).

### Exit criteria for activation

- The aha moment is named in `PROJECT.md # Activation` and `STATE.yaml decisions.aha_moment`.
- A Playwright test asserts time-to-aha < 60s for a brand-new user.
- The four state-specific screens have visual baselines.
- The "just signed up" empty-state copy is voice-consistent and verb-led.

## AUTONOMOUS — set up the test rig (do this BEFORE leaving 02)

Sub-skill 02 is the last skill where the project is "small enough to install dev tooling without breaking anything." Every later skill writes tests against this rig.

This section is non-negotiable. The user does not test the product (see SKILL.md operating rules); the agent does. The rig must be in place before that work begins.

### 1. Install

```bash
npm install -D vitest @vitest/ui jsdom @testing-library/react @testing-library/jest-dom @testing-library/user-event
npm install -D @playwright/test playwright @axe-core/playwright
npx playwright install chromium firefox webkit
```

### 2. Directory layout (create these now, even if empty)

```
project-root/
├── tests/
│   ├── unit/                          # Vitest, isolated logic + components
│   │   └── .gitkeep
│   ├── integration/                   # Vitest, hits DB / external SDKs
│   │   └── .gitkeep
│   └── e2e/                           # Playwright
│       ├── routes.ts                  # shared route manifest (used by crawl + visual + a11y)
│       ├── helpers/                   # auth fixtures, factories, test users
│       ├── flows/                     # journey specs (signup → core → sign-out)
│       │   └── .gitkeep
│       ├── crawl.spec.ts              # added by 16-e2e-testing
│       ├── a11y.spec.ts               # added by 09-accessibility (was 10-accessibility — confirm via STATE.yaml's mode plan)
│       └── visual.spec.ts             # screenshot regression
└── tests/e2e/visual.spec.ts-snapshots/  # baseline screenshots, committed to git
```

The `-snapshots` directory naming is Playwright's built-in convention for `toHaveScreenshot()`. Don't rename it.

### 3. Configs

`vitest.config.ts`:
```ts
import { defineConfig } from 'vitest/config';
import react from '@vitejs/plugin-react';

export default defineConfig({
  plugins: [react()],
  test: {
    environment: 'jsdom',
    setupFiles: ['./tests/unit/setup.ts'],
    include: ['tests/unit/**/*.test.{ts,tsx}'],
  },
});
```

Add a separate `vitest.integration.config.ts` for `tests/integration/**` that uses `environment: 'node'` and a real DB connection from `.env.test` — the integration suite is allowed to be slow (≥ a few seconds per test) because it's catching real-boundary bugs.

`playwright.config.ts` (extend the one from sub-skill 16):
```ts
import { defineConfig, devices } from '@playwright/test';

export default defineConfig({
  testDir: './tests/e2e',
  webServer: {
    command: 'npm run dev',
    url: 'http://localhost:3000',
    reuseExistingServer: true,
    timeout: 120_000,
  },
  use: {
    baseURL: process.env.E2E_BASE_URL ?? 'http://localhost:3000',
    trace: 'retain-on-failure',
  },
  // 5 breakpoints — see 16-e2e-testing.md for why these specific viewports.
  projects: [
    { name: 'mobile-sm', use: { ...devices['iPhone SE'] } },
    { name: 'mobile-lg', use: { ...devices['iPhone 14'] } },
    { name: 'tablet',    use: { ...devices['iPad Mini'] } },
    { name: 'desktop',   use: { ...devices['Desktop Chrome'], viewport: { width: 1280, height: 800 } } },
    { name: 'wide',      use: { ...devices['Desktop Chrome'], viewport: { width: 1920, height: 1080 } } },
  ],
  // Visual regression tolerance: pixel-diff with a strict threshold.
  // 0.01 = up to 1% of pixels may differ before failing. Sub-pixel font
  // anti-aliasing on different OSes can cause false positives below this.
  expect: {
    toHaveScreenshot: { maxDiffPixelRatio: 0.01, animations: 'disabled' },
  },
});
```

### 4. npm scripts (paste into `package.json`)

```json
{
  "scripts": {
    "test": "npm run test:unit && npm run test:integration && npm run test:e2e",
    "test:unit": "vitest run --config vitest.config.ts",
    "test:unit:watch": "vitest --config vitest.config.ts",
    "test:integration": "vitest run --config vitest.integration.config.ts",
    "test:e2e": "playwright test",
    "test:visual": "playwright test tests/e2e/visual.spec.ts",
    "test:visual:update": "playwright test tests/e2e/visual.spec.ts --update-snapshots",
    "test:a11y": "playwright test tests/e2e/a11y.spec.ts",
    "test:crawl": "playwright test tests/e2e/crawl.spec.ts"
  }
}
```

### 5. The first visual baseline (lock the design surface BEFORE any feature code)

Before exiting 02, the agent runs:

```bash
npm run test:visual
```

The first run has no baselines, so Playwright captures and saves. The agent commits these baselines to git:

```bash
git add tests/e2e/visual.spec.ts-snapshots/
git commit -m "02-design: lock visual baselines for landing + theme toggle"
```

These become the reference every later skill diffs against. When a skill intentionally changes the UI, the agent runs `npm run test:visual:update` only **after inspecting each diff** to confirm the change was wanted (see SKILL.md's Visual regression workflow rule).

### 6. The first three tests (templates every later skill copies)

Write these three before exiting 02 so subsequent skills have a pattern to follow. Each is the simplest possible example of its layer.

```ts
// tests/unit/format.test.ts — unit test template
import { describe, it, expect } from 'vitest';
import { formatPrice } from '@/lib/format';

describe('formatPrice', () => {
  it('formats cents as dollars', () => {
    expect(formatPrice(2000)).toBe('$20.00');
  });
});
```

```ts
// tests/integration/db.test.ts — integration test template
import { describe, it, expect, beforeAll } from 'vitest';
import { db } from '@/lib/db';
import { users } from '@/lib/db/schema';

beforeAll(async () => {
  // Reset to a known state; uses .env.test DATABASE_URL pointing at a sandbox DB.
  await db.delete(users);
});

describe('users table', () => {
  it('round-trips an insert', async () => {
    const [u] = await db.insert(users).values({
      email: 't@example.com', firstName: 'T', lastName: 'T',
    }).returning();
    expect(u.email).toBe('t@example.com');
  });
});
```

```ts
// tests/e2e/visual.spec.ts — visual regression template
import { test, expect } from '@playwright/test';
import { ROUTES } from './routes';

for (const route of ROUTES.filter(r => !r.auth)) {
  test(`visual: ${route.path}`, async ({ page }) => {
    await page.goto(route.path);
    await page.waitForLoadState('networkidle');
    // The toHaveScreenshot matcher captures on first run, diffs on later runs.
    // Filename is auto-derived: tests/e2e/visual.spec.ts-snapshots/<test name>-<project>.png
    await expect(page).toHaveScreenshot();
  });
}
```

### 7. Tell the user (one line, then exit)

> *"Test rig is set up. From here on, every skill writes its own tests as it builds — you won't need to test anything except subjective things like 'does this email look right?' I'll run the suite and only flag things that need your eye."*

## Anti-patterns to avoid

- Inventing custom color tokens before the theme is locked.
- Adding a UI component library *and* DaisyUI ("just in case").
- **Gradient buttons or gradient headings.** Solid colors only.
- **Text under 16px** for anything the user is expected to read.
- Verbose copy. Cut every word that isn't earning its place.
- Animations on every element. Reserve motion for state changes (reveal, success, error).
- Stock illustrations. Use type, color, and whitespace instead.
- Mixing icon libraries &mdash; one of anything is fine, two of anything is a smell.
- Putting nav links directly in the header. Top-level pages live inside the hamburger dropdown; the header has at most 5 elements (logo, title, theme toggle, bell-if-notifications, hamburger).
- Letting the header scroll with content. Header is `sticky top-0` and always visible; never `position: static` and never inside `<main>`. If the header disappears when the user scrolls, that's a bug.
- Making the footer sticky / fixed. Footer scrolls with content — it's a reference surface, not an always-visible chrome. Sticky footers steal viewport real estate that belongs to content.
- Variable header or footer height between routes. Both must be the same height on every page (header: 56px; footer: ~96px desktop, ~140px mobile when items wrap). A jumping header on navigation is jarring and fixable in one shared layout component.
- Footer CTAs (signup buttons, "buy now" cards). The footer is reference, not conversion — these belong on the page itself, not the footer of every page.
- A "footer that holds primary nav." If the user needs it daily, it's not footer material. Move it into the hamburger.
- Shipping a single-theme product without explicit user confirmation. Both light and dark are the default; only deviate when the user has a strong product reason.
- Building the dark variant by inverting the light one. Dark themes need their own L-curve adjustments; recoloring with negative-of-light produces low-contrast surfaces.
- Defaulting to the system stack as the display font, or picking a color palette before the tone label is named. Tone first, then palette and font.
- Picking colors by feel without checking contrast ratios or cultural connotations. The agent always proposes palettes that pre-pass WCAG AA &mdash; accessibility is a *constraint* on color choice, not a follow-up audit.
- Shipping the framework's default empty / 404 / loading copy. These are dead giveaways and tank trust.
- A voice that doesn't match the audience (children's education in dev-tool voice, fintech in playful-app voice). Read the audience back through the voice rules.
- An activation flow longer than 60 seconds for a first-time user. Cut every step that doesn't earn its place.
- Tooltips as the only way to understand a control. Labels + clear icons first; tooltips add reinforcement, not basic comprehension.

## Exit criteria

- The user has approved the landing page screenshot or has clicked through it on `localhost`.
- `globals.css` contains the locked theme (OKLCH values) and the chosen display font loaded via `@font-face` from Google Fonts.
- `public/favicon.svg` exists and renders correctly as both the favicon (browser tab) and the inline header logo.
- A tone label has been picked from the curated list, and color palette + display font were chosen against that tone with the user's approval.
- Color palette was selected via the color-theory analysis (six dimensions), passes WCAG AA contrast, and is recorded in `STATE.yaml # Decisions` with the reasoning.
- **Header**: `position: sticky; top: 0` (never scrolls), uniform height of 56px on every page, up to 5 elements (logo, title, theme toggle, bell if notifications, hamburger). The hamburger contains every top-level page plus Settings (when auth exists).
- **Footer**: scrolls with content (NOT sticky), uniform height across every page, holds only the defined classes — identity (©  YYYY  Product), informational (About, FAQ), legal (Terms, Privacy, "Do Not Sell or Share" if CCPA-comply path), optional links if those surfaces actually exist, social if real presence. No primary nav, no CTAs, no user-account controls in the footer.
- Hamburger menu includes a "Feedback" item when feedback collection is enabled in sub-skill 07. The item opens the modal defined there.
- Both light and dark themes ship by default and pass WCAG AA contrast in both modes. The header theme toggle cycles light → dark → system and persists to `localStorage`.
- The user-flow critique has been written, discussed with the user, and applied where agreed.
- A `# Design` section in `PROJECT.md` captures the chosen tone label, color decisions, display font, logo concept, theme modes (light/dark/both), the core user journey sentence, and any flow-critique items deferred to post-MVP.
- Test rig is scaffolded: Vitest (unit + integration), Playwright (e2e), axe-core, baseline screenshot dir under `tests/e2e/visual.spec.ts-snapshots/`. The three template tests (unit, integration, visual) exist and pass. `npm run test` runs the full suite. Visual baselines for the landing page in both light and dark are committed to git.
- `<Tooltip>` component shipped in `components/Tooltip.tsx`; `/settings` has the global hide-tooltips toggle.
- Voice axes scored (Formal/Casual, Spare/Generous, Sincere/Witty, Authoritative/Conversational) and recorded in `STATE.yaml decisions.voice_axes`.
- Voice rules and word-list captured in `PROJECT.md # Voice`.
- Aha moment named; time-to-aha < 60s asserted by `tests/e2e/activation.spec.ts`; the four activation-state screens have visual baselines.

Move on to `03-compliance.md`.
