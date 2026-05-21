---
title: about migr
template: blog
date: 22-05-2026
---

# I Forgot Where I Put My Notes. So I Built an IR.

Let me tell you something embarrassing.

I once lost a note. Not in the dramatic "years of research vanished" way.
Just a `.md` file I had written in Vim, somewhere in my filesystem, that I could not find.
I eventually found it with some `grep magic`. But while I was looking for it, I kept thinking that
"this file has no idea where it belongs. It doesn't know what it's related to.
It doesn't remember the chain of thought that produced it..."

It was just...sitting there. All alone (like me). Orphaned from context.

And so that icked me more than the lost note itself.

---

## The Real Problem With Note-Taking

Most note-taking systems, even the good ones, are really just fancy drawers.

You put something in. You can get it back out. Maybe you can search by filename, or date. Maybe you have folders. But the *relationships* between notes? Those are on you to maintain. Manually. In your head. And your head, as I'm sure you know, is not a reliable filesystem.

This is fine when you have twenty notes. But as the system grows, as your notes multiply
and your ideas sprawl, structure stops being a nice-to-have and becomes load-bearing.
A thousand notes with no meaningful connections isn't a knowledge base.
It's just noise with a directory listing.

I discovered Obsidian, and for a while it felt like magic. Backlinks! A visual graph!
Notes that could reference other notes and **know** about it.
The container had become a network.

But here's the thing..., Obsidian was not mine (Déjà vu). It was polished and capable and closed.
I could use it, but I couldn't understand it, not really. I couldn't see the gears. And as a systems developer, that bothers me. A lot.

I didn't just want a tool that worked. I wanted a tool I could *look into*.

So I started building `Creolynator`. And somewhere along the way, I ended up inventing an IR.

---

## Why Markdown Was The Wrong Starting Point

When I decided to parse markup, I naturally reached for Markdown. Everyone uses Markdown. There are a thousand parsers for it. How hard could it be?

Harder than it looks.

Markdown is intentionally loose. That's not a bug, it was designed to be human-readable ASCII, casual and forgiving.
But that looseness becomes a liability the moment you try to build *on top of it*.
GitHub Markdown and Slack Markdown are both "Markdown," but they interpret the same syntax differently.
The spec reads more like a suggestion than a law. Every edge case is a decision.
Every decision compounds. Before long, you're not writing a parser; you're writing a referee.

> now ofc I could have used already existing parser like I used for `rattip` but I was into parsers
at that time and wanted to write my own for that project...yeah.

So, I needed a grammar I could trust.

That's when I found **Creole**.

Creole is different. It has rules. Not many but firm ones. A heading is `= text =`. A link is `[[target]]`. The syntax is constrained. The grammar is unambiguous. Creole was actually designed by a consortium to solve this exact problem: markup that is both human-readable *and* formally parseable. 

When I switched to Creole, parsing became pleasant. And parsing Creole produced something beautiful: a tree.

---

## The Tree Is Not Enough Though

Every parser produces a tree. It's natural, the structure of a document is inherently hierarchical.
Headings contain paragraphs, paragraphs contain text and links.
The Abstract Syntax Tree captures all of this: it abstracts away the syntax
(goodbye `[[`, `]]`, etc.) and preserves the structure and content underneath.

ASTs are powerful. Every compiler, every language processor you've ever used relies on one as far as I know.
You can traverse them, transform them, generate output from them.
For a lot of problems, an AST is exactly what you need.

But for a *note-taking system*, the AST falls short. And the reason is interesting.

So, see a tree is a hierarchy. It models containment and order beautifully. But knowledge isn't containment.
Knowledge is *relationship*. And relationships often don't fit into a tree.

Think about it:

A note tagged `#project-x` is *grouped with* other notes tagged `#project-x`. That's not a parent-child relationship. The tag doesn't contain the notes. The notes share a label, a membership in a set, a semantic bond.

A note that *links to* another note creates a connection between peers, not between parent and child. In a tree, a node has exactly one parent. In a knowledge graph, a node can be reached from dozens of directions. It can belong to many groups simultaneously. It can reference and be referenced.

