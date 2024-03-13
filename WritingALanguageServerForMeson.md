# Writing a language server for Meson in Swift

This is a blog post about my [language server for meson](https://github.com/JCWasmx86/Swift-MesonLSP), outlining the
technical choices, difficulties, and differences in integration into different editors.

## Motivation
I already started writing a meson language server before. It was written in Vala. I chose Vala because it is
my favorite language. (Albeit Swift endangers this status for certain applications😉)

I wanted to write a language server for meson, because it would be a cool thing to have and because it would
be interesting to learn more about this.

My prior experience with language servers was primarily doing some hacking around the Vala Language Server.

This first attempt in Vala worked somewhat, but due to not planning things before implementing them, I had
one huge file that implemented everything, from hovering to diagnostics. This was a mess to untangle
and I wasn't that motivated anymore, so I moved on to other stuff.

After a few months, I revisited the idea of writing a language server, but after looking at the code I decided
that the time needed to fix the code would be as long as writing a new one. That's why I started another
attempt from scratch.

## Technical decisions

### Implementation language
First I checked what most language servers are written with. Microsoft has a [listing
of implementations together with their implementation languages](https://microsoft.github.io/language-server-protocol/implementors/servers).
A lot of language servers are written using either TypeScript or the language they are written for. So I made a list of languages with their pros and cons:
#### Vala
Foremost: Vala is a really cool language. If I designed a language, it would be nearly a 1-1 copy of Vala. If I rewrote the language server in Vala again, I would know what I have to do, so I could probably be a bit faster. Another pro for Vala is the amount of high-quality libraries available. (Often written in C) And most of these would be already installed on the system of users if they use GNOME.

The drawbacks of Vala are that it is quite a niche language and that I know what I have to do, as I wanted to learn something new during the rewrite
#### Rust
I'm sure a lot of people would think of using Rust here. The strength of rust are its ecosystem and the safety guarantees. (And its cute mascot🦀)

Now to the downsides: I once contributed a minimal amount of code to a rust program. The compile times made me - besides not that much free time - stop contributing there. I did a change, waited a few minutes for a recompile and then had to test my changes. As my programming style depends on very quick compile-test cycles, this was really annoying.

Another aspect I don't like about rust is the size of dependency trees. In this aspect it's like a less bad version of NPM.
#### C
It's a fun language, with a good and wide ecosystem of libraries. With C I could probably reduce memory usage to a minimum and have the best performance.

While C is a good language, it's showing its age. Manual memory management is something that often goes wrong. C Strings are awful.
#### Javascript/Typescript
A lot of language servers are written in JS/TS. That means there are battle-tested implementations of the LSP. Couple that together with the huge ecosystem, and you probably could write a language server quite quick.

The drawbacks are numerous: No types in JS, NPM with its huge dependency trees (Couple that e.g. typo squatting, and you have a huge problem. That's also a reason why I'm hesitant to use Node.js based programs)

#### Swift
I always wanted to look into using Swift, even if I don't even use any Apple device. Swift compiles to native code, an aspect I really like, and it is memory-safe unlike C. You have a huge ecosystem, and you can - if needed - easily bind C libraries.

The drawbacks are obvious: It's still an Apple-first language. A lot of tooling is only on macOS. Just a minimal amount of Linux systems have Swift compilers installed as opposed to e.g. rustc, gcc, etc.

But as it compiles to native code, I can often reuse tooling from C like Sysprof/Heaptrack. The symbols may be mangled, but demangling is just one patch away.
 
#### Python
For some people Python may be the obvious choice, as meson uses it, too and e.g. the parser could be reused. Couple it with the huge ecosystem, and you could write a language server probably a lot faster than in other languages.

The disadvantages are obvious. Python is simply slow. It's good for gluing together native libraries (See the ML ecosystem) or doing things that are e.g. IO/Network-bound, not CPU bound.

After looking around what languages I should use, I decided to use Swift. While I ruled out Rust for now, I will give it a second chance at some later point in other programs.

### Parsing
I just decided to use a tree-sitter based parser as I did before. For that, I reused [this one](https://github.com/bearcove/tree-sitter-meson) and
forked it for some bug fixes. A package for Swift was used that has bindings to the C API.

At some point, I will try to use a custom parser (Given it outperforms tree-sitter in every aspect), but that is a task for later versions.

### JSON-RPC Server
For that, I just decided to reuse as much as possible from the Swift language server (sourcekit-lsp), as it is a very stable and battle-hardened implementation. It allowed me to reuse the JSON-RPC implementation, the plumbing for making the language server work and the type definitions that allow to automatically (de-)serialize JSON.

## Implementation
I implemented the language server in several modules:

- `Swift-MesonLSP`: Just contains the CLI interface that starts the language server and some other tasks like testing.
- `MesonAST`: Contains the AST-Nodes, the type definitions, and some machinery to convert a string/file to an AST
- `MesonAnalyze`: Is responsible for stitching together a tree. It starts at the root file, extracts an AST from it, and for each `subdir('somedir')`, it parses the corresponding
`meson.build` file and patches the AST to reference this source file. Furthermore, this module contains the type analyzer that annotates each node with the possible resulting types, emits diagnostics, and caches metadata for language server usage. (This tree is called `MesonTree`)
- `LanguageServer`: Combines `MesonAST` and `MesonAnalyze`

Sadly getting all the type definitions for the meson API can't be automated. So I was forced to manually extract all possible arguments with all their possible types from the meson source code.
Meson can generate a JSON file with the API description, but sadly this is only implemented for the "core" functions and objects, while for modules the input YAML file is not written yet.
That's why I had to manually copy the documentation for hovering, too.
So it is possible, that there are some minor mistakes, but over time this should be fixable.

The entire flow for building a `MesonTree` is this:
1. Parse source file using tree-sitter, returning a tree-sitter node
2. Convert tree-sitter node to an AST implemented in Swift for ease of use
3. Walk over it to find `subdir` calls.
4. For each subdircall:
 - Go to (1)
 - Patch the call to reference the source file
5. Traverse the AST and annotate all nodes with their possible types, collect diagnostics and metadata

To avoid the mistakes of the first language server, I made a lot of work based on visitors that traverse the AST, allowing me to split up the work into several files.
### First naive implementation
I just decided to use the same approach for parsing as the first language server:
As soon as I get a `textDocument/didChange` notification, I reparse the tree and do the type checking, etc... Yes, this seems
to be inefficient and it was inefficient.

Another thing I noticed was the high memory usage. Each time I created a `MesonTree`, the memory usage rose by a few hundred MB.
A language server for a *buildsystem* shouldn't use 10GB of RAM.

That was the first time I read about memory management in Swift. Long story short, Swift has basically two types:
- Classes: Reference counted
- Value types: E.g. integers, `struct`s. These are always copied.

I checked what I used for all type definitions and changed them from `class` to `struct`, as I probably had some reference cycles
in the type definitions.
Sure this probably came with a minor performance overhead due to copying, but I decided that for now, this fix is good enough. Base memory usage was depending on the project size
between 60MB and 180MB, just slowly rising with each reparse. (To compare: My Vala implementation was somewhere between 2MB and 10MB)

Another aspect of reducing memory leakage was simply caching a `TypeNamespace` (Contains all functions/types known to meson)

Until I released Version 1.0, I was not capable of improving the performance or memory usage. An attempt to cache parts of the AST was failing due to a lot of
concurrency bugs etc... I created an instrumentation module that allows me to measure and aggregate the performance of certain sections in the code. I expose
this information using a web server on `localhost:65000`.

But as soon as I had the features I set for 1.0 I released it despite these flaws, as the performance was good enough for small projects (<5KLOC)

### Improving memory usage and performance

Swift was created by Apple. A consequence of this is that a lot of tooling is only available on macOS. The best thing about native executables is, that
you can often reuse C tooling with just some minor drawbacks (E.g. missing demangling of names).

What do we need to reduce memory usage and improve performance? - Measurements.

In my toolbelt there are two tools that I use for C and Vala:
- Heaptrack for tracking allocations
- Sysprof for profiling and tracking allocations

There is some overlap, but some aspects of Heaptrack regarding the listing of allocations are a bit better than in Sysprof.

So I ran my program using Sysprof and got somewhat expected results: A lot of the allocations come from the `MesonAST` module, especially the type definitions.
Another major factor in the number/size of the allocations was the copying of all the `struct`s.

A minor excurse: At that point, I simply decided to add support for Swift demangling in Sysprof, as it was annoying having to read all mangled symbol names. Sadly I have not found a good way for demangling without starting a process for each symbol name or pulling in LLVM, that's why I didn't attempt to upstream it.

Back to those type definitions. For each meson object type, I created a corresponding Swift struct/class for some convenience.

Suppose you have some meson type `foo`. It has a method `bar`, taking a few arguments like this:
```meson
bool|int|str foo.bar(bool|int|str a,
                     dict(str)|list(str)|env b,
                     c: int)
```
So each time, I instantiated a `foo` object, all method definitions would be recreated.
This means:
- 2 `bool`-Objects
- 3 `int`-Objects
- 4 `str`-Objects
- 1 `dict`-Object
- 1 `list`-Object
- 1 `env`-Object
This goes on recursively. `bool` has some methods taking/returning `str`. That means a few more `str`-objects.
So instead of having just one instance of each type, I have probably several thousand, especially of e.g. `bool`, `int`, or `str`.

So I moved the methods from the `struct`s to `TypeNamespace` as a dictionary `[String: [Method]]`. That still created a lot of objects,
but reduced the memory increase during each parsing process by a huge margin.

I continued with my journey to reduce memory usage or improve performance. In the `TypeAnalyzer` the most allocations occur. I wanted to reduce them, so I looked around and decided to try
converting all my meson type structs to classes. This had the opposite effect, as classes are bigger than structs due to the need for a bit of book-keeping (E.g. ref-counting). So I went
to my `TypeNamespace` and instead of creating those thousands of objects, I created just one instance of each class (Except dicts/lists, as they can have different child types) and reused it. This reduced memory usage and increased
performance by a wide margin.

I did (somewhat unscientific) measurements you can see in the [Changelog](https://github.com/JCWasmx86/Swift-MesonLSP/releases/tag/v1.1) and released 1.1.

### Improving the performance further.

There was still the problem, that if you typed fast (Let's say `n` letters), the entire tree was reparsed `n` times.

This was a huge problem, as those seem to reduce the performance of each other. For example, just parsing once took 700ms,
but parsing `n` times concurrently needed 1.3s.

So as a first measure, I decided to cache the AST that is created from the tree-sitter nodes. I just did that for unopened `meson.build` files,
as I can assert that they are unchanged (Ignoring on-disk modifications without the editor, maybe implementing cache invalidation could help)

This reduced the time to rebuild the `MesonTree` by 30% at best, depending on several factors like the number of open files in the editor.

But still, I had the problem with unnecessary reparses. So I just added an atomic counter, each time I started to rebuild the tree, I incremented it
and stored the new value in a local variable.
After each step, I checked, whether the counter was equal to the local variable. If not, I canceled the process.

### Monitoring the performance and memory usage
Now after all these optimizations, I had to introduce a way to find regressions easier. So I added a CI workflow on Github Actions,
that uses the language server and parses some projects, measuring the time needed, the number and amount of allocations outputting a
JSON file. This JSON file is pushed to another repo. Currently, I have no use for that data, but I plan to write an HTML-Generator for
shiny graphs. While it is not entirely accurate (Sometimes the workflow takes 50 minutes, sometimes an hour) it will show major regressions.

Just for the releases, I did it manually. You can find the dashboard here: https://github.com/JCWasmx86/Swift-MesonLSP/tree/master/Benchmarks (Just run `make serve` and open your browser at http://0.0.0.0:8000/)

## Limitations of this language server
At least at this point, I don't want to implement an interpreter for meson.

### set_variable/get_variable
Meson has support for dynamically creating variables:
```meson
set_variable('abc', 1)
# abc exists now 
```
My language server is able to recognize this pattern and evaluate it correctly, but for example for this code (From Glib), it will fail:
```meson
foreach f : functions
  if cc.has_function(f)
    set_variable('have_func_' + f, true)
  else
    set_variable('have_func_' + f, false)
  endif
endforeach
```
Without an interpreter, it's impossible to tell if e.g. this code is correct:
```meson
x = have_func_strcpy
```
For now, the language server will emit an error, as `have_func_strcpy` was not found.

### subdir
Another bit of example code (From GTK) where the missing interpreter is showing:
```meson
foreach backend : ['broadway', 'wayland', 'win32', 'x11', 'macos']
  if get_variable('@0@_enabled'.format(backend))
    subdir(backend)
    gdk_deps += get_variable('gdk_@0@_deps'.format(backend))
    gdk_backends += get_variable('libgdk_@0@'.format(backend))
    # Special-case this for now to work around Meson bug with get_variable()
    # fallback being an empty array, or any array (#1481)
    if backend == 'wayland'
      gdk_backends_gen_headers += get_variable('gdk_@0@_gen_headers'.format(backend))
    endif
  endif
endforeach
```
Especially not being capable of following these `subdir` calls causes a lot of issues, if variables defined there are simply missing.

### Solving these issues without interpreting.
An limited interpreter could solve this, but that's something I currently don't have the time to do.

But for later releases it is probably a requirement that I don't miss any `(s|g)et_variable/subdir` calls.
## Integration into IDEs

### GNOME-Builder
I used GNOME-Builder for writing this project. Obviously, I need a plugin to test my server.

Adding support for a language server to GNOME Builder is quite easy.
#### As an external plugin
Here the most difficult or time-consuming part is not implementing the plugin, but setting up the build system.
Luckily I have [a repo with my personal plugins](https://github.com/JCWasmx86/GNOME-Builder-Plugins), so I didn't have to implement that from scratch.

You need two components: A shared object to load and a plugin description file

The shared object contains 
```vala
// A helper function, implemented in C
[CCode (cname = "bind_client")]
extern void bind_client (Ide.Object self);
public class MesonService : Ide.LspService {
    construct {
        this.set_program ("Swift-MesonLSP");
    }

    public override void prepare_run_context (Ide.Pipeline pipeline, Ide.RunContext run_context) {
        run_context.append_argv ("--lsp");
    }

    public override void configure_client (Ide.LspClient client) {
        client.add_language ("meson");
    }
}

public class MesonDiagnosticProvider : Ide.LspDiagnosticProvider, Ide.DiagnosticProvider {
    public void load () {
        bind_client (this);
    }
}
// ... some more classes ...
public void peas_register_types (TypeModule module) {
    var obj = (Peas.ObjectModule) module;
    obj.register_extension_type (typeof (Ide.DiagnosticProvider), typeof (MesonDiagnosticProvider));
    // ... some more classes ...
}
```
The plugin description file contains:
```ini
[Plugin]
Authors=JCWasmx86
Copyright=JCWasmx86
Description=Language server integration for meson
Loader=C
Module=mesonlsp
Name=MesonLSP
X-Category=lsps
X-Hover-Provider-Languages=meson
X-Symbol-Resolver-Languages=meson
X-Symbol-Resolver-Languages-Priority=800
X-Highlighter-Languages=meson
X-Highlighter-Languages-Priority=100
X-Formatter-Languages=meson
X-Completion-Provider-Languages=meson
X-Builder-ABI=SOMEVERSION
```
Compile it, copy it to the correct location and that's it! That's trivial
#### Upstream
*Note* My language server has no integration upstream yet, but for the sake of a complete entry I added this section.

You just need three components. (Shown here for the Vala language server):

A `settings.json`
```json
{
  "vala-language-server" : {
  }
}
```
A `vala-language-server.gresource.xml`:
```xml
<?xml version="1.0" encoding="UTF-8"?>
<gresources>
  <gresource prefix="/plugins/vala-language-server">
    <file>vala-language-server.plugin</file>
    <file>settings.json</file>
  </gresource>
</gresources>
```
and the most interesting thing: `vala-language-server.plugin`:
```ini
[Plugin]
Authors=Princeton Ferro, Ben Iofel, Christian Hergert
Copyright=Copyright © 2020 Princeton Ferro, Ben Iofel, Copyright © 2022 Christian Hergert
Description=Vala code intelligence provided by vala-language-server
Embedded=ide_lsp_plugin_register_types
Module=vala-language-server
Name=Vala Language Server
Website=https://github.com/vala-lang/vala-language-server
X-Category=lsps
X-Code-Action-Languages=vala
X-Completion-Provider-Languages=vala
X-Diagnostic-Provider-Languages-Priority=100
X-Diagnostic-Provider-Languages=vala
X-Formatter-Languages-Priority=100
X-Formatter-Languages=vala
X-Highlighter-Languages-Priority=100
X-Highlighter-Languages=vala
X-Hover-Provider-Languages=vala
X-LSP-Command=vala-language-server
X-LSP-Languages=vala;genie;
X-LSP-Settings=settings.json
X-Rename-Provider-Languages=vala
X-Symbol-Resolver-Languages-Priority=100
X-Symbol-Resolver-Languages=vala
```
### Kate
I've used Kate before, but haven't integrated a language server yet. It was just a matter of a few lines of JSON:
```json
{
  "servers": {
    "meson": {
      "command": [
        "Swift-MesonLSP",
        "--lsp"
      ],
      "rootIndicationFileNames": [
        "meson.build",
        "meson_options.txt"
      ],
      "url": "https://github.com/JCWasmx86/Swift-MesonLSP",
      "highlightingModeRegex": "^Meson$"
    }
  }
}
```
Even easier than in GNOME Builder. That's well done.

### Neovim
While testing a bit around, what editors support LSP, I found that editor. A few people
I know use it, so I decided to give it a try.
It was adding this JSON to `:CocConfig`:
```json
{
    "languageserver": {
        "meson": {
            "command": "Swift-MesonLSP",
	    "args": ["--lsp"],
	    "rootPatterns": ["meson.build"],
            "filetypes": ["meson"]
        }
    }
}
```
Quite easy.

### VSCode
I expected a low-code solution like in Kate or GNOME-Builder, as VSCode seems to be strongly related to the language server protocol.
Either I didn't find a way (I don't use VSCode and don't like it that much if I have to use it) or there is none.

I decided to fork the meson VSCode extension and just copied whatever the extension for the vala-language-server did. It did spawn,
but sadly I always got the error that a `Content-Length` header is expected. This caused me a lot of headaches, as my language server worked in Kate
and GNOME Builder. In the end, it was a misplaced log statement before I redirected all output that would have gone to stdout to stderr. (Neovim reuses
the implementation and didn't work until I fixed this)

Yes, it was just a bit of code to get it running, but it was still the most difficult and annoying integration I tested. To be fair, this is probably partially 
due to the ecosystem, as I try to avoid everything around NPM/JS/TS, unless necessary.

### Conclusion
Kate, and Neovim are trivial, GNOME-Builder requires a bit of setup, and VSCode is a lot to write. On a scale from 0-10, how easy it is (10=Trivial) from the POV
of somebody who never did that:
- Kate: 10
- Neovim: 10
- GNOME-Builder: 7-9 (Depending on the method used)
- VSCode: 5

## Wrap-up
During writing this language server I learned a lot about Swift and its memory management, and I experimented with different editors. I will try to use Swift
for more programs in the future (Except for GTK-based programs, there I will continue to use Vala). I think as my first big Swift project it turned out
to be quite usable. I've just written one small (2KLOC) project before to test the waters.

The most important thing I learned was how to use a Profiler efficiently, as that knowledge is universally applicable to every language.

I will continue my journey to make this program a high-performance, low memory-usage language server.
