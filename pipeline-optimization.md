# Henri Lloyd — Pipeline Optimisation Guide
## Token-to-HTML via DOM Script (not line-by-line regex)

---

## Why this file exists

Applying spacing tokens to an HTML file by having Claude read fragments of the file, pattern-match class strings, and issue `Edit` tool calls one by one is the **slowest and most error-prone approach possible**. It costs:

- **Input tokens** — every `Read` call loads raw HTML into the context window. A 1 200-line email template uses ~15 000–20 000 tokens just to read, and Claude reads it 4–8 times across a single spacing pass.
- **Output tokens** — each `Edit` call requires Claude to reproduce the old and new strings in full. Dozens of `Edit` calls add up fast.
- **Fragility** — class-list order varies across Stripo-generated files. A regex that matches `es-p30l es-p50b es-p30r` on one template silently fails on another that stores the same classes in a different order.
- **Manual exception handling** — Claude stops to reason about "the dark footer structure" or "line 712" because exact string matching breaks on edge cases. This burns context.

**The fix:** Claude writes and runs a DOM-manipulation script once. The script handles all edge cases algorithmically in milliseconds. Claude never needs to open the HTML in its context window.

---

## The optimised prompt — copy-paste for new templates

Use this prompt verbatim when applying spacing tokens to a new HTML file. Adjust the file paths as needed.

```
We need to apply the spacing tokens from the Figma JSON files to a new HTML template.

DO NOT read the HTML file manually into context.
DO NOT attempt line-by-line regex replacements with Edit or Read tool calls.

Instead:
1. Read `Figma-local-variables/Spacing Tokens - New/Desktop.json` and
   `Figma-local-variables/Spacing Tokens - New/Mobile.json` to resolve all token values.
2. Write a Node.js script using `cheerio` (or Python using `BeautifulSoup`) that
   implements the DOM manipulation rules listed in `pipeline-optimization.md §4`.
3. Run the script against the target HTML file.
4. Verify the file was written correctly by reading only the key changed lines.
5. Update `figma-typography-custom.css` with heading margin-bottom rules from the tokens.

Target HTML: [PATH_TO_TARGET.html]
```

---

## Why the script approach is superior

| Dimension | Read → Edit approach | Script approach |
|---|---|---|
| HTML context tokens | ~15 000–20 000 per pass | 0 — HTML never enters context |
| Edit calls for spacing | 20–40 individual calls | 1 (run the script) |
| Class-order sensitivity | Fragile — order must match | Immune — operates on the DOM, not strings |
| Edge-case handling | Manual, token-consuming | Algorithmic, zero extra cost |
| Scalability to 5 000-line templates | Hours | Milliseconds |
| Risk of missing a structure | High | Zero — selector finds all matching elements |

---

## DOM manipulation rules — the script must implement these exactly

These rules encode the Stripo token architecture (see `token-map.md` §5–§6 and `stripo-rules.md` §3).

### Rule 1 — `esd-structure` padding

**Target selector:** all elements that have `esd-structure` in their class list.

**Action:**
1. Strip all existing `es-p{N}t`, `es-p{N}r`, `es-p{N}b`, `es-p{N}l` classes (desktop padding).
2. Strip all existing `es-m-p{N}t`, `es-m-p{N}r`, `es-m-p{N}b`, `es-m-p{N}l`, and `es-m-p{N}` (all-sides shorthand) classes (mobile padding).
3. Append the following desktop classes: **`es-p30r es-p40b es-p30l`** (top is 0 — omit).
4. Append the following mobile classes: **`es-m-p25r es-m-p30b es-m-p25l`** (top is 0 — omit).

**Do NOT change:** `esd-structure` itself, `bgcolor`, `style`, or any non-spacing classes.

**Exception — structures with intentional no-bottom:** A small number of `esd-structure` TDs deliberately omit a bottom class (e.g. a heading label between two padded rows). The script should log a warning for any `esd-structure` that had NO existing `es-p{N}b` or `es-m-p{N}b` before modification, so a human can review whether adding the bottom class is correct for that specific structure. Do not auto-add bottom to these without the warning.

### Rule 2 — `esd-block-text` element padding

**Target selector:** all elements that have `esd-block-text` in their class list.

