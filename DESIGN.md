# DESIGN.md: Halfsies

Visual and frontend design language for the Halfsies iOS and Android apps. `REQUIREMENTS.md` owns product scope; this file never adds features. `style-guide.html` renders these tokens and components and MUST change in the same commit as this file.

Rule keywords follow RFC 2119. Rule IDs (`C-*`, `R-*`, `P-*`, `A-*`) are stable.

## Design Inputs

- Audience: the Initiator and Invitee (`REQUIREMENTS.md` 1.3), in US English (NFR-L10N-01).
- Platforms: PLAT-01 to PLAT-04. No web client (`REQUIREMENTS.md` 1.2).
- Brand personality: warm and playful; friendly, fair, a little cheeky (see Brand Voice).
- Color modes: light and dark in v1. The app follows the system setting (see Dark Mode).

## Brand and Logo

The logo and color palette are final.

**Concept.** `logo.svg` pairs a mark with a wordmark. The mark is a circle split into two halves, Tangerine for Person A and Iris for Person B. A thin gap separates them, and a Midnight point sits where they meet. The mark says "two sides, one meeting point." The wordmark is outlined to paths, so it has no font dependency.

**Usage.**
- Clear space: at least the diameter of the center point (30 units at the 128-unit mark height) on every side.
- Minimum size: full lockup 120 px/pt wide. Below that, use the mark alone (the `#mark` group, viewBox `0 0 128 128`) at 16 px/pt or larger. The mark-only file is `logo-mark.svg`.
- Backgrounds: on light backgrounds use `logo.svg` as is. On dark backgrounds use the dark-background variant: the wordmark fill changes to dark `text` (`#F2F3F8`); the halves, center point, and white ring stay unchanged. No other variants.
- Do not: recolor or swap the halves, fill the gap, blend the halves with a gradient (C-5), stretch or rotate, add effects, or re-typeset the wordmark in another font.
- Small sizes: below 32 px/pt, drop the inner detail (the white ring around the center point) so the mark stays legible.
- Logo files and variants MUST meet `SECURITY.md` SC-WEB-03.

**App icon.** The app icon is the mark (`#mark`) centered on a `primary` (Midnight) background.
- iOS: a full-bleed square; the system applies the corner mask. Keep the mark within the central 80%.
- Android: an adaptive icon with a `primary` background layer and the mark as the foreground layer, kept within the 66 dp safe zone so any mask shape (circle, squircle, rounded square) leaves it whole. Provide a monochrome layer for themed icons.
- Below 32 px/pt the small-size rule above applies.

## Color Palette

Seven core tokens and three person tokens.

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

**Logo colors:** `person-a`, `person-b`, and `primary`, plus the white ring `#FFFFFF`.

**Verified contrast** (WCAG 2.x relative luminance):

| Foreground on background | Ratio | Permitted use |
|---|---|---|
| `text` on `background` / `surface` | 16.81 / 16.13 | All text |
| `secondary` on `background` / `surface` | 6.21 / 5.96 | All text; input borders |
| `#FFFFFF` on `primary` | 16.81 | Primary button label |
| `person-a-text` on `background` / `surface` | 5.23 / 5.01 | All text |
| `person-b` on `background`; `#FFFFFF` on `person-b` | 5.39 | All text |
| `person-b` on `surface` | 5.17 | All text |
| `success` on `background` / `success-tint` | 5.36 / 4.54 | All text |
| `error` on `background` / `surface` | 5.62 / 5.39 | All text |
| `text` on `person-a` | 6.44 | Letters on Person A markers |
| `person-a` on `background` | 2.61 | Decorative fills only |

- C-1: `person-a` MUST NOT be used for text or as the only boundary of a control on light surfaces.
- C-2: Text on a `person-a` fill MUST use `text`, never white.
- C-3: Person identity MUST NOT rely on color alone. Every marker, avatar, and legend carries "A", "B", or an initial.
- C-4: `person-a` and `person-b` are reserved for the two people. They MUST NOT be reused for categories, promos, or status.
- C-5: No gradients between `person-a` and `person-b`.

### Dark Mode

Dark mode ships in v1 and follows the system setting; there is no in-app toggle. Dark tokens are Midnight-based and replace the light values by token name.

