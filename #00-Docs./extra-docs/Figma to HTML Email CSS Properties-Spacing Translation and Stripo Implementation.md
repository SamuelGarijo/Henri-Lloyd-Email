# Figma → HTML Email: CSS Properties, Spacing Translation & Stripo Implementation
## Overview
The intuition is correct. The single biggest conceptual gap between Figma and HTML email is the absence of `gap`. In Figma, vertical Auto Layout has one `gap` value that distributes space between all children uniformly. In HTML email, that concept doesn't exist — inter-element vertical spacing must be constructed per-element through padding on table cells, or with dedicated spacer elements inserted between blocks. Everything else flows from understanding this fundamental difference.[^1][^2]

***
## Part 1: The Layout Model Difference
### Figma Auto Layout vs. Email Tables
Figma's Auto Layout is conceptually a Flexbox model: direction (row/column), gap between children, padding inside the parent, alignment. HTML email is a **table model** — layout is expressed through nested `<table>`, `<tr>`, and `<td>` elements, not through CSS layout properties.[^3][^4][^5][^6]

`display:flex` has ~85% email client support, but its associated properties (`flex-direction`, `flex-wrap`, `justify-content`, `align-items`) are poorly supported and unreliable. CSS `gap` specifically is not supported in email contexts. `display:grid` and `grid-template-*` are even less supported. The practical conclusion: **nothing from Figma Auto Layout translates to CSS layout properties in email**. It all translates to table structure.[^1][^7][^8][^2]
### Stripo's 4-Level Hierarchy
Stripo maps this table structure to a four-level system:[^4][^9]

```
Stripe (esd-stripe)
  └── Structure (esd-structure)          ← horizontal layout, up to 11 columns
        └── Container (esd-container-frame)  ← one column
              └── Block(s) (esd-block-*)     ← content elements, stacked vertically
```

Each level has its own padding settings. These are distinct scopes — padding at the structure level controls column spacing, padding at the container level controls inset from the column edges, padding at the block level controls spacing around the content element itself.[^9][^10]

***
## Part 2: The Gap Problem — Three Email Equivalents
A Figma vertical `gap` of 24px between two Auto Layout children maps to **three different things in email**, depending on context:
### Option A — Block padding (most common)
Apply `padding-bottom` to the upper block's `<td>` or `padding-top` to the lower block's `<td>`. In Stripo HTML, this appears as the `es-p{n}b` / `es-p{n}t` utility classes on the block wrapper.[^11][^12]

```html
<td class="esd-block-text es-p24b" style="padding-bottom:24px">
  <!-- content -->
</td>
```
### Option B — Spacer block (section-level gaps)
A dedicated `esd-block-spacer` element inserted between blocks. This is Stripo's preferred method for larger gaps between content zones, because it's visible in the right panel, controllable independently, and renders consistently.[^13][^14]

```html
<td class="esd-block-spacer es-p24t es-p24b" style="padding:24px 0;">
  <table cellpadding="0" cellspacing="0" width="100%">
    <tr><td height="0" style="line-height:0px;font-size:0px;"> </td></tr>
  </table>
</td>
```
### Option C — Container or structure padding
When the gap is really the inset of a whole content zone from the stripe edge, it belongs at the container or structure padding level — not at the block level. This corresponds to Figma's frame padding, not to gap between children.[^15][^9]

**The key token-mapping consequence**: a single Figma `gap` token may need to be split into multiple email tokens — one for block padding-bottom, one for spacer height — depending on where in the hierarchy the gap is being expressed.

***
## Part 3: CSS Property Translation Table
The following table maps Figma design properties to their email HTML/CSS equivalents, with support status across the three major client categories.[^8][^16][^2][^17][^18]

