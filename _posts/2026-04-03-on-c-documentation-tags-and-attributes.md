---
title: On C documentation, tags and attributes
date: 2026-04-03 19:00:00 +0200
categories: [Blogging, Programming languages]
tags: [c, documentation]
---

While working on my recent project (a small compiler in C), I've become more and more
frustrated with how difficult it is to remember what functions do. Reading
documentation (which you usually do not even write at the beginning of a project)
every time you want to use a function is unpleasant - takes time, requires you to
read (who likes reading?), and just annoys you. Even if you have documented your
code, or if you're using an external function that has documentation, sometimes it
might be either too big for you to find what you need, or too small so that you do
not find what you are looking for.

# Example 1

For example, say you want to use strdup and want to check what happens if you pass a
NULL pointer. Running [`man 3
strdup`](https://man7.org/linux/man-pages/man3/strdup.3.html) will not answer that
question; neither will
[`cppreference`](https://en.cppreference.com/w/c/experimental/dynamic/strdup). In
fact, I was not able to find what happens in that case anywhere on the internet
(aside from ChatGPT, which told me it results in UB - how nice).

Signature for `strdup` is given as:
```c
char *strdup(const char *s)
```

What can we learn about `strdup` based on this single line of code?

* That it will not modify the provided string
* That the returned string can be modified

Sadly, that is pretty much it. We have to check the documentation if we want to see
some other details. Without looking for the docs, we cannot tell if

* the function might free `s`
* the function can work with `s` being `NULL`
* the returned string can be `NULL`

Now, what if the signature was different in a way which would allow us to learn more
about the function? Take a look at this and try to answer all 3 questions that were
not possibly answerable before.
```c
char *YESNULL MUSTFREE strdup(const char *s NOFREE YESNULL);
```

Now, that is a totally different story! The function clearly does not free `s` and
can return a `NULL` string which must be freed, and *likely* returns `NULL` if the
input string string is also `NULL`.

We did not need to look at the documentation; the function signature itself answers
*most* of the questions we may have!

# Example 2

Let's take a look at another (made up for this post) function:
```c
int add_to_list(list_t *list, item_t *item);
```

This function is from a library working with dynamic lists, and adds a new item to
the end of the list. Now, say you do not read the documentation, but want to use it.
The questions are immediately popping up in my head:

* What if any of the arguments are `NULL`?
* What is returned? Length, true/false, something else?
* What if memory allocations fails?
* What happens to `item` after it is added to the list?

Let us apply our "tags" (as I like to call them) to this function:
```c
int STATUSCODE
add_to_list(list_t *NONULL list, item_t *NONULL SINK item) MEMSAFE;
```

Now, we can see that:

* Arguments must not be `NULL`
* The function returns a status code - which, in context of C, means that 0 is
  success and everything else is a failure
* `SINK` here is taken from Nim - where it means that the data is essentially 'moved'
  (i.e. you lose ownership of it). Therefore, `item` is not copied, and is instead
moved to the list
* `MEMSAFE` at the end of the function means that the function is safe in context of
  memory - definition of the `MEMSAFE` tag is that "in case of a memory allocation
failure the program will crash with a message about the failure"

Admittedly, one has to know what each tag means if one wants to understand the
signature. However, once you learn what each tag is, you should have no issues with
understanding functions in seconds.

# Tags

In my project, I introduced [several
tags](https://github.com/lis05/bbb/blob/main/src/common/tags.h) defined as macros
that expand into nothing and carry information that only the developer can / should
use. Here are some of them with their definitions (reworded because my original
definitions are so-so):
```c
/* The subject should never be null, otherwise UB will happen. */
#define NONULL

/* Explicitly specifies that the subject being NULL is a well defined behavior,
and that it will not cause a crash / UB. */
#define YESNULL

/* The subject originates from a static buffer and will likely be corrupted after
the subsequent function call; use immediately. */
#define ONETIME

/* Ownership over the subject will be transferred to the callee; after passing
this argument you are no longer the owner of it. */
#define SINK

/* The subject must not be freed by you. */
#define NOFREE

/* The subject must be freed by you! */
#define MUSTFREE

/* If memory allocations fail, the program will crash with a relevant message and
the function will not return. */
#define MEMSAFE

/* Integer is a status code (0 is success, everything else is failure). */
#define STATUSCODE

/* Integer is a boolean (1 is true, 0 is false). */
#define BOOL
```

I've started adapting the codebase to use these tags extensively. While it is a
difficult process (since rewriting half your code is BAD), there are some examples of
it.

[Safe memory related
functions](https://github.com/lis05/bbb/blob/main/src/common/mem.h):
```c
void *NONULL MUSTFREE memdup_safe(const void *ptr NONULL, size_t size) MEMSAFE;
void *NONULL MUSTFREE realloc_safe(void *ptr NONULL SINK, size_t size) MEMSAFE;
```

[Working with scopes](https://github.com/lis05/bbb/blob/main/src/codegen/scope.h)
(i.e. looking up a variable in a scope):
```c
int BOOL scope_has(struct scope_t *scope NONULL, pstr_t token NONULL) MEMSAFE;
```

[Generating labels for assembly
code](https://github.com/lis05/bbb/blob/main/src/codegen/lblg.h):
```c
const char *NONULL NOFREE lblg_gen(struct label_generator_t *lblg NONULL) MEMSAFE;
```

Even if you do not know my codebase, you can tell that `lblg_gen` returns something
you must not free (`NOFREE`), `scope_has` returns a boolean instead of a status code,
and `realloc_safe` does not tolerate NULL pointers. All this without ever reading
(nonexistent) documentation!

One could argue that returning `int BOOL` instead of typedefing a custom boolean type
(or using the builtin ones) is stupid. While I think this is a valid solution, I like
my tags more as they do not limit us to a single type (i.e. you can return both `int
BOOL` and `long BOOL` without having to define two separate types).

I should also mention that tags work like pointers, which mean that they apply to
everything to the left of them. Therefore, you can write the following:
```c
char *NONULL MUSTFREE *YESNULL NOFREE ptr;
// pointer that may be NULL, must not be freed, and points to an array of char*
// that must not be null and must be freed.
```

Since the type ends with the identifier, you can place the 'top-level' tags either
before or after the identifier. I like placing them after, but it is just a matter of
choice.

Another thing is that you can apply tags to local variables and structures, not just
to functions:
```c
// array of status codes, like a result of calling N programs
struct array {
    int STATUSCODE *ptr NONULL MUSTFREE;
    size_t n;
};
```

# Attributes

Modern compilers support attributes, which are in a way similar to my tags. For
example, these can be effectively used as `NONULL` / `MUSTFREE`
```c
[[gnu::nonnull(1)]]
int foo(char *p); // p may not be NULL, basically NONULL

[[gnu::malloc]]
[[gnu::alloc_size(1)]]
void *alloc(size_t n); // function is malloc-like, kinda like MUSTFREE
```

There are many more tags, and almost all of them are targeted at compilers to let
them optimize our code more efficiently. While they are somewhat similar, I believe
they serve a different purpose than tags, which are for the developers to understand
code faster.

# Afterthought

I am **hardly** an experienced C programmer, and I am sure that someone has already
come up with the idea of tags - and I will gladly hear that this whole post is
reinventing something mentioned in some old post on a long forgotten forum 30 years
ago. However, I felt that this was something cool that I wanted to share, and that it
might be interested to those who like C and its limitations.

That being said, tags should not be a replacement of documentation - they are just a
way to let the programmers write code easier, without having to waste time looking
for stuff that might be encoded in the function signature itself.

Thanks for reading and have a good day!
