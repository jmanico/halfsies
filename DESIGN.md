# DESIGN.md: Halfsies

Visual and frontend design language for the Halfsies iOS and Android apps. `REQUIREMENTS.md` is the source of truth for product scope; this file never adds features. `style-guide.html` is the reference implementation of these tokens and MUST change in the same commit as this file.

Markers: `UNKNOWN` = not provided by available documents. `TO BE DECIDED` = decision not yet made. Rule keywords follow RFC 2119. Rule IDs (`C-*`, `R-*`, `P-*`, `A-*`) are stable; `REQUIREMENTS.md` cites P-2, P-4, and A-4.

## Required Design Inputs

- Brand personality: TO BE DECIDED
- Primary audience: Two people planning to meet in person. The Initiator signs in and creates a session; the Invitee joins by invite link, optionally as a guest (REQUIREMENTS 1.3, FR-ACC-01/02). US English in v1 (NFR-L10N-01).
- Platform targets (web / mobile / both): mobile. Native iOS and Android phone apps in portrait; tablet layouts should scale. No web client in v1 (REQUIREMENTS 1.1, 1.2, PLAT-01 to PLAT-04).
- Light / dark mode: TO BE DECIDED
- Existing brand assets: UNKNOWN (`REQUIREMENTS.md` cites `BRAND_GUIDELINES.md`, which does not exist in the repository)

## Brand and Logo

Brand direction: TO BE DECIDED. The product name is Halfsies. The logo and color palette below are provisional. They come from the initial design draft and are not an approved brand.

**Concept.** `logo.svg` pairs a mark with a wordmark. The mark is a circle split into two halves, Tangerine for Person A and Iris for Person B. A thin gap separates them, and a Midnight point sits where they meet. The mark says "two sides, one meeting point." The wordmark is outlined to paths, so it has no font dependency.

**Usage.**
- Clear space: at least the diameter of the center point (30 units at the 128-unit mark height) on every side.
- Minimum size: full lockup 120 px/pt wide. Below that, use the mark alone (the `#mark` group, viewBox `0 0 128 128`) at 16 px/pt or larger. A dedicated mark-only file and app icon are TO BE DECIDED (DQ-3).
- Backgrounds: white or `surface` only. Dark-background and single-color variants are TO BE DECIDED (DQ-2).
- Do not: recolor or swap the halves, fill the gap, blend the halves with a gradient (C-5), stretch or rotate, add effects, or re-typeset the wordmark in another font.
- The SVG MUST NOT contain scripts, event handlers, `foreignObject`, or external references. Any variant MUST keep that property.

## Color Palette

Seven core tokens and three person tokens. All values are provisional until brand direction is decided.

| Token | Hex | Role |
|---|---|---|
| `primary` | `#161A3D` | Midnight. Primary button fill, focus ring, wordmark, midpoint |
| `secondary` | `#5A6078` | Slate. Secondary text, metadata, input borders |
| `background` | `#FFFFFF` | Screen background |
| `surface` | `#FAFAFC` | Cards, sheets |
| `text` | `#161A3D` | Body and heading text |
| `error` | `#C62828` | Validation errors, destructive actions |
| `success` | `#157A4B` | Even trip badge, confirmations |
| `person-a` | `#FF7A1A` | Tangerine. Person A fills and markers; logo left half. Never text on light (C-1) |
| `person-a-text` | `#B84A00` | Person A text and icons on light |
| `person-b` | `#5B4BFF` | Iris. Person B fills, markers, and text; logo right half |

Utility values: `divider` `#E4E6EF` is for decorative dividers only, never control boundaries. `success-tint` `#E3EFE9` is the Even trip badge background.

**Logo colors:** `#FF7A1A`, `#5B4BFF`, `#161A3D`, plus the white ring `#FFFFFF`.

**Verified contrast** (WCAG 2.x relative luminance):

