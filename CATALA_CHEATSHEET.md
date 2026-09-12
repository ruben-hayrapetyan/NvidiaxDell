# Catala Cheatsheet

> **We ship a patched Catala.** Seven defects found while probing the toolchain
> with real legal documents are fixed on branch `fix/legal-ingestion-bugs`
> (commit `ec608147`) in `../catala`, with upstream's suite green (707/707, and
> 66/66 for proof). Sections below marked **[FIXED IN OUR BUILD]** describe
> stock `nightly-137` behaviour and what our build does instead — they still
> matter if anything ever runs against an unpatched compiler.

**Pinned to:** `nightly-137-gb7623302` (the compiler checked out at `../catala`).
Syntax reference v1.2.1. Everything below is lifted from that tree — primarily
`doc/syntax/syntax_en.catala_en` (the file upstream uses to generate the official
cheat sheet, and which is itself typechecked in CI) and `tests/`. If the compiler
is bumped, re-derive this file from those sources rather than editing it by hand.

**Every agent and every runtime prompt loads this file.**

## The one non-negotiable rule

Catala is **never hand-written and hoped for**. It is only ever produced by:

```
generate  ->  catala typecheck  ->  repair  ->  (repeat until clean)
```

Never emit a module the typechecker has not signed off on. §9 is the repair
lookup table: the compiler's error messages are precise, and almost every one of
them maps to a mechanical fix.

---

## 0. Toolchain — how to actually run it

Built and verified on this box (macOS arm64). The compiler lives in a local opam
switch inside the checkout; nothing is installed globally.

```sh
# one-time, already done:
#   brew install opam ninja pkgconf
#   opam init --bare
#   cd catala && opam switch create . ocaml-base-compiler.5.3.0 --no-install
#   opam install ./catala.opam --deps-only --switch "$PWD" --confirm-level=unsafe-yes
#   dune build catala.install --promote-install-files
#   opam install ./catala.opam --working-dir --assume-built --switch "$PWD"

export PATH="<repo>/catala/_opam/bin:$PATH"   # gives you catala + clerk 1.2.1
```

Use `--switch "$PWD"` / a direct `PATH` prepend rather than `opam exec --switch`
from an unrelated directory — the latter fails with
`[ERROR] Variable lib not found in the global configuration`.

`make compiler` fails at the end on `Library "z3" not found`. That is the proof
backend only; `catala.exe` and `clerk.exe` are already built when it happens, and
`dune build catala.install` succeeds regardless. Do not install z3.

### A Catala project needs clerk

Running `catala` on a bare file fails:

```
The standard library module Stdlib_en could not be found at "_build/libcatala".
Hint: run command 'clerk start' first to setup the standard library
```

So every working directory needs a `clerk.toml`:

```toml
[project]
build_dir = "_build"
```

then `clerk start` once to stage the stdlib. After that both `catala typecheck
FILE` and the `clerk` subcommands work. **Prefer `clerk` over calling `catala`
directly** — it resolves module dependencies; bare `catala` does not.

---

## 1. File shape (literate programming)

A Catala file is a Markdown document. Prose is the law; fenced blocks are the
code. Headings (`#`, `##`) structure the document and are echoed in error
messages as context, so they are worth getting right.

~~~markdown
## Article 7 — Base allowance

The allowance is set at $1,000 per month.

```catala
scope Allowance:
  definition amount equals $1,000
```
~~~

Block kinds:

| Fence | Meaning |
|---|---|
| ```` ```catala ```` | code |
| ```` ```catala-metadata ```` | declarations that are exported from a module |
| ```` ```catala-test-cli ```` | an expected-output test (see §8) |

Module directives are written as block quotes, outside code fences:

```catala
> Module ModuleBar
> Using ModuleFoo
> Include: foo.catala_en
```

**The clause text goes directly above the code it compiles to, and each clause
gets its own heading and its own fenced block.** This is not cosmetic. The
compiler attaches the enclosing heading stack to every rule it parses and reports
it as `law_headings` (see §8.3) — so one big code block at the end of the file
makes every rule claim provenance from whatever heading happens to precede it.
Verified: collapsing three articles into one block made all three rules report
`Article 3`. Split per article and each rule reports its own.

Repeat `scope Foo:` in each block; the definitions accumulate across blocks.

---

## 2. Types and literals

```catala
declaration x content boolean equals true
declaration x content integer equals 65536
declaration x content decimal equals 65536.262144
declaration x content decimal equals 37%
declaration x content money equals $1,234,567.89
declaration x content date equals |2024-04-01|
declaration x content duration equals 254 day + -4 month + 1 year
declaration x content optional of money equals Present content $34
declaration x content optional of money equals Absent
declaration x content list of integer equals [ 12; 24; 36 ]
declaration x content (date, money, decimal) equals (|2024-04-01|, $30, 1%)
```

Dates are always `|YYYY-MM-DD|`. Money is a distinct type from decimal, and
`37%` is a decimal literal, not a separate percent type.

Toplevel constants and functions:

```catala
declaration const content decimal
  equals 17.1

declaration square content decimal
  depends on x content decimal
  equals x * x
```

---

## 3. Operators

```catala
not a        a and b       a or b        # "or otherwise"    a xor b
- a          a + b         a - b         a * b               a / b
a = b        a != b        a < b         a <= b    a > b     a >= b
```

Type-disambiguated forms, needed when inference cannot pick one:

```catala
a +! b    # integer
a +. b    # decimal
a +$ b    # money
a +^ b    # duration
```

Conversions are explicit — **there is no implicit numeric coercion**, and
forgetting this is the single most common type error:

```catala
decimal of 44      money of 23.15      round of $9.99
Date.get_month of |2003-01-02|         Date.first_day_of_month of |2003-01-02|
```

---

## 4. Declarations: struct, enum, scope

```catala
declaration structure Struct1:
  data fld1 content integer
  data fld2 content decimal

declaration enumeration Enum1:
  -- Case1 content integer
  -- Case2