| Figma Property | Email CSS/HTML Equivalent | Gmail / Apple Mail | Outlook Desktop | Notes |
|---|---|---|---|---|
| Auto Layout gap (vertical) | `padding-bottom` / `padding-top` on `<td>` + spacer block | ✅ | ✅ | `gap` CSS property: ❌ not supported |
| Auto Layout gap (horizontal) | Column width via table `<td width>` | ✅ | ✅ | No CSS column gap equivalent |
| Frame padding (all sides) | `padding` on `<td>` | ✅ | ✅ on `<td>` only | `padding` on `<div>` fails in some Outlook |
| Background color | `background-color` inline | ✅ | ✅ | Safe everywhere |
| Background image | `background-image` inline or `background` attribute | ✅ | ⚠️ partial | Outlook renders with VML fallback needed |
| Border | `border` inline on `<td>` or `<table>` | ✅ | ✅ | Solid borders are safe |
| Border radius | `border-radius` inline | ✅ | ❌ ignored | Outlook desktop strips it silently |
| Box shadow | `box-shadow` | ✅ | ❌ | Avoid for structural elements |
| Opacity | `opacity` | ✅ | ⚠️ partial | Not reliable for Outlook |
| Font family | `font-family` inline | ✅ | ✅ (system fonts fallback) | Web fonts via `@font-face`: partial |
| Font size | `font-size` in px inline | ✅ | ✅ | px only; em/rem unreliable |
| Font weight | `font-weight` inline | ✅ | ✅ | Use numeric values (400, 700) |
| Line height | `line-height` in **px absolute** | ✅ | ✅ | Percentage `line-height` fails in Outlook.com[^19] |
| Letter spacing | `letter-spacing` in px inline | ✅ | ⚠️ partial | Outlook desktop: inconsistent |
| Text align | `text-align` inline | ✅ | ✅ | Safe |
| Text decoration | `text-decoration` inline | ✅ | ✅ | Safe |
| Text transform | `text-transform` inline | ✅ | ✅ | Safe |
| Text color | `color` inline | ✅ | ✅ | Safe |
| Width / height | `width`/`height` attributes on `<table>`/`<td>` | ✅ | ✅ | Use HTML attributes, not just CSS |
| display:flex | `display:flex` | ✅ (~85%) | ❌ | Flex properties beyond display are unreliable |
| display:grid | `display:grid` | ⚠️ partial | ❌ | Avoid entirely |
| position: absolute | `position:absolute` | ⚠️ | ❌ | Avoid for layout |
| CSS variables | `var(--token)` | ❌ stripped | ❌ | Not supported — use resolved values |
| z-index | `z-index` | ⚠️ | ❌ | Avoid |
| clamp() / min() / max() | — | ❌ | ❌ | Not email contexts |
| CSS Grid gap | `gap` / `column-gap` / `row-gap` | ❌ | ❌ | Use padding instead |
| margin | `margin` | ✅ Gmail/Apple | ⚠️ Outlook | Outlook.com needs capital `Margin`[^19]; Outlook desktop: `padding` on `<td>` is safer[^18] |

***
## Part 4: How Stripo Specifically Implements Spacing
### Padding hierarchy in Stripo HTML
Stripo applies padding at every level of its table hierarchy. Understanding which level corresponds to which Figma concept is the core of token mapping:[^15][^9]

**Stripe padding** — controls the outer breathing room of the entire content band. Set in Stripo's Appearance → General Styles → padding defaults. In HTML, this appears as padding on the outermost `<table>` or its wrapper `<td>`. Equivalent to the outer padding of a top-level frame in Figma.

**Structure padding** — controls the space between the structure's content area and the stripe edges, and optionally the indent between columns in a multi-column structure. In HTML, it appears on the `<td>` that wraps the inner layout table. Corresponds to the padding of a section-level frame.

**Container padding** — the inset inside each column. This is the direct equivalent of Auto Layout padding in a Figma component frame. Applied via inline style on the `esd-container-frame` `<td>`.[^9]

**Block padding** — top and bottom spacing specific to one block within a container. Expressed via the `es-p{n}t` and `es-p{n}b` utility classes and inline padding on the `esd-block-*` wrapper `<td>`.[^12]
### Why margin is avoided in Stripo
Stripo's HTML is table-based for a reason: Outlook desktop only supports `padding` reliably on `<td>` elements — not on `<div>` or `<p>`. Margins on `<div>` elements fail silently in Outlook. Stripo's table structure ensures that padding always lands on a `<td>`, making it Outlook-safe.[^20][^18]
### The spacer block in HTML
When Stripo generates a spacer block (`esd-block-spacer`), the HTML pattern is a `<td>` with padding + an inner empty table with a single row set to `height: 0`, `line-height: 0`, `font-size: 0`. The visual height comes from the `<td>` padding, not from the inner table. The empty table exists for Outlook compatibility — without it, Outlook can collapse the cell.[^14][^21]