| Token | Dark hex | Role in dark |
|---|---|---|
| `primary` | `#C9CCF2` | Primary button fill (label `#161A3D`), focus ring, links |
| `secondary` | `#A9AEC4` | Secondary text, metadata, input borders |
| `background` | `#161A3D` | Screen background (Midnight) |
| `surface` | `#20254D` | Cards, sheets |
| `text` | `#F2F3F8` | Body and heading text |
| `error` | `#FF8A80` | Validation errors, destructive actions |
| `success` | `#5FD19A` | Even trip badge text, confirmations |
| `person-a` | `#FF7A1A` | Unchanged. Person A fills and markers |
| `person-a-text` | `#FF9A4D` | Person A text and icons on dark |
| `person-b` | `#A69CFF` | Person B fills, markers, and text on dark |

Dark utility values: `divider` `#2E3460` (decorative only); `success-tint` `#173A2E`. The logo keeps its own colors (see Brand and Logo).

**Verified contrast, dark** (WCAG 2.x relative luminance):

| Foreground on background | Ratio | Permitted use |
|---|---|---|
| `text` on `background` / `surface` | 15.17 / 13.22 | All text |
| `secondary` on `background` / `surface` | 7.64 / 6.66 | All text; input borders |
| `#161A3D` on `primary` | 10.73 | Primary button label |
| `primary` on `background` / `surface` | 10.73 / 9.35 | Links, focus ring, secondary button |
| `person-a-text` on `background` / `surface` | 7.99 / 6.96 | All text |
| `person-b` on `background` / `surface` | 7.06 / 6.15 | All text |
| `success` on `background` / `success-tint` | 8.86 / 6.59 | All text |
| `error` on `background` / `surface` | 7.36 / 6.42 | All text |
| `#161A3D` on `person-a` / `person-b` | 6.44 / 7.06 | Letters on person markers |
| `person-a` on `background` | 6.44 | Fills |

- C-6: In dark mode, letters on `person-b` fills MUST use `#161A3D`, never white (white on dark `person-b` is 2.38). C-2 still applies to `person-a`.
- C-7: Every screen and component MUST be verified in both modes before release.

## Typography

The platform system font: SF Pro on iOS and Roboto on Android. There are no brand typefaces. Fallback stack for reference renderings: `system-ui, -apple-system, "Segoe UI", Roboto, "Helvetica Neue", Arial, sans-serif`. The system font needs no license or bundling, is highly legible, and supports Dynamic Type and Android font scaling out of the box (A-6).

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

- Sizes are in pt/sp at the default text size and scale per A-6. The minimum body size is 16.
- Travel times MUST use tabular numerals, so A and B values line up.
- Use sentence case everywhere. No all-caps labels.

## Iconography

- Icon set: Lucide (ISC license) for mode, category, and UI icons. Record its license notice with the app's third-party notices.
- Default size 24 (20 inside dense rows), stroke 2, `currentColor` so icons follow the text token and color mode.
- Icons that carry meaning have an accessibility label or sit next to visible text (A-8). Do not mix in icons from other sets.
- Icon SVGs MUST meet `SECURITY.md` SC-WEB-03.

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

**Inputs.** Every input has a visible label; placeholders are never the only label. At rest, the border is 1 `secondary`. On focus, the border is 2 `primary`. Address search shows suggestions per FR-ORG-02. The other person's slot is read-only (FR-ORG-05) and shows a status: invited, joined, or their area label.

**Links.** Link text uses `primary` (or `person-b` inside Person B contexts) and is always underlined. It names the destination, never "click here".

**Focus.** A 2 `primary` ring with a 2 offset, on every interactive element. On `primary` or `person-b` fills, the offset gap keeps the ring visible.

**Form feedback.** Errors appear below the field in `error`, with an icon and text, never color alone. They state the problem and the fix: "We couldn't find that address. Try adding a city." The field is marked invalid for assistive technology. Success messages use `success` with an icon. Unequal travel times are information, not errors (R-3).