| Foreground on background | Ratio | Permitted use |
|---|---|---|
| `text` on `background` / `surface` | 16.81 / 16.13 | All text |
| `secondary` on `background` / `surface` | 6.21 / 5.96 | All text; input borders |
| `#FFFFFF` on `primary` | 16.81 | Primary button label |
| `person-a-text` on `background` / `surface` | 5.23 / 5.01 | All text |
| `person-b` on `background`; `#FFFFFF` on `person-b` | 5.39 | All text |
| `success` on `background` / `success-tint` | 5.36 / 4.54 | All text |
| `error` on `background` / `surface` | 5.62 / 5.39 | All text |
| `text` on `person-a` | 6.44 | Letters on Person A markers |
| `person-a` on `background` | 2.61 | Decorative fills only |

- C-1: `person-a` MUST NOT be used for text or as the only boundary of a control on light surfaces.
- C-2: Text on a `person-a` fill MUST use `text`, never white.
- C-3: Person identity MUST NOT rely on color alone. Every marker, avatar, and legend carries "A", "B", or an initial.
- C-4: `person-a` and `person-b` are reserved for the two people. They MUST NOT be reused for categories, promos, or status.
- C-5: No gradients between `person-a` and `person-b`.

## Typography

**Provisional default:** the platform system font, which is SF Pro on iOS and Roboto on Android. Fallback stack for reference renderings: `system-ui, -apple-system, "Segoe UI", Roboto, "Helvetica Neue", Arial, sans-serif`. The system font needs no license or bundling, is highly legible, and supports Dynamic Type and Android font scaling out of the box (NFR-A11Y-02). Brand typography is TO BE DECIDED (DQ-4).

Weights: 400 regular, 600 semibold, 700 bold.

| Token | Size / line height | Weight | Use |
|---|---|---|---|
| `type-display` | 34 / 40 | 700 | Onboarding headline only |
| `type-h1` | 28 / 34 | 700 | Screen titles |
| `type-h2` | 22 / 28 | 700 | Section titles, place name in detail view |
| `type-h3` | 18 / 24 | 600 | Place name in result card |
| `type-body` | 16 / 24 | 400 | Default |
| `type-body-strong` | 16 / 24 | 700 | Travel times, emphasis |
| `type-small` | 14 / 20 | 400 | Metadata, hours |
| `type-caption` | 12 / 16 | 600 | Badges, map labels |

- Sizes are in pt/sp at the default text size and MUST scale with the OS text size setting. The minimum body size is 16.
- Travel times MUST use tabular numerals, so A and B values line up.
- Use sentence case everywhere. No all-caps labels.

## Layout and Spacing

- Spacing scale (4-unit base, in pt/dp): `space-1` 4, `space-2` 8, `space-3` 12, `space-4` 16, `space-5` 24, `space-6` 32, `space-7` 48, `space-8` 64.
- Screen side padding: `space-4` on phones, `space-6` on wider layouts. Maximum content width for lists and forms: 720.
- Grid: single column on phones. Wider layouts center the content column, or pair the list with the map side by side (FR-RES-02).
- Breakpoints (window width): compact under 600 (phone, primary target), medium 600 to 839, expanded 840 and up (tablet; SHOULD scale per PLAT-04).
- Radius: pill 999 for buttons, chips, and badges; 12 for cards; 8 for inputs; 24 for the top corners of bottom sheets; 0 for the map.
- Touch targets: at least 48×48.

## Components

**Buttons.** Labels name the action ("Send invite", "Open in Maps"), never "Submit" or "OK". A confirmation reuses the verb ("Invite sent"). Minimum height is 48.
- Primary: `primary` fill with white label. Use one per screen.
- Secondary: transparent fill, `primary` label, 1.5 `primary` border.
- Destructive: transparent fill, `error` label and border. Examples: leave session, end session, delete account.
- Hover and pressed states darken the fill or tint the background. Disabled buttons show 40% opacity and explain why nearby when the reason isn't obvious.

**Inputs.** Every input has a visible label; placeholders are never the only label. At rest, the border is 1 `secondary`. On focus, the border is 2 `primary`. Address search shows suggestions after 3 characters (FR-ORG-02). Each participant edits only their own starting point and mode (FR-ORG-05). The other person's slot shows a status: invited, joined, or their area label.

