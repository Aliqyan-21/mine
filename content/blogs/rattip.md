---
title: my own ssg
date: 20-05-2026
template: blog
---

# RATTIP: MY OWN SSG

It all started when finally I decided that there are many things going on in my mind that should be written, and then decided to make my blog.

Then what I noticed was that people use either html,css from sratch or better, use static site generators like Hugo, Jekyll, etc.

First my insticts told me to just use `Hugo` or `Jekyll` or something... I couldn't.

Not because they are bad. they are fine. But *fine* was never the point.

The point was - I wanted to understand what actually happens when a markdown file becomes a webpage. I wanted to build the tool I build with. And I wanted it to be mine, completely, every line of it.

like just now, I realized that there is some extra space in starting of code block, so fixed it [here](https://github.com/Aliqyan-21/rattip/commit/f4eb18fdba438050ee3102e99bfc342a7f13c813). *Thus one of the benefit of coding our own tool.*

Like atleast for this time, because when I am using something to make my personal website, atleast let me know that tool as well as I built it, so... I did.

And thus rattip came into existence. **This website runs on it.**

## where it started

So, It started simple. A markdown file goes in, an HTML file comes out. That's it. How hard could it be?

The answer is - not that hard, but interestingly not trivial either.

I picked, after much ~~configuration~~ consideration (fight was between `md4c` and `markdownpp`)
[md4c](https://github.com/mity/md4c) as the markdown parser. It's a C library, fast, battle tested. It fires callbacks as it walks through the markdown (streaming parser), enter a heading, leave a heading, enter a paragraph, text, leave a paragraph. My job was to implement those callbacks and build HTML from them.

So, I did and made handlers for the blocks, spans and text. Easy...

`HTMLGen` : a C++ class that wraps md4c and accumulates HTML into a buffer. Every markdown element maps to a handler. `handle_h_enter`, `handle_p_enter`, `handle_code_enter` and so on. Simple, mechanical, satisfying.

The first time it generated a real HTML file from a markdown file I'd written, was so cool to see, when you use some library or pre-existing code and it works well, it seems like magic, then u get into it and see, ha this is predictable but still *COOL* !

## the pipeline

Once HTML generation worked, I needed the actual site generator. Enter `SSGen`.

The pipeline is simple:
1. Walk `content/` recursively, find all `.md` files
2. Parse frontmatter from each file
3. Generate HTML with `HTMLGen`
4. Inject into a template
5. Write to mirrored path in `public/`

The function doing this looks like this:

```cpp
/*
 * a recursive functin that will walk through the
 * main folder and find all the 'md' files and generate
 * html out of them and store in public folder (?)
 */
void SSGen::generate_site() {
  content_walker();
  generate_html();
}
```

and then:

```cpp
void SSGen::content_walker() {
  for (const std::filesystem::directory_entry &en :
       std::filesystem::recursive_directory_iterator(main_dir_)) {
    if (en.is_regular_file() && en.path().extension() == ".md") {
      // md_files_.push_back(en.path());
      std::string content = load_file(en.path());
      FMatter     fm      = parse_front_matter(content, en.path());
      files_.push_back({en.path(), content, fm});

      if (fm.nav) {
        std::filesystem::path rel =
          std::filesystem::relative(en.path(), main_dir_);
        std::filesystem::path url = std::filesystem::path("/") / rel;
        url.replace_extension(".html");
        nav_pages_.push_back({fm.title, url.string(), fm.nav_order});
      }
    }
  }
  V66V("Found ", files_.size(), " markdown files");
  V66V("Found ", nav_pages_.size(), " nav items");

  std::sort(nav_pages_.begin(), nav_pages_.end(),
            [](const NavL &a, const NavL &b) { return a.order < b.order; });

  navbar_ = "<nav class=\"rattip-nav\">\n";
  for (auto &item : nav_pages_) {
    navbar_ += "  <a href=\"" + item.url + "\" class=\"rattip-nav-a\">" +
               item.title + "</a>\n";
  }
  navbar_ += "</nav>";
}
```

Frontmatter looks like this:

```
---
title: My Post
date: 2024-01-15
template: blog
nav: 1
---
```

The template system is just placeholder replacement — `{title}`, `{md_content}`, `{nav_bar}`, `{blog_date}`. No template engine, no logic, just string replacement. Turns out that's all you actually need.

The directory mirroring was a nice moment - using `std::filesystem::relative()` to compute output paths. `content/blogs/first_post.md` becomes `public/blogs/first_post.html` automatically, including creating intermediate directories. aah `std::filesystem` also one of the magical tools to use.

## theming

I wanted the styling to be clean and separable. My solution was to give every HTML element a `rattip-` prefixed class. `rattip-h1`, `rattip-p`, `rattip-code`, `rattip-nav` and so on. User writes CSS targeting these classes, done.

No config files for class names, no theme maps. Just CSS. The themes are purely CSS files that live in `themes/name/global.css` and get copied to `public/styles/` on build. And u can make seperate themes, just make templates for linking those themes and using those templates for the particular markdown file, done scene!

The default theme is noir - dark amber gold on near-black... claude made it.

## incremental builds

Regenerating everything on every run could be wasteful. So there is a feature in rattip that compares timestamps -
if `public/index.html` is newer than `content/index.md`, skip it. Only regenerate what changed.

> p.s. again another commit to fix something crucial: [here](https://github.com/Aliqyan-21/rattip/commit/14f7c83bd6243a82c2e1956a0b787e903beccd85) mid writing this blog

`std::filesystem::last_write_time()` does the heavy lifting. The tricky part was a classic short-circuit evaluation bug, calling `last_write_time` on a file that doesn't exist yet because `exists()` check wasn't actually preventing it. Classic case of computing a `bool` before the `if` statement.

## the server

`rattip --serve` starts a local HTTP server. Using just `socket` and `netinet`.

`socket()`, `bind()`, `listen()`, `accept()` in a loop. Parse the HTTP request, find the file, send it back with correct MIME type. About 80 lines of C++.

The part that surprised me - it's really not that complicated for a static file server. HTTP/1.1 for serving files is just reading a path and writing bytes back. The hard parts of HTTP are the parts I didn't even needed for this, and silly me was thinking about using `cpp-httplib1` and all at one time!.

## file watching and live reload

This part was also very fun.

First I thought of using `inotify`, but that was not necessary (as I realized), like...another stupid dependency and it did not worked recursively throught directories too. what??

Instead now a polling watcher runs on a background thread and every 500ms, walk `content/`, compare timestamps against a snapshot, regenerate anything that changed. Simple, no inotify, works recursively by default.

But regenerating without the browser knowing is useless. Enter `SSE`

At first I was thinking of using websockets, then I read about things and came to know about SSE(Server Sent Events).

Think like this Websockets is like two people meeting, communication from both sides, complicated.
SSE on the other hand is like listening to your wife, single way communication.

Server Sent Events. The browser opens a persistent HTTP connection to `/livereload`. When a file changes, the server sends `data: reload\n\n`. The browser gets it and calls `location.reload()`.

The elegant part is SSE is just HTTP with `Content-Type: text/event-stream` and a connection that stays open. No WebSocket handshake, no binary framing protocol. One line of JS: `new EventSource('/livereload').onmessage = () => location.reload()`. Injected into every HTML response in memory, files on disk stay clean.

The watcher watches four directories — `content/`, `templates/`, `themes/`, `assets/`. Each has its own reload flag. CSS change → copy theme to public → reload. Template change → reload templates → regenerate all HTML → reload. Content change → regenerate that file → reload.
As I said above.

Save a file, see the change in the browser in under a second. That's the moment. That was so beatiful to watch,
haha like boom, writing here rendering there, tadaaa!

## the fun features

**Navbar** : pages with `nav: 1` (or any number) get added to the navbar, sorted by that number. Collected during the content walk, built once, injected into every page.

**Spoilers** : `||hidden text||` becomes a blurred span. Click to reveal. No CSS dependency, no JS file, just an inline `onclick` and `filter: blur(4px)`. Works on any theme, any colors, completely self-contained. *(coming in next version release, though present in main)*

**Task lists** : `- [ ] todo` and `- [x] done` render as disabled checkboxes. md4c provides the `is_task` and `task_mark` fields, I just handle them in `handle_li_enter`. *(coming in next version release, though present in main)*

**Syntax highlighting** : highlight.js via CDN in the blog template. The code blocks already have `lang-cpp`, `lang-python` etc. classes from the generator. highlight.js picks them up automatically.

**--init** : run `rattip --init` in an empty directory and it scaffolds everything. Templates, theme, sample content. The initial setup data lives as raw string literals in a header file, no external files needed,
just take the binary with you brother, in your chest, and use it when u need it.

## what I learned

The filesystem API in C++17 is genuinely good. `recursive_directory_iterator`, `relative()`, `last_write_time()`, `create_directories()`,
once you know these exist, you reach for them naturally.

Short circuit evaluation applies to `if` conditions, not to `bool` variable assignments. Obvious in retrospect, bites u though, bit me once, maybe in future too, I am careless.

SSE is criminally underused. For anything one-directional like notifications, live updates, progress, it's simpler than WebSocket in every way ; Use It!

The best solutions are sometimes the ones that aren't solutions. The "blog listing" feature was just... `blogs/index.md`. The walker treats it like any other file. It works because the pipeline is uniform.

Complexity is usually a failure of design. The features that feel magical like spoilers, live reload, auto-navbar are all under 50 lines each. Just because everytime I stopped and thought,
bro I don't need to make it complex, I want the most simples and laziest solution, because me lazy.

## what's next

- [ ] LaTeX math support. Going to make own c latex math parser (I'll call it lx4c *wink wink*).
- [ ] Post-processing and HTML minification for cleaner output and faster html.

---

rattip is [open source](https://github.com/aliqyan-21/rattip) under apache-2.0 license. if you want to build your own site with it, `rattip --init` is the beginning.