**Result card and travel pair** (FR-RES-01).
- R-1: A result card shows the FR-RES-01 fields. Times appear in A-then-B order, each with the mode icon and a person label. Displayed times are rounded to the nearest minute ("22 min · 25 min"); the API supplies the rounded values.
- R-2: The Even trip badge shows when the API marks the result as even (FR-SRCH-06). It is a pill with `success-tint` fill, `success` text in `type-caption`, a leading check, and the text "Even trip". The UI MUST NOT recompute the threshold.
- R-3: When a trip is not even, state the difference in plain `secondary` text, computed from the rounded displayed values ("3 min difference" or "B travels 3 min longer"). Use no red and no warning icon. The Even trip badge follows the API flag, which the server computes from unrounded values, so a pair such as "22 min · 23 min" can still show "Even trip".
- The UI MUST keep the API's ranking order (FR-SRCH-07).

**Person markers.** Person markers are 32 circles with a white ring. Person A: `person-a` fill with a `text` letter. Person B: `person-b` fill with a white letter. Meeting places use a `primary` pin. The other person's marker uses the snapped position and area label the API returns (API-SHP-01).

**Privacy patterns.**
- P-2: The pre-prompt PRIV-07 requires explains why the permission is needed. Location copy: "Halfsies uses your location to calculate travel times. They'll see your approximate area and travel times, never your exact starting point." Decline has the same visual weight as accept. The copy MUST meet `SECURITY.md` SC-PRIV-11. Never say "exact location".
- P-4: A persistent "Who can see what" row on the session screen opens a plain-language summary (PRIV-08). The summary MUST say that the other person sees your travel time to each result (`SECURITY.md` SC-PRIV-11). It uses the same sentence as P-2: "They'll see your approximate area and travel times, never your exact starting point."
- P-5: Screenshot and app-switcher protection (decision SQ-14). On iOS, screens showing your own precise origin or an active invite link are replaced by a neutral cover (logo on `background`) in the app switcher snapshot. On Android, those screens set `FLAG_SECURE`. Other screens are not protected.
- P-6: Age attestation. Sign-in and guest join show a required, unchecked checkbox "I am 16 or older" above the continue button. Continue stays disabled until it is checked, with the reason stated nearby. The checkbox has a visible label and meets the 48 touch target.

**Safety: block and report.** Both live in an overflow menu ("More") on the session screen, labeled "Block [name]" and "Report [name]".
- Block (accounts only): a confirmation sheet explains "You won't be able to start or join sessions with each other. They won't be told." The confirm button uses the destructive style, labeled "Block". Guests see "Leave session" instead of Block.
- Report (accounts and guests): a sheet with a required reason (single-select list) and an optional text field with a visible 500-character counter. The copy says no location data is sent. Buttons: "Send report" (primary) and "Cancel". Confirmation: "Report sent. We review reports within 7 days." Account holders are then offered Block.
- Neither surface uses `person-a` or `person-b` colors (C-4).

## Brand Voice

Owner: Product. Warm and playful: friendly, fair, a little cheeky. Plain words, short sentences, US English, sentence case.
- Be kind about unequal trips; it's information, not blame (R-3).
- A little cheek in empty states and confirmations ("Halfway there!"); none in errors, privacy, safety, or permission copy, which stay plain and calm.
- Never guilt, pressure, or overclaim privacy.
- Invite share text (FR-SES-08): "Let's meet halfway! Join me on Halfsies: <link>". It MUST NOT include any location, address, or area.

## Accessibility

Target: WCAG 2.2 AA, applied to native apps per the W3C guidance on applying WCAG to mobile (NFR-A11Y-01).

- A-1: Use only the text and background pairs listed in Color Palette and Dark Mode. Non-text control boundaries MUST reach at least 3:1.
- A-2: Focus is always visible, as specified under Focus. Focus order follows reading order. Everything works with a keyboard, switch control, VoiceOver, and TalkBack (NFR-A11Y-04).
- A-3: The map has a list equivalent that carries identical information (FR-RES-02).
- A-4: Travel pairs are announced as one sentence: "18 minutes for you, 19 minutes for them. Even trip."
- A-5: Result updates are announced politely: "12 places found."
- A-6: Text scales with Dynamic Type (iOS) and font scaling (Android) up to 200% without clipping or horizontal scrolling.
- A-7: When the OS reduce-motion setting is on, transitions and route animations are replaced by instant state changes. Motion never carries meaning on its own.
- A-8: Every interactive element has an accessibility label.

## Open Questions

None open.
