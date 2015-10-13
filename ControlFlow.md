### Goals

 - Provide a natural compilation target for optimizing compilers. The proposal
   here lets compilers emit code that has a lot less redundant branching that
   has to be optimized away to expose what the code is really doing.

 - Simplify the task of baseline JITs and other simple consumers. With less
   branching to optimize away, baseline JITs can be simpler.

 - Simplify the task of debuggers and related tools, both at the WebAssembly
   level, and at the high-level source language level.

 - Make it possible to have a simple lossless bidirectional mapping between
   the AST form and a linear stack-machine-like language. While many users
   won't need this, it's a form that is very natural for some low-level
   tools.

These goals align with WebAssembly's [High Level Goals](HighLevelGoals.md),
specifically "to serve as a compilation target", and to produce an
"MVP [...] primarily aimed at C/C++".

They further facilitate the goals to "build a new LLVM backend for WebAssembly
and an accompanying clang port", "promote other compilers and tools targeting
WebAssembly", and "enable other useful tooling".

### An end-to-end example

Consider this C code:
```
   for (i = 0; i < n; ++i)
       y[i] = a * x[i] + y[i];
```

This is *the* hot loop in some applications. And for the purposes of this
section, it's a broadly representative loop.

The high-level optimized LLVM IR for this code looks like this:
```
  %cmp.9 = icmp sgt i32 %n, 0
  br i1 %cmp.9, label %for.body.preheader, label %for.end

for.body.preheader:
  br label %for.body

for.body:
  %i.010 = phi i32 [ %inc, %for.body ], [ 0, %for.body.preheader ]
  %arrayidx = getelementptr double, double* %x, i32 %i.010
  %0 = load double, double* %arrayidx, align 8
  %mul = fmul double %0, %a
  %arrayidx1 = getelementptr double, double* %y, i32 %i.010
  %1 = load double, double* %arrayidx1, align 8
  %add = fadd double %mul, %1
  store double %add, double* %arrayidx1, align 8
  %inc = add i32 %i.010, 1
  %exitcond = icmp eq i32 %inc, %n
  br i1 %exitcond, label %for.end.loopexit, label %for.body
```

The key thing to notice here is that while the C semantics for a for loop say
that the loop condition is tested at the top, `n + 1` times, this code does one
check outside the loop, so that the loop itself only iterates `n` times. LLVM
has duplicated the loop condition to arrange this, an optimization it calls
"loop rotation".

This isn't an obscure optimization; LLVM does this to all loops it can, and
considers this form to be a kind of canonical form for loops:

 - Optimizations that reason about the trip count work out a lot nicer when
   the trip count is `n` rather than `n + 1`.
 - Code motion optimizations like LICM have a place to hoist expressions to
   outside the loop -- in the `for.body.preheader` block in this example, which
   is control-equivalent to executing the loop body at least once.
 - This form is closest to what the compiler will eventually want to emit in
   machine code. By moving the branch to the bottom of the loop, we eliminate
   an unconditional branch back to the top, making the loop one instruction
   shorter.

Also, this isn't just an LLVM thing; GCC does the same thing (where it's called
"loop header cloning"), and I'm familiar with several other C/C++ compilers
that do this too.

#### The status quo

First, consider how LLVM will compile this to WebAssembly as currently
defined by ml-proto. `loop` is defined to be an infinite loop, from which one
exits by using an explicit break. Using WebAssembly as it is currently defined,
the following is the best WebAssembly code LLVM could possibly produce:

```
   if (n > 0) {
     loop {
       y[i] = a * x[i] + y[i];
       i = i + 1;
       if (i == n) break; // conditional branch over unconditional branch...
     } // branch back to top
     // ... to here
   }
```

This achieves the desired semantics. But now instead of one branch inside the
loop, we have three, as indicated in the comments.