And backlinks, the set of all notes that point to *you*, those don't even exist in the tree. You'd have to reconstruct them by scanning every other note.

So here u can come to the conclusion that for note-taking you might need something more
than a tree. But what?

---

## The Graph, and the Problem With Just Using One

The obvious answer is: use a? ||graph||.

Tools like Obsidian do this. The document is parsed into a tree (for structure),
and then a semantic graph is layered on top, nodes for tags, nodes for links, edges connecting everything.
This is better. It captures the meaning, not just the shape.

But there's a tension here. Most implementations treat the tree and the graph as *separate systems*.
The tree tells you what's in the document. The graph tells you what's in the document.

You end up with two data structures that know about each other but don't really *understand* each other. Querying across them is awkward. Updating one doesn't automatically update the other. The seams show.

I stared at this for a while, and then something clicked.

---

## The Insight: Layers

What if the tree and the graph weren't separate systems? What if they were *layers of the same representation*?

What if a single node could simultaneously participate in a structural hierarchy *and* a semantic network?
What if you could traverse either layer independently, or query across both, with the same code?

This is the core idea behind **MIGR** - **Multilayered Intermediate Graph Representation**.

It's not any revolutionary idea in absolute terms. Graph databases have existed for decades.
Layered representations appear in compiler design and database theory.
But as I saw, it's uncommon in document processing.
Most intermediate representations were mainly AST-based or semantic-based.

MIGR chooses blends both, in creamy smoothie way...yum yum.

---

## What MIGR Actually Is

Let me break down the name, because it earns it.

**Intermediate:** MIGR is not the source markup. It's not the output format.
It lives in between, in the same way that compiler IRs live between source code and machine code.
Parse Creole → MIGR → (HTML, PDF, EPUB, whatever you want). The IR is the working surface.

**Graph:** Fundamentally, at its core, MIGR is a graph. Nodes and edges, not a tree. Though trees are in there too, as you'll see.

**Multilayered:** Not a single graph, but multiple graphs, each with its own semantics, yet interconnected through shared nodes.

**Representation:** It represents a document. Not just the content, but the structure, the meaning, the connections.

Now layers huh?...

### The Structural Layer

This is the tree. Headings, paragraphs, lists, images, the skeleton of the document.
What you'd see if you collapsed all the formatting and looked at the outline.

This layer answers: *What is the document about, structurally? What is the hierarchy?*

### The Inline Layer

Nested within the structural layer. This is the fine grain; bold, italic, links, inline code. The texture of the content.

This layer answers: *How is content formatted within blocks?*

### The Semantic Layer

This is the graph. References, tags, backlinks, cross-references. The web of meaning that connects notes to each other and to concepts.

This layer answers: *What does this document mean? What does it reference? What other notes point to it? What does it belong to?*

Three layers. Each independent. Each queryable on its own. But connected through a shared abstraction.

---

## The Node: The Heart of Everything

Here's the elegant part.

Every node, across every layer, is the same object. One unified interface. Each node has:

- An **identity** (a unique ID)
- A **type** (what kind of thing it is — heading, paragraph, tag, link...)
- **Content** (what it holds)
- **Metadata** (information *about* it)
- A **version** (how many times it's changed)
- A **hash** (a fingerprint of its content)

And the best part is that each node participates in both kinds of edges:
- structural edges (parent-child, for the tree layers) and
- semantic edges (cross-references for the semantic layer, but able to point to *any* node anywhere).

Why does this matter?

Because it means you can write generic code, a visitor, a traversal algorithm,
a serializer that works on *any layer* without modification. The code doesn't need to know whether it's walking the structure or the semantic graph. It just walks nodes and edges. The layers are logical distinctions, not physical ones.

That unification is what lets you ask interesting cross-layer questions naturally.
Things like: *"What are all the headings tagged #important?"*

In a system with separate tree and graph structures, that's a join across two systems.
In MIGR, a heading node can simply have semantic edges to tag nodes. The question becomes a natural traversal.

---

## How MIGR Stacks Up Against the Alternatives

I think it's worth taking a moment to put MIGR in context,
because these comparisons helped me understand what I was actually building.

**Versus Traditional ASTs:** An AST is a linear pipeline: parse, get a tree, traverse the tree, generate output.
It answers *"what is the document?"* MIGR adds layers that let it also answer
*"what does the document mean, and what is it connected to?"* ASTs are focused.
MIGR is richer, at the cost of some complexity, which is a trade-off I consciously accepted.

**Versus Semantic Webs and RDF:** RDF represents everything as triples: `note1 references note2`, `note1 hasTag project-x`.
Very flexible and powerful. But completely disconnected from the document's structure.
An RDF store doesn't know that `note1` has a heading followed by two paragraphs.
That structural information is lost, or stored separately.
MIGR keeps both. Structure and semantics coexist in the same model as we read.

**Versus Obsidian's Model:** As I understand it, Obsidian parses documents into a tree *and* separately extracts a semantic graph.
They're connected at the application level, but implemented as separate data structures.
MIGR unifies them at the model level. Both tree edges and semantic edges attach to the same nodes. The separation is logical, not physical.

**Versus Hypergraphs:** A hypergraph lets a single edge connect multiple nodes at once.
MIGR uses only simple pairwise edges for now. Less expressive, but also simpler to implement and reason about.
For note-taking, pairwise relationships turn out to be sufficient most of the time anyways,
a note can have multiple tags, and you can query by multiple tags. The same effect emerges, just through different paths.

---

## The Mechanical Clock

I want to talk about the philosophy for a moment, because this is the part I care about most.

There's an analogy I keep coming back to: `the mechanical clock`.

A mechanical clock is a complex machine that is easy to use.
You look at it and read the time. Simple.
But inside, something remarkable is happening; gears meshing with precision, springs storing and releasing energy, escapements regulating the rhythm. Everything is visible. Open the case, and you can see how it works. The complexity is *organized*, and it is *accessible*.

Most software today is the opposite. It's opaque.
You use the surface. The internals are hidden, not by necessity, but by design.
The system was built to be *used*, not understood.

I find this frustrating as both a developer and a user. If you don't understand the tool, you're at its mercy.
You can't extend it. You can't fix it when it breaks in unexpected ways.
You can't even know *why* it behaves the way it does. Frustrating...

MIGR is designed to be open like it's dancing in a strip club...ahem. The layers are transparent.
The node model is learnable. You can look at the code and understand what it does.
You are not insulated from the machinery, you are *invited into it*...ahem.

`Richard Stallman` has argued something on the lines of "software freedom matters because users are not passengers.
I find this compelling. MIGR is built in that spirit.
Not perfectly, *is any system perfect?* but in spirit.

The layers are also a direct expression of this philosophy. Each layer has a single, clear responsibility.
Structure does structure. Semantics does semantics. They don't bleed into each other.
This is why the model is learnable:
you can understand one layer without needing to understand all of them.

---

## What MIGR Borrows From

Good systems steal. Here's where MIGR's ideas come from.

**From compiler design:** The IR concept itself.

**From database design:** Separate the schema from the query language.
Define the structure once. Then query it in any way you want. MIGR does this across layers: the node/edge model is the schema, and traversal strategies are the query language.

**From graph theory:** Graphs are more natural for representing meaning than trees.
I had to learn from mistakes by trying to use a tree for something it couldn't express.
I learned that, trade discussions you can do, but sometimes you have to feel the limitation before the alternative makes sense.

**From Unix philosophy:** Keep things small and composable. Each layer is independent.
Each operation is single-purpose. Chain them together for complex answers.

---

## What MIGR Is Not (Being Honest Here)

Let me be clear about what MIGR doesn't do, because ofc, this is necessary to know.

- It is not a database. MIGR lives in memory. It's not optimized for millions of nodes.
It's not persistent by default. It's a document-scale tool, and that's intentional. Don't try to use it for enterprise data.

- It is not a semantic web engine. It's not RDF, not OWL, not a knowledge representation standard.
It's simpler and more specific. Designed for notes and documents, not arbitrary knowledge graphs.

- It is not a replacement for Obsidian. Obsidian is a UI, a sync service, a community, an ecosystem.
MIGR is an IR. These live at different layers of abstraction.
**MIGR** could *power* a note-taking system. **Obsidian** could theoretically *use* MIGR as its IR.
But they're not comparable things.

- And most importantly: **it's not finished**...hehe.

- The traversal layer isn't completed yet. The builder API isn't written yet. Optimization hasn't been touched. MIGR is a foundation. A foundation I believe in, and one that is sound. But not a house.

---

## The Problems Still Ahead

I think it's worth spending a moment on the open problems, because they're instructive about what "finished" would even mean.

**Persistence:** Right now, MIGR disappears when the process ends. Serialization is implemented; deserialization isn't.
For MIGR to be a real document format, it needs to survive to disk and come back.

**Incremental updates:** Currently, any change rebuilds everything from scratch.
For a batch tool, this is fine. For real-time editing this is wasteful in a way that would become very obvious very fast.

**Extensibility:** Node types and edge types are currently enums. Fixed.
To add a new semantic type, you modify the core code.
That's just as is for now, but needs changing in future.

**Performance:** Queries are currently O(n). ID generation isn't thread-safe. Hashing is placeholder-quality. None of this matters at document scale. All of it starts to matter at corpus scale. I'm aware of this. I'm not worried about it yet, and that's an intentional call.

---

## The Lesson I Keep Relearning

I built MIGR incrementally. I didn't spend a year in theory first. I built, hit limitations, redesigned, built again. The current architecture emerged from that process, not from a whiteboard.

I think this is the right way to build.

Theory without building is speculation. You can produce beautiful diagrams of systems you have no idea how to implement, or that solve problems you haven't actually experienced.

Building without theory is flailing you accumulate code without understanding, and then you're surprised when it breaks.

The sweet spot is: understand the space, sketch the design, build to learn, refactor when patterns emerge.
This is pragmatism. The result isn't theoretically perfect, but it's *practically sensible*.
The layers are clean but not over-engineered. The node model is flexible but not baroque.

There's something almost biological about this. A cell looks simple from the outside.
Inside, it's doing thousands of operations simultaneously,
organized by an architecture that evolved incrementally over billions of years,
not designed top-down, but refined through iteration. The complexity is organized.
Layered. Each layer serves the whole.

**MIGR** won't take billions of years to mature, ||hopefully||. But the principle holds.

---

## Why This Matters (For You, Not Just For Me)

You might be thinking: "Cool project. Niche problem. Why should I care?"

Fair question. Let me answer it honestly.

You might not care about Creole, or note-taking, or Creolynator specifically.
But the *ideas* here apply anywhere you're building a system that processes structured information which is most systems.

The tension between trees and graphs is not unique to documents.
It shows up in AST design for programming languages,
in UI component hierarchies that need to express cross-cutting concerns,
in any system where hierarchy and relationship coexist and neither alone is sufficient.

The layered IR approach is a pattern, not just a solution.
Separate your concerns. Give each layer its own model and its own query logic.
Connect them through a shared node abstraction. This produces systems that are learnable, extensible, and honest about their complexity.

And the mechanical clock philosophy, *building things that are transparent,
not just usable* is something I genuinely believe the software world needs more of.
We've spent decades building walls around our systems so that users don't have to think about what's inside.
I think we've overcorrected. Some users *want* to think about what's inside.
Some users *need* to. Those users deserve better.

---

## What Comes Next

Creolynator and MIGR are early. There's a lot of unfinished ground. But the vision is clear.

A mechanical clock that happens to manage knowledge.

I'm going to keep building it incrementally. That's the only honest approach. I don't know what refinements will be necessary. I don't know what scale this will reach. I don't know what use cases will emerge. What I know is what I need to do next, and while I do that, I'll figure out what comes after.

Build. Learn. Refactor.

If you want to look at what exists so far, here's the [repo](https://github.com/aliqyan-21/creolynator). It's early, and it shows. But the foundation is there.

And if any of this resonates the frustration with opaque tools,
the desire to understand your systems from the inside,
the conviction that complexity deserves respect rather than burial; I'd love to hear from you.

---

*Thanks for reading. This one ran long, but I think MIGR earned it.*