```html
<td class="esd-block-spacer" style="font-size:0;padding:40px 0;">
  <table cellpadding="0" cellspacing="0" width="100%">
    <tr><td style="font-size:0;line-height:0;">&#160;</td></tr>
  </table>
</td>
```

***
## Part 5: Stripo's Mobile Override Pattern for Spacing
On desktop, spacing is controlled by inline styles and `es-p{n}` utility classes. On mobile, overrides go into the Custom CSS layer inside a `@media (max-width:600px)` block. Stripo generates unique block IDs (e.g., `es-text-8927`) to scope overrides to specific blocks.[^22]

For font sizes, the pattern is:
```css
@media only screen and (max-width:600px) {
  .es-text-8927 .es-text-mobile-size-16,
  .es-text-8927 .es-text-mobile-size-16 * {
    font-size: 16px !important;
  }
}
```

For padding overrides on mobile, Stripo generates similar scoped selectors. The `!important` is required because it must override the inline styles set at desktop level.[^11][^22]

**Important implication for the token pipeline**: mobile spacing tokens (from the Mobile mode in Figma) cannot be applied as inline styles — they must be applied inside media queries in the Custom CSS panel. This means the mobile portion of your token JSON maps to a different output target than the desktop portion.

***
## Part 6: Complete Figma → Stripo Token Mapping Model
Bringing the above together, a complete token from Figma maps as follows:

| Token category | Figma source | Stripo target | Layer | How |
|---|---|---|---|---|
| Heading font-size (desktop) | `typography/Desktop/H1/size` | `<h1 style="font-size:48px">` | Inline style on element | Direct inline on heading tag |
| Heading font-size (mobile) | `typography/Mobile/H1/size` | `@media` → `.es-text-{id} .es-text-mobile-size-48` | Custom CSS panel | Media query override with `!important` |
| Body font-family | `typography/Desktop/body/font-family` | `body, td, p { font-family: ... }` in General Styles | Default CSS panel | System-wide rule, not inline |
| Button background | `colour/button-fill` | `<a class="es-button" style="background-color:#xxx">` | Inline on `<a>` | Direct inline |
| Container padding | `spacing/container/pad-h` | `<td class="esd-container-frame" style="padding:0 24px">` | Inline on container `<td>` | Direct inline |
| Inter-block gap | `spacing/gap/section` | Spacer `esd-block-spacer` with `padding:24px 0` | Inline on spacer block `<td>` | Spacer block height |
| Between-element gap | `spacing/gap/element` | `padding-bottom` on upper block's `<td>` | Inline on block `<td>` | Block-level padding |
| Line height | `typography/Desktop/body/line-height` | `<p style="line-height:24px">` | Inline on element | px absolute, never % |
| Letter spacing | `typography/Desktop/H1/letter-spacing` | `<h1 style="letter-spacing:-1px">` | Inline on heading | px, Outlook-partial |
| Border radius (button) | `spacing/button/radius` | `<a style="border-radius:6px">` | Inline on `<a>` | Works except Outlook desktop |

CSS custom properties (`var(--token-name)`) must never be used in email HTML output — all token values must be resolved to their literal px, hex, or named values before insertion, because email clients strip or ignore CSS variables.[^2][^17]

---

## References

