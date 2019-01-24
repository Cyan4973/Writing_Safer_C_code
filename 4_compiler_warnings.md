Compiler warnings
===============

One way to improve C code quality is to reduce the number of strange constructions that the standard does not explicitly forbid. This will greatly help code reviewers, who want less surprises, and try to understand what a segment of source code is achieving and impacting.

A straightforward way to create such a "constrained"  C variant is to add compiler-specific warning flags. They will trigger warnings on detecting certain constructions considered dubious, if not downright dangerous.

A simple example is the condition `if (i=1) {`. This test seems to check if `i` equal `1`, but that's not what it does : it _assigns_ the value `1` to `i`. Also, as a consequence, it is always true. This is _most likely_ a typo, the programmer probably wanted to express an equality test `if (i==1) {`. Yet, it's not invalid, strictly speaking. So a compiler is [allowed to accept it at face value](https://godbolt.org/z/Q5swoj) and generate a binary without any warning. That may take a while to debug ...

Modern compilers [won't let such an obvious flaw silently happen](https://godbolt.org/z/6uM2iL). They will issue a warning.
At the very least, the warning is an invitation to spell the intention more clearly.
Sometimes, it was a genuine error, and the compiler just helped us catch this issue before it ever reaches production, saving some considerable debug time.

Multiplying the number of flags will increase the number of warnings. But sifting through a large list of warnings to find which ones are interesting and which one are merely informational can be daunting. Moreover, collaborative programming requires simple rules, that anyone can abide by.

Using warnings should be coupled with a strict "zero-warning" policy. Every warning must be considered an error to be dealt with immediately. This is a clean signal that everyone understand, and that any CI environment can act upon. If a warning message is considered not fixable, or not desirable to fix, it's preferable to remove the associated flag from the build chain.

On `gcc`, ensuring that no warning can be ignored can be enforced by the `-Werror` flag, which makes any warning a fatal error. Visual has ["treat warnings as errors"](https://i2.wp.com/dailydotnettips.com/wp-content/uploads/2016/03/image11.png).
More complex policies are possible, such as activating more warnings and only make some of them fatal (for example `-Werror=vla`) but it makes the setup more complex, and logs more difficult to follow.  

As a consequence, it's not a good idea to just "enable everything". Each additional flag increases the number of false-positive to deal with. When too many warnings are generated, it will feel like a discouraging and low-value task, leading to its abandon. Only warnings which bring some value deserve to be tracked, fixed, and continuously maintained. Therefore, it is preferable to only add a flag when its benefit is clearly understood.

That being said, the best moment to crank up the warning level is at the beginning of a project. What tends to be difficult is to _add_ new flags to an existing project, because new flags will reveal tons of programming patterns that where silently allowed and must now be avoided, anywhere within the repository. On the other hand, keeping an existing code clean is much simpler, as issues appear only in new commits, and can therefore be located and fixed quickly.

## MS Visual

My programming habits have largely switched from Windows to Unix these last few years, so I'm no longer up to date on this topic.
By default, Visual organizes its list of optional warnings into "levels". The higher the level, the more warnings it generates. It's also possible to opt-in for a single specific warning, but I have not enough experience to comment that usage.

By default, Visual compiler uses [level 1 on command line , and level 3 on IDE](https://docs.microsoft.com/en-us/cpp/build/reference/compiler-option-warning-level?view=vs-2017).
Level 3 is already pretty good, but I recommend to aim for level __4__ if possible. That level will catch additional tricky corner cases, making the code cleaner and more portable.
Obviously, on an existing project, move up progressively towards higher levels, as each of them will generate more warnings to clean up.

The exact way to change the warning level may depend on the IDE version. On command line, it's always `/W4`, so that one is pretty clear. On IDE, it's generally accessible in the `properties->C` tab, which is one of the first displayed, [as shown here](https://www.learncpp.com/cpp-tutorial/configuring-your-compiler-warning-and-error-levels/).

Do not use `/Wall` as part of your regular build process. It contains too many warnings of "informational" value, which are not meant to be suppressed, hence will continuously drown the signal and make "zero warning policy" impossible.


## `gcc` and `clang`

`gcc` and by imitation `clang` offer a command line experience with a large list of compatible flags for warnings.
Overtime, I've developed my own selection, which has become pretty long. I would recommend it to any code base. I'm going to detail it below. It is by no means a "final" or "ultimate" version. The list can always evolve, integrating more flags, either because I missed them, or they end up being more useful than I initially anticipated, or because they become more broadly supported.

For simplicity purpose, I tend to concentrate on flags that are well supported by `gcc` and `clang`, and present since a few revisions. Flags which only work on "latest version of X" are not considered in this list, because they can cause trouble for compilation on targets without version X. This issue can be solved by adding yet another machinery to maintain version-specific flags, complete with its own set of problems, but I would not recommend to start with such complexity.

If your project does not include those flags yet, I suggest to only enable them one after another. A project developed without a specific flag is likely to have used the flagged pattern in many places. It's important to clean one flag completely before moving to next one, otherwise, the list of warnings to fix becomes so large that it will seem insurmountable. Whenever it is, just drop the flag for the time being, you'll come back to it later.

### Basics

- `-Wall`  : This is the "base" warning level for `gcc`/`clang`. In contrast to what its name implies, it does not enable "all" warnings, far from it, but a fairly large set of flags that the compiler team believes is generally safe to follow. For a detailed list of what it includes, you can [consult this page](https://gcc.gnu.org/onlinedocs/gcc/Warning-Options.html), which is likely applicable to latest compiler version. The exact list of flags evolves with the specific version of the compiler. It's even different depending on `gcc` or `clang`. It's okay, because the flag itself doesn't change.
     I would recommend to start with this flag, and get to the bottom of it before moving on to more flags. Should the generated list of warnings be overwhelming, you can break it down into a more narrow set of flags, or selectively disable a few annoying warnings with `-Wno-###`, then plan to re-enable them progressively later.

- `-Wextra` : This is the second level for `gcc` and `clang`. It includes an additional set of flags, which constrains the code style further, improving maintainability. For example, this level will [raise a warning](https://godbolt.org/z/9Axq8E) whenever a [`switch() { case: }` uses a fall-through implicitly](https://godbolt.org/z/pGfevo), which is generally (but not always) a mistake.
This flag used to be called `-W`, but I recommend the `-Wextra` form, which is more explicit.

### Correctness

- `-Wcast-qual` :  This flag ensures that a QUALifier is respected during a cast operation. This is especially important for the `const` "read-only" qualifier: it ensures that a pointer to a read-only area [cannot](https://godbolt.org/z/09ejGE) be [silently transformed into another pointer with write capability](https://godbolt.org/z/T_6Ga9), which is quite an essential guarantee. I even don't quite get it how come this is an optional warning, instead of a compulsory property of the language.

- `-Wcast-align` : the C standard requires that a type must be stored at an address suitable for its alignment restriction. For example, on 32-bits systems, an `int` must be stored at an address which is a multiple of 4. This restriction tends to be forgotten nowadays because x86 cpus have always been good at dealing with unaligned memory accesses, and ARM ones have become better at this game (they used to be terrible). But it's still important to respect this property, for portability, for performance (avoid inter-pages accesses), and for compatibility with deep transformations such as auto-vectorization. Casting can unintentionally violate this condition. A typical example is when [casting a memory area previously reserved as a table of `char*`](https://godbolt.org/z/LIu8pK), hence without any alignment restriction, in order to store `int` value, which require an alignment of 4. `-Wcast-align` [will detect the violation](https://godbolt.org/z/pcPHRw), and fixing it will make sure the code respect alignment restrictions, making it more portable.

- `-Wstrict-aliasing` : [Strict aliasing](https://www.approxion.com/pointers-c-part-iii-strict-aliasing-rule/) is a complex and badly known rule. It states that, in order to achieve better performance, compilers are allowed to consider that 2 pointers of different types never reference the same address space, so their content cannot "collide". If they nonetheless do, it's an [undefined behavior](https://en.wikipedia.org/wiki/Undefined_behavior), hence anything can happen unpredictably.
To ensure this rule is not violated, compilers may optionally offer some code analysis capabilities, that will flag suspicious constructions. `gcc` offers `-Wstrict-aliasing`, with various levels of caution, with `1` being the most paranoid.
Issues related to strict aliasing violation only show up in optimized codes, and are among the most difficult to debug. It's best to avoid them. I recommend using this flag at its maximum setting. If it generates too much noise, try more permissive levels. `-Wstrict-aliasing=3` is already included as part of `-Wall`, so if `-Wall` is already clean, the next logical step is level `2`, then `1`.
One beneficial side-effect of this flag is that it re-inforces the separation of types, which is a safer practice. Cross-casting a memory region with pointers of different types is no longer an easy option, as it gets immediately flagged by the compiler. There are still ways to achieve this, primarily through the use of `void*` memory segments, which act as wildcards. But the extra-care required is in itself protective, and should remind the developer of the risks involved.

- `-Wpointer-arith` forbids pointer arithmetic on `void*` or function pointer. C unfortunately lacks the concept of "memory unit", so a `void*` is not a pointer to an address: it's pointer to an object "we don't know anything about". Pointer arithmetic is closely related to the concept of table, and adding `+1` is always relative to the size of the table element (which must be a constant). With `void*`, we have no idea what this element size could be, so it's not possible to `+1` it, nor do more complex pointer arithmetic.
To perform operation on bytes, it's necessary to use a pointer to a byte type, be it `char*`, `unsigned char*` or `int8_t*`.
This is a strict interpretation of the standard, and helps make the resulting code more portable.

### Variable declaration

- `-Winit-self` : prevents a fairly silly corner case, where a variable is initialized with itself, such as `int i = i+1;`, which can not be right. `clang` and `g++` make it part of `-Wall`, but not `gcc`.

- `-Wshadow` : A variable `v` declared at a deep nesting level shadows any other variable with same name `v` declared at an upper level. This means that invoking `v`at the deeper level will target the deeper `v`. This is legal from a C standard perspective, but it's considered bad practice, because it's confusing for the reviewer. Now 2 different variables with different roles and lifetime carry the same name. It's better to differentiate them, by using different names.
_Sidenote_ : name shadowing can be annoying when using a library which unfortunately defines very common symbol names as part of its interface. Don't forget that the C namespace is global. For this reason, whenever publishing an API, always ensure that no public symbol is too "common" (such as `i`, `min`, `max`, etc.). At a minimum, add a `PREFIX_` to the public symbol name, so that opportunities of collision get drastically reduced.

- `-Wswitch-enum` : This flag ensures that, in a `switch(enum) { case: }` construction, all declared values of the `enum` have a `case:` branch. This can be useful to ensure that no `enum` value has been forgotten (even if there is a `default:` branch down the list to deal with them). Forgetting an `enum` value is a fairly common scenario when the `enum` list changes, typically by adding an element to it. The flag will issue a warning on all relevant `switch() { case: }`, simplifying code traversal to ensure that no case has been missed.


### Functions

- `-Wstrict-prototypes` : historically, C functions used to be declared with just their name, without even detailing their parameters. This is considered bad practice nowadays, and this flag will ensure that a function is declared with a fully formed prototype, including all parameter types.
A common side effect happens for functions without any parameter. Declaring them as `int function()` seems to mean "this function takes no argument", but it's not correct. Due to this historical background, it actually means "this function may have any number of arguments of any type, it's just not documented". Such definition will limit the effectiveness of the compiler in controlling the validity of an invocation, so it's bad, and this flag will issue a warning. The correct way to tell that a function has no (zero) argument is `int function(void)`.

- `-Wmissing-prototypes` :  this flag enforces that any public function (non-`static`) has a declaration somewhere. It's easy to game that condition by just writing a prototype declaration right in front of the function definition itself, but it misses the point : this flag will help find functions which are (likely) no longer useful.
The problem with public functions is that the compiler has no way to ensure they are not used anymore. So it will generate them, and wait for the linking stage to know more. In a library, such "ghost" function will be present, occupy valuable space, and more importantly will still offer a public symbol to be reached, remaining within the library's attack surface, and offering a potential backdoor for would-be attackers. Being no longer used, these functions may also not be correctly tested anymore, and might allow unintended state manipulations. So it's better to get rid of them.
If a kind of "private function just for internal tests" is needed, and should not be exposed in the official `*.h` header, create a secondary header, like `*-debug.h` for example, where the function is declared. And obviously `#include` it in the `*.c` unit. This will be cleaner and compatible with this flag.

- `-Wredundant-decls` : A prototype should be declared only once, and this single definition should be `#include` everywhere it's needed. This policy avoids multiple source of truth, with associated synchronization problems.
This flag will trigger a warning if it detects that a function prototype is declared twice (or more).

### Floating point

- `-Wfloat-equal` : this flag prevents usage of `==` equality operator between `float` value. This is because floating point values are lossy representations of real numbers, and any operation with them will incur an inaccuracy uncertainty, which exact detail depends on target platform, hence is not portable. Two floating-point values should not be compared with equality, it's not supposed to make sense given the lossy nature of the representation. Rather ensure that the distance between 2 floats is below a certain threshold to consider them "equivalent enough".

### Preprocessor

- `-Wundef` : forbids evaluation of a macro symbol that's not defined. Without it, `#if SYMBOL_NOT_EXIST` is silently translated into `#if 0`, which may or may not generate the intended outcome. This is useful when the list of macro symbols evolves : whenever a macro symbol disappears, all related preprocessor tests get flagged with this warning, which makes it possible to review and adapt them.

### Standard Library

- `-Wformat=2` : this will track potential `printf()` issues which can be abused to create security hazard scenarios.
An example is when the formatting chain itself can be under control of an external source, such as `printf(message)`, with `char* message` being externally manipulated. This can be used to read _and write_ out of bound and take remote control of the system. Yep, it's that dangerous.
The solution to this specific issue is to write `printf("%s", message)`. It may look equivalent, but this second version is safer, as it interprets `message` only as a pure `char*` string to display, instead of a formatting string which can trigger read/write orders from inside `printf()`.
`-Wformat=2` will flag this issue, and many more, such as ensuring proper correspondence between the argument type and control string statement, leading to a safer program.
These issues go beyond the C language proper, and more into `stdio` library territory, but it's good to enable more options to be protected from this side too.

### Extended compatibility

- `-Wvla` : prevents usage of Variable Length Array.
VLA were supported in `C99`, but are now optional since `C11` (support can be tested using `__STDC_NO_VLA__` macro). They allow nice things such as allocating on stack a table of variable size, depending on a function parameter. However, VLA have a pretty poor reputation. I suspect a reason is that they were served by sub-par implementations leading to all sort of hard issues, such as undetected stack-overflow, with unpredictable consequences.
Note that even "good" implementations, able to dynamically expand stack to make room for larger tables, and correctly detect overflow issue to properly `abort()`, cannot provide any way for the program to be informed of such issue and react accordingly. It makes it impossible to create a program that is guaranteed to not `abort()`.  
For better portability, it's enough to know that some implementations of VLA are/were poor, and that VLA is no longer required in `C11` to justify avoiding it. VLA is also not available for `C90`.

- `-Wdeclaration-after-statement` : this flag is useful for `C90` compatibility. Having all declarations at the top of the block, before any statement, is a strict rule that was dropped with `C99`, and it's now possible to declare new variables anywhere in a block. This flag is mostly useful if the goal is to be compatible with `C90` compilers, such as MS Visual Studio C before 2015  as an example.

- `-Wc++-compat` : this flag ensures that the source can be compiled unmodified as both valid C and C++ code. This will require a few additional restrictions, such as casting from `void*`, which is unnecessary in C, but required in C++.
This it handy for highly portable code, because it's not uncommon for some users to just import the source file in their project and compile it as a C++ file, even though it's clearly labelled as C. Moreover, when targeting `C90` compatibility, `C++` compatibility is not too far away, so the remaining effort is moderate.

### Discovering new flags

While it's not recommended to use too many warnings in the production build chain, it can be sometimes interesting to look at more options. Special mention can be given to `-Weverything` on `clang`, which will activate every possible warning flag.
Now, [`-Weverything` is not meant to be used in production](https://quuxplusone.github.io/blog/2018/12/06/dont-use-weverything/). It's mostly a convenient "discovery" feature for `clang` developers, which can track and understand new warnings as they are added to "trunk".
But for the purpose of testing if the compiler can help find new issues, it can be an interesting temporary digression. One or two of these warnings might uncover real issues, inviting to re-assess the list of warning flags used in production.

### Summary

All the flags presented so far can be combined into the following list, provided below for copy-pasting purposes :
`-Wall -Wextra -Wcast-qual -Wcast-align -Wstrict-aliasing -Wpointer-arith -Winit-self -Wshadow -Wswitch-enum -Wstrict-prototypes -Wmissing-prototypes -Wredundant-decls -Wfloat-equal -Wundef -Wvla -Wdeclaration-after-statement -Wc++-compat`

Quite a mouthful. Adopting as-is this list into an existing project might result in an abundant list of warnings if they were not already part of the build. Don't be afraid, your code is not completely broken, but consider having a look: it might be fragile in subtle ways that these flags will help find. Enable additional warnings one by one, selectively, pick those which add value to your project. In the long run, these flags will help keep the code better maintained.

Compiler warning flags can be seen as a giant list of patterns that the compiler is pre-trained to detect. It's great. But beyond these pre-defined capabilities, one might be interested in adding one's own set of conditions for the compiler to check and enforce. That's the purpose of next blog post.

#### Special Thanks
An early version of this article was commented by Nick Terrell and Evan Nemerson.