**Links.** Link text uses `primary` (or `person-b` inside Person B contexts) and is always underlined. It names the destination, never "click here".

**Focus.** A 2 `primary` ring with a 2 offset, on every interactive element. On `primary` or `person-b` fills, the offset gap keeps the ring visible.

**Form feedback.** Errors appear below the field in `error`, with an icon and text, never color alone. They state the problem and the fix: "We couldn't find that address. Try adding a city." The field is marked invalid for assistive technology. Success messages use `success` with an icon. Unequal travel times are information, not errors (R-3).

**Result card and travel pair** (FR-RES-01).
- R-1: Every result shows the place name, category, price level, open status at the meeting time, and both travel times. Times appear in A-then-B order, each with the mode icon and a person label.
- R-2: The Even trip badge shows when the API marks the result as even (FR-SRCH-06). It is a pill with `success-tint` fill, `success` text in `type-caption`, a leading check, and the text "Even trip". The UI MUST NOT recompute the threshold.
- R-3: When a trip is not even, state the difference in plain `secondary` text ("B travels 12 min longer"). Use no red and no warning icon.
- The UI MUST keep the API's ranking order (FR-SRCH-07).

**Person markers.** Person markers are 32 circles with a white ring. Person A: `person-a` fill with a `text` letter. Person B: `person-b` fill with a white letter. Meeting places use a `primary` pin. The other person's marker sits at the snapped position the API returns and is labeled with their area label, never an address (API-SHP-01, PRIV-03).

**Privacy patterns.**
- P-2: Before any OS location, notification, or calendar prompt, show a pre-prompt explaining why (PRIV-07). Location copy: "Halfsies uses your location to calculate travel times. The other person sees only your general area, never your exact location." Decline has the same visual weight as accept.
- P-4: A persistent "Who can see what" row on the session screen opens a plain-language summary. That summary MUST match API behavior (PRIV-08).

## Accessibility

Target: WCAG 2.2 AA, applied to native apps (NFR-A11Y-01).

- A-1: Use only the text and background pairs listed in Color Palette. Non-text control boundaries MUST reach at least 3:1.
- A-2: Focus is always visible, as specified under Focus. Focus order follows reading order. Everything works with a keyboard, switch control, VoiceOver, and TalkBack (NFR-A11Y-04).
- A-3: The map has a list equivalent that carries identical information (FR-RES-02).
- A-4: Travel pairs are announced as one sentence: "18 minutes for you, 19 minutes for them. Even trip." (NFR-A11Y-03)
- A-5: Result updates are announced politely: "12 places found."
- A-6: Text scales to 200% without clipping or horizontal scrolling (NFR-A11Y-02).
- A-7: When the OS reduce-motion setting is on, transitions and route animations are replaced by instant state changes. Motion never carries meaning on its own.

## Open Questions

- DQ-1: What is the brand personality and direction? Until it is decided, the logo and palette remain provisional.
- DQ-2: Is dark mode in v1? It decides whether dark tokens and a dark-background logo variant are needed. The initial draft proposed a Midnight-based dark theme.
- DQ-3: What are the app icon and mark-only logo file, including the iOS and Android icon masks and small-size simplification?
- DQ-4: Should there be brand typefaces, or do we keep system fonts? The initial draft proposed Bricolage Grotesque for headings and Atkinson Hyperlegible Next for body text (both OFL 1.1). If adopted, they must be bundled in the apps, support tabular numerals, and scale with Dynamic Type.
- DQ-5: `BRAND_GUIDELINES.md` (voice and invite share text, cited by FR-SES-08) does not exist. Who owns it, and does it absorb the brand parts of this file?
- DQ-6: Should the other person's travel time be displayed rounded (OD-06, PRIV-06)? It changes how the travel pair and difference text read.
- DQ-7: What are the mode and category icons? No icon set has been chosen. Candidates must be open-licensed.
