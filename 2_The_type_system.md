
The type system is a simple yet powerful way to ensure that the code knows enough about the data it is manipulating. It is used to declare conditions at interfaces, which are then enforced at each invocation.
The compiler is very good to check types. The condition is trivial to enforce, and doesn't cost much compilation resource, so it's a powerful combination.

## `typedef` and weak types

C is sometimes labelled a "weakly typed" language, presumably because it is associated to the behavior of one of its keywords `typedef`.
The keyword itself implies that `typedef` DEFines a new TYPE, but it's unfortunately a misnomer.

As an example, `typedef` can be used this way :
```C
typedef int meters;
typedef int kilograms;
```
This defines 2 new "types", `meters` and `kilograms`, which can be used to declare variables.
```C
meters m;
kilograms k;
```
One could logically expect that, from now on, it's no longer allowed to mix `meters` and `kilograms`, since they represent different types, hence should not be compatible.

Unfortunately, that's not the case : `meters` and `kilograms` are still considered as `int` from the C type system perspective, and [mixing them works](https://godbolt.org/z/UOFA0g) without a single warning, even when every possible compiler warning is enabled.

As such, `typedef` must be considered a mere tagging system. It's still useful from a code reviewer perspective, since it has documenting value, and that may help notice discrepancies, such as this example. But the compiler won't be able to provide any signal.

## Strong types in C