It is of course not hard for a clever JIT to optimize this back down to a
one-branch loop. However, for baseline JITs that just want to emit code
quickly, this is extra work that's going to have to be done for every loop
that works this way, which given how common this pattern is in C++ compilers,
is a lot. Also, this is just a simple example; more complicated examples
happen too, especially when `tableswitch` is involved.

It's also suboptimal for debuggers, profilers, code coverage, and other tools.
Consider someone single-stepping through this code at the WebAssembly level.
Should stepping through an `if(x) break` be one step or two? Should the loop
backedge be a separate step? As things stand right now, tools are going to
want to do some work to present the developer with view of their code with
"obvious" optimizations applied, and then translate back and forth between
that view and the actual code.

Things are also complicated for high-level source language debugging. The task
of mapping from WebAssembly code back to high-level code is complicated in the
presence of implicit branches and instructions that evoke effects at multiple
locations.

#### A quick look at my proposal

Next, consider the WebAssembly code we could get for this same loop if we
use my proposal. In this proposal, the `loop` statement doesn't implicitly
branch, and there's a conditional branch instruction (`br_if`) which does
exactly what we want:

```
   if (n > 0) {
     loop {
       y[i] = a * x[i] + y[i];
       i = i + 1;
       br_if top (i != n); // conditional branch
     }
   }
```

With this code, the literal semantics has the same number of branches as
the machine code has, which is the same number of branches LLVM thought it
had. Baseline JIT compilers can just translate this one operation at a time
and it'll be fast.

Debuggers, profilers, and code coverage tools can consider every branch to
be a literal branch that a WebAssembly-level developer might care about,
and there's no need to optimize or to translate between literal and optimized
representations (other to apply obvious syntax sugarings, but those are
conceptually simpler).

And, mappings to high-level languages are easy to describe with this proposal,
because every construct that "does" something is an instruction which can
have a location, and since it only does one thing, it only needs one
location.

### Proposed AstSemantics text (brief descriptions)

 * `block`: a fixed-length sequence of statements with a label at the end
 * `loop`: a fixed-length sequence of statements with a label at the end
           and a loop header label at the top
 * `if`: an if statement with a label at the end and a body containing an
         optional else label in the middle
 * `br`: branch to a given label in an enclosing construct (see below)
 * `br_if`: conditionally branch to a given label in an enclosing construct
 * `tableswitch`: a jump table transferring which may jump either to enclosed
                  `case` blocks or to labels in enclosing constructs
 * `case`: must be an immediate child of `tableswitch`; has a label declared
           in the `tableswitch`'s table and a body

References to labels must occur within an *enclosing construct* that defined
the label. This means that references to an AST node's label can only happen
within descendents of the node in the tree. For example, references to a
`block`'s label can only happen from within the `block`'s body. In practice,
one can arrange `block`s to put labels wherever one wants to jump to, except
for one restriction: one can't jump into the middle of a loop from outside
it. This restriction ensures the well-structured property discussed below.

### Big-Picture Ideas

In this proposal:

 - Control flow is *well-structured*. This means:
    - No multiple-entry loops.
    - Control flow is a tree structure which fits naturally into the wasm AST.
    - Statements=Expressions works naturally if we choose to do that.
    - Fast SSA construction and type checking algorithms that work on ASTs
      with labeled break and continue (i.e. JS) work, because the control flow
      here follows the same patterns.

 - There is some high-level structure, and key low-level simplifying invariants
   are preserved as well.

 - There is no implicit branching.

 - Every instruction that has side effects, including control transfers, does
   everything it does together, in one place.

### Grammar (pseudo-code)

```
"(" "block" label_opt expr expr_list ")"

"(" "loop" label_opt label_opt expr expr_list ")"

"(" "if" expr label_opt label_opt expr_list else_opt ")"

else_opt :
 | /* empty */
 | label expr_list

"(" "br" label ")"

"(" "br_if" label expr ")"

"(" "tableswitch" expr label label_list label case_list ")"

"(" "case" label expr_list ")"

```

