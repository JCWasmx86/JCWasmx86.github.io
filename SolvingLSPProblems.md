# Writing a meson language server -- Addressing my issues
This will be a post for my [meson language server](https://github.com/JCWasmx86/Swift-MesonLSP) detailing which problems I encountered and how I solved them. And I will outline the future of Swift-MesonLSP.

## Problems
### Frequent reparses
In the first few months, I simply rebuilt the entire tree (=Parsing every meson source file in a project) on every `textDocument/didChange`. If you press 5 keys a second, I will rebuild the tree 5 times. (Unless your editor
combines them client-side, but GNOME Builder doesn't do that)

So I simply used [`Task`s](https://developer.apple.com/documentation/swift/task). These are simple a "unit of asynchronous work". So each time I get notified by the editor the file has changed, I will cancel the old task and start a new one.
You just have to sprinkle a lot of `if Task.isCancelled {}` over the code in order to ensure the time spent on doing unnecessary computations is minimal. Examples for good points to do this check are:
- After parsing the meson source files
- After analyzing the AST
- After parsing each subproject
- After clearing/sending diagnostics

You still have the problem, that a lot of unnecessary work is done, and you can drive up the CPU usage by just typing really fast. So what's the best way to avoid that? Simply collapse "typing sequences" into just one tree-rebuild.
The code for the task is now like this:
```swift
do {
  // 100 ms
  try await Task.sleep(nanoseconds: 100 * 1_000_000)
} catch {
  // Sleep was cancelled
  return
}
// Rebuild the tree now.
```
My experiments have shown that this reduces the number of rebuilds by a lot and thus reduces CPU Usage.
### Calculating non-constant values to `subdir(x)` and `(set|get)_variable(x)`
The fact that meson allows passing strings that are not literals to `subdir`, `set_variable` and `get_variable` made my life a lot harder. But if my language server can't handle that, it would be simply useless for a lot of projects.
Especially for a lot of projects in the GNOME space, I consider my "main" priority with everything. So I implemented a really weird interpreter. While it cannot derive `x` for every meson project, it does it for probably more than 99% of all the meson projects.

We have this code (Copied+Shortened from GLib):
```meson
foreach f : functions
  if cc.has_function(f)
    set_variable('have_func_' + f, true)
  else
    set_variable('have_func_' + f, false)
  endif
endforeach
```
My code is seeing this expression in the first argument of the `set_variable` call: `'have_func_' + f`. While for the left-hand side of the binary expression, we have to do nothing, we have to find out, what `f` is. Let's derive `f`.
We walk up, from the current line, until we encounter a definition of `f`. It's a value from the iteration over `functions`. Now let's search for functions. The fancy part is not finding the definition of `functions`, but
all possible values. `functions` is an array of function names:
```
functions = [
  'accept4',
  'close_range',
  'copy_file_range',
  # truncated
]
```
So far, so good. We have our strings. Now we build all possible combinations of our binary expression:
```
'have_func_' + 'accept4' => 'have_func_accept4'
'have_func_' + 'close_range' => 'have_func_close_range'
'have_func_' + 'copy_file_range' => 'have_func_copy_file_range'
```
Why do we still get errors? - Because of code like this:
```meson
if glib_conf.has('HAVE_SYS_STATVFS_H')
  functions += ['statvfs']
endif
```
So we need to add all other values that could be possibly in this list to our `functions` list. And now we can finally derive all our values in this loop.

This interpreter is one of the things I'm most proud of. It supports even more complex cases like this one from Harfbuzz:
```meson
tests = [
    'hb-shape-fuzzer.cc',
    'hb-subset-fuzzer.cc',
    'hb-set-fuzzer.cc',
    'hb-draw-fuzzer.cc',
]
if get_option('experimental_api')
    tests += 'hb-repacker-fuzzer.cc'
endif
foreach file_name : tests
    test_name = file_name.split('.')[0]
    set_variable('@0@_exe'.format(test_name.underscorify()), true)
endforeach
```
I've implemented a limited set of methods and functions and expressions I can evaluate. This allows me to trace *a lot* of variable transformations. (As long as they are in the same source file, as I will never walk up to the parent source file due to potential performance issues)

One disadvantage of this method is that I "pollute" the list of existing variables with variables that may never exist on some code-paths. But this is probably unavoidable.

### Creating releases
In the beginning, I just compiled a binary on Ubuntu (As it is probably universal if you ignore stuff like Alpine Linux and nixOS) and macOS and attached it to the latest release automatically. I ran my benchmarks, created a Windows binary in my Windows VM (As the GitHub Actions Runner is too weak for a release build) and there was my release. Invested time: Probably 10-20 minutes.

But then for some reason I added more supported Linux packaging/distributions: 3 Debian versions, 3 Ubuntu versions, Arch Linux AUR, Fedora COPR. My time investment into creating releases increased drastically to about 60 minutes.

I had a lot of free time for a week and decided to make it nearly automatic. [Here](https://github.com/JCWasmx86/Swift-MesonLSP/blob/main/.github/workflows/ReleaseUpload.yml) is my result. What does it do?

- Creates Ubuntu/macOS/Windows binaries, attaches some metadata for license stuff, zips them up and attaches them to the latest release
- Updates the AUR package
- Updates the COPR package
- Creates .deb files for Ubuntu/Debian and updates my [apt repo](https://github.com/JCWasmx86/swift-mesonlsp-apt-repo).

So the amount of work I have to do is basically zero now. But you may ask: "You said Windows release builds don't work on GitHub Actions?" The magic are pagefiles.
```yaml
      - name: Configure pagefile
        uses: al-cheb/configure-pagefile-action@v1.3
        with:
          minimum-size: 3GB
          maximum-size: 8GB
          disk-root: "D:"
```
Just add this snippet somewhere in your workflow and, you have a bit more "RAM". Took a long time to find this action, previous attempts using Powershell weren't so successful. (For some reason the build sometimes still fails without any indication what went wrong, but that's outside my debugging capabilities)

A few weeks later, I refactored the workflows a lot more to decrease the time needed by introducing a bit more parallelism. Sure, it's probably a bit over-engineered, but where would be the fun?

The only bad aspect of this is the vendor lock-in to GitHub Actions, but if needed, I can just go back to doing some stuff manually again.

## Future of Swift-MesonLSP
### More features
I thought my language server was somewhat feature-complete, but after the inclusion in vscode-meson I got a lot of feature requests and new ideas for features:
- Semantic tokens (Shoutout to tristan957 for helping with that one) got introduced for e.g. highlighting the variables/placeholders in format strings.
- A few more configuration options, like allowing to disable linting of variable names and some other special cases.
- Folding ranges got implemented as it was easy and basically an effort of a few minutes.

There are still a few things left:

- Code intelligence for wrap files (Probably implemented in December 2023)
- Code intelligence for `meson_options.txt`/`meson.options` files (Will be implemented in November 2023)

### Switching the library used for the language server
I started using the infrastructure from sourcekit-lsp, as it was basically the only maintained and easily usable implementation of the types/protocol. I could assume it was battle-hardened, as it was used by the official language server for Swift. It worked fine until the middle of October 2023. After that, the internal API was refactored heavily twice. I was able to port my language server after the first refactoring, but the second refactoring increased my porting effort so much, I gave up. Yes, I knew it was an internal API, so it's purely my fault I ended up in this position, but it was the best decision to use it when I began writing Swift-MesonLSP 

So I was searching for other libraries I could use. I had a lot of luck: I found [LanguageServerProtocol from ChimeHQ](https://github.com/ChimeHQ/LanguageServerProtocol) I already use the SwiftTreeSitter library from this organization, so basically my search was over. But I'm currently waiting for an open PR to be merged that improves the API a bit.

I hope to switch to using ChimeHQ's LanguageServerProtocol in December 2023. There are many advantages:

- API stability as I won't use an internal API anymore
- Ease of contributing due to being a smaller project than sourcekit-lsp.
- Reduced binary size: Pulling in sourcekit-lsp means pulling in a lot of other platform libraries from the Swift project. While I can't guess the impact in numbers, I'm sure it will reduce the binary size. That will improve the compilation and linking times.

### Rewriting in Rust (Maybe)
I had a lot of problems with Swift: Huge compile times, a somewhat dead ecosystem/community outside of Apple and Swift on Server and huge binary sizes due to statically linking Foundation (Or shipping it as DLL)

Rust is the best choice here as opposed to other languages:

- Python: I won't write a huge program like my language server in a language without compiler (Swift-MesonLSP has 18kLOC at the time of writing) Yes, mypy exists, and it used successfully by many, but I don't like that you can mess up so much at runtime. (Except for Django, as it's the only web framework I can grasp)
- C: Nope. I'm doing a lot of work with strings, and using C for that is just a vulnerability in the making. Imagine opening a malicious meson.build that exploits my language server. That's not a risk I'm willing to take. Furthermore, it would be a lot more code - probably somewhere around 100-120kLOC
- Go: Solid choice, both Rust and Golang are equally good for writing language servers, but I went with Rust.

Personally, I have a few fears with Rust:

- Borrow checker: Currently I'm doing a lot of modifying of e.g. the AST and my AST has "cycles" (Each node has a reference to the parent and to their children). If I want to rewrite it, it would be a 1-1 rewrite.
- Async stuff: Every time I worked with something async in the last few months, something went wrong, or it was a lot more difficult than e.g. just using threads.
- Introducing new, exciting bugs. I already invested a lot of time into the Swift version, fixing bugs, adding features. A rewrite will always bring in new bugs. (And that same argument is the reason why there are still companies running stuff on Windows 95 or Java 6)

I won't drop everything and start rewriting Swift-MesonLSP in Rust now, (Would that be Rust-MesonLSP then?🤔) but in the coming years, it may be possible. (Earliest moment possible would be October 2024)
