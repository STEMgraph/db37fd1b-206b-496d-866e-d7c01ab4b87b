<!---
{
  "id": "db37fd1b-206b-496d-866e-d7c01ab4b87b",
  "teaches": "GNU Make: Automatic Header Dependencies with gcc -MMD",
  "depends_on": [
    "0b44c096-b285-4c06-b5ca-7480f72ca524"
  ],
  "author": "Stephan Bökelmann",
  "first_used": "2026-10-02",
  "keywords": [
    "make",
    "gcc -MMD",
    "-MP",
    "dependency files",
    "include",
    "header dependencies"
  ]
}
--->

# GNU Make: Automatic Header Dependencies with `gcc -MMD`

> In this exercise you will stop listing header files by hand. The compiler will write a small dependency file next to every object file, and the `Makefile` will include those files, so that a changed header always triggers exactly the right rebuilds. No `sudo` is required.

## Introduction

In the exercise on dependencies you saw what happens when a header is missing from the dependency list: `make` says "up to date" and the program is built from stale objects. Writing `main.o: vec2.h util.h` by hand works for three files and breaks down at thirty, because nobody keeps those lists in sync with the `#include` lines.

The compiler already knows every header it opened. With `-MMD` it writes that knowledge into a file `name.d` in `make` syntax, as a side effect of normal compilation:

```make
main.o: main.c vec2.h util.h
```

The `Makefile` only has to `-include` those files. The leading minus means "do not complain if the file does not exist yet", which is the case on the very first build. `-MP` adds an empty rule for every header, so that deleting a header does not make `make` fail with "no rule to make target".

### 1.1) Further Readings and Other Sources

* [GCC Manual: Preprocessor Options (`-MMD`, `-MP`, `-MF`)](https://gcc.gnu.org/onlinedocs/gcc/Preprocessor-Options.html)
* [GNU Make Manual: Automatic Prerequisites](https://www.gnu.org/software/make/manual/html_node/Automatic-Prerequisites.html)
* [GNU Make Manual: Including Other Makefiles](https://www.gnu.org/software/make/manual/html_node/Include.html)

## Tasks

### Task 1: Look at What the Compiler Knows

Start in `~/make-intro` from the previous exercises. Ask the compiler for the dependencies of `main.c` without compiling:

```bash
gcc -MM main.c
gcc -M main.c | head -5
```

`-MM` lists only your own headers, `-M` also the system headers. Which of the two do you want in a `Makefile`, and why?

### Task 2: Generate Dependency Files as a Side Effect

Add `-MMD -MP` to the compiler flags and remove every hand-written header rule such as `vec2.o: vec2.h`:

```make
CC      = gcc
CFLAGS  = -Wall -Wextra -O2 -MMD -MP
SRCS   := $(wildcard *.c)
OBJS   := $(SRCS:.c=.o)
DEPS   := $(OBJS:.o=.d)

all: main_program

main_program: main.o libvec2.a libutil.a
	$(CC) $< -o $@ -L. -lvec2 -lutil -lm

libvec2.a: vec2.o
	ar rcs $@ $^

libutil.a: util.o
	ar rcs $@ $^

%.o: %.c
	$(CC) $(CFLAGS) -c $< -o $@

clean:
	rm -f $(OBJS) $(DEPS) *.a main_program

.PHONY: all clean
```

```bash
make clean && make
ls *.d
cat main.d
```

### Task 3: Include the Dependency Files

Append to the `Makefile`:

```make
-include $(DEPS)
```

```bash
make
touch vec2.h
make
```

Both `vec2.o` and `main.o` are rebuilt, although no rule in your file mentions `vec2.h`. Remove the leading `-`, run `make clean && make`, and read the error. Put the `-` back.

### Task 4: A New Header Appears

Create a header that `util.c` starts to include:

```c
// file: util.h
#ifndef UTIL_H
#define UTIL_H
#define UTIL_FACTOR 2
int UTIL_twice(int x);
#endif
```

```c
// file: util.c
#include "util.h"
int UTIL_twice(int x) { return UTIL_FACTOR * x; }
```

```bash
make
cat util.d
sed -i 's/UTIL_FACTOR 2/UTIL_FACTOR 3/' util.h
make
```

The first `make` rebuilt `util.o` because `util.c` changed and wrote a new `util.d`. The second `make` rebuilt it again because `util.h` is now known. Nobody edited the `Makefile`.

### Task 5: A Header Disappears

```bash
sed -i '/#include "util.h"/d; s/UTIL_FACTOR/2/' util.c
rm util.h
make
```

Without `-MP` this would fail: `util.d` still names `util.h` as a prerequisite and `make` has no rule to create it. Remove `-MP` from `CFLAGS`, run `make clean && make`, recreate and delete `util.h` as above, and watch the error. Put `-MP` back.

### Task 6: Where Do the `.d` Files Go?

The `.d` files clutter the source directory. Tell the compiler where to write them with `-MF`:

```make
CFLAGS  = -Wall -Wextra -O2 -MMD -MP -MF .deps/$*.d
DEPS   := $(patsubst %.c,.deps/%.d,$(SRCS))

%.o: %.c | .deps
	$(CC) $(CFLAGS) -c $< -o $@

.deps:
	mkdir -p $@
```

Adjust `clean` to `rm -rf .deps` and verify with `make clean && make && ls -a .deps`. `$*` is the stem matched by `%` in the pattern rule.

### Task 7: Compare With the Dry Run

```bash
make clean && make
touch vec2.h
make -n
```

Explain, line by line, why each command in the dry run is necessary.

## Questions

- What is the difference between `-M`, `-MM`, `-MD` and `-MMD`? Which one also produces an object file?
- Why must the dependency files be included at the *end* of the `Makefile`, not at the top?
- The first build has no `.d` files yet. Why is that not a problem?
- What does `-MP` generate, and which error does it prevent?
- Why is `.deps` an order-only prerequisite in Task 6?
- The dependency file is written *during* compilation. What happens if the compilation fails halfway through? Look up `-MT` and think about why some projects generate dependency files in a separate step.

## Advice

`-MMD -MP` plus `-include $(DEPS)` is five tokens that turn a fragile `Makefile` into a correct one. Almost every hand-written `Makefile` for C or C++ that you will meet in the wild contains exactly this pattern, sometimes with `-MT` or `-MF` on top. CMake and Ninja do the same thing internally; when you move on to CMake you will find `.d` files or their equivalent in the build directory.

If you remember one thing from this exercise: never list headers by hand again.
