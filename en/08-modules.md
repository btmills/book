Files, Modules, and Programs
============================

We've so far experienced OCaml only through the toplevel.  As you move
from exercises to real-world programs, you'll need to leave the
toplevel behind and start building programs from files.  Files are
more than just a convenient way to store and manage your code; in
OCaml, they also act as abstraction boundaries that divide your
program into conceptual components.

In this chapter, we'll show you how to build an OCaml program from a
collection of files, as well as the basics of working with modules and
module signatures.

## Single File Programs ##

We'll start with an example: a utility that reads lines from `stdin`,
computing a frequency count of the lines that have been read in.  At
the end, the 10 lines with the highest frequency counts are written
out.  Here's a simple implementation, which we'll save as the file
`freq.ml`.  Note that we're using several functions from the
`List.Assoc` module, which provides utility functions for interacting
with association lists, _i.e._, lists of key/value pairs.

~~~~~~~~~~~~~~~~ { .ocaml }
(* file.ml: basic implementation *)

open Core.Std

(* build_counts recursively builds up a mapping from lines to
   number of occurences of that line. *)
let rec build_counts counts =
  match In_channel.input_line stdin with
  | None -> counts (* EOF, so return the counts accummulated so far *)
  | Some line ->
    (* get the number of times this line has been seen before,
       inferring 0 if the line doesn't show up in [counts] *)
    let count =
      match List.Assoc.find counts line with
      | None -> 0
      | Some x -> x
    in
    (* increment the count for line by 1, and recurse *)
    build_counts (List.Assoc.add counts line (count + 1))

let () =
  (* Compute the line counts *)
  let counts = build_counts [] in
  (* Sort the line counts in descending order of frequency *)
  let sorted_counts = List.sort ~cmp:(fun (_,x) (_,y) -> descending x y) counts  in
  (* Print out the 10 highest frequency entries *)
  List.iter (List.take 10 sorted_counts) ~f:(fun (line,count) ->
    printf "%3d: %s\n" count line)
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

<sidebar><title>Where is the main function?</title>

Unlike C, programs in OCaml do not have a unique `main` function. When
an OCaml program is evaluated, all the statements in the
implementation files are evaluated in order.  These implementation
files can contain arbitrary expressions, not just function
definitions. In this example, the role of the `main` function is
played by the expression `let () = process_lines []`, which kicks off
the actions of the program.  But really the entire file is evaluated
at startup, and so in some sense the full codebase is one big `main`
function.

</sidebar>

If we weren't using Core or any other external libraries, we could
build the executable like this:

~~~~~~~~~~~~~~~
ocamlc freq.ml -o freq
~~~~~~~~~~~~~~~

But in this case, this command will fail with the error `Unbound
module Core`.  We need a somewhat more complex invocation to get Core
linked in:

~~~~~~~~~~~~~~~
ocamlfind ocamlc -linkpkg -thread -package core freq.ml -o freq
~~~~~~~~~~~~~~~

Here we're using `ocamlfind`, a tool which itself invokes other parts
of the ocaml toolchain (in this case, `ocamlc`) with the appropriate
flags to link in particular libraries and packages.  Here, `-package
core` is asking `ocamlfind` to link in the Core library, `-linkpkg` is
required to do the final linking in of packages for building a
runnable executable, and `-thread` turns on threading support, which
is required for Core.

While this works well enough for a one-file project, more complicated
builds will require a tool to orchestrate the build.  One great tool
for this task is `ocamlbuild`, which is shipped with the OCaml
compiler.  We'll talk more about `ocamlbuild` in chapter
{{{OCAMLBUILD}}}, but for now, we'll just walk through the steps
required for this simple application.  First, create a `_tags` file,
containing the following lines.

~~~~~~~~~~~~~~~
true:package(core)
true:thread
~~~~~~~~~~~~~~~

The purpose of the `_tags` file is to specify which compilation
options are required for which files.  In this case, we're telling
`ocamlbuild` to link in the `core` package and to turn on threading
for all files (the pattern `true` matches every file in the
project.)

We then create a build script `build.sh` that invokes `ocamlbuild`:

~~~~~~~~~~~~~~~
#!/usr/bin/env bash

TARGET=freq
ocamlbuild -use-ocamlfind $TARGET.byte && cp $TARGET.byte $TARGET
~~~~~~~~~~~~~~~

If you invoke `build.sh`, you'll get a bytecode executable.  If we'd
used a target of `unique.native` in `build.sh`, we would have gotten
native-code instead.

Whichever way you build the application, you can now run it from the
command-line.  The following line extracts strings from the `ocamlopt`
executable, and then reports the most frequently occurring ones.

~~~~~~~~~~~~~~~~~
$ strings `which ocamlopt` | ./freq
 13: movq
 10: cmpq
  8: ", &
  7: .globl
  6: addq
  6: leaq
  5: ", $
  5: .long
  5: .quad
  4: ", '
~~~~~~~~~~~~~~~~~

<sidebar><title>Byte-code vs native-code</title>

OCaml ships with two compilers---the `ocamlc` byte-code compiler, and
the `ocamlopt` native-code compiler.  Programs compiled with `ocamlc`
are interpreted by a virtual machine, while programs compiled with
`ocamlopt` are compiled to native machine code to be run on a specific
operating system and processor architecture.

Aside from performance, executables generated by thet two compilers
have nearly identical behavior.  There are a few things to be aware
of.  First, the byte-code compiler can be used on more architectures,
and has some better tool support; in particular, the OCaml debugger
only works with byte-code.  Also, the byte-code compiler compiles
faster than the native code compiler.

As a general matter, production executables should usually be built
using the native-code compiler, but it sometimes makes sense to use
bytecode for development builds.  And, of course, bytecode makese
sense when targetting a platform not supported by the native code
compiler.

 </sidebar>


## Multi-file programs and modules ##

Source files in OCaml are tied into the module system, with each file
compiling down into a module whose name is derived from the name of
the file.  We've encountered modules before, for example, when we used
functions like `find` and `add` from the `List.Assoc` module.  At it's
simplest, you can think of a module as a collection of definitions
that are stored within a namespace.

Let's consider how we can use modules to refactor the implementation
of `freq.ml`.  Remember that the variable `counts` contains an
association list representing the counts of the lines seen so far.
But updating an association list takes time linear in the length of
the list, meaning that the time complexity of processing a file is
quadratic in the number of distinct lines in the file.

We can fix this problem by replacing association lists with a more
efficient datastructure.  To do that, we'll first factor out the key
functionality into a separate module with an explicit interface.  We
can consider alternative (and more efficient) implementations once we
have a clear interface to program against.

We'll start by creating a file, `counter.ml`, that contains the logic
for maintaining the association list used to describe the counts.  The
key function, called `touch`, updates the association list with the
information that a given line should be added to the frequency counts.

~~~~~~~~~~~~~~~~ { .ocaml }
(* counter.ml: first version *)

open Core.Std

let touch t s =
  let count =
    match List.Assoc.find t s with
    | None -> 0
    | Some x -> x
  in
  List.Assoc.add t s (count + 1)
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

We can now rewrite `freq.ml` to use `Counter`.  Note that the
resulting code can still be built with `build.sh`, since `ocambuild`
will discover dependencies and realize that `counter.ml` needs to be
compiled.

~~~~~~~~~~~~~~~~ { .ocaml }
(* freq.ml: using Counter *)

open Core.Std

let rec build_counts counts =
  match In_channel.input_line stdin with
  | None -> counts
  | Some line -> build_counts (Counter.touch counts line)

let () =
  let counts = build_counts [] in
  let sorted_counts = List.sort counts
    ~cmp:(fun (_,x) (_,y) -> Int.descending x y)
  in
  List.iter (List.take sorted_counts 10)
    ~f:(fun (line,count) -> printf "%3d: %s\n" count line)
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~


### Signatures and Abstract Types ###

While we've pushed some of the logic to the `Counter` module, the code
in `freq.ml` can still depend on the details of the implementation of
`Counter`.  Indeed, if you look at the invocation of `build_counts`:

~~~~~~~~~~~~~~~~ { .ocaml }
  let counts = build_counts [] in
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

you'll see that it depends on the fact that the empty set of frequency
counts is represented as an empty list.  We'd like to prevent this
kind of dependency, so that we can change the implementation of
`Counter` without needing to change client code like that in
`freq.ml`.


The first step towards hiding the implementation details of `Counter`
is to create an interface file, `counter.mli`, which controls how
`counter` is accessed.  Let's start by writing down a simple
descriptive interface, _i.e._, an interface that describes what's
currently available in `Counter` without hiding anything.  We'll use
`val` declarations in the `mli`, which have the following syntax

~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
val <identifier> : <type>
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

and are used to expose the existence of a given value in the module.
Here's an interface that describes the current contents of `Counter`.
We can save this as `counter.mli` and compile, and the program will
build as before.

~~~~~~~~~~~~~~~~ { .ocaml }
(* counter.mli: descriptive interface *)

val touch : (string * int) list -> string -> (string * int) list
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

To actually hide the fact that frequency counts are represented as
association lists, we need to make the type of frequency counts
_abstract_.  A type is abstract if its name is exposed in the
interface, but its definition is not.  Here's an abstract interface
for `Counter`:

~~~~~~~~~~~~~~~~ { .ocaml }
(* counter.mli: abstract interface *)

open Core.Std

type t

val empty : t
val to_list : t -> (string * int) list
val touch : t -> string -> t
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Note that we needed to add `empty` and `to_list` to `Counter`, since
otherwise, there would be no way to create a `Counter.t` or get data
out of one.

Here's a rewrite of `counter.ml` to match this signature.

~~~~~~~~~~~~~~~~ { .ocaml }
(* counter.ml: implementation matching abstract interface *)

open Core.Std

type t = (string * int) list

let empty = []

let to_list x = x

let touch t s =
  let count =
    match List.Assoc.find t s with
    | None -> 0
    | Some x -> x
  in
  List.Assoc.add t s (count + 1)
~~~~~~~~~~~~~~~~~~~~~~~~~~~

If we now try to compile `freq.ml`, we'll get the following error:

~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
File "freq.ml", line 11, characters 20-22:
Error: This expression has type 'a list
       but an expression was expected of type Counter.t
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

This is because `freq.ml` depends on the fact that frequency counts
are represented as association lists, a fact that we've just hidden.
We just need to fix the code to use `Counter.empty` instead of `[]`
and `Counter.to_list` to get the association list out at the end for
processing and printing.

Now we can turn to optimizing the implementation of `Counter`.  Here's
an alternate and far more efficient implementation, based on the `Map`
datastructure in Core.

~~~~~~~~~~~~~~~~ { .ocaml }
(* counter.ml: efficient version *)

open Core.Std

type t = (string,int) Map.t

let empty = Map.empty

let touch t s =
  let count =
    match Map.find t s with
    | None -> 0
    | Some x -> x
  in
  Map.add t s (count + 1)

let to_list t = Map.to_alist t
~~~~~~~~~~~~~~~~~~~~~~~~~~~

## More on modules and signatures

### Concrete types in signatures

In our frequency-count example, the module `Counter` had an abstract
type `Counter.t` for representing a collection of frequency counts.
Sometimes, you'll want to make a type in your interface _concrete_, by
including the type definition in the interface.

For example, imagine we wanted to add a function to `Counter` for
returning the line with the median frequency count.  If the number of
lines is even, then there is no precise median, so the function would
return the two lines before and after the median instead.  We'll use a
custom type to represent the fact that there are two possible possible
return values.  Here's a possible implementation.


~~~~~~~~~~~~~~~~~~~~~~~~~~~ { .ocaml }
type median = | Median of string
              | Before_and_after of string * string

let median t =
  let sorted_strings = List.sort (Map.to_alist t)
      ~cmp:(fun (_,x) (_,y) -> Int.descending x y)
  in
  let len = List.length sorted_strings in
  if len = 0 then failwith "median: empty frequency count";
  let nth n = List.nth_exn sorted_strings n in
  if len mod 2 = 1
  then Median (nth (len/2))
  else Before_and_after (nth (len/2) (len/2 + 1))
~~~~~~~~~~~~~~~~~~~~~~~~~~~

Now, to expose this usefully in the interface, we need to expose both
the function and the type `median` with its definition.  We'd do that
by adding these lines to the `counter.mli`:

~~~~~~~~~~~~~~~~~~~~~~~~~~~ { .ocaml }
type median = | Median of string
              | Before_and_after of string * string

val get_median : t -> median
~~~~~~~~~~~~~~~~~~~~~~~~~~~

The decision of whether a given type should be abstract or concrete is
an important one.  Abstract types give you more control over how
values are created and accessed, and makes it easier to enforce
invariants beyond the what's enforced by the type itself; concrete
types let you expose more detail and structure to client code in a
lightweight way.  The right choice depends very much on the context.

### The `include` directive ###

OCaml provides a number of tools for manipulating modules.  One
particularly useful one is the `include` directive, which is used to
include the contents of one module into another.

One natural application of `include` is to create one module which is
an extension of another one.  For example, imagine you wanted to build
an extended version of the `List` module, where you've added some
functionality not present in the module as distributed in Core.  We
can do this easily using `include`:

~~~~~~~~~~~~~~~~~~~~~~~~~~~ { .ocaml }
(* ext_list.ml: an extended list module *)

open Core.Std

(* The new function we're going to add *)
let rec intersperse list el =
  match list with
  | [] | [ _ ]   -> list
  | x :: y :: tl -> x :: el :: intersperse (y::tl) el

(* The remainder of the list module *)
include List
~~~~~~~~~~~~~~~~~~~~~~~~~~~

Now, what about the interface of this new module?  It turns out that
include works on the signature language as well, so we can pull
essentially the same trick to write an `mli` for this new module.  The
only trick is that we need to get our hands on the signature for the
list module, which can be done using `module type of`.

~~~~~~~~~~~~~~~~~~~~~~~~~~~ { .ocaml }
(* ext_list.mli: an extended list module *)

open Core.Std

(* Include the interface of the list module from Core *)
include (module type of List)

(* Signature of function we're adding *)
val intersperse : 'a list -> 'a -> 'a list
~~~~~~~~~~~~~~~~~~~~~~~~~~~

And we can now use `Ext_list` as a replacement for `List`.  If we want
to use `Ext_list` in preference to `List` in our project, we can
create a file of common definitions:

~~~~~~~~~~~~~~~~~~~~~~~~~~~ { .ocaml }
(* common.ml *)

module List = Ext_list
~~~~~~~~~~~~~~~~~~~~~~~~~~~

And if we then put `open Common` after `open Core.Std` at the top of
each file in our project, then references to `List` will automatically
go to `Ext_list` instead.

### Modules within a file ###

Up until now, we've only considered modules that correspond to files,
like `counter.ml`.  But modules (and module signatures) can be nested
inside other modules.  As a simple example, consider a program that
needs to deal with some class of identifier like a username.  Rather
than just keeping usernames as strings, you might want to mint an
abstract type, so that the type-system will help you to not confuse
usernames with other string data that is floating around your program.

Here's how you might create such a type, within a module:

~~~~~~~~~~~~~~~~~~~~~~~~~~~ { .ocaml }
open Core.Std

module Username : sig
  type t
  val of_string : string -> t
  val to_string : t -> string
end = struct
  type t = string
  let of_string x = x
  let to_string x = x
end
~~~~~~~~~~~~~~~~~~~~~~~~~~~

The basic structure of a module declaration like this is:

~~~~~~~~~~~~~~~~~~~~~~~~~~~ { .ocaml }
module <name> : <signature> = <implementation>
~~~~~~~~~~~~~~~~~~~~~~~~~~~

We could have written this slightly differently, by giving the
signature its own top-level `module type` declaration, making it
possible to in a lightweight way create multiple distinct types with
the same underlying implementation.

~~~~~~~~~~~~~~~~~~~~~~~~~~~ { .ocaml }
module type ID = sig
  type t
  val of_string : string -> t
  val to_string : t -> string
end

module String_id = struct
  type t = string
  let of_string x = x
  let to_string x = x
end

module Username : ID = String_id
module Hostname : ID = String_id

(* Now the following buggy code won't compile *)
type session_info = { user: Username.t;
                      host: Hostname.t;
                      when_started: Time.t;
                    }

let sessions_have_same_user s1 s2 =
  s1.user = s1.host
~~~~~~~~~~~~~~~~~~~~~~~~~~~

### Opening modules ###

One useful primitive in OCaml's module language is the `open`
directive.  We've seen that already in the `open Core.Std` that has
been at the top of our source files.

The basic purpose of `open` is to extend the namespaces that OCaml
searches when trying to resolve an identifier.  Roughly, if you open a
module `M`, then every subsequent time you look for an identifier
`foo`, the module system will look in `M` for a value named `foo`.
This is true for all kinds of identifiers, including types, type
constructors, values and modules.

`open` is essential when dealing with something like a standard
library, but it's generally good style to keep opening of modules to a
minimum.  Opening a module is basically a tradeoff between terseness
and explicitness - the more modules you open, the harder it is to
look at an identifier and figure out where it's defined.

Here's some general advice on how to deal with opens.

  * Opening modules at the top-level of a module should be done quite
    sparingly, and generally only with modules that have been
    specifically designed to be opened, like `Core.Std` or
    `Option.Monad_infix`.

  * One alternative to local opens that makes your code terser without
    giving up on explicitness is to locally rebind the name of a
    module.  So, instead of writing:

    ~~~~~~~~~~~~~~~~~~~~~~~~~~~ { .ocaml }
    let print_median m =
       match m with
       | Counter.Median string -> printf "True median:\n   %s\n"
       | Counter.Before_and_after of before * after ->
         printf "Before and after median:\n   %s\n   %s\n" before after
    ~~~~~~~~~~~~~~~~~~~~~~~~~~~

    you could write

    ~~~~~~~~~~~~~~~~~~~~~~~~~~~ { .ocaml }
    let print_median m =
       let module C = Counter in
       match m with
       | C.Median string -> printf "True median:\n   %s\n"
       | C.Before_and_after of before * after ->
         printf "Before and after median:\n   %s\n   %s\n" before after
    ~~~~~~~~~~~~~~~~~~~~~~~~~~~

    Because the module name `C` only exists for a short scope, it's
    easy to read and remember what `C` stands for.  Rebinding modules
    to very short names at the top-level of your module is usually a
    mistake.

  * If you do need to do an open, it's better to do a _local open_.
    There are two syntaxes for local opens.  For example, you can write:

    ~~~~~~~~~~~~~~~~~~~~~~~~~~~ { .ocaml }
    let average x y =
      let open Int64 in
      x + y / of_int 2
    ~~~~~~~~~~~~~~~~~~~~~~~~~~~

    In the above, `of_int` and the infix operators are the ones from
    `Int64` module.

    There's another even more lightweight syntax for local opens, which
    is particularly useful for small expressions:

    ~~~~~~~~~~~~~~~~~~~~~~~~~~~ { .ocaml }
    let average x y =
      Int64.(x + y / of_int 2)
    ~~~~~~~~~~~~~~~~~~~~~~~~~~~


### Common errors with modules

When OCaml compiles a program with an `ml` and an `mli`, it will
complain if it detects a mismatch between the two.  Here are some of
the common errors you'll run into.

#### Type mismatches

The simplest kind of error is where the type specified in the
signature does not match up with the type in the implementation of the
module.  As an example, if we replace the `val` declaration in
`counter.mli` by swapping the types of the first two arguments:

~~~~~~~~~~~~~~~~~~~~~~~~~~~ { .ocaml }
val touch : string -> t -> t
~~~~~~~~~~~~~~~~~~~~~~~~~~~

and then try to compile `Counter` (by writing `ocamlbuild
-use-ocamlfind counter.cmo`), we'll ge the following error:

~~~~~~~~~~~~~~~~~~~~~~~~~~~
File "counter.ml", line 1, characters 0-1:
Error: The implementation counter.ml
       does not match the interface counter.cmi:
       Values do not match:
         val touch :
           ('a, int) Core.Std.Map.t -> 'a -> ('a, int) Core.Std.Map.t
       is not included in
         val touch : string -> t -> t
~~~~~~~~~~~~~~~~~~~~~~~~~~~

This error message is a bit intimidating at first, and it takes a bit
of thought to see where the first type, which is the type of [touch]
in the implementation, doesn't match the second one, which is the type
of [touch] in the interface.  You need to recognize that [t] is in
fact a [Core.Std.Map.t], and the problem is that in the first type,
the first argument is a map while the second is the key to that map,
but the order is swapped in the second type.

#### Missing definitions

We might decide that we want a new function in `Counter` for pulling
out the frequency count of a given string.  We can update the `mli` by
adding the following line.

~~~~~~~~~~~~~~~~~~~~~~~~~~~ { .ocaml }
val count : t -> string -> int
~~~~~~~~~~~~~~~~~~~~~~~~~~~

Now, if we try to compile without actully adding the implementation,
we'll get this error:

~~~~~~~~~~~~~~~~~~~~~~~~~~~
File "counter.ml", line 1, characters 0-1:
Error: The implementation counter.ml
       does not match the interface counter.cmi:
       The field `count' is required but not provided
~~~~~~~~~~~~~~~~~~~~~~~~~~~

A missing type definition will lead to a similar error.

#### Type definition mismatches

Type definitions that show up in an `mli` need to match up with
corresponding definitions in the `ml`.  Consider again the example of
the type `median`.  The order of the declaration of variants matters
to the OCaml compiler so, if the definition of `median` in the
implementation lists those options in a different order:

~~~~~~~~~~~~~~~~~~~~~~~~~~~ { .ocaml }
type median = | Before_and_after of line * line
              | Median of line
~~~~~~~~~~~~~~~~~~~~~~~~~~~

that will lead to a compilation error:

~~~~~~~~~~~~~~~~~~~~~~~~~~~
File "counter.ml", line 1, characters 0-1:
Error: The implementation counter.ml
       does not match the interface counter.cmi:
       Type declarations do not match:
         type median = Before_and_after of string * string | Median of string
       is not included in
         type median = Median of string | Before_and_after of string * string
       Their first fields have different names, Before_and_after and Median.
~~~~~~~~~~~~~~~~~~~~~~~~~~~

Order is similarly important in other parts of the signature,
including the order in which record fields are declared and the order
of arguments (including labelled and optional arguments) to a
function.

#### Cyclic dependencies

In most cases, OCaml doesn't allow circular dependencies, _i.e._, a
collection of definitions that all refer to each other.  If you want
to create such definitions, you typically have to mark them specially.
For example, when defining a set of mutually recursive values, you
need to define them using `let rec` rather than ordinary `let`.

The same is true at the module level.  By default, circular
dependencies between modules is not allowed, and indeed, circular
dependencies among files is never allowed.

The simplest case of this is that a module can not directly refer to
itself (although definitions within a module can refer to each other
in the ordinary way).  So, if we tried to add a reference to `Counter`
from within `counter.ml`:

~~~~~~~~~~~~~~~~~~~~~~~~~~~ { .ocaml }
let singleton l = Counter.touch Counter.empty
~~~~~~~~~~~~~~~~~~~~~~~~~~~

then when we try to build, we'll get this error:

~~~~~~~~~~~~~~~~~~~~~~~~~~~
File "counter.ml", line 17, characters 18-31:
Error: Unbound module Counter
Command exited with code 2.
~~~~~~~~~~~~~~~~~~~~~~~~~~~

The problem manifests in a different way if we create circular
references between files.  We could create such a situation by adding
a reference to Freq from `counter.ml`, _e.g._, by adding the following
line:

~~~~~~~~~~~~~~~~~~~~~~~~~~~ { .ocaml }
let build_counts = Freq.build_counts
~~~~~~~~~~~~~~~~~~~~~~~~~~~

In this case, `ocamlbuild` will notice the error and complain:

~~~~~~~~~~~~~~~~~~~~~~~~~~~
Circular dependencies: "freq.cmo" already seen in
  [ "counter.cmo"; "freq.cmo" ]
~~~~~~~~~~~~~~~~~~~~~~~~~~~