**Action:**
1. Strip any existing `es-p{N}b` class.
2. Add `es-p15b` (element.pad-bottom = S = 15px).

**Do NOT change:** `es-p{N}t`, `es-p{N}r`, `es-p{N}l` — these are intentional per-block overrides, not covered by the global element token. Do not touch classes like `es-p20b`, `es-p10b`, `es-p12b`, `es-p8b` that were set intentionally on specific blocks (product cards, step connectors, etc.) — **only replace `es-p30b`** to avoid collateral damage.

> **Safer sub-rule:** Rather than stripping ALL `es-p{N}b` classes, strip only `es-p30b` specifically. This is the default element padding that the token changes (30→15). Other values are intentional exceptions.

### Rule 3 — `es-button-border` span and `es-button` anchor

**Target selector:**
- `span.es-button-border` where the button is a solid-background CTA (NOT the transparent text-link variant that has `padding: 0` or `background: transparent`)
- `a.es-button` inside the same span

**How to detect solid-background buttons:** the `<span class="es-button-border">` has NO `background: transparent` in its `style` attribute (or has no `style` at all). Transparent buttons (padding:0) are intentional and must not be touched.

**Action on solid-background button spans:**
- Merge into the existing `style` attribute: `border-radius: 12px; display: block;`
- Preserve all other existing inline style properties.

**Action on solid-background button anchors:**
- Merge into the existing `style` attribute: `padding: 15px 30px; border-radius: 12px;`
- Preserve all other existing inline style properties (color, font-weight, etc.).

### Rule 4 — Heading margin-bottom (CSS only, not HTML)

This rule applies to `figma-typography-custom.css`, not to the HTML.

After the script finishes the HTML, update the CSS file:

**Desktop (inside the base h1–h5 rules, not inside `@media`):**
```css
h1 { margin-bottom: 24px; }
h2 { margin-bottom: 20px; }
h3 { margin-bottom: 16px; }
h4 { margin-bottom: 12px; }
h5 { margin-bottom: 10px; }
```

**Mobile (inside `@media only screen and (max-width: 600px)`):**
```css
h1 { margin-bottom: 20px !important; }
h2 { margin-bottom: 16px !important; }
h3 { margin-bottom: 12px !important; }
h4 { margin-bottom: 10px !important; }
h5 { margin-bottom: 8px !important; }
```

---

## Reference script — Node.js with Cheerio

Save as `apply-spacing-tokens.js` in the project root. Run with `node apply-spacing-tokens.js`.

