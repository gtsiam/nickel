# LSP Semantics

These are work-in-progress notes on how we want the language server to behave.

## Goto definition

The LSP has support for going to declarations, definitions, type definitions, and implementations. It
also has a "find references" method, which is sort of like a goto.

Here is the easy base for goto definition: when a variable is bound by a
`let`, `fun`, or `match`, the variable binding (possibly in a pattern) is an LSP
"definition" and any uses of that variable are LSP "references".

For example, in `let foo = 3 in 4 + foo` a "list references" request on the
first `foo` should return the second `foo`, and a "goto definition" on the
second `foo` should return the first `foo`.

Static accesses are a little more challenging: given a path like `foo.bar.baz`
and a "goto definition" request for `baz`, it would be nice if we could find
where the value for `baz` is set (i.e. some part of the code that looks like
`{ ..., baz = 2, ...}`). This cannot be done in general (at least, without
evaluation) because `foo` could be the result of some arbitrary computation.
Nevertheless, we make some effort at static analysis for the common cases.

Let's describe a few simple cases first:

- The fields in a record literal
  are LSP "definitions" and any static accesses of them are LSP "references".
  For example, in `{bar = 3}.bar` the first occurrence of `bar` is
  a definition and the second is a reference to it.

- For all path elements except the last, `let` bindings should be transparent.
  For example, in each of

    - `let foo = { bar = 3 } in foo.bar`,
    - `let baz = { bar = 3 } in let foo = baz in foo.bar`, and

  the second occurrence of `bar` references the first.

- The definitions should be repeatedly "resolved" for all path elements
  except the last.
  In `let foo = { baz = { bar = 3 } } in foo.baz.bar`, the second occurrence
  of `bar` references the first.

- In the merge of two records, fields defined on both sides are definitions.
  We don't take merge priorities into account, because it's probably useful for the user to
  see the overridden values in addition to the values that "win." Note that the LSP allows
  multiple responses to a "goto definition" request.

  For example, in `let x = { foo | default = 3, bar = 4 } & { foo = 2 } in [x.foo, x.bar]`
  the second instance of `bar` references the first instance, while the last instance of `foo`
  references the first two instances.

- If-then-else acts like a merge: both branches provide definitions, so that
  in `let x = if ... then { foo = 1 } else { foo = 2 } in x.foo`, the last instance of `foo`
  references the first two instances.

- We should see through function applications to some extent (with details TBD).
  For example, sprinkling around the identity function shouldn't break any of the examples
  above. Other examples: in each of

   - `let f = fun x => {bar = 1} in (f 0).bar`
   - `let f = fun x => x.foo in (f { foo = { bar = 1 } }).bar`

  the second instance of `bar` references the first.

  `typescript-language-server` is worth looking at for inspiration here.
  For example, in

  ```typescript
  function foo() {
    return { foo : 1 };
  }

  const x = foo();
  x.foo;
  ```
  then "goto definition" on the final `foo` points to the `foo` in the record literal.
  The algorithm seems to be mainly type-directed. For example, annotating `foo()`
  as `function foo(): any { ... }` breaks going to the definition of `foo`

### Proposed algorithm

The core part of the algorithm is a function mapping terms to sets of
record literals, giving for each term the set of record literals that might
"contribute" (through, e.g., merging) to that term. This will only be an
approximation, of course, because we aren't evaluating. Let's call this process
"record literal resolution," and it is defined by:

- a record literal resolves to itself
- a variable resolves to the term that it's bound to (e.g. in a let binding)
- an import statement resolves to the contents of the file that was imported
- a merge resolves to the union of the resolutions of the two merged terms
- an if-then-else resolves to the union of the resolutions of the two branches
- a function application resolves to the resolution of its body (with the
  function arguments bound appropriately). There probably needs to be some
  protection against recursion (TBD)
- everything else resolves to the empty set (but this can be extended later)

Here is some pseudocode roughly corresponding to what's above: we have
a struct `DefInfo` that represents some object with fields having locations:

```rust
struct DefInfo {
  /// What are the definitions available on this field? There could be many:
  /// in the presence of a merge, a field might be defined in both branches.
  definitions: HashMap<Ident, Vec<DefInfo>>,
  // The location of the definition, if any.
  body: TermPos,
}
```

Then we have a function `field_infos(RichTerm) -> DefInfo` that looks something like:

```text
field_infos(e1 & e2) = field_infos(e1) U field_infos(e2)
field_infos(let x = e1 in e2) = field_infos(e2)
field_infos(fun x => body) = field_infos(body)
field_infos(head x) = field_infos(head)
field_infos(var) = field_infos(goto_definition(var))
field_infos(e1 | C) = field_infos(e1) U field_infos(C)
field_infos(foo.bar) = field_infos(goto_definition(bar in foo.bar))
other cases => empty
```

Now to find the definition of `baz` in `foo.bar.baz`, we first resolve `foo`
to a set of record literals. For each of those literals containing a `bar` field,
we resolve their values to a set of record literals. For each of those literals
containing a `baz` field, we report it as a definition of `baz`.

## Goto type definition

Since we cannot (yet) name custom types, it probably doesn't make sense to use "goto type definition"
for static types. Instead, we could use the LSP "goto type definition" request for going
to contract definitions: given `x | Foo`, a "goto type definition" request for `x` should be the same
as a "goto definition" request for `Foo`. This would respect record contracts, so that
in `{x = 1, y = 2} | { x | Foo, y | Bar }` a "goto type definition" request for `x` should again
be the same as a "goto definition" request for `Foo`.

## Completion

Here is a list of items that we might want to provide completions for:

- record fields in static paths, as in `let x = { foo = 1 } in x.fo`
- record fields in record literals, as in `{ fo } | { foo | Number }`
- enum variants, as in `let x | [| 'Foo, 'Bar |] = 'Fo`
- variables in scope, as in `let foo = 1 in 2 + fo`
- filenames in imports, as in `import "fo` when `foo.ncl` exists on disk
- maybe keywords? They're pretty short in nickel

One of the trickier parts of handling completion is that the input will be incomplete
and may not parse. Let's ignore that for now, and assume that we have a full AST.

In LSP, the editor (and not the language server) is in charge of text-based filtering.
That is, if the user enters `2 + fo` and requests completion, we can return all of the
names of variables in scope; the editor is in charge of filtering out all those that
don't start with "fo". When types are involved, the responsibilities swap: when
completing `x.fo` the language server should return only the field names belonging
to `x` (it still doesn't need to care about whether they start with "fo").
Type information can also be used to filter completions for enum variants and
variables in scope (more on that below).

Completion behavior for everything except a record field is fairly straightforward:

- when completing an enum variant, if we know the type of the term that's being
  completed (and it's an enum type), return all of that type's known enum variants.
  Otherwise, just return all the enum variants we've seen ever.
- when completing a variable in scope, if we know the type of the term that's being
  completed then return all of the in-scope variables that are either `Dyn` or have
  the right type. Otherwise, just return all the variables in scope.
- when completing an import filename, we could return all files in the directory tree
  and let the editor sort them out, but this is probably slow. Instead, take the
  basename of the path so far and (if it points to a directory that exists) return
  all files in that directory
- if we decide we want to complete keywords, just return all the keywords and let
  the editor filter them

Record fields are more complicated. Completing record fields is quite related
to going to definitions, and so they may be able to share some of the implementation,
including the `field_infos` function described above. One potential difference
is in how they treat contract annotations. For example, in
`(x | { foo | Number }).foo` we certainly want to use the contract annotation
for completion, but do we want to use it for goto definition?
