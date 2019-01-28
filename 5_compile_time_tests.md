# Compile-time tests

A code snippet operates on states and parameters. The code snippet is deemed valid if its inputs respect a number of conditions. It can be as simple as saying that a size should be positive, and a state should be already allocated.
The usual way to check if conditions are effectively met is to `assert()` them. This adds a runtime check, which is typically active during tests. If tests are thorough enough, it's hoped that any scenario which can violate the conditions will be found during tests thanks to `assert()`, and fixed.

As one can already guess, this method features a few weaknesses. Don't get me wrong: adding `assert()` is _way way better_ than not adding them, but the whole precinct is to hope that tests will be good enough to find the bad paths leading to a condition violation, and one can never be sure that all bad paths were uncovered.

For some conditions, it's possible to transfer the check at compile time instead.
Granted, it's only a subset of what can be checked. But whatever can be validated at compiler stage is "good to go": no further runtime test required. It's like a mini-proof that holds for whatever state the program is in.
On top of that, the condition is checked immediately, while editing, right in the short-loop feedback. No need to wait for long test results ! Failures are identified and fixed quickly.

Not all checks can be transferred to compile-time, but whatever can be should seriously consider it.

static assert
-----------------

Invariant guarantees can be checked at compile time with a [`static_assert()`]. Compilation will stop, with an error, if the invariant condition is not satisfied. A successful compilation necessarily means that the condition is always respected (for the compilation target).
A typical usage is to ensure that `int` type of target system is wide enough. Or that some constants respect a pre-defined order. Or, as mentioned in an earlier article, that a shell type is necessarily large enough to host its target type.

It has all the big advantages mentioned previously : no trace in the generated binary : no run time nor space cost ! The condition doesn't have to be disabled depending on debug or release mode. And no need to rely on tests to verify that the condition holds.

[`static_assert()`]:https://en.wikichip.org/wiki/c/assert.h/static_assert

### C90 compatibility

[`static_assert()`] is a special macro added in the `C11` standard. While most modern compilers are compatible with this version of the standard, if you plan on making your code portable on a wider set of compilers, it's a good thing to consider an alternative which is compatible with older variants, such as `C90`.

Fortunately, it's not that hard. `static_assert()`  started its life as a "compiler trick", and many of them can be found over Internet. The basic idea is to transform a condition into an invalid construction, so that the compiler _must_ issue an error at the position of the `static_assert()`. Typical tricks include :
- defining an `enum` value as a constant divided by `0`
- defining a table which size is negative

For example :
```C
#define STATIC_ASSERT(COND,MSG) typedef char static_assert_##MSG[(COND)?1:-1]
```

One can find multiple versions, with different limitations. The macro above has the following ones :
- cannot be expressed in a block after the first statement for `C90` compatibility (declarations before statements)
- require different error messages to distinguish multiple assertions
- require the error message to be a single uninterrupted word, without double quotes, differing from `C11` version

The 1st restriction can be circumvented by putting brackets around `{ static_assert(); }` whenever needed.
The 2nd one can be improved by adding a `__LINE__` macro as part of the name, thus making it less probable for two definitions to use exactly the same name.
The last one is more concerning. It's an incompatibility and a strong limitation compared to `C11`.

That's why I rather recommend [this more complete version by Joseph Quinsey](https://godbolt.org/z/K9RvWS), which makes it possible to invoke the macro the same way as the `C11` version, allowing to switch easily from one to another. The declaration / statement limitation for `C90` is still present, but as mentioned, easily mitigated.

### Limitations

Unfortunately, static asserts can only reason about constants, which values are known at compile time.

Constants, in the C dialect, regroup a very restricted set :
- Literals value, e.g. `222`.
- Macros which result in literals value, e.g. `#define value 18`
- `enum` values
- `sizeof()` results
- Mathematical operations over constants which can be solved at compile time, e.g. `4+1` or even `((4+3) << 2) / 18`.

As a counter-example, one might believe that `const int i = 1;` is a constant, as implied by the qualifier `const`. But it's a misnomer : it does not define a constant, merely a "read-only" (immutable) value.