```javascript
/**
 * apply-spacing-tokens.js
 * Henri Lloyd — Apply Figma Spacing Tokens to a Stripo HTML template
 *
 * Usage: node apply-spacing-tokens.js <path-to-target.html>
 *
 * Requires: npm install cheerio
 * Cheerio docs: https://cheerio.js.org/
 */

const fs = require('fs');
const cheerio = require('cheerio');

const targetPath = process.argv[2];
if (!targetPath) {
  console.error('Usage: node apply-spacing-tokens.js <path-to-target.html>');
  process.exit(1);
}

// ─── TOKEN VALUES (resolved from Figma JSON) ────────────────────────────────
// Update these if token values change. Source of truth: Spacing Tokens - New/
const TOKENS = {
  desktop: {
    structure:  { r: 30, b: 40, l: 30 },   // t = 0, omit
    element:    { b: 15 },                  // only bottom is non-zero
    button:     { padding: '15px 30px', borderRadius: '12px' },
  },
  mobile: {
    structure:  { r: 25, b: 30, l: 25 },   // t = 0, omit
    button:     { padding: '10px 25px' },   // borderRadius same as desktop, no change
  },
};

// ─── HELPERS ────────────────────────────────────────────────────────────────

/** Remove spacing classes matching a pattern from a class string. Returns cleaned string. */
function removeSpacingClasses(classStr, patterns) {
  return classStr
    .split(/\s+/)
    .filter(cls => !patterns.some(rx => rx.test(cls)))
    .join(' ');
}

const DESKTOP_PADDING_RX = [
  /^es-p\d+[trbl]$/,   // es-p30l, es-p40b, es-p50t, etc.
];

const MOBILE_PADDING_RX = [
  /^es-m-p\d+[trbl]$/, // es-m-p25r, es-m-p30b, etc.
  /^es-m-p\d+$/,       // es-m-p20 (all-sides shorthand)
];

const ELEMENT_BOTTOM_TARGET_RX = /^es-p30b$/; // only replace the default 30px; leave intentional values

/** Merge new key:value pairs into an existing inline style string, preserving existing props. */
function mergeStyles(existingStyle = '', newProps = {}) {
  const existing = {};
  (existingStyle || '').split(';').forEach(decl => {
    const [prop, ...rest] = decl.split(':');
    if (prop && rest.length) {
      existing[prop.trim()] = rest.join(':').trim();
    }
  });
  Object.assign(existing, newProps);
  return Object.entries(existing)
    .filter(([, v]) => v)
    .map(([k, v]) => `${k}: ${v}`)
    .join('; ');
}

/** Build structure desktop class additions */
function structureDesktopClasses(t) {
  const classes = [];
  if (t.r) classes.push(`es-p${t.r}r`);
  if (t.b) classes.push(`es-p${t.b}b`);
  if (t.l) classes.push(`es-p${t.l}l`);
  return classes;
}

/** Build structure mobile class additions */
function structureMobileClasses(t) {
  const classes = [];
  if (t.r) classes.push(`es-m-p${t.r}r`);
  if (t.b) classes.push(`es-m-p${t.b}b`);
  if (t.l) classes.push(`es-m-p${t.l}l`);
  return classes;
}

// ─── MAIN ────────────────────────────────────────────────────────────────────

const html = fs.readFileSync(targetPath, 'utf8');
const $ = cheerio.load(html, { decodeEntities: false, xmlMode: false });

let structureCount = 0;
let structureNoBottomWarnings = [];
let blockTextCount = 0;
let buttonCount = 0;

// ── Rule 1: esd-structure padding ──────────────────────────────────────────
$('[class*="esd-structure"]').each((i, el) => {
  const $el = $(el);
  let cls = $el.attr('class') || '';

  // Check if this structure had NO existing bottom class before we strip
  const hadNoBottom = !/es-p\d+b/.test(cls) && !/es-m-p\d+b/.test(cls);

  // Strip all old spacing classes
  cls = removeSpacingClasses(cls, [...DESKTOP_PADDING_RX, ...MOBILE_PADDING_RX]);

  // Append new token classes
  const additions = [
    ...structureDesktopClasses(TOKENS.desktop.structure),
    ...structureMobileClasses(TOKENS.mobile.structure),
  ];
  cls = (cls + ' ' + additions.join(' ')).replace(/\s+/g, ' ').trim();

  $el.attr('class', cls);
  structureCount++;

  if (hadNoBottom) {
    structureNoBottomWarnings.push(`  → Element index ${i}: ${$el.attr('class')}`);
  }
});

// ── Rule 2: esd-block-text element bottom padding ──────────────────────────
$('[class*="esd-block-text"]').each((i, el) => {
  const $el = $(el);
  let cls = $el.attr('class') || '';

  if (ELEMENT_BOTTOM_TARGET_RX.test(cls.split(/\s+/).find(c => ELEMENT_BOTTOM_TARGET_RX.test(c)) || '')) {
    cls = cls.split(/\s+/)
      .map(c => ELEMENT_BOTTOM_TARGET_RX.test(c) ? `es-p${TOKENS.desktop.element.b}b` : c)
      .join(' ');
    $el.attr('class', cls);
    blockTextCount++;
  }
});

// ── Rule 3: solid-background buttons ──────────────────────────────────────
$('span.es-button-border').each((i, el) => {
  const $span = $(el);
  const existingStyle = $span.attr('style') || '';

  // Skip transparent/text-link buttons
  if (existingStyle.includes('background: transparent') ||
      existingStyle.includes('background:transparent')) {
    return;
  }

  // Update span
  const newSpanStyle = mergeStyles(existingStyle, {
    'border-radius': TOKENS.desktop.button.borderRadius,
    'display': 'block',
  });
  $span.attr('style', newSpanStyle);

  // Update inner anchor
  const $a = $span.find('a.es-button').first();
  if ($a.length) {
    const aStyle = $a.attr('style') || '';
    if (aStyle.includes('padding: 0') || aStyle.includes('padding:0')) return; // text-link
    const newAStyle = mergeStyles(aStyle, {
      'padding': TOKENS.desktop.button.padding,
      'border-radius': TOKENS.desktop.button.borderRadius,
    });
    $a.attr('style', newAStyle);
    buttonCount++;
  }
});

// ─── WRITE OUTPUT ──────────────────────────────────────────────────────────
fs.writeFileSync(targetPath, $.html(), 'utf8');

// ─── REPORT ────────────────────────────────────────────────────────────────
console.log(`\n✅ Done.\n`);
console.log(`  esd-structure elements updated : ${structureCount}`);
console.log(`  esd-block-text elements updated: ${blockTextCount}`);
console.log(`  Solid-background buttons updated: ${buttonCount}`);

if (structureNoBottomWarnings.length > 0) {
  console.log(`\n⚠️  REVIEW REQUIRED — ${structureNoBottomWarnings.length} esd-structure element(s) had no existing bottom`);
  console.log(`  padding class. Bottom padding was added (token: es-p${TOKENS.desktop.structure.b}b).`);
  console.log(`  Verify these are intentional or revert if they should stay at 0:\n`);
  structureNoBottomWarnings.forEach(w => console.log(w));
}

console.log(`\nFile written: ${targetPath}\n`);
```

