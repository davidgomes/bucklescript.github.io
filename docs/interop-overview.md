---
id: interop-overview
title: Overview
---

"Interop" is short for "interoperability". In the context of BuckleScript, this means communicating with JavaScript. One of BuckleScript's **major** goals is to interop as smoothly as possible with JS across the stack.

In a real-life, decently-sized codebase, one cannot simply convert from one technology to another within a few days, weeks, or sometimes months or years. For large codebases like Bloomberg's or Facebook's, sometimes the interop stays seemingly forever. If the interop isn't carefully designed enough to wiggle the language into a codebase, the conversion cost could dwarf the initial value of the language and become a deal breaker.

A well-crafted interop system that is simple, performant, local and non-prescriptive allows devs to use familiar concepts while trying out the technology, to be confident enough of the (lack of) unknowns to recommend the solution to managers, and to be productive while converting the team's project.

## Implementation

The above section might sound a bit buzzword-y; here's is how we've designed BuckleScript's interop, from the lowest part of the stack to the highest.

### Variables

Most variables' name compile to clean JS names. `hello` compiles to `hello`, whereas other compilers might turn it to e.g. `__$INTERNAL$hello$1`. Clean names (i.e. "no name mangling") help debugging and one-off experiments & bindings. We've seen emergency fixes where a team mate less familiar with BuckleScript and Reason comes into a file and drop a raw JS code block in the middle of a BuckleScript file. It's reassuring to know that last-resort escape hatches are possible.

### Data Structures

Major data structures in BuckleScript **map over cleanly to JS**. For example, a BS string is a JS string. A BS array is a JS array. The following BS code:

```ocaml
let messages = [| "hello"; "world"; "how"; "are"; "you" |]
```

Reason syntax:

```reason
let messages = [|"hello", "world", "how", "are", "you"|];
```

Roughly compiles to the JS code:

```js
var messages = ["hello", "world", "how", "are", "you"]
```

There is zero mental and performance overhead while using such value. Naturally, the value on the BuckleScript side is automatically typed to be an array of strings.

This behavior doesn't hold for _all_ the BS data structures; the dedicated sections for each offer more info.

### Functions

In most cases, you can directly call a JS function from BS, and vice-versa! These calls are free of performance overhead in most cases.

### Module/File

Every `let` declarations in a BS file is exported by default and usable from JS. For the other way around, you can declare in a BS file what JS module you want to use inside BS. We can both output and consume CommonJS, ES6 and AMD modules.

The generated output is clean enough that it could be passed as (slightly badly indented) hand-written JavaScript code. Try a few snippets in our playground!

<!-- TODO: playground link -->

### Build system

BS comes with a lightning-fast (fastest?) build system. It starts up in a few **milliseconds** and shuts down as fast. Incremental compilation is **two digits of milliseconds**. This allows the build system to be inserted invisibly into your whole JS build pipeline without embarrassing it. Unless your JS build pipeline is already embarrassingly slow. That's ok.

**1** BS file compiles to **1** JS file. The build can be configured to generate JS files alongside or outside your `ml`/`re` source files. This means you don't have to ask the infra team's help in trying out BuckleScript at the company; simply generate the JS files and check them in. From the perspective of the rest of the compilation pipeline, it's as if you've written these JS files by hand. This is how [Messenger successfully introduced Reason into the codebase](https://reasonml.github.io/community/blog/#messengercom-now-50-converted-to-reason).

### Package Management

We use NPM and Yarn. Since the generated output is clean enough, you can publish them at NPM `prepublishOnly` time and remove all trace of BuckleScript beforehand. The consumer wouldn't even know you wrote in BS, not JS! Check your `node_modules` right now; you might have been using some transitive BS code without knowing! time. The consumer wouldn't even know you wrote in BS, not JS.

## Conclusion

Hopefully you can see from the previous overview that we've poured _lots_ of thoughts into simplicity, familiarity and performance in the interop architecture. The next few sections describe each of these points in detail.