To ensure that two types cannot be accidentally mixed, it's necessary to strongly separate them. And that's actually possible.
C has one thing called a `struct` (and its remote relative called `union`).
Two `struct` defined independently are considered completely foreign, even if they contain _exactly the same members_.
[They can't be mixed unintentionally](https://godbolt.org/z/UVdyLi).

This gives us a basic tool to strongly segregate types.

### Operations

Using `struct` comes with severe limitations. To begin with, the set of default operations is much more restricted. It's possible to allocate `struct` on stack, and make it part of a larger `struct`, it's possible to assign with `=` or `memcpy()`, but that's pretty much it. No simple operation like `+ - * /`, no comparison like `< <= => >`, etc.

Users may also access members directly, and manipulate them. But it breaks the abstraction.
When structures are used as a kind of "bag of variables", to simplify transport, and enforce naming for clarity, it's fine to let users access members directly. Compared to a function with a ton of parameters, an equivalent function with a structure as input will help readability tremendously, just because it enforces naming parameters.
But in the present case, when structures are used to enforce abstractions, users should be clearly discouraged from accessing members directly. Which means, all operations must be achieved at `struct` level directly.

To comply with these limitations, it's now necessary to create all allowed operations one by one, giving a uniquely named symbol to each one. So if `meters` and `kilograms` can be added, both operations need their own function signature, such as `add_meters()` and `add_kilograms()`. This feels like a hindrance, and indeed, if there are many types to populate, it can require a lot of glue code.

But on the plus side, only what's allowed is now possible. For example, multiplying `meters` with `meters` shouldn't produce some `meters`, but rather a `square_meters` surface, which is a different concept. Allowing additions, but not multiplications, is an impossible subtlety for basic `typedef`.

### Composition

There is no "intermediate" situation, where a type would be "compatible" with another type, yet different. In the mechanisms explained so far, types are either compatible and identical, using `typedef`, or completely incompatible, using a new definition and a new name.

In contrast, in Object Oriented languages, a `cat` can also be an `animal`, thanks to inheritance, so it's possible to use `cat` to invoke `animal` methods, or use functions with `animal` parameter(s).

`struct` strongly leans towards composition. A `struct cat` can include a `struct animal`, which makes it possible to invoke `animal` related functions, though it's not transparent : it's necessary to explicitly spell the substructure (`cat.animal`) as a parameter or return value of the `animal` related function.

Note that even Object Oriented languages generally approve the [composition over inheritance](https://en.wikipedia.org/wiki/Composition_over_inheritance) guiding principle. The guiding principle states that, if there isn't a very good reason to employ inheritance, composition must always be preferred, because it generally fares better as the code evolves and becomes more complex (multiple inheritances quickly translate into a nightmare).

`struct` can be made more complex, with tables of virtual function pointers, achieving something similar to inheritance and polymorphism. But this is a whole different level of complexity. I will rather avoid this route for the time being. The current goal is merely to separate types in a way which can be checked by the compiler. Enforcing a unified interface on top of different types is a more complex topic, better left for a future article.

## Opaque types

`struct` are fine as strong types, but publishing their definition implies that their members are public, meaning any user can access and modify them.
When it's the goal, it's totally fine.

But sometimes, one could wish that, in order to protect users from unintentional mis-usage, it would be better to make structure members unreachable. This is called an [opaque type](https://en.wikipedia.org/wiki/Opaque_data_type). An additional benefit is that whichever is inaccessible cannot be relied upon, hence may be changed in the future without breaking user code.

Object oriented language have the `private` tag, which allows exactly that : some members might be published, but they are nonetheless unreachable from the user (well, [in theory](http://bloglitb.blogspot.com/2011/12/access-to-private-members-safer.html)...).

A "poor man" equivalent solution in C is to comment the code, clearly indicating which members are public, and which ones are private. No guarantee can be enforced by the compiler, but it's still a good indication for users.
Another step is to give private members terrible names, such as `never_ever_access_me`, which provides a pretty serious hint, and is less easy to forget than a code comment.

Yet, sometimes, one wishes to rely on stronger compiler-backed guarantee, to ensure that no user will access private structure members. C doesn't have `private`, but can do something equivalent.
It relies on the principles of [incomplete type](https://docs.oracle.com/cd/E19205-01/819-5265/bjals/index.html).

My own preference is to declare an incomplete type by pairing it with `typedef` :
```C
typedef struct house_s house;
typedef struct car_s car;
```

Notice that we have not published anything about the internals of `struct house_s`. This is intentional. Since nothing is published, nothing can be accessed, hence nothing can be misused.

Fine, but what can we do about such a thing ? To begin with, we can't even allocate it, since its size is not known.
That's right, the only thing that can be declared at this stage is a pointer to the incomplete type, like this :
```C
house* my_house;
car* my_car;
```

And now ?
Well, only functions with `house*` or `car*` as parameter or return type can actually do something with it.
These functions must access `struct house_s` and `struct car_s` internal definitions. These definitions are therefore published in a relevant unit `*.c` file, rather than the header `*.h`. Being not part of the public interface, the structure's internal remains effectively private.

The first functions required are allocators and destructors.
For example, I'm used to the following name convention :
```C
thing* PREFIX_createThing();
void PREFIX_freeThing(thing* t);
```

Now, it's possible to allocate space for `thing*`, and eventually do something with it (with additional functions).
A good convention is that functions which accept `thing*` as mutable argument should have `thing*` as first parameter, like in this example :
```C
int PREFIX_pushElement(thing* t, element e);
element PREFIX_pullElement(thing* t);
```
Notice that we are getting pretty close to object oriented programming with this construction. Functions and data members, while not declared in an encompassing "object", must nonetheless be defined together: the need to know the structure content to do anything about it forces function definitions to be grouped  into the unit that declares the structure content. It's fairly close.

Compared with a direct `struct`, a few differences stand out :
- Members are private
- Allocation is implemented by a function, it can only be invoked
 	- no way to allocate on stack
 	- no way to include a `thing` into another `struct`
 	     - but it's possible to include a pointer `thing*`
    - Initialization can be enforced directly in the constructor
        - removes risks of garbage content due to lack of initialization.
- The caller _is in charge of invoking the destructor_.
    - The pattern is exactly identical to `malloc()` / `free()` (see future article on [Resource Control]())

The responsibility to invoke the destructor after usage is very important.
It's no different than invoking `free()` after a `malloc()`,
but that's still an additional detail to take care of, with the corresponding risk to forget or mismanage it.

To bypass this responsibility, and take control of the allocation process, it can be preferable to consider opaque types with static allocation. That's the topic of the next article.

## Summary

This closes this first chapter on the type system. We have seen that it's possible to create [strong types](https://en.wikipedia.org/wiki/Strong_and_weak_typing), and we can use this property to ensure users can't mix up different types accidentally. We have seen that it's possible to create [opaque types](https://en.wikipedia.org/wiki/Opaque_data_type), and ensure users can only invoke allowed operations, or can't rely on secret internal details, clearing the path of future evolution. These properties are compiler-checked, so they are always automatically enforced.

That's not bad. Just using these properties will seriously improve code resistance to potential mis-usages.

Yet, there is more the compiler can do to detect potential bugs in our code. To be continued...