---

## Reference script — Python with BeautifulSoup

Save as `apply-spacing-tokens.py`. Run with `python3 apply-spacing-tokens.py target.html`.

```python
"""
apply-spacing-tokens.py
Henri Lloyd — Apply Figma Spacing Tokens to a Stripo HTML template

Usage: python3 apply-spacing-tokens.py <path-to-target.html>

Requires: pip install beautifulsoup4 lxml
"""

import sys, re
from pathlib import Path
from bs4 import BeautifulSoup

if len(sys.argv) < 2:
    print("Usage: python3 apply-spacing-tokens.py <path-to-target.html>")
    sys.exit(1)

target_path = Path(sys.argv[1])

# ─── TOKEN VALUES ────────────────────────────────────────────────────────────
TOKENS = {
    "desktop": {
        "structure":  {"r": 30, "b": 40, "l": 30},
        "element_b":  15,
        "button_padding": "15px 30px",
        "button_radius":  "12px",
    },
    "mobile": {
        "structure":  {"r": 25, "b": 30, "l": 25},
        "button_padding": "10px 25px",
    },
}

# ─── HELPERS ─────────────────────────────────────────────────────────────────

DESKTOP_PAD_RX = re.compile(r'^es-p\d+[trbl]$')
MOBILE_PAD_RX  = re.compile(r'^es-m-p\d+([trbl]?)$')
ELEMENT_B_RX   = re.compile(r'^es-p30b$')  # only replace default 30px

def strip_spacing_classes(cls_list):
    return [c for c in cls_list if not DESKTOP_PAD_RX.match(c) and not MOBILE_PAD_RX.match(c)]

def build_structure_classes(t_desk, t_mob):
    desk = [f"es-p{t_desk[s]}{s}" for s in ("r", "b", "l") if t_desk.get(s)]
    mob  = [f"es-m-p{t_mob[s]}{s}"  for s in ("r", "b", "l") if t_mob.get(s)]
    return desk + mob

def parse_style(style_str):
    props = {}
    for decl in (style_str or "").split(";"):
        if ":" in decl:
            k, _, v = decl.partition(":")
            props[k.strip()] = v.strip()
    return props

def render_style(props):
    return "; ".join(f"{k}: {v}" for k, v in props.items() if v)

def merge_styles(existing_str, new_props):
    props = parse_style(existing_str)
    props.update(new_props)
    return render_style(props)

# ─── MAIN ─────────────────────────────────────────────────────────────────────

html = target_path.read_text(encoding="utf-8")
soup = BeautifulSoup(html, "lxml")

structure_count = 0
no_bottom_warnings = []
block_text_count  = 0
button_count      = 0

# ── Rule 1: esd-structure ─────────────────────────────────────────────────
for el in soup.find_all(class_=re.compile(r'\besd-structure\b')):
    cls_list = el.get("class", [])
    had_no_bottom = not any(re.match(r'^es-p\d+b$', c) for c in cls_list)

    cls_list = strip_spacing_classes(cls_list)
    cls_list += build_structure_classes(TOKENS["desktop"]["structure"],
                                        TOKENS["mobile"]["structure"])
    el["class"] = cls_list
    structure_count += 1

    if had_no_bottom:
        no_bottom_warnings.append(el.get("class"))

# ── Rule 2: esd-block-text ────────────────────────────────────────────────
for el in soup.find_all(class_=re.compile(r'\besd-block-text\b')):
    cls_list = el.get("class", [])
    if any(ELEMENT_B_RX.match(c) for c in cls_list):
        el["class"] = [
            f"es-p{TOKENS['desktop']['element_b']}b" if ELEMENT_B_RX.match(c) else c
            for c in cls_list
        ]
        block_text_count += 1

# ── Rule 3: solid-background buttons ──────────────────────────────────────
for span in soup.find_all("span", class_="es-button-border"):
    style = span.get("style", "")
    if "background: transparent" in style or "background:transparent" in style:
        continue

    span["style"] = merge_styles(style, {
        "border-radius": TOKENS["desktop"]["button_radius"],
        "display": "block",
    })

    a = span.find("a", class_="es-button")
    if a:
        a_style = a.get("style", "")
        if "padding: 0" in a_style or "padding:0" in a_style:
            continue
        a["style"] = merge_styles(a_style, {
            "padding": TOKENS["desktop"]["button_padding"],
            "border-radius": TOKENS["desktop"]["button_radius"],
        })
        button_count += 1

# ─── WRITE ────────────────────────────────────────────────────────────────
target_path.write_text(str(soup), encoding="utf-8")

# ─── REPORT ──────────────────────────────────────────────────────────────
print(f"\n✅ Done.")
print(f"  esd-structure updated : {structure_count}")
print(f"  esd-block-text updated: {block_text_count}")
print(f"  Buttons updated       : {button_count}")

if no_bottom_warnings:
    print(f"\n⚠️  REVIEW — {len(no_bottom_warnings)} structure(s) had no prior bottom class.")
    print(f"  Bottom class es-p{TOKENS['desktop']['structure']['b']}b was added. Verify these are intentional:")
    for w in no_bottom_warnings:
        print(f"  → {' '.join(w)}")

print(f"\nFile written: {target_path}\n")
```