Note that this omits support for
[Statements=Expressions](https://github.com/WebAssembly/design/issues/223),
however this is only for brevity; such support could naturally be re-introduced.

### More detailed descriptions

#### `block`

It's just a non-empty list of statements with a label at the end.

#### `loop`

`loop` is identical to `block` except that it also has a label at the top that
can be branched back to, making a loop.

Note that this means that `loop` does not automatically branch from the bottom
to the top. It falls through by default.

#### `if`

A plain `if` is a fairly normal-looking if statement. When it has an else arm,
it differs from a high-level if statement in that there is an explicit branch
to branch past the else body, and there's a label defining the beginning of
the else body.

#### `br`

`br` branches to a label in an enclosing construct, such as a `block`, `loop`,
`if`, or other construct. As with all other control transfers, it can't branch
anywhere else; this constraint keeps control flow well-structured.

#### `br_if`

This is the conditional-branch form of `br`. It has an extra expression operand
which determines at runtime whether the branch is taken.

#### `tableswitch`

This is conceptually a jump table. It has an array of labels, indexed starting
from 0, and an index operand, and it selects which label to branch to with that
index. When the index is out of bounds, it has a default label as well that it
uses in that case.

Labels may either be references to labels in the enclosing scope, or
declarations of labels of case blocks enclosed within the body. The body of a
`tableswitch` can contain only `case` blocks. Control falls through between the
`case` blocks.

#### `case`

`case` statements can only appear as immediate children of `tableswitch`
statements. They have a label, which must be declared in the `tableswitch`
statement's table, and a body. Control falls through the end of a `case`
block into the following `case` block, or the end of the `tableswitch` in
the case of the last `case`.

### Examples.

I've collected several example sketches [here](ControlFlowExamples.md).

### Questions?

I've collected answers to common questions [here](ControlFlowFAQ.md).

### Differences relative to the current ml-proto code

Relative to what's currently in ml-proto, this is somewhat lower-level in feel.
In particular, it has the property that there are no implicit branches. However,
it still has a higher-level feel than the previous `br_if`/`br_switch`
proposals.

`tableswitch` is generalized to be able to branch to enclosing block labels,
in addition to enclosed cases. This is follows LLVM's
[`switch`](http://llvm.org/docs/LangRef.html#switch-instruction) which also
works in essentially the same way, and the flexibility is useful so that
we don't need to generate branch-to-branch constructs.

Cases fall through by default, and the override is to add a `br`. While
everyone agrees that this is a terrible default for humans writing high-level
language code, it's simpler for compilers generating code, and it simplifies
the language.

I took the liberty of converting `tableswitch` to use a zero-indexed dense
table instead of a sparse list of cases. I believe this idea has been
discussed in several circles and is generally well-received. However,
this proposal could work either way, so if this is unpopular, I can
change it back.

### Differences relative to the earlier `br_if`/`br_switch` proposal

It re-introduces an `if` construct and re-merges `br_switch` and `switch`
(and renames it to `tableswitch` to emphasize the table nature) which
eliminates the need for deeply nested block stacks, making the polyfill
story simpler, and eliminating problems with strictly recursive parsers.
It also gives the proposal a higher-level feel overall.

It will take more work in LLVM to fully utilize this proposal, but if it
gets a favorable response, I'm prepared to do this work, and I believe
it'll deliver the comparable benefits.

### Deferred Ideas

These are things that aren't included in the current proposal. They're
interesting, but they have unresolved questions, and they're not worth bogging
down the rest of the discussion right now. But they are interesting to note, for
people interested in the topic.

 * `br_unless`: it's handy for simple low-level tools to be able to invert
   branches, as by just changing an opcode from `br_if` to `br_unless` or back.
   But it's not game-changing.

 * Remove the default from `tableswitch`, or perhaps just make it optional. It
   would be more conceptually clean from a certain perspective to have
   `tableswitch` trap if it gets an index which is out of bounds on its table.
   Application code can do its own range check and branch if it needs to. But
   it's more work to optimize on present-day platforms.
