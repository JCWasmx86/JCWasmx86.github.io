# Reflection - Swift and Meson language server

I did the first commit to Swift-MesonLSP on January 4th 2023. Except one small program I wrote in Swift, I've
never worked with it before. Now nearly 20 kLOC later, I want to reflect a bit about the usage of Swift for my
Meson language server.

## Reflection - Swift
### Why did I choose Swift
I outlined my choice a bit further [here](https://jcwasmx86.github.io/WritingALanguageServerForMeson), but the summary is basically this:

> Swift compiles to native code, an aspect I really like, and it is memory-safe unlike C. You have a huge ecosystem, and you can - if needed - easily bind C libraries.

### What was good?
The IDE setup was basically trivial. I just had to write a few plugins for GNOME Builder to set up `swift-format`, `swiftlint` and the Swift language server
`sourcekit-lsp`. (Shoutout to [Christian Hergert](https://gitlab.gnome.org/chergert) for making plugins so nice and easy to write!)
These plugins landed in upstream Builder later on, except the integration for sourcekit-lsp, as it was added independently.

The language itself is quite readable and powerful. The compiler generates quite fast code, despite the ARC. (It's around 15-18% of the runtime)

Furthermore, the ecosystem is quite evolved. I never had the feeling that I have to reimplement some basic stuff. Sure some things are quite weird design choices
in my opinion. Why is logging not in the standard library or Foundation? Same for cryptographic operations like hashing.

Another aspect I like are the small dependency graphs. In other languages (Looking at you JS and Rust), adding one new dependency can pull in a dozen of new dependencies,
I like it, as it makes auditing the code and the dependencies easier.

### What was bad?
The tooling on Linux was rather mixed. I had the lldb debugger (From the swift-lang package) crashing multiple times during printing a backtrace, when my language server crashed.
The normal lldb and gdb worked fine, albeit the name mangling makes things difficult to read.

The Windows support could be described as experimental at best. I have CI builds failing without any apparent reason, even in a debug build. I tried release builds before, but they
all stopped at some point. Just after a few weeks I was able to find the root cause: It's not enough RAM available in the GitHub Actions runners. I had a few very weird crashes I couldn't
track down, except forking a dependency and removing support for a feature. (It was just an ancient encoding, so no big loss)

Static linking `FoundationNetworking` is weird. I want to link everything statically, as the amount of Linux users having Swift installed is probably just a rounding mistake. This worked fine, until
I needed to download files for the wrap support. Statically linking `FoundationNetworking` requires you to statically link libcurl. First of all most distributions have no static libcurl and statically
linking it sounds like a huge mistake from a security perspective. What if there is a critical vulnerability? That's why I now rely on the fact that there is probably `curl` or `wget` installed. It works
on all major operating systems and is a good compromise for now.

The compilation time is quite long, as it is basically the case for every "modern" language that depends on LLVM. This increases the iteration times, which my usual workflow depends on to be low.

The Swift Package Manager is - to say it politely - disappointing. I wanted to embed a file into the binary. That's a task you probably need for a lot of programs. But there was no straightforward way.
One suggestion was to write a SPM-Plugin. Sadly the API is quite weird, and I simply used a worse solution by using a hackier solution (=Don't use a file, embed the information in a source file) Another aspect
that is missing is installation support. Why is there no `swift install` that copies the binary to `$PREFIX/bin` or some other specific location. Every other build system does this. Meson, CMake, cargo, go etc.

### What would I do different?
To be honest, I wouldn't use Swift anymore for this use case. I have a somewhat broken debugger, something that's a basic that should work. SPM that I'm required to use as the language server does not support anything else
is basically unusable for everything that does over "Here's the code, here are the tests, do compile them and then test it!". Cross-platform support is mediocre at best. macOS works perfectly, Linux has tooling
issues and Windows is basically broken in some aspects.

I think my choice for another language would be either Vala, as the glib-jsonrpc optimizations are quite cool, tooling works as expected, I have a great build system or Rust because its memory safety, its
(sadly somewhat drama-prone) community and the ecosystem.

Sure, Swift was fun, but I don't see myself writing another cross-platform CLI tool in it in a near future. Maybe doing web stuff with vapor? - Yes. Doing weird stuff with GTK and Swift? - Yes.
But anything else? - At the time of writing, no.

## Reflection - Meson language server
### Current status
My language server is completed from an editor POV. I implemented basically all features that are useful in a meson context.
- Diagnostics
- Auto-completion
- Renaming
- Codeactions
- Hover with documentation
- Go-to definition

Internally I've set up a benchmark suite that monitors memory usage for a few selected projects and the performance for a huge number of projects. (300 kLOC) These include
all GNOME projects, all ElementaryOS projects and a few other codebases.

I'm currently setting up automated LSP tests that emulate an editor. This will help to find more bugs and ensures there are no regressions.

### What did work fine?
I think my architecture is well-structured into several modules. I'm monitoring the performance of my application all the time. This means there are no unexpected performance regressions
in the releases. I've setup regression testing. For guessing the possible values in `subdir(x)`, where x is not a constant I've extracted real word samples and added them to my test suite.

Furthermore, I found my own development style for all the stuff around release management and feature sets. (Think of it as some very weird combination of waterfall and agile models)

### What didn't work fine?
Some parts of my architecture are simply badly designed. The `MesonDocs` module is basically the documentation embedded as `Map<string,string>`. I wanted to move the documentation
to a JSON file, embed it and load it at runtime, but sadly I wasn't able to do this due to SPM being weird.

The same for the listing of methods and functions. It's currently a 4.5kLOC file that contains all parameters and return types of each function/method. I would like to move it to an external
file and embed it into the binary, but it's the same issue as above.

Furthermore, many files are quite long. I have several files with more than a thousand LOC, e.g. the type analyzer or the file that glues everything together.

The last bad aspect is the lack of automatic language server testing, but I'm currently working on it.

### Outlook
My language server is complete from a feature POV. What does this mean? - In future there will be fewer releases (Think of once every two to three months, instead of several each month).
I will only fix bugs and update the docs and the function definitions for meson. But it does not mean the feature set is frozen, instead features will only be added on demand. But contributions,
as long as they follow certain standards will still be accepted.