---

## What Claude still does manually (not delegated to the script)

The script handles **structural HTML class sync**. The following tasks remain Claude's responsibility and require manual file edits or CSS writes — but they are cheap (small files, targeted edits):

| Task | Why it stays manual | Cost |
|---|---|---|
| `figma-typography-custom.css` heading margin-bottom | Small CSS file, targeted `margin-bottom` additions to h1–h5 rules | Very low |
| Mobile `@media` heading font-size / line-height rules | `es-text-{ID}` values are Stripo-assigned — must be read from the actual HTML once, then CSS is written | Low (one read, one write) |
| `es-text-mobile-size-{XX}` sync on heading elements | Requires reading Mobile tokens and comparing to existing class values | Low |
| Custom CSS cascade protection (§9 in stripo-rules.md) | Mandatory three-rule block; written once per template | Trivial |
| Button colour / brand tokens | Color tokens, not spacing — out of script scope | Low |

---

## Quick-start checklist for a new template

```
1. [ ] Resolve spacing tokens from Desktop.json + Mobile.json
2. [ ] Run `node apply-spacing-tokens.js <target.html>` (or Python equivalent)
3. [ ] Review any ⚠️  no-bottom warnings in the script output
4. [ ] Read the heading blocks (es-text-{ID}) — one focused read
5. [ ] Sync es-text-mobile-size-{XX} HTML classes where needed
6. [ ] Write @media CSS for heading sizes (using synced IDs and token values)
7. [ ] Add cascade protection block (stripo-rules.md §9) to the @media block
8. [ ] Add/update heading margin-bottom in Custom CSS (desktop + mobile)
9. [ ] Confirm button styles (padding, border-radius) are correct
```
