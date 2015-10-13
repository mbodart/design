### FAQ

Q. Why have `br_if` when we have both `if` and `br` already?

A. Conditional branches are a very common pattern, and this reduces the burden
   of pattern-matching in baseline JITs, since all popular hardware platforms
   have conditional branch instructions.

   Also, if one is aiming to do a conditional branch, `br_if` is conceptually
   simpler than `if` with `br` because the latter is a branch-over-a-branch.

Q. Why is `if` not just a ternary AST node?

A. That would require an implicit branch over the else arm; avoiding implicit
   branching is one of the goals of this proposal. Also, a ternary if statement
   would not have localized semantics; it would need to start by doing a
   conditional branch, stick around for a while, and then perform another
   branch. This proposal is seeking to make every branch an explicit construct.

Q. Why merge `break` and `continue`?

A. Because there isn't a great need for the distinction. Also, `tableswitch`
   statements want to be able to branch to any enclosing construct label. It'd
   be confusing to make the distinction for branches but not jump tables.

Q. Why rename `break` to `br`?

A. To reflect that this operator is the generalization of not just `break` but
   `continue` too, so it's closer to a general-purpose branch. `br` is a common
   abbreviation for branch in similar languages.

Q. `block` and `loop` are very similar; why not merge them?

A. Because it's useful to tell WebAssembly consumers up front where the loops
   are. This lets even very simple consumers do some loop-aware optimizations,
   and lets somewhat more sophisticated consumers save some compile time. And,
   it should make code easier to pretty-print.

Q. Why not merge `br` and `br_if`?

A. We could, by having `br` be `br_if` with a constant condition, but `br` is
   a fairly common operation and it seems more convenient to keep it separate.
   Note that the total number of opcodes is not something we need to worry
   about too much due to per-module opcode translation tables.

Q. Why doesn't `loop` loop by default?

A. In compiled code, it's common for loops end with an explicit conditional
   branch, so this alleviates WebAssembly consumers from having to recognize the
   pattern of a conditional branch at the end of a `loop`, invert the branch
   condition, rewrite the branch destinations, and skip the implicit branch they
   otherwise would have inserted.

   I'm open to better names for the `loop` operator, if people have suggestions.

Q. Why do `tableswitch` cases fall through by default?

A. It simplifies the semantics to say that `tableswitch` and `case` are just a
   jump table and syntactic constructs to place labels, and not also a series of
   branches mixed in.

   Also, WebAssembly is a compiler target, not something that most humans should
   be writing a lot manually. Compilers have a significantly easier time
   remembering to insert branches where they need them.

Q. Can this be polyfilled to JS?

A. Yes. It's even an improvement over earlier proposals because it doesn't
   require the deeply nested branch stacks, so it should be easier on existing
   JS parsers.

Q. If we were freed of hardware constraints, would we still be doing this?

A. Yes. Macroassemblers were popular in the 60's and 70's. They provided a
   wealth of high-level control constructs for convenience. But with the rise
   in popularity of compilers, macroassemblers gradually diminished to become
   relatively obscure specialized tools. Compiler writers almost universally
   chose to target the lower-level, simpler ISAs, rather than go through
   macro-assemblers. One of the reasons for this is that while structure is
   mildly convenient when it matches the structure of what a compiler wants
   to do, it's quite inconvenient when it doesn't, and compilers most often
   find the inconveniences outweigh the advantages.

   Other systems to consider include Java bytecode, CIL bytecode, LLVM IR,
   and many other similar bytecodes, IRs, and virtual ISAs designed in the
   absence of immediate hardware execution, and which generally choose plain
   branching, because it's simple and flexible.

Q. What about the use case of translating code directly from ASTs?

A. This is indeed a very important use case, but consider a few things about
   what this means. High-level ASTs are going to have numerous constructs
   that aren't in WebAssembly, such as proper `while` loops, so some
   lowering is going to be needed in any case. And, high-level ASTs aren't
   often going to be very optimized.

   One of the strengths of asm.js that we're hoping to preserve in WebAssembly
   is that WebAssembly engines don't need to do a lot of the optimization
   that could easily be done on developer machines ahead of time.

   So, how are we going to support languages that translate directly from ASTs?
   JIT libraries. Producer libraries can offer a high-level interface which has
   higher-level control constructs than WebAssembly has, such as proper `for`
   loops, proper struct and array types, and other essentials, and they can have
   the ability to perform some optimization before producing the underlying
   WebAssembly code.