1. [Using CSS flexbox and grid for email templates](https://stackoverflow.com/questions/72343323/using-css-flexbox-and-grid-for-email-templates) - Can we use CSS flexbox and grid for making layouts in html email templates? And apart form using tab...

2. [Why Inline CSS Is Still Essential for HTML Emails](https://www.francescatabor.com/articles/2025/12/12/why-inline-css-is-still-essential-for-html-emails) - CSS support varies wildly between clients. Layout techniques like Flexbox or Grid are unreliable or ...

3. [Auto Layout in Figma - Epic Web Dev](https://www.epicweb.dev/tips/auto-layout-in-figma) - Figma's Auto Layout feature makes designing layouts feel more like building with Flexbox. Instead of...

4. [(New editor) Understanding of Stripo Template Layout](https://support.stripo.email/en/articles/6414520-new-editor-understanding-of-stripo-template-layout) - The structure of Stripo emails is based on the HTML table layout. The core elements of the email are...

5. [Understanding of Stripo Template Layout](https://support.stripo.email/en/articles/3173888-understanding-of-stripo-template-layout) - Each structure has a container and may have up to 8 containers in a row. And each container has a bl...

6. [Guide to auto layout – Figma Learn - Help Center](https://help.figma.com/hc/en-us/articles/360040451373-Guide-to-auto-layout) - Elements in an auto layout frame are automatically arranged in a frame based on direction, spacing, ...

7. [CSS Issues in Interactive Emails](https://www.emailservicebusiness.com/blog/css-issues-interactive-emails/) - While Flexbox enjoys decent compatibility, CSS Grid is still widely unsupported, forcing many design...

8. [HTML and CSS in Emails: What Works in 2026? - Designmodo](https://designmodo.com/html-css-emails/) - Designing emails with HTML and CSS allows for professional, branded, and visually appealing content....

9. [(New editor) What are the structures and containers? How to use ...](https://support.stripo.email/en/articles/6424840-new-editor-what-are-the-structures-and-containers-how-to-use-them) - This article is going to teach you about structures and containers, as well as how to work with them...

10. [What are the Structures and Containers? How to use them?](https://support.stripo.email/en/articles/3173911-what-are-the-structures-and-containers-how-to-use-them) - A structure can include up to 8 containers in a row. Just like other email elements, structures can ...

11. [HTML Email Spacing Techniques that (Usually) Work](https://www.emailonacid.com/blog/article/email-development/spacing-techniques-in-html-email/) - Spacing in HTML emails is a challenge. Learn tricks for creating spacing in HTML emails, such as cel...

12. [Modules - Stripo](https://viewstripo.email/branded-export/guidelines/8fd2c229-4375-46dd-9d34-5f0fe3606550/Modules/Modules.html) - This guide provides the ability to easily navigate between all created modules related to Sample ema...

13. [How to use Spacer block in Stripo](https://www.youtube.com/watch?v=SIFksNLf2eA) - To use the Spacer block in Stripo, you can follow these steps:

1. Open the email template you want ...

14. [(New Editor) What are the Basic Blocks? How to use them?](https://support.stripo.email/en/articles/6446095-new-editor-what-are-the-basic-blocks-how-to-use-them) - Blocks contain the basics to create email elements: image, text, button, spacer, social networks, me...

15. [How to Adjust Paddings and Indents in Newsletter Templates with ...](https://stripo.email/blog/adjust-paddings-indents-newsletter-templates-stripo/) - In this post, we are going to share some tricks on how to set indents, margins between content eleme...

16. [Email Spacing: Tips for Margins and HTML Email Padding](https://www.emailonacid.com/blog/article/email-development/7-tips-and-tricks-regarding-margins-and-padding-in-html-emails/) - Cellpadding and cellspacing attributes are safe but it's best to avoid CSS margins and padding withi...

17. [Can I email… "css" search results](https://www.caniemail.com/search/?s=css) - css column properties#. Support for the columns shorthand and longhand properties. Estimated Support...

18. [Outlook email display problems: understanding and solving them](https://www.badsender.com/en/2024/04/16/outlook-email-display-problems/) - Master Outlook email display problems: learn how to identify and correct common challenges with this...

19. [Code snippets for and tips for Outlook.com Inboxes - Email on Acid](https://www.emailonacid.com/tip/outlook-com/) - Tips for coding emails for Outlook.

20. [CSS padding is not working as expected in Outlook](https://stackoverflow.com/questions/21474239/css-padding-is-not-working-as-expected-in-outlook/21474504) - I have the following html in an email template. I am getting different view in MS Outlook and in Gma...

21. [HTML Emails - Empty Tables/TR/TD as spacers](https://stackoverflow.com/questions/7601442/html-emails-empty-tables-tr-td-as-spacers) - Has anyone had any experience with using empty tables, table rows or table cells as layout spacers? ...

22. [(New editor) The General styles for Desktop and Mobile views](https://support.stripo.email/en/articles/6433986-new-editor-the-general-styles-for-desktop-and-mobile-views) - Modify the Message Content width, which is 600px by default. Today, this is the most standard size a...