## Doc for Scope1
#[test]
declaration scope Scope1:
  internal var1 content integer
    state before
    state after
  internal var2 condition
  sub1 scope Scope0

  output var3 content integer
  input var4 content integer
  input output var5 content integer
  context var6 content integer
  context output var7 content integer
  output sub2 scope Scope0
```

Scope variable qualifiers — **this is the scope's input signature, and it is what
the chat layer reads to elicit missing facts**:

| Qualifier | Meaning |
|---|---|
| `input` | must be supplied by the caller; the scope cannot define it |
| `output` | visible to the caller |
| `input output` | both |
| `internal` | neither; private to the scope |
| `context` | has a default in the scope, but the caller may override it |
| `context output` | as above, and visible |
| `condition` | a boolean defined with `rule` rather than `definition` |
| `<name> scope S` | a subscope instance |

`state before` / `state after` declare successive states of one variable; refer
to an earlier one with `var1 state before`.

---

## 5. Scope definitions, labels, exceptions

```catala
scope Scope1:
  definition var1 equals 0

  definition var1
    under condition 0
    consequence equals 0

  rule var2
    under condition var1 >= 2
    consequence fulfilled

  rule var2 under condition false
    consequence not fulfilled

  definition f of x, y equals 0

  label lbl1 definition var1 equals 0
  exception lbl1 definition var1 equals 0
  exception definition var1 equals 0

  definition var1
    state before
    equals 0

  assertion 0
  date round down
  date round up
```

A whole block of definitions can share a guard:

```catala
scope Scope1
  under condition var1 >= 2:
  ...
```

`definition` assigns a value; `rule ... consequence fulfilled / not fulfilled`
defines a `condition` variable. Function definitions repeat the parameter names
from the declaration — **the names must match exactly** (§9.9).

### The exception mechanism

This is the reason to use Catala at all, and it must not be flattened into
nested `if`s. The structure *is* the meaning: a base case, and exceptions that
override it.

- `label L definition x ...` names a definition (or a *group* of definitions —
  reusing the same label on several definitions groups them).
- `exception L definition x ...` declares this definition as overriding the one
  labelled `L`. It wins whenever its condition holds.
- `exception definition x ...` (unlabelled) means "overrides the only other
  definition of `x`" — legal only when there is exactly one candidate (§9.2).
- Exceptions nest: an exception can itself carry a `label` and be overridden.

At runtime exactly one definition must apply. Two applicable definitions with no
exception relation between them is a **conflict error** (§9.1) — not a silent
last-one-wins. That error is the corpus's contradiction detector.

---

## 6. Expressions

```catala
let x equals 36 - 5 in ...

if cond then a else b

match expr with pattern
  -- Case1 content x : 0
  -- Case2 : 0
  -- anything : 0            # wildcard, must be last

expr with pattern Case1                      # is it this case?
expr with pattern Case1 content x and x >= 2 # ...and a guard on the payload

struc1 but replace { -- fld2: 8% }   # functional record update
struc1.fld2                          # field access
tuple1.2                             # tuple projection (1-indexed)
sub1.var0                            # subscope variable access
f of $44.50, 1/3                     # function application

output of Scope1 with { -- fld1: 9 -- fld2: 15% }   # direct scope call

assertion x > 0
#[error.message = "err"] impossible   # unreachable branch
#[debug.print = "message"] 0
```

Struct and enum construction:

```catala
Struct1 { -- fld1: 9 -- fld2: 7% }
Case1 content 12
Case2
```

---

## 7. List operations

```catala
lst contains 3
exists x among lst such that x > 2
for all x among lst we have x > 2
map each x among lst to x + 2
list of x among lst such that x > 2
map each x among lst such that x > 2 to x - 2
map each (x, y) among (lst1, lst2) to x + y
lst1 ++ lst2
Integer.sum of lst    # NOT 'sum integer of' — deprecated, see below
number of lst
maximum of lst or if list empty then -1
content of x among lst such that x.fld1 = 0
content of x among lst such that x * x is minimum or if list empty then -1
sort lst in decreasing order
sort all x among lst in increasing order of x.fld2 and then -x.fld1
combine all x among lst in acc initially 0 with acc + x
```

Partial operations (`maximum`, `content of ... is minimum`) require an
`or if list empty then ...` fallback. The compiler will not let you skip it.

**`sum <type> of` is deprecated** — the compiler warns at run time that it
"will be removed in the next release. Use the function 'Money.sum of' instead".
It is still shown in `doc/syntax/syntax_en.catala_en`, so **the official syntax
reference teaches a construct that is going away**; prefer `Money.sum of`,
`Integer.sum of`, `Decimal.sum of`.

Empty lists behave well: `number of [] = 0`, `Money.sum of [] = $0.00`,
`maximum ... or if list empty then $0` returns the fallback, and
`List.first_element of []` / `List.nth_element of l, 5` return `Absent`.

---

## 8. Modules and tests

### Modules

Define (`tests/modules/good/mod_def.catala_en`): the `> Module` directive names
it, and `catala-metadata` blocks are its exported interface.

```catala
> Module Mod_def

declaration enumeration Mod_def:
  -- Yes
  -- No
  -- Maybe
  -- Something content integer

#[test] declaration scope S:
  output sr content money
```

Use (`tests/modules/good/mod_use.catala_en`):

```catala
> Using Mod_def

#[test] declaration scope T2:
  t1 scope Mod_def.S
  output o3 content money

scope T2:
  definition o3 equals t1.sr
  definition o4 equals Mod_def.half of 10
```

Names are qualified `Module.name` when ambiguous, bare when not.

### A module's exported declarations MUST be in `catala-metadata`

This is the single biggest trap in a module-per-document corpus, because the
failure is invisible.

Put `declaration scope …` in a plain ```` ```catala ```` block and the module
**typechecks perfectly on its own**. The moment another document references it:

```
clerk test   ->   exit 2, stdout EMPTY, stderr EMPTY
```

Nothing at all. Only `clerk test --debug` reveals the cause, and it is an
*OCaml-level* error from the code generator, not a Catala diagnostic:

```
FAILED: [code=2] _build/ocaml/BenefitsAct.cmi
Error: Unbound module "HousingAct.EligibleRent"
```

Move the declarations into ```` ```catala-metadata ```` — keeping the
definitions in a plain ```` ```catala ```` block — and the same corpus builds,
runs ($72.00 across a two-statute chain), proves, and traces with cross-module
`law_headings` intact.

```catala
> Module HousingAct
```

~~~markdown
```catala-metadata
declaration scope EligibleRent:      ← interface: MUST be here
  input actual content money
  output eligible content money
```

```catala
scope EligibleRent:                  ← implementation: plain block is fine
  definition eligible equals if actual < cap then actual else cap
```
~~~

Note also: **interpreting a module requires it compiled to a native OCaml
`.cmxs`** (`Compiled OCaml object "_build/ocaml/X.cmxs" not found`), so `catala
interpret -I .` cannot stand alone on a module corpus — run `clerk` first.

`clerk test` builds **only** the OCaml backend (verified: a working two-module
corpus produced `_build/ocaml/*.cmxs` and no C or Java output). `clerk build`
is the one that compiles the configured target backends — there the C backend
needs `gmp.h` on the include path (`export CPATH=/opt/homebrew/include` on
Homebrew macOS) and Java needs `javac`, and a missing toolchain there produces
the same silent exit 2. All four backends (ocaml, python, c, java) generate
code fine for legal constructs — structs, enums, lists, dates with rounding,
money, nested exceptions and assertions — so backend failures are environment
problems, never language limits.

### Tests (`clerk`)

Tests live **inside** the Catala file as expected-output blocks:

~~~markdown
```catala-test-cli
$ catala test-scope A
┌─[RESULT]─ A ─
│ x = 2
└─
```
~~~

Workflow (from `tests/README.md`):

| Command | Effect |
|---|---|
| `clerk test <file> --reset` | run and **record** the actual output as expected |
| `clerk test <file>` | run and diff against the recorded output |
| `clerk test` | run the whole suite |

`catala test-scope S` behaves like `catala Interpret -s S` but lets clerk vary
the flags (optimisations on/off, alternate interpreters) across the same test.

**Every adversarial counterexample becomes one of these blocks, permanently.**
Write the fact pattern as a `#[test] declaration scope` that feeds the module
under test, record it with `--reset`, commit it.

### 8.3 Machine-readable introspection — *use these, don't parse prose*

Two `clerk` subcommands hand over as structured data what would otherwise need
scraping or inference.

**`clerk exceptions FILE -s SCOPE -v VAR --output-format=json`** returns the
whole exception tree for one variable. Each rule carries its source span, its
`law_headings` (the enclosing Markdown heading stack — i.e. the clause), and
`condition_text` (the condition, rendered). Verified shape:

```json
{"scope":"Allowance","variable":"amount","trees":[
  {"label":"base","rules":[{"pos":{"start_line":18,
     "law_headings":["Allowance Act","Article 1 — Base rate"]}}],
   "exceptions":[
     {"label":"reduced","rules":[{"condition_text":"income > $50,000.00",
        "pos":{"law_headings":["Allowance Act","Article 2 — Reduced rate"]}}],
      "exceptions":[
        {"label":"exception_to_reduced","rules":[{"condition_text":"dependants > 2",
           "pos":{"law_headings":["Allowance Act","Article 3 — Exemption"]}}],
         "exceptions":[]}]}]}]}
```

Unlabelled exceptions are auto-named (`exception_to_reduced`). **This is the
"which exception branch was taken" mechanism**: the tree gives the hierarchy and
the clause text for every branch, with no model in the loop.

**`clerk json-schema FILE --scope SCOPE`** returns a two-element array —
input schema, then output schema — as JSON Schema draft-04. The input schema's
`required` array is literally the list of facts the scope needs:

```json
{"title":"Scope Allowance input",
 "definitions":{"Allowance_in":{"type":"object",
   "properties":{"income":{"$ref":"#/definitions/money"},
                 "dependants":{"$ref":"#/definitions/integer"}},
   "required":["dependants","income"],
   "additionalProperties":false}}}
```

**This is the missing-fact elicitation mechanism.** Diff the facts extracted from
the question against `required`; what's left is exactly what to ask for, with its
type. Do not reconstruct this by reading the declaration.

### 8.4 `--trace` — the branch actually taken

`clerk exceptions` (§8.3) gives the *static* hierarchy. For what fired on a
given fact pattern, use the execution trace:

```sh
catala interpret FILE -s SCOPE --trace --trace-format=json
```

Nodes have `kind` ∈ {`scope_call`, `scope_var`, `exception`}, each with `pos`
and `law_headings`. The `exception` node with `value: true` inside the scope
under test is the branch that decided the answer:

```
EXCEPTION TAKEN: line=43 heading=Article 3 — Exemption for large families value=True
```

Three practical caveats:

1. **The output is not one JSON document.** The trace array is on **line 1**;
   the human `┌─[RESULT]─` block follows on subsequent lines of the same
   stdout. Parse the first line only.
2. **`name` is `null` on exception nodes** — the label is not carried. Correlate
   `pos` against `clerk exceptions` output to recover the label.
3. **Filter by scope.** The test scope's own definitions appear as `exception`
   nodes too; only nodes whose `law_headings` belong to the module under test
   are clause provenance.

Note the recurring pattern across all three machine-readable outputs: warnings
land in the `json-schema` stdout (§8.3), `proof` writes only to stderr (§12),
and the JSON trace is followed by human text. **Never pipe any of them straight
into a parser** — isolate the payload first.

### Compiler commands