Therefore it's [not possible to `static_assert()` conditions on variables](https://godbolt.org/z/SO8FLZ), not even `const` ones. It's also not possible to [express conditions using functions](https://godbolt.org/z/Q0K5wV) (though macro replacements are valid).

This obviously strongly restrains the set of conditions that can be expressed with a `static_assert()`.

Nonetheless, every time `static_assert()` is a valid option, it's recommended to use it. It's a cheap, efficient zero-cost abstraction which guarantees an invariant, contributing to a safer outcome.

Arbitrary conditions validated at compile time
------------------------

Checking arbitrary conditions at compile time, like say a runtime `assert()` ? That sounds preposterous. Yet, there's indeed something close enough.

The question asked changes in a subtle way : it's no longer "prove that the condition holds given current value(s) in memory", but rather "prove that the condition can never be false", which is a much stronger statement.

The benefits are similar to `static_assert()` : since the condition is guaranteed to be met, there's no need to check it at run time, hence this check is completely runtime free.

Enforcing such a strong property may seem a bit overwhelming. Yet, that's exactly the kind of condition which is silently required by any operation featuring [undefined behavior] due to a [narrow contract](https://alexpolt.github.io/contract.html). Problem is, by default, there is no signal at compilation time to warn the programmer when these conditions are broken, resulting in unpredictable consequences at run time. This is a big source of bugs.

This method is not suitable for situations involving an unpredictable runtime event. For example, compiler cannot guarantee that a certain file will exist, so trying to open a file will always require a runtime check.
But it's appropriate for conditions that the programmer expect to be always true, and which violation constitute a programming error. The compiler will now be in charge of proving that the invariant is necessarily true, and issue a warning if that's not the case.

[undefined behavior]: https://en.wikipedia.org/wiki/Undefined_behavior

Let's give a simple example :
dereferencing a pointer requires that, as a bare minimum, the pointer is not `NULL`. It's not a loose statement, like "this pointer is probably not `NULL` in general", it must be 100% true, otherwise, [undefined behavior] is invoked.
How to ensure this property then ?
Simple : test if the pointer is `NULL`, and if it is, do not dereference it, and branch elsewhere.
Passing the branch test guarantees the pointer is now non-`NULL`.

This example is trivial, yet very applicable to everyday code.
It's extremely common to forget such a test, since there's no warning for the programmer. And `NULL` pointer can happen due to exceptional conditions which can be difficult to trigger during tests, such as a rare `malloc()` failure for example.
And that's just a beginning : most functions and operations feature a set of (hopefully documented) conditions to be respected for their behavior and result to be correct. Want to divide ? better be by non-zero. Want to add signed values ? Better be sure they don't overflow. Let's call `memcpy()` ? Better ensure memory segments are allocated and don't overlap. And on, and on, and on.

The usual way to ensure that these conditions are met is to `assert()` them (when they can be expressed). Yet, since `assert()` is a runtime test with associated runtime cost, it is designed to be disabled once in production. Even if `assert()` are let enabled, they will terminate the process quite abruptly, which is still not great for production environment.

A better solution is to ensure that the condition always hold. This is where a compile-time guarantee comes in.

We want the compiler to emit a warning whenever a condition cannot be guaranteed to be true. Technically, this is almost like an `assert()`, though one that leaves no trace in generated binary.

This outcome, by the way, already happens regularly, though silently, whenever an `assert()` can be proven to be always true, through a fairly common optimization stage called Dead Code Elimination (DCE).
Therefore, the idea is to design an `assert()` that is meant to be removed from final binary through DCE, and emits a warning if it does not. Since no such instruction exists in the base language, we'll have to rely on some compiler-specific extensions.

`gcc` for example offers a [function attribute](https://gcc.gnu.org/onlinedocs/gcc/Common-Function-Attributes.html) which does exactly that :

> `warning ("message")`
> If the `warning`  attribute is used on a function declaration and a call to such a function is not eliminated through dead code elimination or other optimizations, a warning that includes `"message"` is diagnosed. This is useful for compile-time checking.

This makes it possible to create this macro :
```C
__attribute__((noinline))
__attribute__((warning("condition not guaranteed")))
static void never_reach(void) { abort(); } // must define a side effect, to not be optimized away

// EXPECT() : will trigger a warning if the condition is not guaranteed to be true
#define EXPECT(c) (void)((c) ? (void)0 : never_reach())
```

The resulting macro is called `EXPECT()`, for consistency with a recent C++20 proposal, called [attribute contract](https://en.cppreference.com/w/cpp/language/attributes/contract), which suggests the notation `[[expects: expression]]` to achieve something similar (though not  identical, proposed behavior is different in important ways).

`EXPECT()` is designed to be used the same way as `assert()`, the difference being it will trigger a warning at compile time whenever it cannot be optimized away,  underlying that the condition can not be proven to be always true.

### Limitations

It would be too easy if one could just start writing `EXPECT()` everywhere as an `assert()` replacement. Beyond the fact that it can only be used to test programming invariants, there are additional limitations.

First, this version of `EXPECT()` macro only works well on `gcc`. I have not found a good enough equivalent for other compilers, though it can be emulated using other tricks, such as an [incorrect assembler statement](https://godbolt.org/z/OyoUjL), or linking to some non existing function, both of which feature significant limitations : do not display the line at which condition is broken, or do not work when it's not a program with a `main()` function.

Second, checking the condition is tied to compiler's capability to combine [Value Range Analysis](https://en.wikipedia.org/wiki/Value_range_analysis) with [Dead Code Elimination](https://en.wikipedia.org/wiki/Dead_code_elimination). That means the compiler must use at least a bit of optimization. These optimizations are not too intense, so `-O1` is generally enough. Higher levels can make a difference if they increase the amount of inlining (see below).

However, `-O0` definitely does not cut it, and all `EXPECT()` will fail. Therefore, `EXPECT()` must be disabled when compiling with `-O0`. `-O0` can be used for fast debug builds for example, so it cannot be ruled out. This issue makes it impossible to keep `EXPECT()` always active by default, so its activation must be tied to some explicit build macro.

Third, Value Range Analysis is limited, and can only track function-local changes. It cannot cross function boundaries.

There is a substantial exception to this last rule for `inline` functions : for these cases, since function body will be included into the caller's body, `EXPECT()` conditions will be applied to both sides of the interface, doing a great job at checking conditions and inheriting VRA outcome for optimization.
`inline` functions are likely the best place to start introducing `EXPECT()` efficiently in an existing code base.

## Function pre-conditions

Beyond `inline` functions, `EXPECT()` must be used differently compared to `assert()`.
A typical way to check that input conditions are respected is to `assert()` them at the beginning of the function.
This wouldn't work with `EXPECT()`.

Since VRA does not cross function boundaries, `EXPECT()` will not know that the function is called with bad parameters. Actually, it will also not know that the function is called with good parameters. With no ability to make any assumption on function parameters, `EXPECT()` will just always fail.

```C
// Never call with `v==0`
int division(int v)
{
    EXPECT(v!=0);  // This condition will always fail :
                   // the compiler cannot make any assumption about `v` value.
    return 1/v;
}

int lets_call_division_zero(void)
{
    return division(0);   // No warning here, though condition is violated
}
```

To be useful, `EXPECT()` must be declared _on the caller side_, where it can properly check input conditions.
Yet, having to spell input conditions on the caller side at every invocation is cumbersome. Worse, it's too difficult to maintain : if conditions change, all invocations must be updated !
A better solution is to spell all conditions in a single place, and encapsulate them as  part of the invocation.

```C
// Never call with `v==0`
int division(int v)
{
    return 1/v;
}

// The macro has same name as the function, so it masks it.
// It encapsulates all preconditions, and deliver the same result as the function.
#define division(v) ( EXPECT(v!=0), division(v) )

int lets_call_division_zero(void)
{
    return division(0);   // Now, this one gets flagged right here
}

int lets_call_division_by_something(int divisor)
{
    return division(divisor);   // This one gets flagged too : there is no guarantee that it is not 0 !
}

int lets_divide_and_pay_attention_now(int divisor)
{
    if (divisor == 0) return 0;
    return division(divisor);   // This one is okay : no warning
}

```

Here are some more [example usages](https://godbolt.org/z/E2QgA0). Note how `EXPECT()` are combined with a function signature into a macro, so that compile time checks get triggered every time the function is called.

### Limitations

This construction solves the issue on the caller side, which is the most important one.

You may note that the macro features a typical flaw : its argument `v` is present twice. It means that, if `v` is actually a function, it's going to be invoked twice. In some cases, like `rand()`, both invocations may even produce different results.

However, in this specific case, it's not a problem, because it is impossible to successfully invoke the macro using a function as argument.
The reason is, function's return value has no any guarantee attached, beyond its type. So, if it is an `int`, it could be any value, from `INT_MIN` to `INT_MAX`.
As a consequence, no function's return value can ever comply with any condition. It will necessarily generate a warning.

The encapsulating macro can only check conditions on variables.
It will only accept variables which are _guaranteed_ to respect the conditions. If a single one _may_ break any condition, a warning is issued.

However, pre-conditions remain unknown to the function body itself. This is an issue, because the function may benefit from this information to comply with other function pre-conditions. Without it, it will be necessary to re-express the conditions withing function body, which is an unwelcome burden.

While the safest solution is to branch out bad cases, sometimes, in order for code simplicity to benefit from the narrow contract, it's better to rely on the fact that conditions are necessarily respected. It would be a waste, and unwelcome complexity, to generate a branch for them. In which case, a quick solution is to express these guarantees with `assert()`. This is, by the way, what should have been done anyway, even without `EXPECT()`.

An associated downside is that ensuring that `EXPECT()` conditions are respected using `assert()` presumes that `assert()` are present and active in source code, to guide the Value Range Analysis. If `assert()` are disabled, corresponding `EXPECT()` will fail. This suggests that `EXPECT()` can only be checked in debug builds. But with optimization enabled (`-O1`).

Now with all these `assert()` back, it seems like these compile-time checks are purely redundant, and almost useless.

Not quite. It's true that so far, it has not reduced the amount of `assert()` present in the code, but the compiler now actively checks pre-conditions, and mandates the presence of `assert()` for every condition that the local code does not explicitly rule out. This is still a step up : risks of contract violation are now clearly visible, and it's no longer possible to "forget" an `assert()`. As a side effect, tests will also catch condition violations sooner, leading to more focused and shorter debug sessions. This is still a notable improvement.

It nonetheless feels kind of incomplete. One missing aspect is an ability to transfer pre-conditions from the calling site to the function body, so that they can be re-used to satisfy later pre-conditions.
This capability requires another complementary tool. But that's for another blog post.
