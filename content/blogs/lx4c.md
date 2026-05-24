---
title: lx4c
date: 24-05-2026
template: blog
---

# LX4C: MY OWN LATEX MATH PARSER

So if you read the rattip blog post, you must have saw this at the bottom:

> LaTeX math support. Going to make own c latex math parser (I'll call it lx4c *wink wink*).

That wink wink was me being optimistic.
I did not had much idea how to parse the latex math (*haven't used it much before*).
I just knew I wanted math to render on my blog without pulling in a JS library.

## why not just use katex

The obvious answer was use KaTeX...
inject a script tag, done. But then I thought about the spoiler feature in rattip.
It's written in such a way that there is no need for writing seperate CSS no JS,
just an inline onclick. Works everywhere, out of the box.

I wanted math in rattip also to feel like that.

KaTeX is around **280kb** of JavaScript: 
1. It needs a **CDN**.
2. It needs **CSS**.
3. It **runs client side**.
4. Every time the page load, the browser downloads it, parses it,
then renders your equation. For what? So that $E = mc^2$ shows up?

Too much for simple thing, so ofc there must be some alternative for it.

## the insight

I found out that browsers render MathML natively.
Chrome 109+, Firefox always (from the starting), Safari always.
It is just HTML. No JS, no CSS, nothing.
You emit:
```html
<\math><\mi>E<\/mi><\mo>=<\/mo><\mi>m<\/mi><\msup><\mi>c<\/mi><\mn>2<\/mn><\/msup><\/math>
```
and the browser renders it beautifully.

So now the question became: *can I convert LaTeX math to MathML?*,
at HTML generation time, before the browser ever sees it?

Yes. Obviously yes. That is lx4c...

## but wait, latex is turing complete

When I told a friend about my plan, they pointed me to this:

> Parsing LaTeX programmatically is notoriously difficult because LaTeX is technically a
Turing-complete, macro-expanded language.

True! But also, completely irrelevant to what I was doing.

LaTeX the document language is Turing complete.
Macros, conditionals, loops, the whole thing.
I think nobody parses that without an actual TeX engine.

LaTeX math notation is something else entirely.
Content between `$...$` is just tokens and tree structure.
No macros, no expansion, no conditionals.
When you write `\frac{a}{b}` you mean fraction, always, everywhere, no exceptions.

So I had a simple task at hand parse a small, well-defined,
context-free grammar that happens to use LaTeX syntax...

## the design

I wanted lx4c to feel like how it felt to use md4c.
Drop in `lx4c.h` and `lx4c.c`, done. Two files, no dependencies, no build system.

But I also wanted the architecture to be clean.
**md4c** separates parsing from rendering, parser fires callbacks,
you implement what you want...Same idea here.

Two layers:

```
lx4c.h / lx4c.c                     — parser, produces AST
lx_to_mathml.h / lx_to_mathml.c     — visitor, walks AST, emits MathML
```

The public API of the parser is two functions:

```c
lx4c_node *lx4c_parse(const char *latex, size_t len);
void       lx4c_free(lx4c_node *root);
```

That is the whole surface. Parse, get a tree, free when done.
The renderer is separate because tomorrow someone might want LaTeX to plain text,
or a linter, or something I have not thought of. Parser should know nothing about any damn output format!
Just parse my content! you slaveee...

## how it works

Three stages.

**Lexer** turns the raw string into tokens:

```
\frac{x^{2}}{\alpha + 3.14}

CMD:frac  LBRACE  IDENT:x  SUP  LBRACE  NUMBER:2  RBRACE  RBRACE
LBRACE  CMD:alpha  OTHER:+  NUMBER:3.14  RBRACE
```

`\alpha` strips the backslash so the parser gets `alpha` directly.
Single letters are `IDENT`.
Multi-letter names always come through `CMD`.
Numbers consume the dot so `3.14` is one token.

**Parser** builds the AST using recursive descent.
Each grammar rule is one function.

- `parse_row` eats atoms until it hits a stop token.
- `parse_atom` handles one unit.
- `parse_group` handles content inside `{...}`. They call each other recursively.

The key insight was `parse_group`: if it sees `{`, parse until `}`.
If it sees no `{`, parse exactly one atom.
This is what makes `x^2` and `x^{2+1}` both work correctly.

**The lookup table** is the heart:

```c
{"alpha",   CMD_IDENT, "α"},
{"frac",    CMD_FRAC,  NULL},
{"leq",     CMD_OP,    "≤"},
{"sum",     CMD_OP,    "∑"},
```

When the parser sees `\alpha` it looks it up, finds `CMD_IDENT` with symbol `α`,
emits `<mi>α</mi>`. When it sees `\frac` it finds `CMD_FRAC`,
knows to parse two groups, builds an `mfrac` node. The table is like the library.
When u want to add new things, just add it in the library done.

**Renderer** uses a visitor pattern. Each node type has a visit function:

```c
void visit_frac(lx4c_node *n, void *ctx, const lx4c_visitor *v) {
  append_buf(ctx, "<mfrac>");
  for (int i=0; i<n->child_count; ++i) {
  lx4c_accept(n->children[i], v, ctx);
  }
  append_buf(ctx, "</mfrac>");
}
```

Visitor visits the whole tree and emits mathml according to the nodes it finds.

## a bug that taught me something

Early on I had this in `parse_group`:

```c
static lx4c_node *parse_group(lx4c_parser *p) {
  if (p->curr.type == TOK_LBRACE) { parser_advance(p); }
  lx4c_node *row = parse_row(p, TOK_RBRACE);
  ...
}
```

When there was no `{`, it still called `parse_row(p, TOK_RBRACE)` which ate everything
until a `}` or EOF.
So `a_i^2 + b_i^2 = c_i^2` *($a_i^2 + b_i^2 = c_i^2$, hehe)* was producing one giant subscript containing the entire rest of the expression.

The fix that came at that time once I saw it...If there is no `{`, just parse one atom:

```c
if (p->curr.type == TOK_LBRACE) {
  parser_advance(p);
  lx4c_node *row = parse_row(p, TOK_RBRACE);
  if (p->curr.type == TOK_RBRACE) { parser_advance(p); }
  return row;
}
return parse_atom(p);
```

This is what `x^2` and `x^{2+1}` actually mean in LaTeX.
No braces means one token. Braces mean a group.
Once the code matched that mental model everything worked.

## the result

User writes this in markdown:

```
$x = \frac{-b \pm \sqrt{b^2 - 4ac}}{2a}$
```

rattip generates this:

```html
<math class="rattip-lx">
  <\mrow><\mi>x<\/mi><\mo>=</\mo><\mfrac>...<\/mfrac><\/mrow>
</math>
```

Browser renders this:

$$x = \frac{-b \pm \sqrt{b^2 - 4ac}}{2a}$$

Zero JS. Zero CSS. Zero CDN. Just HTML that the browser already knows how to render.

Same philosophy as spoiler. User writes something, magic happens, nothing to configure!

## what lx4c handles

Greek letters, named functions, operators, fractions, roots, square roots, nth roots,
superscripts, subscripts, both together, overlines, vectors, underlines,
text inside math, and graceful fallback for anything unknown.

That covers probably 95% of what appears in a technical blog.
The other 5% falls back to showing the raw token name inside `<mi>`,
which is odd but *I think*...not broken.

## what is next for lx4c

Things I want to add (maybe) later:

- [ ] `\left(` `\right)` stretchy delimiters
- [ ] `\begin{matrix}` basic matrices
- [ ] `\mathbf` font variants

But honestly the current 20 constructs are enough for everything I write...

---

lx4c is [open source](https://github.com/aliqyan-21/lx4c) under **apache 2.0** licence.

If you want math on your static site without pulling in a JS library, it is right there.