| Command | Use |
|---|---|
| `catala typecheck FILE` | the gate in the generate→typecheck→repair loop |
| `catala Typecheck --check-invariants` | stricter; also checks internal invariants |
| `clerk build FILE` | typecheck with dependencies resolved |
| `clerk run FILE -s S` | build deps, then execute scope `S` |
| `catala test-scope S` | execute and print outputs (inside a clerk project) |
| `clerk exceptions FILE -s S -v V` | exception tree (§8.3) |
| `clerk json-schema FILE --scope S` | input/output signature (§8.3) |
| `catala interpret FILE -s S --trace --trace-format=json` | branch actually taken (§8.4) |
| `catala proof FILE` | static conflict/gap detection (§12) |
| `catala scopelang FILE` | liveness check: what actually compiled (§13.1) |
| `clerk test FILE --xml` | machine-readable test counts (§13.3) |

Exit code `123` means the command produced errors. Errors are emitted with the
offending source span *and* the enclosing Markdown heading, so the clause that
caused them is machine-recoverable from the error text.

---

## 9. Common compiler errors and their fixes

Messages below are verbatim from the test suite in the pinned tree.

### 9.1 Conflict between definitions — *the contradiction signal*

```
During evaluation: conflict between multiple valid consequences for
assigning the same variable.
```

Both spans are reported, each with its `law_headings`, which is what lets ingest
surface *both* clauses. Verified output for two clauses that overlap at
`income > $50,000, dependants > 2`:

```
│  During evaluation: conflict between multiple valid consequences for
│  assigning the same variable.
├─➤ conflict.catala_en:56.24-28:      consequence equals $700
├─ Allowance Act
│  └─ Article 4 — Amending regulation (conflicting)
├─➤ conflict.catala_en:44.24-30:      consequence equals $1,000
└─ Allowance Act
   └─ Article 3 — Exemption for large families
```

**`catala typecheck` does not catch this.** Verified: the file above typechecks
clean and the conflict appears only under `clerk run`, when a fact pattern makes
both definitions apply at once. A static warning (`Multiple conflicting
definitions: these have the same conditions and will always trigger a conflict at
runtime`) is emitted only in the easy case where the conditions are *syntactically*
identical, as in `tests/exception/bad/two_exceptions.catala_en`.

The consequence for ingestion: **the conflict gate is the fact-pattern suite, not
the typechecker.** A contradiction is only detected if some test drives both
clauses into scope simultaneously. This is why every exception branch needs test
coverage — the coverage *is* the detector.

Cause: two definitions of the same variable apply simultaneously with no
exception relation ordering them. From `tests/exception/bad/two_exceptions.catala_en`:

```catala
label base_x definition x equals 0
exception base_x definition x equals 1
exception base_x definition x equals 2   # both exceptions apply -> conflict
```

Fix: order them — make one an exception *of the other* (`label` the first
exception, then `exception <that label>`), or tighten the conditions so they are
disjoint. **Do not "fix" this by deleting a clause**; if the two clauses come
from different source documents, this error *is* the finding.

### 9.2 Ambiguous unlabelled exception

```
This exception can refer to several definitions. Try using labels to
disambiguate.
```

Cause: bare `exception definition x ...` when `x` has more than one other
definition. Fix: add `label L` to the intended base and write `exception L`.

### 9.3 Unknown label

```
Unknown label for the scope variable x: "base_y".
```

Cause: `exception base_y definition x ...` where `base_y` labels a definition of
a *different* variable. Labels are scoped to the variable they define. Fix: point
at a label on the same variable.

### 9.4 Exception cycle

```
Exception cycle detected when defining x:
each of these 3 exceptions applies over the previous one, and the first
applies over the last.
```

Fix: break the cycle — the exception graph must be a DAG with a base.

### 9.5 Incompatible types

```
Incompatible types:
  got      decimal,
  expected integer
```

By far the most common error. Catala never coerces silently. Fix: insert the
explicit conversion (`decimal of`, `money of`, `round of`), or use the
suffixed operator (`+!`, `+.`, `+$`, `+^`). The error reports both the offending
expression and the declaration the expected type came from.

### 9.6 No applicable rule

```
During evaluation: no applicable rule to define this variable in this situation.
```

Cause: every definition of the variable was conditional and none of the
conditions held. Fix: supply an unconditional base case. A scope whose base case
is missing is almost always a misread of the clause — the law nearly always has
a default.

### 9.7 Cyclic dependency between scope variables

```
Cyclic dependency detected between the following variables of scope A:
z → x → y → z
```

Fix: Catala has no recursion and no fixpoints; restructure so the dependency is
acyclic.

### 9.8 Wildcard not last

```
Wildcard must be the last match case.
```

Fix: move `-- anything :` to the end.

### 9.9 Function argument name mismatch

```
Function argument name mismatch between declaration ('x') and definition ('y').
```

Fix: use the declared parameter names verbatim in `definition f of ...`.

### 9.10 Ambiguous duration comparison

```
During evaluation: ambiguous comparison between durations in different
units (e.g. months vs. days).
```

Fix: months and days are not commensurable. Normalise to one unit, or compare
dates instead of durations.

### 9.11 Other errors seen in the suite

| Message | Fix |
|---|---|
| `This subscope variable is a mandatory input but no definition was provided.` | define every `input` of the subscope before calling it |
| `Variable a is not a declared output of scope A.` | mark it `output` in the declaration |
| `The name of this constructor has not been defined before` | typo, or a missing `> Using` |
| `Duplicate constructor X2 in enumeration E3` | rename |
| `attempting to compare values with uncomparable types` | insert a conversion |
| `a value is being used as denominator in a division and it computed to zero` | guard the divisor with a condition |
| `In this function definition, the type <t> is specified as anything, not fully known.` | annotate the type |
| `Type Mod_def1 is private and cannot be used here.` | export it via a `catala-metadata` block |
| `» expected 'under condition' followed by a condition, 'equals' followed by ...` | parse error: a `definition` is missing its body |

---

## 10. Three golden examples

Verbatim from the pinned tree. These are the patterns to imitate.

### 10.1 Exception groups — `tests/exception/good/groups_of_exceptions.catala_en`

A base case split across two conditions, an intermediate group overriding it,
and a further group overriding that. This is the shape most legal rules take.

