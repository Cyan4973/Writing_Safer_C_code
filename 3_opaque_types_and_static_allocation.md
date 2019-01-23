# Opaque types and static allocation

In a previous episode, we've seen that it is possible to create opaque types. However, creation and destruction of such type must be delegated to some dedicated functions, which themselves rely on dynamic allocation mechanisms.

Sometimes, it can be convenient to bypass the heap, and all its `malloc()` / `free()` shenanigans. Pushing a structure onto the stack, or within thread-local storage, are natural capabilities offered by a normal `struct`. It can be desirable at times.

The previously described opaque type is so secret that it has no size, hence is not suitable for such scenario.

Fortunately, static opaque types are possible.
The main idea is to create a "shell type", with a known size and an alignment, able to host the target (private) structure.

For safer maintenance, the shell type and the target structure must be kept in sync, by using typically a [static assert](https://en.cppreference.com/w/c/language/_Static_assert). It will ensure that the shell type is always large enough to host the target structure. This check is important to automatically detect future evolution of the target structure.

If it wasn't for the [strict aliasing rule], we would have a winner : just use the shell type as the "public" user-facing type, proceed with transforming it into the private type inside the unit. It would combine properties of `struct` while remaining opaque.

[strict aliasing rule]:http://dbp-consulting.com/tutorials/StrictAliasing.html

## Strict aliasing

Unfortunately, the [strict aliasing rule] gets in the way : we can't manipulate the same memory region from two pointers of different type for the lifespan of the stored value. That's because the compiler is allowed to make assumptions about pointer value provenance for the benefit of performance.

To visualize the issue, I like [this simple example](https://godbolt.org/z/6cSQvx), powered by Godbolt. Notice how the two `+1` get combined into a single `+2`, saving one save+load round trip, and allowing computation over `i` and `f` in parallel, so it's real saving.
But unfortunately, if `f` and `i` have same addresses, the result is wrong : the first `i+1` influences the operation on `f` which influences the final value of `i`.
Of course, this example is silly : I can't think of a single reason which would justify operations on `int` and `float` pointing at the same memory address. It shows that the rule is quite logical : if these pointers have different type, they most likely do not represent the same memory area. And since benefits are substantial, it's tempting to use that assumption.

Interpreting differently the same memory area using different types of pointers is called "type punning". It may work, as long as the compiler serializes operations as expected in the code, but there is no guarantee that it will continue to work safely in the future. A known way to break older programs employing type punning is to recompile them with modern compilers using advanced performance optimizations such as `-O3 -lto`. With enough inlining, register caching and dead code elimination, one will start to experience strange effects, which can be very hard to debug.

This is explained in greater details in this [excellent article from Mike Acton](https://cellperformance.beyond3d.com/articles/2006/06/understanding-strict-aliasing.html). For an even deeper understanding of what can happen under the hood, you can [read this document](http://www.open-std.org/jtc1/sc22/wg14/www/docs/n2263.htm) suggested by Josh Simmons. It demonstrates that there is a lot more to a pointer than just its binary representation.

One line of defense could be disable usage of strict aliasing by the optimizer, with a compilation directive such as [`fno-strict-aliasing`](https://godbolt.org/z/kYaAAb) on `gcc`.
I wouldn't recommend it though. On top of impacting performance, it ties code correctness to a specific compiler setting, which may or may not be present in user's project. Portability is also impacted, since there is no guarantee that this capability will always be available on some different C compiler.

Another line of defense consists in using the `char*` pointer, which is the exception to the rule, and can alias anything. When one memory area is passed as a `char*`, the compiler [will pay attention to serialize `char*` read/write properly](https://godbolt.org/z/2G0dOZ). It works well in practice, at least in my tests. What is worrying though is that in theory, the compiler [is only obliged to guarantee the read in correct order](https://stackoverflow.com/a/28240251/646947). That it pays attention to serialize the write too seems to be "extra care", presumably so that existing programs continue to work as intended. Not sure if it is reliable to depend on it on long term.

Another issue is, our proposed shell type is not a `char*` table. It's a `union`, containing a `char*` table. That's not the same, and in this case, [the exception does not hold](https://godbolt.org/z/4Qly0X).

As a consequence, the shell type must not be confused with the target type. The [strict aliasing rule] makes them non-interchangeable !

## Safe static allocation for opaque types

The trick is to use a 3rd party initializer, to convert the allocated space and return a pointer of appropriate type.
To ensure strict compliance with C standard, it's a multi-steps trick, hence a more complex setup. Consider this technique as "advanced", implying limited usage scenarios.

Here is an example :
```C
typedef struct thing_s thing;   // incomplete (opaque) type

typedef union {
    char body[SIZE];
    unsigned align4;   // ensures `thingBody` is aligned on 4-bytes boundaries
} thingBody;

// PREFIX_initStatic_thing() accepts any buffer as input,
// and returns a properly initialized `thing*` opaque pointer.
// It ensures `buffer` has proper size (`SIZE`) and alignment (4) restrictions
// and will return `NULL` if it does not.
// Resulting `thing*` uses the provided buffer only, it will not allocate further memory on its own.
// Use `thingBody` to define a memory area respecting all conditions.
// On success, `thing*` will also be correctly initialized.
thing* PREFIX_initStatic_thing(void* buffer, size_t size);

// Notice there is no corresponding destructor.
// Since the space is reserved externally, its deallocation is controlled externally.
// This presumes that `initStatic` does Not dynamically allocates further space.
// Note that it doesn't make sense for `initStatic` to invoke dynamic allocation.

/* ====================================== */
/* Example usage */

int function()
{
    thingBody scratchSpace;   /* on stack */
    thing* T const = PREFIX_initStatic_thing(&scratchSpace, sizeof(scratchSpace));
    assert(T != NULL);  // Should be fine. Only exception is if `struct thing_s` definition changes and there is some version mismatch.

    // Now use `T` as a normal `thing*` pointer
    // (...)

    // do Not `free(T)` at function's end, since thingBody is part of the stack
}
```

In this example, the static size of `thingBody` is used to allocate space for `thing` on the stack. It's faster, and there is no need to care about deallocation.

But that's all it does. No data is ever read from nor written to `thingBody`. All usages of the memory region pass through `thing*`, which is safe.

Compared to a usual public `struct`, the experience is not equivalent.
To begin with, the proposed stack allocation is a multi-liner and creates 2 variables : the shell type, and the target pointer. It's not too bad, and this model fits well enough any kind of manual allocation scenario, be it on stack or within a pre-reserved area (for embedded environments typically).

If that matters, stack allocation could have been made a one liner, hidden behind a macro.
But I tend to prefer the variant in above example. It makes it clear what's happening. Since one of C strengths is a clear grasp of resource control, it is better to preserve that level of understanding.

There are more problematic differences though.
It's not possible to use the shell type as a return type of a function: once again, shell type and target incomplete type are different things. On the same line, it's not possible to pass the shell type by value. The memory region can only be passed by reference, and only using the correctly typed pointer.

Embedding the shell type into a larger structure is dangerous and generally not recommended : it requires 2 members (the shell and the pointer), but the pointer is only valid if the `struct` is not moved around, nor copied. That's a too strong constraint to make it safely usable.

### Removing the pointer

Suggested by Sebastian Aaltonen, it is generally possible to bypass the target pointer, and just reuse the address of the shell type instead. Since the shell type is never accessed directly, there is no aliasing to be afraid of.

The only issue is, [some compilers might not like the pointer cast from `shellType*` to target `opaque*`](https://godbolt.org/z/ldGoF2), irrespective of the fact that the `shellType` is never accessed directly. This is an annoying false positive. That being said, newer compilers are [better at detecting this pattern](https://godbolt.org/z/LYduX7), and won't complain.
Note that the explicit casting [is not optional,](https://godbolt.org/z/_My0Tz) so the notation cannot be shortened, hence this method will not save much keystrokes.

The real goal is to guarantee that the address transmitted is necessarily the address of `shell`. This makes sense when the intention is to move `shell` around or copy it : no risk to lose sync with a separate pointer variable.

To be complete, note that, in above proposal, `initStatic()` does more than casting a pointer :
- It ensures that the memory area has correct size & alignment properties
	- `shellType` provides these guarantees too.
		- The only corner case is when the program invokes `initStatic()` from a dynamic library. If runtime library version is different from the one used during compilation of the program, it can lead to a potential discrepancy on size or alignment requirements.
		- No such risk when using static linking.
- It ensures that the resulting pointer references a properly initialized memory area.

The second bullet point, in particular, still needs to be done one way or another, so `initStatic()` is still useful, at least as an initializer.

### Using the shell type directly

Removing the pointer is nice, but the real game changer is to be able to employ the opaque type as if it was a normal `struct`, in particular :
- assign with `=`
- can be passed by value as function parameter
- can be received as return type from a function

These properties can influence the API design, making the opaque type "feel" more natural to use. For example :

```C
// declaration
#define SIZE 8
typedef union {
    char body[SIZE];
    unsigned align4;   // ensures `thing` is aligned on 4-bytes boundaries
} thing;
// No need for a "separate" incomplete type.
// The shell IS the public-facing type for API.

thing thing_init(void);
thing thing_set_byValue(int v);
thing thing_combine(thing a, thing b);

// usage
thing doubled_value(int v)
{
    thing const ta = thing_set_byValue(v);
    thing const tb = ta;
    return thing_combine(ta, tb);
}
```

This can be handy for __small__ [POD types](https://en.wikipedia.org/wiki/Passive_data_structure) (typically less than a few dozens of bytes), giving them a behavior similar to basic types.
Since passing arguments and results by value [implies some memory copy,](https://godbolt.org/z/-Zqqcy) the cost of this approach increases as type size increases. Therefore, whenever the type becomes uncomfortably large, prefer switching to a pointer reference.

The compiler may completely eliminate the memory copy operation if it can somehow inline the invoked functions. That's, by definition, hard to do when these functions are in a separate unit, due to the need to access a private type declaration.
However, `-lto` (Link Time Optimization) can break the unit barrier. As a consequence, functions which were behaving correctly while not inlined might end up being inlined, triggering weird optimization effects.

For example, statements acting directly on `shell*`, such as potential `memset()` initialization, or any kind of value assignment, might be reordered for parallel processing with other statements within inlined functions acting on `internal_type*`, on the assumption that `shell*` and `internal_type*` should not be aliased.
To be fair, I would expect a modern compiler to be clever enough to detect that `shell*` and `internal_type*` reference effectively the same address, and avoid re-ordering or eluding memory read / write operations. Nevertheless, this is a risk, that might be triggered by complex cases or less clever compilers (typically older ones).

[The solution is to use `memcpy()`](https://godbolt.org/z/7_Zw07) to transfer data back and forth between internal type and shell type. `memcpy()` acts as a synchronization point for memory accesses : it guarantees that read and write orders will be serialized, ordered as written in the source code. The compiler will not be able to "outsmart" the code by re-ordering statements under the assumptions that side-effects on 2 pointers of different types cannot alias each other : a `memcpy()` can alias anything, so it has to be performed in the requested order.

### Back to `struct` ?

Adding `memcpy()` everywhere is a small inconvenience. Also, there is always a risk that the compiler will not be smart enough to elide the copy operation.

Due to these limitations and risks, it can be better to give up this complexity and just use a public `struct`. As long as the `struct` is a POD type, all conveniences are available. And without the need to add some private declaration, it's now possible to define implementations directly in header, as explicit `inline` functions, sharply reducing the cost of passing parameters.

To avoid direct accesses to structure member, one can still mention it clearly in code comments, and use scary member names as deterrent. A more involved way to protect `struct` members is to give them scary _and useless_ names, such as `dont_access_me_1`, `dont_access_me_2`, etc.  and rename them with macros in the code section which can actually interpret them. This is a bit more involving, especially if the number of member names is large, potentially leading to confusion. More importantly, the compiler will no longer be able to help in case of contract violation, and protecting the design pattern will now entirely depend on reviewers. Still, it's a very reasonable choice, notably for "internal" types, which are not exposed on user side API, hence should only be manipulated by a small number of skillful contributors subject to review process.

For user facing types though, opacity is more valuable. And if the type size is large enough to begin with, it seems a no brainer : prefer the opaque type, and only use references.
