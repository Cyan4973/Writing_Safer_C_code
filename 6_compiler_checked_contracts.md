# Compiler-validated contracts

[Contract programming](https://en.wikipedia.org/wiki/Design_by_contract) is not a new concept. It's a clean way to design narrow contracts, by spelling explicitly its conditions, and checking them actively at contract's interface. In essence, it's translated into a bunch of `assert()` at the entrance and exit of a function. It's a fairly good formal design, although one associated with a runtime penalty.

We left the previous episode with an ability to express function preconditions and make them checked by the compiler, but no good way to transport the outcome of these checks into the function body. We'll pick up from there.

The proposed solution is to re-express these invariants in the function as `assert()`, as they should have been anyway if `EXPECT()` was absent. It works, but it also means that downstream `EXPECT()` can only be validated when `assert()` are enabled, aka. in debug builds.

Let's try to improve this situation, and keep `EXPECT()` active while compiling in release mode, aka with `assert()` disabled.
What is needed is an `assert()` that still achieves its outcome on Value Range Analysis while disabled. Such a thing exists, and is generally called an `assume()`.

### `assume()`

`assume()` is not part of the language, but most compilers offer some kind of hook to build one. Unfortunately, they all differ.

On `gcc`, `assume()` can be created using `__builtin_unreachable()` :
```C
#define assume(cond) do { if (!(cond)) __builtin_unreachable(); } while (0)
```

`clang` provides `__builtin_assume()`. `icc` and Visual provide `___assume()`. etc.
You get the idea.

An important point here is that, in contrast with all techniques seen so far, `assume()` actually reduces compiler's effectiveness at catching bug. It's an explicit "trust me, I'll tell you what to know" situation, and it's easy to get it wrong.

One way to mitigate this negative impact is to make sure `assume()` are converted into `assert()` within debug builds, so that there is at least a chance that wrong assumptions get caught during tests.

`assume()` however have additional restrictions compared to `assert()`. `assert()` merely need to produce no side-effect, and offer a tractable runtime cost, though even this last point is negotiable. But `assume()` must be transformed into pure compiler hints, leaving no trace in the generated binary (beyond the assumption's impact). In particular, the test itself should not be present in the generated binary.
This reduces eligible tests to simple conditions only, such as `(i>=0)` or `(ptr != NULL)`. A counter example would be `(is_this_graph_acyclic(*graph))`. "complex" conditions will not provide any useful hint to the compiler, and on top of that, may also leave a runtime trace into the generated binary, resulting in a *reduction* of performance.

_Note_ : I haven't found a way to ensure this property on `gcc` : if `assume()` is served a complex condition, it will [happily generate additional asm code](https://godbolt.org/z/lKNMs3), without issuing any warning.
Fortunately, `clang` is [way better at this game](https://godbolt.org/z/aLDyrP), and will correctly flag bad `assume()` conditions, which would generate additional code in `gcc`.
As a consequence, it's preferable to have `clang` available and check the code with it from time to time to ensure all `assume()` conditions are correct.

In our case, `assume()` main objective is not really performance : it is to forward conditions already checked by `EXPECT()` within function's body, so that they can be re-used to automatically comply with downstream `EXPECT()` conditions, even when `assert()` are disabled, aka during release compilation.

So here we are : every time a condition is required by `EXPECT()` and cannot be deducted from the local code, express it using `assume()` rather than `assert()`. This will make it possible to keep `EXPECT()` active irrespective of the debug nature of the build.
Note that, if any condition is too complex for `assume()`, we are back to square one, and need to rely on `assert()` only (hence debug builds only).

Being able to keep `EXPECT()` active in release builds is nice, but not terrific. At this stage, we still need to write all these `assume()` in the code, and we cannot take advantage of pre-conditions already expressed at the entrance of the function.

Worse, since pre-conditions are expressed on one side, in the `*.h` header where function prototype is published, while the corresponding `assume()` are expressed in the function body, within `*.c` unit file, that's 2 separate places, and it's easy to lose sync, when one side changes the conditions.

### Expressing conditions in one place

What we need is to express preconditions in a [single source of truth](https://en.wikipedia.org/wiki/Single_source_of_truth). This place should preferably be close to the prototype declaration, since it can also serve as function documentation. Then the same conditions will be used in the function body, becoming assumptions.

The solution is simple : define a macro to transport the conditions in multiple places.
Here is [an example](https://godbolt.org/z/oWoT3I).
The conditions are transferred from the header, close to prototype declaration, into the function body, using a uniquely named macro. It guarantees that conditions are kept in sync.
In the example, note how the knowledge of `minus2()` preconditions, now considered satisfied within function body, makes it possible to automatically comply with the preconditions of invoked `minus1()`, without adding any `assert()` or `assume()`.

In this example, the condition is trivial `(i>=2`), using a single argument. Using a macro to synchronize such a trivial condition may seem overkill. However, synchronization is important in its own right. Besides, more complex functions, featuring multiple conditions on multiple arguments, will be served by a design pattern which can be just reproduced mindlessly, whatever the complexity of the preconditions : `assume(function_preconditions());`.

There is still a variable element, related to the number of arguments and their order.
To deal with that variance, argument names could be baked directly into the preconditions macro. Unfortunately, this would only work within a function. But since the macro transporting preconditions is itself invoked within a macro, it wouldn't expand correctly.

Another downside is that we just lost a bit of clarity in the warning message : conditions themselves used to be part of the warning message, now only the macro name is, which transmits less information.
Unfortunately, I haven't found a way around this issue.

To preserve clarity of the warning message, it may be tempting to keep the previous format, with conditions expressed directly in the masking function macro, whenever such conditions are not required afterwards in the body. However, it creates a special cases, with some functions which replicate conditions in their body, and those that don't.

Transmitting preconditions compliance into function body makes it easier to comply with a chain of preconditions. A consequence of which, it makes it more tractable to use compile-time pre-conditions onto a larger scope of the code base.

### Post conditions

Yet we are not completely done, because the need to check preconditions implies that all contributors of any parameter are part of the game. Function's return values themselves are contributors.

For example, one may invoke a function `f1()` requiring an argument `i>0`.
The said argument may be provided as a return value of a previous function `f2()`.
`f2()` might guarantee in its documentation that its return value is necessarily `>0`,  hence is compliant,
but the compiler doesn't read the documentation. As far as it's concerned, the return value could be any value the type allows.

The only way to express this situation is to save the return value into an intermediate variable,
and then `assert()` or `assume()` it with the expected guarantee,
then pass it to the second function.
This is a bit more verbose than necessary, especially as `f2()` was already fulfilling the required preconditions. Besides, if `f2()` guarantees change, the local assumption will no longer be correct.

Guarantees on function's outcome are also called post-conditions. The whole game is to pass this information to the compiler.

This could be done by bundling the post-conditions into the macro invoking the function.
Unfortunately, that's a bit hard to achieve with a portable macro, usual woes get in the way : single-evaluation, variable declarations and returning a value are hard to achieve together.

For this particular job, we are better off using an `inline` function.
See [this example](https://godbolt.org/z/jNxO6b) on godbolt.
It works almost fine : the guarantees from first function are used to satisfy preconditions of second function. This works without the need to locally re-assess first function's guarantees.
As an exercise, removing the post-conditions from encapsulating `inline` function immediately triggers a warning on second invocation, proving it's effective.

However, we just lost a big property by switching to an `inline` function : warnings now locate precondition violations _into_ the `inline` function, instead of the place where the function is invoked with incorrect arguments. Without this information, we just know there is a contract violation, but we don't know _where_. This makes fixing it sensibly more difficult.

To circumvent this issue, let's use a macro again. This time we will combine a macro to express preconditions with an inlined function to express outcome guarantees. Here is [an example](https://godbolt.org/z/i4pdFH).
This one gets it right on almost everything : it's portable, conditions and guarantees are transferred to the compiler, which triggers a warning whenever a condition is not met, indicating the correct position of the problem.

There is just one last little problem : notice how the input parameter `v` get evaluated twice in the macros. This is fine if `v` is a variable, but not if it's a function. Something like `f1( f2(v) )`  will evaluate `f2()` twice, which is bad, both for runtime and potentially for correctness, should `f2(v)` return value be different on second invocation.

It's a pity because this problem was solved in the first proposal, using only an `inline` function. It just could not forward the position where a condition was broken. Now we are left with two incomplete proposals.

Let's try it again, using a special kind of macro.
`gcc` and by extension `clang` support a special kind of [statement expression](https://gcc.gnu.org/onlinedocs/gcc-5.2.0/gcc/Statement-Exprs.html), which makes it possible to create a compound statement able to return a value (its last expression). This construction is not portable. In general, I wouldn't advocate it due to portability restrictions. But in this case, `EXPECT()` only works on `gcc` to begin with, so it doesn't feel too bad to use a `gcc` specific construction. It simply must be disabled on non-gcc targets.

The new formulation, reproduced below, works perfectly, and now [enforces the contract while avoiding the double-evaluation problem, and correctly indicates the position at which a condition is violated](https://godbolt.org/z/dukzpt), significantly improving diagnosis.

```C
int positive_plus1(int v);
#define positive_plus1_preconditions(v)   ((v)>=0)     // Let's first define the conditions. Name is long, because conditions must be unique to the function.
#define positive_plus1_postconditions(r)   ((r)>0)     // Convention : r is the return value. Only used once, but published close to the prototype, for documentation.

// Encapsulating macro
// The macro itself can be published in another place of the header,
// to leave complete visibility to the prototype and its conditions.
// This specific type of macro is called a statement-expression,
// a non-portable construction supported by `gcc` (and `clang`)
// It's okay in this case, because `EXPECT()` only works with `gcc` anyway.
// But it will have to be disabled for non-gcc compilers.
#define positive_plus1(iv) ({                                             \
    int const _v = iv;   /* avoid double-evaluation of iv */              \
    int _r;                                                               \
    EXPECT(positive_plus1_preconditions(_v));  /* also used within function body */ \
    _r = positive_plus1(_v);                                              \
    assume(positive_plus1_postconditions(_r)); /* only used here */       \
    _r;   /* last expression is the return value of compound statement */ \
 })
```

### Summary

That's it. This construction gives all the tools necessary to use compiler-checked contracts in a C code base. Such strong checks increase the reliability of the code base, especially during refactoring exercises, by catching _at compile time_ all potential contract breaches, and _requiring_ to deal with them, either through branches or at least explicitly `assert()` them. This is a big step up from a situations where breaking conditions was plain silent at compilation, and _may_ break during tests _if_ `assert()` are not forgotten _and_ the test case is able to break the condition.

It can be argued that applying this design pattern makes declaring functions more verbose, and it's true. The effort though was supposed to be already done in a different way : as part of code documentation, and as part of runtime checks (list of `assert()` within function body). The difference is that they are expressed upfront, and are known to the compiler, which is more powerful.

Nonetheless, it would be even better if conditions could become part of the function signature, making the notation clearer, better supported, and by extension possibly compatible with automatic documentation or IDE's context info, simplifying their presentation.
There is currently a C++20 proposal, called [attribute contract](https://en.cppreference.com/w/cpp/language/attributes/contract), which plans to offer something close. Okay, it's not C, and quite importantly it is a bit different in subtle ways : it's more focused on runtime checks. There is a specific `[[expects axiom: (...)]]` notation which seems closer to what is proposed in this article, because it doesn't silently insert automatic runtime checks. However, as far as I know, it also doesn't guarantee any compiler check, reducing the contract to a simple `assume()`. It implies this topic is left free to compiler's willingness, which may or may not pick it up, most likely resulting in significant behavior differences.

But hopefully, the trick presented in this article is available right now, and doesn't need to wait for any committee, it can be used immediately on existing code bases.

I hope this article will raise awareness on what compilers already know as part of their complex machinery primarily oriented towards better runtime performance, and make a case on how to re-purpose a small part of it to improve correctness too.