```catala
declaration scope Foo:
  input y content integer
  output x content integer

scope Foo:
  label base definition x under condition
    y = 0
  consequence equals 0

  label base definition x under condition
    y = 1
  consequence equals 1

  label intermediate exception base definition x under condition
    y = 2
  consequence equals 2

  label intermediate exception base definition x under condition
    y = 3
  consequence equals 3

  exception intermediate definition x under condition
    y = 4
  consequence equals 4

  exception intermediate definition x under condition
    y = 5
  consequence equals 5

#[test] declaration scope Test:
  f scope Foo
  output x content integer

scope Test:
  definition f.y equals 2
  definition x equals f.x
```

```
$ catala test-scope Test
┌─[RESULT]─ Test ─
│ x = 2
└─
```

Note the reuse of `label base` on two definitions: that makes them one group,
which a single `exception base` can override wholesale.

### 10.2 Module definition and use — `tests/modules/good/mod_def.catala_en`, `mod_use.catala_en`

```catala
> Module Mod_def

declaration enumeration Mod_def:
  -- Yes
  -- No
  -- Maybe
  -- Something content integer

declaration structure Str1:
  data fld1 content Mod_def
  data fld2 content integer

#[test] declaration scope S:
  output sr content money
  output e1 content Mod_def

declaration half content decimal
  depends on x content integer
  equals x / Date.get_year of |0002-01-01|
```

```catala
scope S:
  definition sr equals $1,000
  definition e1 equals Maybe
```

Consumed from another file:

```catala
> Using Mod_def

#[test] declaration scope T2:
  t1 scope Mod_def.S
  output o2 content Mod_def
  output o4 content decimal

scope T2:
  definition o2 equals t1.e1
  definition o4 equals Mod_def.half of 10
  assertion o2 = Maybe
  assertion o4 = 5.0
```

### 10.3 A conflict, caught — `tests/exception/bad/two_exceptions.catala_en`

The failure mode the ingest gate depends on. Two exceptions over the same base,
both unconditional:

```catala
declaration scope A:
  output x content integer

scope A:
  label base_x
  definition x equals 0

  exception base_x
  definition x equals 1

  exception base_x
  definition x equals 2
```

```
$ catala test-scope A
┌─[WARNING]─
│  Multiple conflicting definitions:
│  these have the same conditions and will always trigger a conflict at runtime.
├─➤ two_exceptions.catala_en:12.14-15:   definition x equals 1
├─➤ two_exceptions.catala_en:15.14-15:   definition x equals 2
┌─[ERROR]─
│  During evaluation: conflict between multiple valid consequences for
│  assigning the same variable.
#return code 123#
```

Both conflicting spans are in the output. Parse them out and you have the two
clauses to surface.


---

## 11. Legal encoding traps (verified against the compiler)

Each of these was reproduced on this box. Every one typechecks clean.

### 11.1 Never compare a date difference to a month/year duration

```catala
# WRONG — typechecks, then dies at runtime
definition valid equals termination_date - notice_given >= 3 month
#   During evaluation: ambiguous comparison between durations in different units

# RIGHT — date arithmetic
definition valid equals notice_given + 3 month <= termination_date
```

`date - date` yields days; `3 month` is months; months are not a fixed number of
days so Catala refuses. "Three months' notice" is the most common clause in
commercial contracts — get this wrong once and the module is unusable.

### 11.2 Month/year arithmetic needs an explicit rounding mode

`|2026-01-31| + 1 month` is a runtime error until the scope declares
`date round down` or `date round up`. The two disagree:

| | result |
|---|---|
| `date round down` | `2026-02-28` |
| `date round up` | `2026-03-01` |

Same for age: someone born `2008-02-29` attains 18 on `2026-02-28` or
`2026-03-01`. **The source document almost never says which.** Whatever you pick
is a legal ruling that is not in the text — record it as an explicit assumption,
never bury it.

### 11.3 `List.sequence` is off by one against its docs  **[FIXED IN OUR BUILD: docstring corrected]**

The docstring in `stdlib/list_en.catala_en` says
`sequence of 3, 6 = [ 3; 4; 5; 6 ]`. The implementation returns `[3; 4; 5]`, and
`sequence of 3, 3 = []` — it is half-open `[begin, end)`. Upstream's own recorded
test (`tests/stdlib/list.catala_en`) confirms the half-open behaviour, so **the
documentation is wrong, not the code.**

Consequence: any period-based computation written from the docs is short by one
period, silently. Verified: $1,000 at 5% compounded over 10 years came out as
$1,551.34 (= 1.05⁹) instead of $1,628.89.

```catala
# use an inclusive range explicitly — note the parentheses, see below
combine all y among (List.sequence of 1, (years + 1))
in acc initially principal with acc * 1.05
```

Always unit-test an iteration count before trusting it.

Two things bite here:

**`of` binds looser than arithmetic.** Written without the inner parentheses,
`List.sequence of 1, years + 1` parses as `(List.sequence of 1, years) + 1` and
fails with `I don't know how to apply operator + on types list of integer and
integer` — an error that points at the whole expression, not the precedence.
**Parenthesise every compound argument to `of`.**

**Rounding drift accumulates over iterations.** Even correctly ranged, the loan
above returns **$1,628.91** against an exact $1,628.894…, because money is
rounded to cents at each of the ten steps (§11.4). For statutory interest,
compute in `decimal` and convert to `money` once at the end.

### 11.4 Money rounds to cents; apportionment does not conserve

`$100 / 3.0` → `$33.33`, and `× 3.0` back → `$99.99`. One cent vanishes with no
warning. Where a statute requires the whole fund to be distributed, compute the
remainder explicitly and assign it.

### 11.5 There is no unit system and no currency

`45.0 hours + 250.0 square_metres` = `295.0`. `$1,000 (EUR) + $200 (USD)` =
`$1,200.00`. `money` is a single untagged type. Encode units in the *variable
name* and add an `assertion` where you can; the typechecker will not help.

### 11.6 There is no `string` type

