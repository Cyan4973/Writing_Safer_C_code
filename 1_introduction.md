Writing safer C code may feel like an overwhelming goal. After all, After all, we are told that C gives programmers plenty of opportunities to [shoot their own foot](http://www.stroustrup.com/bs_faq.html#really-say-that).

But that's doesn't mean there is no possible improvement. Actually, in the last decade, programming practices have already evolved dramatically, and for the better, as a consequence of multiple forces, such as improved tooling, shared programming and rising cost of failures, as the numerous Internet exploits tend to remind us all too often.

I expected to start this series with an introduction on C, its strengths, and guiding principles on safer coding practices. But it doesn't fit the blog post format, being too long, boring, and at times potential troll magnet. Suffice to say that "safer" implies writing Reviewer-Oriented source code, aka highly readable, and as much error automation as possible, favoring fast methods (immediate feedback while editing code) over longer ones (long offsite test sessions in dedicated environments).

One thing I can't escape though is to mention a few words on the intended audience. These articles are not meant to learn new things for "experts", which know a lot more than I do. Neither are they intended to guide the first steps towards C programming. The intended audience has good enough C programming skills, and can actually ship products. Shipping real products is important, because the whole concept of "safer programming" is better understood under the pressure and experience of a product's maintenance.

The main driver is to make it more difficult to ship bugs, as the code base lives and evolves, and new team members get onboard, adding much-needed automated controls at every opportunity. Issues are centered around modifying / fixing an existing code base, and managing the cascading impacts on the rest of the project. This requires to prepare the code for this challenge, hence the design patterns proposed are also useful for new codes with an expected "long" life expectancy (beyond a few months).

Now let's shorten this introduction and go directly into the meat of the topic.
I'll start this series with design patterns that leverage compiler checks, to help make C code more resistant to mis-usages and future refactoring.

As a quick background, the compiler is a fairly central part of the development process for compiled language. Compiling a source code incurs a delay, more or less noticeable. That's a cost.
Interpreted languages (most scripts, `python`, `ruby`, `basic`, `bash`, etc.) can evade it, making the initial code writing experience more agreeable, with quick modification / experience feedback loop.
The real cost though comes later, and it is steep : compiled languages have this constraint that the compiler must understand and therefore sanitize the code in order to produce the executable binary. This constraint becomes a huge advantage as it catches many categories of errors before they get a chance to run. This typically includes many flavors of mis-typings. Interpreted languages, in contrast, will have to find a majority of problems at run time (_note:_ a good editor's parser can definitely help both language types there).

And compiler can go much farther. One of the big lessons from modern languages favoring safety like `rust` is that using the compiler as a primary tool to guide design patterns towards safer practices improves code quality substantially. It's a good choice : the compiler is a compulsory part of the development chain, it sits close to the programmer, its diagnosis is part of the valuable "short" feedback loop (in contrast with complementary techniques such as code analyzers, test suites and sanitizers). Whatever the compiler can flag gets solved more quickly, reducing load and risks at later stages of the development.

Hence, this is the first topic to explore : let's make the compiler work for us,  check the validity of our code to the best of its abilities. To reach that goal, we will have to purposely leverage its capabilities, in effect help the compiler help us.

And let's start with its first weapon, the type system.
