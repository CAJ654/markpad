# markpad

A zero-dependency, single-file live Markdown editor with a collapsible syntax block palette. Built with Vanilla JS, HTML, and CSS — no build step, no framework, no external libraries beyond a Google Fonts CDN link.

---

## Features

**Live preview** — the preview pane re-renders on every `input` event, so you see formatted output as you type.

**Syntax block palette** — a code.org-style panel on the left groups 14 insert blocks across 6 color-coded categories. Clicking any block inserts the corresponding Markdown snippet at your cursor position. Categories collapse and expand with an animated chevron.

**Cursor-aware insertion** — blocks are inserted at the current selection point. Block-level syntax (headings, lists, code fences, blockquotes) automatically gets a preceding newline when dropped mid-line, so structure never gets mangled.

**Status bar** — live word count, character count, and line count update with the editor.

**No dependencies** — the entire app is one `.html` file. Drop it in a browser and it works.

---

## Supported Markdown Syntax

| Category    | Syntax                          | Output                        |
|-------------|---------------------------------|-------------------------------|
| Headings    | `# H1` / `## H2` / `### H3`    | `<h1>` – `<h3>`               |
| Bold        | `**text**`                      | `<strong>`                    |
| Italic      | `*text*`                        | `<em>`                        |
| Bold+Italic | `***text***`                    | `<strong><em>`                |
| Strikethrough | `~~text~~`                    | `<del>`                       |
| Inline code | `` `code` ``                    | `<code>`                      |
| Code block  | ` ``` ` fenced block ` ``` `   | `<pre><code>`                 |
| Unordered list | `- item` / `* item`          | `<ul><li>`                    |
| Ordered list | `1. item`                      | `<ol><li>`                    |
| Blockquote  | `> text`                        | `<blockquote>`                |
| Horizontal rule | `---`                       | `<hr>`                        |
| Link        | `[text](url)`                   | `<a target="_blank">`         |
| Image       | `![alt](url)`                   | `<img>`                       |

---

## How the Parser Works

The parser is split into two functions — no external library used.

**`inline(s)`** processes a single line of text using chained regex replacements. Images are matched before links to prevent the `!` from being swallowed by the link pattern. Inline patterns run longest-match-first (`***` before `**` before `*`) to avoid greedy conflicts.

```js
function inline(s) {
  s = esc(s);
  s = s.replace(/!\[([^\]]*)\]\(([^)]+)\)/g, '<img src="$2" alt="$1">');
  s = s.replace(/\[([^\]]+)\]\(([^)]+)\)/g,  '<a href="$2" ...>$1</a>');
  s = s.replace(/`([^`]+)`/g,                '<code>$1</code>');
  s = s.replace(/\*\*\*(.+?)\*\*\*/g,        '<strong><em>$1</em></strong>');
  s = s.replace(/\*\*(.+?)\*\*/g,            '<strong>$1</strong>');
  s = s.replace(/\*(.+?)\*/g,                '<em>$1</em>');
  s = s.replace(/~~(.+?)~~/g,               '<del>$1</del>');
  return s;
}
```

**`parse(md)`** splits the source on `\n` and walks each line, tracking three mutable state flags:

- `inUL` / `inOL` — whether a list is currently open. The parser opens a `<ul>` or `<ol>` when the first matching line is encountered and closes it as soon as the pattern breaks.
- `inPre` — whether a fenced code block is open. Lines inside a fence are buffered raw and HTML-escaped only at close, preserving indentation and special characters.

Block-level patterns (`#`, `>`, `---`) are tested with simple regexes and short-circuit with `continue`, so later branches never run on the same line. Unclosed fences at end-of-input are gracefully closed.

---

## Palette Architecture

Each category in the palette is a `.cat` element wrapping a `.cat-label` and a `.cat-body`:

```html
<div class="cat cat-h">
  <div class="cat-label">
    <svg class="cat-chevron">…</svg>
    Headings
    <div class="cat-line"></div>
  </div>
  <div class="cat-body">
    <button class="block" data-id="h1"># H1</button>
    …
  </div>
</div>
```

Collapse is CSS-driven via `max-height` on `.cat-body`. Toggling `.collapsed` on `.cat` animates the body to `max-height: 0; opacity: 0` and rotates the chevron `−90°`. JavaScript only adds/removes the class — no height measurements, no JS animation loops.

```css
.cat-body {
  max-height: 300px;
  opacity: 1;
  transition: max-height 0.28s cubic-bezier(0.4, 0, 0.2, 1),
              opacity    0.2s  ease;
}
.cat.collapsed .cat-body { max-height: 0; opacity: 0; }
.cat.collapsed .cat-chevron { transform: rotate(-90deg); }
```

Block insertion strings live in a plain JS object keyed by `data-id`:

```js
const BLOCKS = {
  h1: '\n# Heading 1\n',
  bold: '**Bold Text**',
  codeblock: '\n```js\n// your code here\n```\n',
  // …
};
```

---

## File Structure

```
markpad.html      ← entire app (HTML + CSS + JS, self-contained)
README.md         ← this file
```

---

## Usage

Open `markpad.html` in any modern browser. No server required.

To embed in an existing project, copy `markpad.html` and adjust the `body` height strategy if the app is nested inside another layout (the root uses `height: 100%; overflow: hidden` on `html` and `body`).

---

## Extending

**Add a new block** — append an entry to the `BLOCKS` map and add a `<button class="block" data-id="yourKey">` inside any `.cat-body`. The insertion and flash logic pick it up automatically.

**Add a new category** — copy an existing `.cat` div, change the `cat-*` class for the color token, add a matching set of CSS rules following the existing pattern (6 properties per category), and populate the `.cat-body`.

**Add a Markdown feature** — for inline syntax, add a `.replace()` call to `inline()`. For block-level syntax, add a branch inside the `parse()` loop before the paragraph fallback. Mind ordering: more-specific patterns should be tested before less-specific ones.

---

## Known Limitations

- No nested list support (e.g. indented sub-items).
- Table syntax (`| col | col |`) is not parsed.
- Inline HTML passthrough is not supported — angle brackets are escaped.
- `_italic_` and `__bold__` underscore variants are not handled, only `*` syntax.
- No syntax highlighting inside fenced code blocks (language class is applied but no highlighter is loaded).