`content string` → `Unknown built-in type`. Every textual category (licence
class, jurisdiction, reason code) must become an `enumeration`, fixed at encoding
time. Upside: a `match` that misses a constructor is a **compile error**, whereas
a missing `under condition` branch is only a runtime one. Prefer enums over
integer/boolean condition codes for exactly this reason.

### 11.7 A term cannot be redefined per Part

Two `declaration structure Employee` in one file → `struct name "Employee"
already defined`, even when the Act genuinely redefines the term "for the
purposes of this Part". Suffix the type per part (`Employee_Part1`) and record
the mapping, because the identifier no longer matches the legal term.

### 11.8 An exception may override exactly one label

"Notwithstanding sections 3 and 7" cannot be written. Both
`exception s3, s7` and two stacked `exception` lines are syntax errors. The only
option is to give s.3 and s.7 the *same* label and override the group — per-rule
headings and conditions survive in the exception tree, but the two sections are
now structurally one group and cannot be overridden separately later.

### 11.9 Block quotes break the parser

A line beginning with `>` is a module directive. A judgment or memo that quotes
statute the normal Markdown way is a **parse error**, and it cascades into a
misleading `Unclosed block or missing newline at the end of file`.

**Indent quoted text by two spaces** — that parses fine. Tables, numbered and
lettered lists, footnotes, horizontal rules, `#` inside prose and inline triple
backticks were all verified safe.

### 11.10 `context` variables are never elicited

