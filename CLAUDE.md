# Henri Lloyd — Email Design System
## Project context
This repository contains Figma design tokens (exported as JSON) and Stripo email code examples. The goal is to translate Figma tokens into valid, editable Stripo HTML templates.

## Source rules — read before touching any file
- JSON files in Figma-local-variables/ are the design token source. They define values by collection and mode.
- HTML and CSS files in Stripo-code-examples/ are the Stripo grammar reference. They show how Stripo structures markup internally.
- Never treat these two sources as interchangeable. JSON = token values. HTML/CSS = structural grammar.
- The active brand mode is: Dark Base
- The active viewport modes are: Desktop (primary) and Mobile (override layer)

## Files you must read before any generation task
0. On each conversation session, read the stripo-rules.md files in the Beginning to have a good knowledge about Stripo code syntax. (this is mandatory on each session)
1. token-map.md
2. pipeline-optimization.md when using the pipeline from figma to stripo code.

## Files you can read when needed
- If you have doubts regarding token mapping from Figma sources or the token variables collection (in json) to the Stripo HTML or CSS, please read "Figma to HTML Email CSS Properties-Spacing Translation and Stripo Implementation.md"
/Users/samuelgarijocortes/Desktop/Henri Lloyd - Email Design System/Figma to HTML Email CSS Properties-Spacing Translation and Stripo Implementation.md

## Stripo Defaults & Custom CSS — correct pipeline model

**IMPORTANT: Default Styles stays ON. Do NOT turn it off.**

Our actual workflow:

- **Default Styles = ON.** The General Styles panel continues to generate the base Default CSS. Treat it as Stripo-generated base CSS — readable but not editable in code mode.
- **Custom CSS panel** — all global token-driven overrides live here: heading sizes, line-heights, colours, list spacing, link styles, mobile `@media` rules. These override Default CSS via normal CSS cascade / specificity.
- **Inline styles** — only for block-specific overrides: per-block padding, per-block background, one-off font tweaks for a specific module. Applied to the correct TD/element (`esd-container-frame`, `esd-block-*`, `h1–h6`, `a`, `img`).

Behavioural expectations:
- When Stripo's UI shows "The option is disabled because it was overridden in the Code editor" for a token-controlled property → that is **correct and expected**. Do not attempt to fix it.
- Never rewrite docs or token-map to force Default Styles OFF.
- Do not suggest turning the Default Styles toggle off. That breaks our workflow.
- The source of truth is: **Default Styles ON** + token overrides in Custom CSS + inline styles.

## Stripo HTML rules — non-negotiable
- Never remove or alter these classes: esd-stripe, esd-structure, esd-container-frame, esd-block-text, esd-block-image, esd-block-button
- Block-level spacing tokens go as inline styles on the `<td class="esd-block-*">` wrapper. Heading typography tokens (size, weight, line-height, letter-spacing, margin-bottom) go in the Custom CSS panel on `h1–h6` selectors, not inline. Only block-specific overrides go inline on the text element itself (`<h1>`, `<p>`, `<a>`), never on the `esd-block-*` wrapper.
- Global font-family and body font-size belong in the Custom CSS panel, not inline
- Do not invent class names. Only use classes already present in the reference HTML or in the stripo-rules.md

## Output
CSS output files go in /output. The primary CSS file (henri-lloyd-default.css) contains all global token-driven override rules and is pasted into Stripo's Custom CSS panel (Default Styles ON). HTML template files also go in /output. Do not overwrite files in Stripo-code-examples/.