Q. What about humans reading WebAssembly code?

A. Humans doing low-level debugging of WebAssembly code may prefer the simpler
   lower-level structure provided here. The bidirectional mapping to linear form
   makes single-step debugging a much more natural activity.

   Humans preferring to see a higher-level code are better served by seeing
   the actual source code. One doesn't often read x86 code when using a native
   debugger on x86, and similarly one shouldn't often see WebAssembly code when
   using a WebAssembly debugger. One should see the original source code.

   That said, there are times when humans will want to read WebAssembly code
   and will want to see it with some structure. Fortunately, some simple
   heuristics ought to be able to insert basic indentation in most cases,
   since this proposal does preserve a great deal of structural information.

Q. How are we going to support simple JIT use cases?

A. We can run JIT libraries in WebAssembly too. Since WebAssembly has
   near-native performance, it should be able to do most of the same
   optimizations that a WebAssembly engine itself might otherwise do, in close
   to the same speed.

   This approach is aimed at achieving a high level of performance portability,
   because one can count on getting the same high-level optimizations
   everywhere. And, it's aimed at keeping the WebAssembly platform itself
   simple.

Q. Do we need `if` if we can just `tableswitch` with a `0` case and a default?

A. We could make things work that way. But it's nice to think of `if` as
   being a "branch" in hardware terms, and a `tableswitch` as a "lookup table",
   with the associated performance considerations.

Q. What about functional languages translating to WebAssembly?

A. Guaranteed tail calls and GC are separate discussions. But other than that,
   this is another use case for a JIT library. It's not complicated to lower
   functional languages to machine code, and a JIT library could easily provide
   a convenient abstraction layer for doing so.

Q. What if I'm using a JIT library and I want to debug the output?

A. This is indeed a case that is not going to be quite as nice as one might
   like in the MVP timeframe. The MVP is focusing on C/C++, so while there are
   several things we could do here to help people work with the generated
   code, they aren't likely to happen in the MVP timeframe.

Q. Why doesn't WebAssembly just have a
   [CFG](https://en.wikipedia.org/wiki/Control_flow_graph)?

A. A CFG, or even just a sequence of instructions with branches and labels, is
   what all comparable systems use, and probably would have been what
   WebAssembly would have started with, had it not been for the happenstance of
   WebAssembly's JS heritage.

   That said, there are some advantages to structured control flow which motivate
   preserving it. It enables some specialized compression and decoding
   optimizations, as well as fast optimization techniques.

   Fortunately, you can still use `goto` in C/C++ to do just about anything you
   want, and it will be efficient on WebAssembly. Even though WebAssembly's
   control flow is structured, it is very flexible. `br` and `br_if` are simple
   branches, and `block` and `loop` can put labels just about anywhere. The only
   thing that isn't supported well is loops with multiple entries, which is not
   a common pattern in general.

Q. What about multiple-entry loops?

A. There are some important use cases for multiple-entry loops though,
   particularly optimized interpreter loops. In the MVP timeframe, these will be
   supported by adding additional branching or other code to form code that does
   not actually have multiple-entry loops.

   In the future, we expect to add even more expressive control constructs
   in order to support these use cases well.

Q. Why did you wait until now to propose this?

A. The technique of using `br_if`-style constructs to lower CFG-based control
   flow into structured form is a relatively recent discovery. Developing it
   into a more fully fledged idea and exploring how it interacts with other
   things in WebAssembly, and LLVM, has taken some time. I've now implemented
   code for this in LLVM and have been able to do several experiments on
   realistic code, which has significantly informed this proposal.

   Also, when I started proposing `br_if`, it was not well received. I have
   now stepped back and incorporated feedback from several sources, and
   significantly revised the proposal.

Q. Wasn't the code proposed for ml-proto for `br_switch` complicated?

A. This was mostly due to attempts to implement `br_switch` in terms of
   the existing infrastructure which wasn't built for it. In a way, this is
   another demonstration of the awkwardness of achieving the semantics being
   pursued here on top of the constructs currently in ml-proto.

   `br_switch` is actually a very simple instruction. The `tableswitch` proposed
   here is a little more involved due to the contained `cases`, but besides
   placing labels in various places, it's still just a jump table.