A `context` variable (the encoding of "unless the instrument specifies
otherwise") appears in the scope's JSON schema `properties` but **not** in
`required`. Elicitation driven by `required` alone will silently apply the
statutory default: verified, a 5% default returned $1,050 where the lease's 12%
gives $1,120. **Ask about `properties` minus `required` too**, flagged as
"defaults to X unless the document says otherwise".

### 11.12 Use `impossible` for cases the instrument excludes

For "this Act does not apply to X", an unhandled case gives the opaque
`no applicable rule to define this variable`. Prefer an explicit branch:

```catala
-- NonResident : #[error.message = "non-residents are out of scope of this Act"] impossible
```

which fails with the legal reason attached:

```
During evaluation: "impossible" computation reached.
non-residents are out of scope of this Act
```

That message is something the chat layer can show a user. Note `catala proof`
does **not** flag a reachable `impossible` — only tests find it.

### 11.11 Getting the exception direction backwards is undetectable

If the encoder makes the original an exception to the amendment instead of the
reverse, a 2025 claim returns the repealed $200 rather than $260. Typecheck
clean, no conflict, and `clerk exceptions` renders a coherent tree — so the chat
layer produces a **fully cited, confidently wrong** answer. No tool catches this.
Only comparison of artifact against source text does.

---

## 12. `catala proof` — static conflict and gap detection

Requires the proof backend: `opam install z3`, then `dune build
compiler/plugins/`, `dune build catala-proof.install --promote-install-files`,
`opam install ./catala-proof.opam --working-dir --assume-built`.

```sh
catala proof FILE            # whole file
catala proof FILE -s SCOPE   # one scope
```

It discharges exactly the two verification conditions in
`compiler/verification/conditions.mli` — `NoOverlappingExceptions` and
`NoEmptyError` — **and returns a counterexample**:

```
[Toll.charge] At least two exceptions overlap for this variable:
The solver generated the following counterexample to explain the faulty behavior:
--> tonnes : 11

[Gap.rate] This variable might return an empty error:
--> band : 0
```

This is the single most valuable tool for ingestion. It turns conflict detection
from *coverage-dependent* into *static*: two tolls overlapping only on
`10 < t < 20` were found with no test written, and one overlap injected into a
100-statute corpus was located in **0.43 s**. It handles money, dates, and lists
(`--> xs : (length = 0)`), and reported no false positives on correct modules.
Feed each counterexample straight into a `clerk test` case.

**Two operational cautions.**

1. **Exit code 0 even when a defect is proven, and all output goes to
   stderr.** **[FIXED IN OUR BUILD: `--fail-on-unproven`]** — our build adds an
   opt-in flag that returns 123 when any VC is undischarged, so `proof` can be a
   CI gate (default behaviour unchanged, so upstream's 47 proof tests still
   pass). The stderr point stands regardless: Verified: on a file with a proven overlap, `$?` is 0 and *stdout is
   0 bytes* — every finding is on **stderr**. A gate must do
   `catala proof FILE 2>&1 >/dev/null | grep -q WARNING`. Redirecting the way
   you normally would (`> out.txt`) silently captures nothing.

   **Constrain inputs with assertions or the counterexamples are nonsense.**
   Unconstrained, the solver offered `area : -51, value : -1`. Adding
   `assertion area >= 0.0` etc. moved it to `area : 51, value : 1` — a fact
   pattern worth turning into a test. Nonlinear conditions (products of two
   inputs) were handled fine, in 0.03 s.

   Scaling is practical: 2000 scopes proved in 9.8 s, a 500-deep exception
   chain in 0.10 s.
2. **It proves only those two conditions.** Verified *not* caught — proof mode
   reports "No errors" for all of these:

   | Defect | Still fails how |
   |---|---|
   | months-vs-days comparison (§11.1) | runtime error |
   | missing date rounding mode (§11.2) | runtime error |
   | division by zero | runtime error |
   | money rounding drift (§11.4) | silent wrong answer |
   | reversed exception direction (§11.11) | silent wrong answer |
   | contradiction across *separate scopes* | invisible by construction |
   | a reachable `impossible` branch (§11.12) | runtime error |

### Python backend  **[FIXED IN OUR BUILD]**

Stock `nightly-137` emits Python packages that cannot be imported at all:
`__init__.py` contains `__all__ = [Fragile]` (bare identifiers, not strings →
`NameError`), and generated modules do `from . import Stdlib_en` although the
stdlib is installed as the separate `libcatala` package (→ `ImportError`). Both
are fixed. With them fixed, generated Python **agrees with the interpreter to
the cent** across compound interest, division and rounding — verified on five
input sets, so there is no interpreter/codegen divergence to worry about.

### The scope-locality limit

Conflict detection — by test **or** by proof — only sees definitions of the
**same variable in the same scope**. Two statutes ingested as separate scopes
(`FeeA` = $500, `FeeB` = $750) coexist with no diagnostic of any kind. So
"compile each document into its own module" defeats contradiction detection
entirely. Contradicting clauses must land in the *same scope* as competing
definitions, which means the pipeline has to decide that two clauses are about
the same thing **before** Catala can help. Catala does not solve that step.


---

## 13. Silent-drop hazards — the clause that isn't there

These are the worst failures in the toolchain, because the artifact looks
healthy. All verified.

### 13.1 Only two fence labels are live  **[FIXED IN OUR BUILD: now warns]**

| fence | compiled? |
|---|---|
| ```` ```catala ```` | **yes** |
| ```` ```catala-metadata ```` | **yes** |
| ```` ```catala_en ```` | **no — silently prose** |
| ```` ```catala-en ```` | **no** |
| ```` ```Catala ```` / ```` ```CATALA ```` | **no** |
| anything else, or bare ```` ``` ```` | **no** |

` ```catala_en ` matches the *file extension*, so it is an entirely natural
thing to write — and it produces **zero diagnostics**. An entire statute plus
its tests placed in such a block gave:

```
catala typecheck  ->  Typechecking successful!     (exit 0)
catala proof      ->  No errors found              (exit 0)
clerk test        ->  NO TESTS WERE RUN            (exit 0)
```

The law simply is not in the program, and nothing says so.

**Mitigation — assert liveness after every ingest.** `catala scopelang` lists
what actually compiled:

```sh
catala scopelang FILE | grep -oE 'let scope [A-Za-z_0-9]+'
```

Compare that against the scopes the generator intended to emit. Never treat
"typecheck OK" as evidence the code was read.

### 13.2 `#[tests]` compiles and never runs  *(warns already; treat warnings as errors)*

`#[test]` runs; `#[ test ]` runs; `#[Test]` is a **compile error**; but
`#[tests]` yields only `[WARNING] Unrecognised attribute "tests"`, exit 0, and
the test silently does not run. Treat warnings as errors on ingest.

### 13.3 Test blocks have the same hazard  *(still silent — assert the `--xml` count)*

Only the exact label ```` ```catala-test-cli ```` runs. `catala-test`,
`catala-cli`, `catala_test_cli`, `Catala-test-cli`, `catala-test-CLI` all give
`NO TESTS WERE RUN`, exit 0, no diagnostic — and a deliberately wrong expected
value in such a block is never checked. The permanent counterexample ledger can
silently hold zero live tests.

**Mitigation — assert the test count.** `clerk test --xml` emits JUnit:

```xml
<testsuites tests="2" failures="0">
```

Gate on `tests` being greater than zero and equal to what you expect.
`--json` (VSCode format) is also available.

### 13.4 Coverage  **[FIXED IN OUR BUILD: verbose percentage corrected]**

`clerk test --code-coverage` reports line coverage, and because each exception
branch occupies its own lines it is a decent proxy for branch coverage — one
test on a three-branch module gave `3 / 13` uncovered (76%), and covering every
branch gave 100%.

**Use the summary table, never the `--verbose` line.** The per-file verbose
output divides by the wrong denominator
(`build_system/clerk_report.ml:410-414` uses `total_unreached_lines` where it
means `total_reachable_lines`):

|actual | summary table | `--verbose` line |
|---|---|---|
| 8 / 10 | `80 %` | `8 / 10 lines (400 %)` |
| 10 / 13 | `76 %` | `10 / 13 lines (333 %)` |
| 14 / 14 | `100 %` | `14 / 14 lines (-1 %)` |

At full coverage it divides by zero and `int_of_float infinity` yields **-1**.
A gate parsing that line would reject perfect coverage and wave through partial.

### 13.5 The gate matrix

Exit codes, one document per isolated project:

| defect | `typecheck` | `proof` | `clerk test` |
|---|---|---|---|
| clean module | 0 | 0 | 0 |
| statute vanished (§13.1) | 0 | 0 | **0** |
| unrecognised attribute (§13.2) | 0 | 0 | **0** |
| contradiction across scopes (§12) | 0 | 0 | **0** |
| proven exception overlap | 0 | **0** | 1 |
| proven statutory gap | 0 | **0** | 1 |

`catala typecheck && catala proof && clerk test` **passes all six.** A usable
gate needs: exit codes, *plus* `proof` stderr grepped for `WARNING`, *plus* a
`scopelang` liveness assertion, *plus* warnings-as-errors.

### 13.6 clerk is project-wide

`clerk test <one-file>` builds and tests the **whole project**, not the named
file. One malformed document anywhere in the corpus fails every subsequent
`clerk` invocation, whatever file you name. Ingest candidates should be
validated in an isolated directory before being added to the corpus.

---

## 14. Encoding robustness (verified)

Safe in **prose**: CRLF line endings, a UTF-8 BOM, non-breaking spaces, smart
quotes, em-dashes, `§`, soft hyphens, tabs, tables, numbered and lettered lists,
footnotes, horizontal rules, `#` mid-prose, inline triple backticks, headings
seven deep, duplicate headings, no headings at all, and a fenced block indented
inside a list item (which *does* compile).

Unsafe **inside a code block**: U+2212 MINUS SIGN, en dash, and fullwidth `＄`
are parse errors — but non-breaking space is accepted as whitespace. The
diagnostic is poor: it underlines the *space before* the character, says
`Parsing error after token " "`, never mentions an unexpected character, and
then adds a spurious `Unclosed block or missing newline at the end of file. Did
you forget a ``` delimiter ?`. The rendered line looks identical to valid code,
so a repair loop will regenerate it unchanged or start chasing fences. **Strip
Unicode punctuation to ASCII before the typecheck step.**

Parser performance is a non-issue: 22,000 lines / 2,000 sections in 0.19 s, 500
stacked exceptions in 0.03 s, 400-deep nested conditionals in 0.02 s.


---

## 15. Structural failures that are loud (and one that isn't)

Verified as clean, blocking diagnostics — safe to rely on:

| situation | result |
|---|---|
| duplicate scope name | `scope name "S" already defined` |
| duplicate struct name (term redefined per Part) | `struct name "Employee" already defined` |
| two files claiming one module name | `Conflicting module name Same` |
| `> Module Foo` in a file not named `Foo` | `Module declared as Foo, which does not match the file name` |
| `> Include:` a missing file | `Included file '…' is not a regular file or does not exist` |
| `> module` / `> using` (lowercase), `> Modul` | parse error |
| non-exhaustive `match` over an enum | compile error, exit 123 |

**Circular includes — [FIXED IN OUR BUILD].** Two documents that `> Include:`
each other used to recurse until file descriptors ran out
(`System error: cyc2.catala_en: Too many open files`, exit 125). Three separate
traversals needed guarding (the compiler's `expand_includes`, clerk's
`count_test_scopes`, and `clerk_rules.inclusion_map`, which looped forever
rather than crashing). Our build reports:

```
Circular file inclusion: 'b.catala_en' is already being included further up
the '> Include' chain.                                        (exit 123)
```

Note `clerk` still refuses a cyclic project with ninja's own
`dependency cycle` error, and because clerk is project-wide (§13.6) one cyclic
document blocks the whole corpus — so screen for cycles at ingest anyway.

**`clerk test --reset` is safe on prose.** It rewrites only the
`catala-test-cli` block. Verified byte-identical preservation of accented text,
guillemets, `§`, non-breaking spaces, soft hyphens, tabs, trailing whitespace
and blank runs — so recording a counterexample never disturbs the authoritative
clause text above it.


---

## 16. Second-wave findings

### 16.1 Scope variables are evaluated eagerly — one bad output breaks every query

A scope computes **all** its variables, not just the ones the caller reads. A
module with outputs `n`, `total`, `max` and `average`, queried on an empty list
for `n` alone, failed with

```
During evaluation: a value is being used as denominator in a division and it
computed to zero.
```

because `average` divides by `number of xs`. Deleting the unused `average`
made the identical query return `n = 0, total = $0.00, max = $0.00` correctly.

For a chat product this means **a question that is perfectly answerable fails
because of an unrelated output in the same scope.** Guard every partial
computation (`under condition (number of xs) > 0`) even when the caller will
never read it — or split fragile derivations into their own scope.

### 16.2 A clause that can never apply is invisible

An exception whose condition is unsatisfiable — `x > 100 and x < 50`, a
plausible mis-encoding of "over 100" plus "under 50" from two sub-clauses — is
reported by **nothing**. `catala typecheck` is silent; `catala proof` says
`No errors found`. Answers quietly fall through to the base case.

The solver could decide this trivially, but the VC set is only
`NoOverlappingExceptions` and `NoEmptyError`. The **only** detector is coverage:

```
clerk test dead.catala_en --code-coverage
   Lines covered    1    14    15    93 %      <- the 1 uncovered line is the dead clause
```

Two tests either side of the threshold both passed and both returned the base
case. **Require 100% line coverage on every ingested module** — anything less is
either a missing test or a clause that can never fire. This is also the only
detector for a reachable `impossible` branch (§11.12), which proof likewise
ignores.

### 16.3 The verified conflict-surfacing recipe

`catala proof` names the *variable* and gives a counterexample, but with three
or more clauses on one variable it does **not** say which pair conflicts. Feed
the counterexample back to get them:

1. `catala proof FILE 2>&1 >/dev/null | grep -A9 counterexample`
   → `--> turnover : 1000000.01 $`
2. Write that as a `#[test]` scope.
3. `catala interpret FILE -s ThatScope` → the runtime conflict error names
   **exactly** the two clauses with positions and headings — verified to pick
   out Act B and Act C while correctly excluding Act A.
4. Record it with `clerk test --reset`; it is now a permanent case.

### 16.4 Identifier character set

Accepted: Latin with diacritics (`Société`, `impôt`, `Ustawał`, `Gesetzß`,
`İstanbul`), **Cyrillic** (`Закон`) and **Greek** (`Νόμος`).

Rejected: **CJK** (`法律`), **Arabic/RTL** (`قانون`), emoji, and a combining
accent (`cafe` + U+0301, as against precomposed `café`) — so homoglyph collisions
are impossible, but non-Latin-script corpora outside Cyrillic/Greek cannot be
encoded with native-language identifiers.

A **zero-width space** inside an identifier is rejected, but the message is
`Parsing error after token "a"` on a line that looks perfectly normal. Strip
U+200B along with the other invisibles (§14).

### 16.5 Robustness confirmed — no findings

These were probed and behaved correctly, so they need no special handling:

- **PDF/Word conversion noise in prose**: form feeds, vertical tabs, NUL and
  control bytes, `ﬁ`/`ﬂ` ligatures, hyphenation across line breaks, page
  headers mid-clause, margin line numbers, footnote daggers, mixed CRLF, and a
  single 40 KB line — all parse, scope stays live.
- **Boundary values**: `|0001-01-01|` to `|9999-12-31|`, `$0.01`, `-$0.01`,
  `1000000 day`. An invalid calendar date `|2023-02-29|` is a **compile
  error** ("does not correspond to a correct calendar day").
- **Bad references**: `Field "x" does not belong to structure "S"`,
  `No scope named X found`, `Variable y is not a declared output of scope A` —
  all precise.
- **Deep nesting**: a 30-level struct chain and 9-deep tuples typecheck.
- **Error recovery**: three independent syntax errors in three blocks are all
  reported in one run (`ERROR 1/4 … 3/4`), so the repair loop sees them together.
- **Feature interactions**: a `state` variable carrying an exception hierarchy,
  a `rule` (condition variable) with its own exception, and a `context`
  variable overridden by the caller *flipping* which exception branch fires —
  all compose correctly ($0 / $18,000 / $12,000 across three fact patterns).
