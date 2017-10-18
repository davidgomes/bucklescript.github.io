\#\# BuckleScript annotations for Unicode and JS FFI support

To make OCaml work smoothly with JavaScript, we introduced several
extensions to the OCaml language. These BuckleScript extensions
facilitate the integration of native JavaScript code and improve the
generated code.

\# Unicode support (@since 1.5.1)

> **Warning**
>
> In this section, we assume the source is encoded using UTF8.

In OCaml, string is an immutable byte sequence (like GoLang), so if the
user types some Unicode

    Js.log "你好"

It will be translated into

    console.log("\xe4\xbd\xa0\xe5\xa5\xbd");

Luckily, OCaml allows customized multiple-line string support,
BuckleScript **reserves** the delimiter `js` and `j` (`j` is unused so
far).

**Input:.**

    Js.log {js|你好，
    世界|js}

**Output:.**

    console.log("你好，\n世界");

Inside the `js` delimiter, the escape convention is like JavaScript

    Js.log {js|\x3f\u003f\b\t\n\v\f\r\0"'|js}

**Output:.**

    console.log("\x3f\u003f\b\t\n\v\f\r\0\"\'");

\#\# Unicode support with string interpolation (@since 1.7.0)

Like `{js||js}`, `{j||j}` not only allow unicode point, but also
variable interpolation.

For example

    let world = {j|世界|j}
    let hello_world = {j|你好，$world|j}

Users can parenthesize the interpreted variable as below:

    let hello_world = {j|你好，$(world)|j}

Note the syntax of interpolated variable is intentionally designed to be
simple. Its lexical convention is as below:

    identifier := leading_identifier_char identifier_chars
    leading_identifier_char := 'a' .. 'z' | '_'
    identifier_chars := 'a' .. 'z' | 'A' .. 'Z' | '0' .. '9' | '_' | '\'

\# FFI

Like TypeScript, when building type-safe bindings from JS to OCaml,
users have to write type declarations. In OCaml, unlike TypeScript,
users do not need to create a separate `.d.ts` file, since the type
declarations are an integral part of OCaml.

The FFI is divided into several components:

-   Binding to simple functions and values

-   Binding to high-order functions

-   Binding to object literals

-   Binding to classes

-   Extensions to the language for debugger, regex, and embedding
    arbitrary JS code

\# Binding to simple JS functions values

This part is similar to [traditional
FFI](http://caml.inria.fr/pub/docs/manual-ocaml-4.02/intfc.html), with
syntax as described below:

    external value-name : typexpr = external-declaration attributes
    external-declaration := string-literal

Users need to declare types for foreign functions (JS functions for
BuckleScript or C functions for native compiler) and provide customized
`attributes`.

\#\# Binding to global value: `bs.val`

    external imul : int -> int -> int = "Math.imul" [@@bs.val]
    type dom
    (* Abstract type for the DOM *)
    external dom : dom = "document" [@@bs.val]

`bs.val` attribute is used to bind to a JavaScript value, it can be a
function or plain value.

> **Note**
>
> -   If `external-declaration` is the same as `value-name`, the user
>     can leave `external-declaration` empty. For example:
>
>         external document : dom = "" [@@bs.val]
>
> -   If you want to make a single FFI for both C functions and
>     JavaScript functions, you can give the JavaScript foreign function
>     a different name:
>
>         external imul : int -> int -> int =
>           "c_imul" [@@bs.val "Math.imul"]
>
\#\# Scoped values: `bs.scope` (@since 1.7.2)

In JS library, it is quite common to use a name as namespace, for
example, if the user want to write a binding to
`vscode.commands.executeCommand`, assume `vscode` is a module name, the
user needs to type `commands` properly before typing `executeCommand`,
and in practice, it is rarely useful to call `vscode.commands` alone,
for this reason, we introduce a convient sugar: `bs.scope`

**Example.**

    type param
    external executeCommands : string -> param array -> unit = ""
    [@@bs.scope "commands"] [@@bs.module "vscode"][@@bs.splice]

    let f a b c  =
      executeCommands "hi"  [|a;b;c|]

**Output:.**

    var Vscode = require("vscode");
    function f(a, b, c) {
      Vscode.commands.executeCommands("hi", a, b, c);
      return /* () */0;
    }

NOTE `bs.scope` can also be chained as below:

**Example.**

    external makeBuffer : int -> buffer = "Buffer"
    [@@bs.new] [@@bs.scope "global"]
    external hi : string = ""
    [@@bs.module "z"] [@@bs.scope "a0", "a1", "a2"]
    external ho : string = ""
    [@@bs.val] [@@bs.scope "a0","a1","a2"]
    external imul : int -> int -> int = ""
    [@@bs.val] [@@bs.scope "Math"]
    let f2 ()  =
      makeBuffer 20 , hi , ho, imul 1 2

**Output:.**

    var Z      = require("z");
    function f2() {
      return /* tuple */[
              new (global.Buffer)(20),
              Z.a0.a1.a2.hi,
              a0.a1.a2.ho,
              Math.imul(1, 2)
            ];
    }

\#\# Binding to JavaScript constructor: `bs.new`

`bs.new` is used to create a JavaScript object.

    type t
    external create_date : unit -> t = "Date" [@@bs.new]
    let date = create_date ()

**Output:.**

    var date = new Date();

\#\# Binding to a value from a module: `bs.module`

**Input:.**

    external add : int -> int -> int = "add" [@@bs.module "x"]
    external add2 : int -> int -> int = "add2"[@@bs.module "y", "U"] // 
    let f = add 3 4
    let g = add2 3 4

-   `"U"` will hint the compiler to generate a better name for the
    module, see output

**Output:.**

    var U = require("y");
    var X = require("x");
    var f = X.add(3, 4);
    var g = U.add2(3, 4);

> **Note**
>
> -   if `external-declaration` is the same as `value-name`, it can be
>     left empty, for example,
>
>         external add : int -> int -> int = "" [@@bs.module "x"]
>
\#\# Binding the whole module as a value or function

    type http
    external http : http = "http" [@@bs.module] // 

-   `external-declaration` is the module name

> **Note**
>
> -   if `external-declaration` is the same as `value-name`, it can be
>     left empty, for example:
>
>         external http : http = "" [@@bs.module]
>
\#\# Binding to method: `bs.send`, `bs.send.pipe`

`bs.send` helps the user send a message to a JS object.

    type id (** Abstract type for id object *)
    external get_by_id : dom -> string -> id =
      "getElementById" [@@bs.send]

The object is always the first argument and actual arguments follow.

**Input:.**

    get_by_id dom "xx"

**Output:.**

    document.getElementById("xx");

`bs.send.pipe` is similar to `bs.send` except that the first argument,
i.e, the object, is put in the position of last argument to help user
write in a *chaining style*:

    external map : ('a -> 'b [@bs]) -> 'b array =
      "" [@@bs.send.pipe: 'a array] // 
    external forEach: ('a -> unit [@bs]) -> 'a array =
      "" [@@bs.send.pipe: 'a array]
    let test arr =
        arr
        |> map (fun [@bs] x -> x + 1)
        |> forEach (fun [@bs] x -> Js.log x)

-   For the `[@bs]` attribute in the callback, see
    [???](#Binding to callbacks (high-order function))

> **Note**
>
> -   if `external-declaration` is the same as `value-name`, it can be
>     left empty, for example:
>
>         external getElementById : dom -> string -> id =
>           "" [@@bs.send]
>
\#\# Binding to dynamic key access/set: `bs.set_index`, `bs.get_index`

This attribute allows dynamic access to a JavaScript property

**Input:.**

    type t
    external create : int -> t = "Int32Array" [@@bs.new]
    external get : t -> int -> int = "" [@@bs.get_index]
    external set : t -> int -> int -> unit = "" [@@bs.set_index]

    let _ =
      let i32arr = (create 3) in
      set i32arr 0 42;
      Js.log (get i32arr 0)

**Output:.**

    var i32arr = new Int32Array(3);
    i32arr[0] = 42;
    console.log(i32arr[0]);

\#\# Binding to Getter/Setter: `bs.get`, `bs.set`

This attribute helps get and set the property of a JavaScript object.

    type textarea
    external set_name : textarea -> string -> unit = "name" [@@bs.set]
    external get_name : textarea -> string = "name" [@@bs.get]

\# Splice calling convention: `bs.splice`

In JS, it is quite common to have a function take variadic arguments.
BuckleScript supports typing homogeneous variadic arguments. For
example,

    external join : string array -> string = "" [@@bs.module "path"] [@@bs.splice]
    let v = join [| "a"; "b"|]

**Output:.**

    var Path = require("path");
    var v = Path.join("a","b");

> **Note**
>
> For the external call, if the `array` arguments is not a compile time
> array, the compiler will emit an error message.

\# Special types on external declarations: `bs.string`, `bs.int`,
`bs.ignore`, `bs.as`, `bs.unwrap`

\#\# Using polymorphic variant to model enums and string types There are
several patterns heavily used in existing JavaScript codebases, for
example, the string type is used a lot. BuckleScript FFI allows the user
to model string type in a safe way by using annotated polymorphic
variant.

    external readFileSync :
      name:string ->
      ([ `utf8
       | `my_name [@bs.as "ascii"] // 
       ] [@bs.string]) ->
      string = ""
      [@@bs.module "fs"]

    let _ =
      readFileSync ~name:"xx.txt" `my_name

-   Here we intentionally made an example to show how to customize a
    name

**Output:.**

    var Fs = require("fs");
    Fs.readFileSync("xx.txt", "ascii");

Polymorphic variants can also be used to model *enums*.

**Input:.**

    external test_int_type :
      ([ `on_closed
       | `on_open [@bs.as 3] // 
       | `in_bin // 
       ]
       [@bs.int]) -> int =
      "" [@@bs.val]

    let _ =
      test_int_type `in_bin

-   *\`on\_closed* will be encoded as 0

-   *`on_open`* will be 3 due to the attribute \`bs.as&lt;/literal&gt;

-   *\`in\_bin* will be 4

**Output:.**

    test_int_type(4);

\#\# Using polymorphic variant to model event listener

BuckleScript models this in a type-safe way by using annotated
polymorphic variants.

    type readline
    external on :
        (
        [ `close of unit -> unit
        | `line of string -> unit
        ] // 
        [@bs.string])
        -> readline = "" [@@bs.send.pipe: readline]
    let register rl =
      rl
      |> on (`close (fun event -> () ))
      |> on (`line (fun line -> print_endline line))

-   This is a very powerful typing: each event can have its own
    *different types*.

**Output:.**

    function register(rl) {
      return rl.on("close", function () {
                    return /* () */0;
                  })
               .on("line", function (line) {
                  console.log(line);
                  return /* () */0;
                });
    }

\#\# Using polymorphic variant to model arguments of multiple possible
types (@since 1.8.3)

Sometimes a JavaScript function will accept an argument that could have
different types depending upon how it’s used.

    function padLeft(string, padding) {
      if (typeof padding === "number") {
        return Array(padding + 1).join(" ") + value;
      }
      if (typeof padding === "string") {
        return padding + value;
      }
      throw new Error(`Expected string or number, got '${padding}'.`);
    }

You can model such a function in BuckleScript using `[@bs.unwrap]`.

    external padLeft :
      string
      -> ([ `String of string
          | `Int of int
          ] [@bs.unwrap])
      -> string
      = "" [@@bs.val]

Polymorphic variants with `[@bs.unwrap]` will "unwrap" the variant at
the call site so that the JavaScript function is called with the
underlying value.

    let _ = padLeft "Hello World" (`Int 4)
    let _ = padLeft "Hello World" (`String "bs: ")

Output:

    padLeft("Hello World", 4);
    padLeft("Hello World", "bs: ");

> **Warning**
>
> -   These `[@bs.string]`, `[@bs.int]`, and `[@bs.unwrap]` annotations
>     will only have effect in `external` declarations.
>
> -   The runtime encoding of using polymorphic variant is internal to
>     the compiler.
>
> -   With these annotations mentioned above, BuckleScript will
>     automatically transform the internal encoding to the designated
>     encoding for FFI. BuckleScript will try to do such conversion at
>     compile time if it can, otherwise, it will do such conversion in
>     the runtime, but it should be always correct.
>
\#\# Phantom Arguments and ad-hoc polymorphism

`bs.ignore` allows arguments to be erased after passing to JS functional
call, the side effect will still be recorded.

For example,

    external add : (int [@bs.ignore]) -> int -> int -> int = ""
    [@@bs.val]
    let v = add 0 1 2 // 

-   the first argument will be erased

**Output:.**

    var v = add (1,2);

This is very useful to combine GADT:

**Input.**

    type _ kind =
      | Float : float kind
      | String : string kind
    external add : ('a kind [@bs.ignore]) -> 'a -> 'a -> 'a = "" [@@bs.val]

    let () =
      Js.log (add Float 3.0 2.0);
      Js.log (add String "x" "y");

**Output.**

    console.log(add(3.0, 2.0));
    console.log(add("x", "y"));

User can also have a payload for the GADT:

    let string_of_kind (type t) (kind : t kind) =
      match kind with
      | Float -> "float"
      | String -> "string"

    external add_dyn : ('a kind [@bs.ignore]) -> string -> 'a -> 'a -> 'a = ""
    [@@bs.val]

    let add2 k x y =
      add_dyn k (string_of_kind k) x y

> **Note**
>
> Using a GADT as a `[@bs.ignore]` argument as described to achieve a
> single polymorphic argument can now also be accomplished with
> `[@bs.unwrap]` above. The GADT + `[@bs.ignore]` approach is slightly
> more flexible and is always guaranteed to have no runtime overhead,
> where `[@bs.unwrap]` could incur slight runtime overhead in some cases
> but could be a more intuitive API for end users.

\#\# Fixed Arguments

Contrary to the Phantom arguments, `_ [@bs.as]` is introduced to attach
constant data.

For example:

    external process_on_exit : (_ [@bs.as "exit"]) -> (int -> unit) -> unit =
      "process.on" [@@bs.val]

    let () =
        process_on_exit (fun exit_code ->
            Js.log( "error code: " ^ string_of_int exit_code ))

**Output:.**

    process.on("exit", function (exit_code) {
          console.log("error code: " + exit_code);
          return /* () */0;
        });

It can also be used in combination with other attributes, for example:

    type process

    external on_exit : (_ [@bs.as "exit"]) -> (int -> unit) -> process =
        "on" [@@bs.send.pipe: process]
    let register (p : process) =
            p |> on_exit (fun i -> Js.log i)

**Output:.**

    function register(p) {
      return p.on("exit", (function (i) {
                    console.log(i);
                    return /* () */0;
                  }));
    }

**Input:.**

    external io_config :
        stdio:(_ [@bs.as "inherit"]) -> cwd:string -> unit -> _ = "" [@@bs.obj]

    let config = io_config ~cwd:"." ()

**Output:.**

    var config = {
      stdio: "inherit",
      cwd: "."
    };

\#\# Fixed Arguments with arbitrary JSON literal (@since 1.7.0)

So the payload can be more flexiblie with JSON literal support

    type t
    external x: t = "" [@@bs.val]

    external on_exit_slice5 :
        int
        -> (_ [@bs.as 3])
        -> (_ [@bs.as {json|true|json}])
        -> (_ [@bs.as {json|false|json}])
        -> (_ [@bs.as {json|"你好"|json}])
        -> (_ [@bs.as {json| ["你好",1,2,3] |json}])
        -> (_ [@bs.as {json| [{ "arr" : ["你好",1,2,3], "encoding" : "utf8"}] |json}])
        -> (_ [@bs.as {json| [{ "arr" : ["你好",1,2,3], "encoding" : "utf8"}] |json}])
        -> (_ [@bs.as "xxx"])
        -> ([`a|`b|`c] [@bs.int])
        -> (_ [@bs.as "yyy"])
        -> ([`a|`b|`c] [@bs.string])
        -> int array
        -> unit
        =
        "xx" [@@bs.send.pipe: t] [@@bs.splice]

    let _ = x |> on_exit_slice5 __LINE__ `a `b [|1;2;3;4;5|]

**Output:.**

    x.xx(114, 3, true, false, ("你好"), ( ["你好",1,2,3] ), ( [{ "arr" : ["你好",1,2,3], "encoding" : "utf8"}] ), ( [{ "arr" : ["你好",1,2,3], "encoding" : "utf8"}] ), "xxx", 0, "yyy", "b", 1, 2, 3, 4, 5)

\# Binding to NodeJS special variables: [bs.node](../api/Node.html)

NodeJS has several file local variables: `__dirname`, `__filename`,
`_module`, and `require`. Their semantics are more like macros instead
of functions.

BuckleScript provides built-in macro support for these variables:

    let dirname : string option = [%bs.node __dirname]
    let filename : string option = [%bs.node __filename]
    let _module : Node.node_module option = [%bs.node _module]
    let require : Node.node_require option = [%bs.node require]

\# Binding to callbacks (high-order function)

High order functions are functions where the callback can be another
function. For example, suppose JS has a map function as below:

    function map (a, b, f){
      var i = Math.min(a.length, b.length);
      var c = new Array(i);
      for(var j = 0; j < i; ++j){
        c[j] = f(a[i],b[i])
      }
      return c ;
    }

A **naive** external type declaration would be as below:

    external map : 'a array -> 'b array -> ('a -> 'b -> 'c) -> 'c array = "" [@@bs.val]

Unfortunately, this is not completely correct. The issue is by reading
the type `'a -> 'b -> 'c`, it can be in several cases:

    let f x y = x + y

    let g x = let z = x + 1 in fun y -> x + z

In OCaml they all have the same type; however, `f` and `g` may be
compiled into functions with different arities.

A naive compilation will compile `f` as below:

    let f = fun x -> fun y -> x + y

    function f(x){
      return function (y){
        return x + y;
      }
    }
    function g(x){
      var z = x + 1 ;
      return function (y){
        return x + z ;
      }
    }

Its arity will be *consistent* but is *1* (returning another function);
however, we expect *its arity to be 2*.

Bucklescript uses a more complex compilation strategy, compiling `f` as

    function f(x,y){
      return x + y ;
    }

No matter which strategy we use, existing typing rules **cannot
guarantee a function of type `'a -> 'b -> 'c` will have arity 2.**

\#\# `[@bs]` for explicit uncurried callback

To solve this problem introduced by OCaml’s curried calling convention,
we support a special attribute `[@bs]` at the type level.

    external map : 'a array -> 'b array -> ('a -> 'b -> 'c [@bs]) -> 'c array
    = "map" [@@bs.val]

Here `('a -> 'b -> 'c [@bs])` will *always be of arity 2*, in general,
`'a0 -> 'a1 ... 'aN -> 'b0 [@bs]` is the same as
`'a0 -> 'a1 ... 'aN -> 'b0` except the former’s arity is guaranteed to
be `N` while the latter is unknown.

To produce a function of type `'a0 -> .. 'aN -> 'b0 [@bs]`, as follows:

    let f : 'a0 -> 'a1 -> .. 'b0 [@bs] =
      fun [@bs] a0 a1 .. aN -> b0
    let b : 'b0 = f a0 a1 a2 .. aN [@bs]

A special case for arity of 0:

    let f : unit -> 'b0 [@bs] = fun [@bs] () -> b0
    let b : 'b0 = f () [@bs]

Note that this extension to the OCaml language is *sound*. If you add an
attribute in one place but miss it in other place, the type checker will
complain.

Another more complex example:

    type 'a return = int -> 'a [@bs]
    type 'a u0 = int -> string -> 'a return [@bs] // 
    type 'a u1 = int -> string -> int -> 'a [@bs] // 
    type 'a u2 = int -> string -> (int -> 'a [@bs]) [@bs] // 

-   `u0` has arity of 2, return a function with arity 1

-   `u1` has arity of 3

-   `u2` has arity of 2, return a function with arity 1

\#\# `[@bs.uncurry]` for implicit uncurried callback (@since 1.5.0)

Note the `[@bs]` annotation already solved the problem completely, but
it has a drawback that it requires users to write `[@bs]` both in
definition site and call site.

For example:

    external map : 'a array -> ('a -> 'b[@bs]) -> 'b array = "" [@@bs.send] // 
    map [|1;2;3|] (fun [@bs] x -> x + 1) // 

-   `[@bs]` annotation in definition site

-   `[@bs]` annotation in call site

This is less convenient for end users, so we introduce another implicit
annotation `[@bs.uncurry]` so that the compiler will automatically wrap
the curried callback (from OCaml side) to JS uncurried callback. In this
way, the `[@bs.uncurry]` annotation is defined only once.

    external map : 'a array -> ('a -> 'b [@bs.uncurry]) -> 'b array = "" [@@bs.send] // 
    map [|1;2;3|] (fun x -> x+ 1) // 

-   `[@bs.uncurry]` annotation in definition site

-   Idiomatic OCaml code

> **Note**
>
> In general, `bs.uncurry` is recommended, and compiler will do lots of
> optimizations to resolve the `curry` to `uncurry` calling convention
> at compile time. However, there are some cases the compiler optimizer
> could not do it, in that case, it will be converted runtime.
>
> This means `[@bs]` are completely static behavior (no any runtime
> cost), while `[@bs.uncurry]` is more convenient for end users but in
> some very rare cases it might be slower than `[@bs]`

\#\# Uncurried calling convention as an optimization

**Background:.**

As we discussed before, we can compile any OCaml function as arity 1 to
support OCaml’s curried calling convention.

This model is simple and easy to implement, but the native compilation
is very slow and expensive for all functions.

    let f x y z = x + y + z
    let a = f 1 2 3
    let b = f 1 2

can be compiled as

    function f(x){
      return function (y){
        return function (z){
          return x + y + z
        }
      }
    }
    var a = f (1) (2) (3)
    var b = f (1) (2)

But as you can see, this is *highly inefficient*, since the compiler
already *saw the source definition* of `f`, it can be optimized as
below:

    function f(x,y,z) {return x + y + z}
    var a = f(1,2,3)
    var b = function(z){return f(1,2,z)}

BuckleScript does this optimization in the cross module level and tries
to infer the arity as much as it can.

\# Callback optimization

However, such optimization will not work with *high-order* functions,
i.e, callbacks.

For example,

    let app f x = f x

Since the arity of `f` is unknown, the compiler can not do any
optimization (unless `app` gets inlined), so we have to generate code as
below:

    function app(f,x){
      return Curry._1(f,x);
    }

`Curry._1` is a function to dynamically support the curried calling
convention.

Since we support the uncurried calling convention, you can write `app`
as below

    let app f x = f x [@bs]

Now the type system will infer `app` as type `('a ->'b [@bs]) -> 'a` and
compile `app` as

    function app(f,x){
      return f(x)
    }

> **Note**
>
> In OCaml the compiler internally uncurries every function declared as
> `external` and guarantees that it is always fully applied. Therefore,
> for `external` first-order FFI, its outermost function does not need
> the `[@bs]` annotation.

\#\# Bindings to `this` based callbacks: `bs.this`

Many JS libraries have callbacks which rely on `this` (the source), for
example:

    x.onload = function(v){
      console.log(this.response + v )
    }

Here, `this` would be the same as `x` (actually depends on how `onload`
is called). It is clear that it is not correct to declare `x.onload` of
type `unit -> unit [@bs]`. Instead, we introduced a special attribute
`bs.this` allowing us to type `x` as below:

    type x
    external x: x = "" [@@bs.val]
    external set_onload : x -> (x -> int -> unit [@bs.this]) -> unit = "onload" [@@bs.set]
    external resp : x -> int = "response" [@@bs.get]
    let _ =
      set_onload x begin fun [@bs.this] o v ->
        Js.log(resp o + v )
      end

**Output:.**

    x.onload = (function (v) {
        var o = this ; // 
        console.log(o.response + v | 0);
        return /* () */0;
      });

-   The first argument is automatically bound to `this`

`bs.this` is the same as `bs` : except that its first parameter is
reserved for `this` and for arity of 0, there is no need for a redundant
`unit` type:

    let f : 'obj -> 'b [@bs.this] =
      fun [@bs.this] obj -> ....
    let f1 : 'obj -> 'a0 -> 'b [@bs.this] =
      fun [@bs.this] obj a -> ...

> **Note**
>
> There is no way to consume a function of type
> `'obj -> 'a0 .. -> 'aN -> 'b0 [@bs.this]` on the OCaml side. This is
> an intentional design choice, we **don’t encourage** people to write
> code in this style.
>
> This was introduced mainly to be consumed by existing JS libraries.
> User can also type `x` as a JS class too (see later)

\# Binding to JS objects

**Convention:.**

All JS objects of type `'a` are lifted to type `'a Js.t` to avoid
conflict with OCaml’s native object system (we support both OCaml’s
native object system and FFI to JS’s objects), `\##` is used in JS’s
object method dispatch and field access, while `#` is used in OCaml’s
object method dispatch.

**Typing JavaScript objects:.**

OCaml supports object oriented style natively and provides structural
type system. OCaml’s object system has different runtime semantics from
JS object, but they share the same type system, all JS objects of type
`'a` are typed as `'a Js.t`

OCaml provides two kinds of syntaxes to model structural typing:
`< p1 : t1 >` style and `class type` style. They are mostly the same
except that the latter is more feature rich (supporting inheritance) but
more verbose.

\#\# Simple object type

Suppose we have a JS file `demo.js` which exports two properties:
`height` and `width`:

**demo.js.**

    exports.height = 3
    exports.width  = 3

There are different ways to writing binding to module `demo`, here we
use OCaml objects to model module `demo`

    external demo : < height : int ; width : int > Js.t = "" [@@bs.module]

There are two kinds of types on the method name:

-   normal type

        < label : int >
        < label : int -> int >
        < label : int -> int [@bs]>
        < label : int -> int [@bs.this]>

-   method

        < label : int -> int [@bs.meth] >

The difference is that for `method`, the type system will force users to
fulfill its arguments all at the same time, since its semantics depends
on `this` in JavaScript.

For example:

    let test f =
      f##hi 1 // 
    let test2 f =
      let u = f##hi in
      u 1
    let test3 f =
      let u = f##hi in
      u 1 [@bs]

-   `##` is JS object property/method dispatch

The compiler would infer types differently

    val test : < hi : int -> 'a [@bs.meth]; .. > -> 'a // 
    val test2 : < hi : int -> 'a ; .. > -> 'a
    val test3 : < hi : int -> 'a [@bs]; .. >

-   `..` is a row variable, which means the object can contain more
    methods.

\#\# Complex object type

Below is an example:

    class type _rect = object
      method height : int
      method width : int
      method draw : unit -> unit
    end [@bs] // 
    type rect = _rect Js.t

-   `class type` annotated with `[@bs]` is treated as a JS class type,
    it needs to be lifted to `Js.t` too.

For JS classes, methods with arrow types are treated as real methods
(automatically annotated with `[@bs.meth]`) while methods with non-arrow
types are treated as properties.

So the type `rect` is the same as below:

    type rect = < height : int ; width : int ; draw : unit -> unit [@bs.meth] > Js.t

\#\# How to consume JS property and methods

As we said: `##` is used in both object method dispatch and field
access.

    f##property // 
    f##property #= v
    f##js_method args0 args1 args2 

-   property get should not come with any argument as we discussed
    above, which will be checked by the compiler.

-   Here `method` is of arity 3.

> **Note**
>
> All JS method application is uncurried, JS’s **method is not a
> function**, this invariant can be guaranteed by OCaml’s type checker,
> a classic example shown below:
>
>     console.log('fine')
>     var log = console.log;
>     log('fine') // 
>
> -   May cause exception, implementation dependent, `console.log` may
>     depend on `this`
>
In BuckleScript

    let fn = f0##f in
    let a = fn 1 2
    (* f##field a b would think `field` as a method *)

is different from

    let b = f1##f 1 2

The compiler will infer as below:

    val f0 : < f : int -> int -> int > Js.t
    val f1 : < f : int -> int -> int [@bs.meth] > Js.t

If we type `console` properly in OCaml, user could only write

    console##log "fine"
    let u = console##log
    let () = u "fine" // 

-   OCaml compiler will complain

> **Note**
>
> If a user were to make such a mistake, the type checker would complain
> by saying it expected `Js.method` but saw a function instead, so it is
> still sound and type safe.

\# getter/setter annotation to JS properties (simplified @since 1.9.2)

Since OCaml’s object system does not have getters/setters, we introduced
two attributes `bs.get` and `bs.set` to help inform BuckleScript to
compile them as property getters/setters.

    type y = <
      height : int [@bs.set no_get] // 
    > Js.t
    type y0 = <
      height : int [@bs.set] [@bs.get null] // 
    > Js.t
    type y1 = <
      height : int [@bs.set] [@bs.get undefined] // 
    > Js.t
    type y2 = <
      height : int [@bs.set] [@bs.get nullable ] // 
    > Js.t
    type y3 = <
      height : int [@bs.get nullable] // 
    > Js.t

-   `height` is setter only

-   getter return `int Js.null`

-   getter return `int Js.undefined`

-   getter return `int Js.nullable`

-   getter only, return `int Js.nullable`

> **Note**
>
> Getter/Setter also applies to class type label

\#\# Create JS objects using bs.obj

Not only can we create bindings to JS objects, but also we can create JS
objects in a type safe way on the OCaml side:

    let u = [%bs.obj { x = { y = { z = 3}}} ] // 

-   `bs.obj` extension is used to mark `{}` as JS objects

**Output:.**

    var u = { x : { y : { z : 3 }}}}

The compiler would infer `u` as type:

    val u : < x : < y : < z : int > Js.t > Js.t > Js.t

To make it more symmetric, extension `bs.obj` can also be applied into
the type level, so you can write:

    val u : [%bs.obj: < x : < y : < z : int > > > ]

Users can also write expression and types together as below:

    let u = [%bs.obj ( { x = { y = { z = 3 }}} : < x : < y : < z : int > > > ]

Objects in a collection also works:

    let xs = [%bs.obj [| { x = 3 } ; { x = 3 } |] : < x : int > array ]
    let ys = [%bs.obj [| { x = 3 } ; { x = 4 } |] ]

**Output:.**

    var xs = [ { x : 3 } , { x : 3 } ]
    var ys = [ { x : 3 } , { x : 4 } ]

\#\# Create JS objects using external

`bs.obj` can also be used as an attribute in external declarations, as
below:

    external make_config : hi:int -> lo:int -> unit -> t = "" [@@bs.obj]
    let v = make_config ~hi:2 ~lo:3

**Output:.**

    var v = { hi : 2 , lo : 3 }

Option argument is also supported:

    external make_config : hi:int -> ?lo:int -> unit -> t = "" [@@bs.obj] // 
    let u = make_config ~hi:3 ()
    let v = make_config ~lo:2 ~hi:3 ()

-   In OCaml, the order of label does not matter, and the evaluation
    order of arguments is undefined. Since the order does not matter, to
    make sure the compiler realize all the arguments are fulfilled
    (including optional arguments), it is common to have a `unit` type
    before the result.

**Output:.**

    var u = {hi : 3}
    var v = {hi : 3 , lo: 2}

Now, we can write JS style code in OCaml too (in a type safe way):

    let u = [%bs.obj {
      x = { y = { z = 3 } };
      fn = fun [@bs] u v -> u + v // 
      } ]
    let h = u##x##y##z
    let a = u##fn
    let b = a 1 2 [@bs]

-   `fn` property is not method, it does not rely on `this`. We will
    show how to create JS method in OCaml later.

**Output:.**

    var u = { x : { y : { z : 3 } }, fn : function (u, v) {return u + v}}
    var h = u.x.y.z
    var a = u.fn
    var b = a(1,2)

> **Note**
>
> When the field is an uncurried function, a short-hand syntax `#@` is
> available:
>
>     let b x y h = h#@fn x y
>
>     function b (x,y,h){
>       return h.fn(x,y)
>     }
>
> The compiler will infer the type of `b` as
>
>     val b : 'a -> 'b -> < fn : 'a -> 'b -> 'c [@bs] > Js.t -> 'c

\#\# Create JS objects with `this` semantics The objects created above
can not use `this` in the method, this is supported in BuckleScript too.

    let v2 =
      let x = 3. in
      object (self) // 
        method hi x y = self##say x +. y
        method say x = x *. self##x ()
        method x () = x
      end [@bs] // 

-   `self` is bound to `this` in generated JS code

-   `[@bs]` marks `object .. end` as a JS object

**Output:.**

    var v2 = {
      hi: function (x, y) {
        var self = this ;
        return self.say(x) + y;
      },
      say: function (x) {
        var self = this ;
        return x * self.x();
      },
      x: function () {
        return 3;
      }
    };

Compiler infers the type of `v2` as below:

    val v2 : <
      hi : float -> float -> float [@bs.meth];
      say : float -> float [@bs.meth];
      x : unit -> float [@bs.meth]
    > [@bs]

Below is another example to consume a JS object :

    let f (u : rect) =
      (* the type annotation is un-necessary,
         but it gives better error message
      *)
       Js.log u##height;
       Js.log u##width;
       u##width #= 30;
       u##height #= 30;
       u##draw ()

**Output:.**

    function f(u){
      console.log(u.height);
      console.log(u.width);
      u.width = 30;
      u.height = 30;
      return u.draw()
    }

\# Method chaining

    f
    ##(meth0 ())
    ##(meth1 a)
    ##(meth2 a b)

\#\# Object label translation convention

There are two cases, where we might want to do name mangling for a JS
object method name.

First, in OCaml, some names are keywords, so we want to add an
underscore to avoid a syntax error.

**Key-word method:.**

    f##_open
    f##_MAX_LENGTH

**OUTPUT:.**

    f.open
    f.MAX_LENGTH

Second, it is common to have several types for a single method. To model
this ad-hoc polymorphism, we introduced a small convention when
translating object labels, which is *occasionally* useful as below

**Ad-hoc polymorphism.**

    f##draw__cat (x,y)
    f##draw__dog (x,y)

**OUTPUT:.**

    f.draw(x,y) // f.draw in JS can accept different types
    f.draw(x,y)

> **Note**
>
> 1.  If `__[rest]` appears in the label, index from the right to left.
>
>     -   If index = 0, nothing mangled
>
>     -   If index &gt; 0, `__[rest]` is dropped
>
> 2.  Else if `_` is the first char
>
>     -   If the following char is not *a* .. *z*, drop the first *\_*
>
>     -   Else if the rest happens to be a keyword, drop the first *\_*
>
>     -   Else, nothing mangled
>
\# Return value checking (@since 1.5.1)

In general, the FFI code is error prone, and potentially will leak in
`undefined` or `null` values.

So we introduced auto coercion for return values to gain two benefits:

1.  More safety for FFI code without performance cost (explained later).

2.  More idiomatic OCaml code for users to consume the FFI.

Below is a contrived core example:

    type element
    type dom
    external getElementById : string -> element option = ""
    [@@bs.send.pipe:dom] [@@bs.return nullable] // 

    let test dom =
        let elem = dom |> getElementById "haha" in
        match elem with
        | None -> 1
        | Some ui -> Js.log ui ; 2

-   `nullable` attribute will automatically convert null and undefined
    to `option`

**Output:.**

    function test(dom) {
      var elem = dom.getElementById("haha");
      if (elem == null) {
        return 1;
      } else {
        console.log(elem);
        return 2;
      }
    }

Currently 4 directives are supported: `null_to_opt`, `undefined_to_opt`,
`nullable`(introduced in @1.9.0) and `identity`. `null_undefined_to_opt`
works the same as `nullable`, but it is deprecated, `nullable` is
encouraged

> **Note**
>
> `null_to_opt`, `undefined_to_opt` and `nullable` will **semantically**
> convert a nullable value to `option` which is a boxed value, but the
> compiler will do smart optimizations to **remove such boxing
> overhead** when the returned value is destructed in the same routine.
>
> The three directives above require users to write literally
> `_ option`. It is in theory not necessary, but it is required to
> reduce user errors.
>
> When the return type is `unit`: the compiler will append its return
> value with an OCaml `unit` literal to make sure it does return `unit`.
> Its main purpose is to make the user consume FFI in idiomatic OCaml
> code, the cost is **very very small** and the compiler will do smart
> optimizations to remove it when the returned value is not used (mostly
> likely).
>
> When the return type is `bool`, the compiler will coerce its return
> value from JS boolean to OCaml boolean. The cost is also **very
> small** and compiler will remove such coercion when it is not needed.
> Note even if your external FFI does return OCaml `bool` or `unit`,
> such implicit coercion will **cause no harm**.
>
> `identity` will make sure that compiler will do nothing about the
> returned value. It is rarely used, but introduced here for debugging
> purpose.

\# Embedding untyped Javascript code

> **Warning**
>
> This is not encouraged. The user should minimize and localize use
> cases of embedding raw JavaScript code, however, sometimes it’s
> necessary to get the job done.

\#\# Detect global variable existence `bs.external` (@since 1.5.1)

Before we dive into embedding arbitrary JS code, a quite common use case
of embedding untyped JS code is detect a global variable (feature
detection), Bucklescript provides a relatively type safe approach for
such use case: `bs.external` (or `external`),
`[%bs.external a_single_identifier]` is a value of `_ option` type, see
examples below

    let test () =
      match [%external __DEV__] with
      | Some _ -> Js.log "dev mode"
      | None -> Js.log "production mode"

**Output:.**

    function test() {
      var match = typeof (__DEV__) === "undefined" ? undefined : (__DEV__);
      if (match !== undefined) {
        console.log("dev mode");
        return /* () */0;
      }
      else {
        console.log("production mode");
        return /* () */0;
      }
    }

    let test2 () =
      match [%external __filename] with
      | Some f -> Js.log f
      | None -> Js.log "non node environment"

**Output:.**

    function test2() {
      var match = typeof (__filename) === "undefined" ? undefined : (__filename);
      if (match !== undefined) {
        console.log(match);
        return /* () */0;
      }
      else {
        console.log("non node environment");
        return /* () */0;
      }
    }

\#\# Embedding arbitrary JS code as an expression

    let keys : t -> string array [@bs] = [%bs.raw "Object.keys" ]
    let unsafe_lt : 'a -> 'a -> Js.boolean [@bs] = [%bs.raw{|function(x,y){return x < y}|}]

We highly recommend writing type annotations for such unsafe code. It is
unsafe to refer to external OCaml symbols in raw JS code.

\#\# Embedding raw JS code as statements

    [%%bs.raw{|
      console.log ("hey");
    |}]

Other examples:

    let x : string = [%bs.raw{|"\x01\x02"|}]

It will be compiled into:

    var x = "\x01\x02"

Polyfill of `Math.imul`

       [%%bs.raw{|
       // Math.imul polyfill
       if (!Math.imul){
           Math.imul = function (..) {..}
        }
       |}]

> **Warning**
>
> -   So far we don’t perform any sanity checks in the quoted text
>     (syntax checking is a long-term goal).
>
> -   Users should not refer to symbols in OCaml code. It is not
>     guaranteed that the order is correct.
>
\# Debugger support

We introduced the extension `bs.debugger`, for example:

      let f x y =
        [%bs.debugger];
        x + y

which will be compiled into:

      function f (x,y) {
         debugger; // JavaScript developer tools will set an breakpoint and stop here
         x + y;
      }

\# Regex support

We introduced `bs.re` for Javascript regex expressions:

    let f = [%bs.re "/b/g"]

The compiler will infer `f` has type `Js.Re.t` and generate code as
below:

    var f = /b/g

> **Note**
>
> `Js.Re.t` can be accessed and manipulated using the functions
> available in the [`Js.Re`](../api/Js.Re.html) module.

\# Examples

Below is a simple example for the [mocha](https://mochajs.org/) library.
For more examples, please visit
<https://github.com/bucklescript/bucklescript-addons>

\#\# A simple example: binding to mocha unit test library

This is an example showing how to provide bindings to the
[mochajs](https://mochajs.org/) unit test framework.

    external describe : string -> (unit -> unit [@bs]) -> unit = "" [@@bs.val]
    external it : string -> (unit -> unit [@bs]) -> unit = "" [@@bs.val]

Since, `mochajs` is a test framework, we also need some assertion tests.
We can also describe the bindings to `assert.deepEqual` from the nodejs
`assert` library:

    external eq : 'a -> 'a -> unit = "deepEqual" [@@bs.module "assert"]

On top of this we can write normal OCaml functions, for example:

    let assert_equal = eq
    let from_suites name suite =
        describe name (fun [@bs] () ->
             List.iter (fun (name, code) -> it name code) suite
             )

The compiler would generate code as below:

     var Assert = require("assert");
     var List = require("bs-platform/lib/js/list");

    function assert_equal(prim, prim$1) {
     return Assert.deepEqual(prim, prim$1);
     }

    function from_suites(name, suite) {
     return describe(name, function () {
       return List.iter(function (param) {
        return it(param[0], param[1]);
          }, suite);
      });
     }