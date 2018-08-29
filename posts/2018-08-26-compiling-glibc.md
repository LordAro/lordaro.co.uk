extends: default.liquid

title: Compiling older glibc versions
date: 26 August 2018 21:30:00 +1000
---

I recently had need to compile glibc 2.5 on an up to date system. This article explains why, what, and how.

## Why?

The GNU C Library, or glibc, is C's standard library. Specifically, it is the GNU version and is the most common
implementation in Linux-land. As a libc implementation, it contains implementations of common functions like `printf`
all the way down to some functions you've not heard of, like `arc4random_uniform`. As such, nearly every compiled binary
on your system links against this library.

Since newer operating systems normally come with a newer version of libc, programs compiled with this newer version of
libc will not work with older systems as the library's internals will be different - functions will have different
signatures, or will just be missing. To provide binaries that have maximum compatibility with older systems, you often
have to find a system with the minimum system requirement and let backwards compatibility mean you can use that same
binary on newer platforms. This can be an issue, as to build your program on the older platform means you need to use
the older compiler toolchain from that system. Which can be a problem if you're missing features (i.e. newer language
features not supported by older compilers), missing functionality in other areas like optimisations or basic stuff like
editors or debuggers not working on the older system.

## What?

Luckily, there is another option. Implementations of libc often "version" the function symbols that programs link
against, allowing a specific version of the function to be used. This is also how they achieve the backwards
compatibility when you have an older program with a newer version of the library.

Using a `.symver` directive in a header file, you can force the linker to use a specific symbol for a function. For
example, glibc contains two different versions of the `realpath` function:

     > objdump -T /usr/lib/libc.so.6 | grep realpath
    0000000000131d10 g    DF .text	0000000000000021 (GLIBC_2.2.5) realpath
    0000000000041fd0 g    DF .text	0000000000000593  GLIBC_2.3   realpath

Normally, `realpath` would link to the GLIBC_2.3 version. But with a .symver directive such as

    __asm__(".symver realpath,realpath@GLIBC_2.2.5");

will force the program to call the older version of the function instead.

By adding a load of these together in a header file, you can build an entire project so that it only uses functions from
an older libc.

## How?

My end goal for this project was a compatibility header for glibc 2.5, the version that is distributed with CentOS 5, or
Red Hat 5, which somehow still has something resembling support until 2020, despite being over a decade old.

Luckily, there was an [existing project](https://github.com/wheybags/glibc_version_header) that built header files down
to glibc 2.13, the version included with Ubuntu 11.04. It was also helpful in resolving an issue where the `clock_*`
functions had been moved from librt to libc around glibc 2.17. This was a good start, but there was a problem - while
the newest versions compiled fine, it didn't compile some older versions, starting with 2.25.

### 2.25

Turns out that my Linux install was too new - the installation of binutils was causing issues. More accurately, older
versions of `ld`, the linker, ironically enough was doing something wrong with the `symver` directive, adding symbols
when it shouldn't have been. This was fixed by adding

[Avoid .symver on common symbols \[BZ #21666\]](https://sourceware.org/git/gitweb.cgi?p=glibc.git;a=commitdiff;h=388b4f1a02f3a801965028bbfcd48d905638b797)

### 2.21

Modern GCC (5.4+) enables PIE by default. PIE (Position-independent executable) is a form of Address Space Layout
Randomisation (ASLR) as implemented by the Linux kernel, which helps in preventing side-channel attacks that examine the
contents of memory at particular addresses. This does not play well with glibc, causing symbol relocation errors when
linking, so must be disabled by setting the LDFLAG `-no-pie`. Newer versions disable it in the build system.

### 2.18

A common problem amongst all the older versions starting with 2.18, was that glibc's autotools-generated configure
script wasn't able to properly detect the newer versions of the build system installed - `gcc`, `ld` & `make` - all
rather important parts of the build system. These checks were simply commented out and ignored.

### 2.17

    Error: `_obstack@GLIBC_2.2.5' can't be versioned to common symbol '_obstack_compat'

This was another issue relating to the newer binutils install. Turns out that all was needed was to initialise the
`_obstack_compat` pointer to 0, or `NULL`. This variable also comes with the comment "A looong time ago (before 1994,
anyway; we're not sure) this global variable was used by non-GNU-C macros to avoid multiple evaluation. The GNU C
library still exports it because somebody might use it." Lovely legacy code. Mmm.

### 2.16

Newer versions of the GCC compiler have the "stack protector" turned on by default, which protects against stack frame
misuse. These issues are fixed in newer glibc, but for older versions we can to turn it off with `-fno-stack-protector`.
Similarly, `_FORTIFY_SOURCE` needs to be disabled, with `-U_FORTIFY_SOURCE`. The optimisation flag goes in as normal.

Only an issue with newer binutils, there was a single `#include <rpc/types.h>` that expected the header to be installed
on the system. This no longer became the case (on my system), but the "original" copy of it is in the repo anyway, so
just switching the statement to `#include "rpc/types.h"` works fine.

There was also a silly issue in this version, one of the manual texi pages was using a directive that had been removed
from texinfo. A quick patch to remove them and all was well. This only affected version 2.16.

* [\[PATCH\] Remove @hsep and @vsep usage from info pages](https://sourceware.org/ml/libc-alpha/2012-12/msg00020.html)

### 2.12

At this point, we hit a point where no one else had bothered trying to compile glibc older than 2.13 on platforms that
were in any way recent. The next few problems involved lots of scouring ancient bug reports and when that failed,
bisecting the revisions until the commit that fixed the issue was found, and then backporting it to the appropriate
versions.

Because bisecting in this way is backwards to normal - we're going forward to see when the compile next succeeds,
instead of the more usual going back through history - I thought I'd run through an example of how to use it.

First off, the bisect "terms" have to be reversed, otherwise it gets very confused if the "good" commit is newer than
the "bad" commit. We can do this by specifying our own terms when initialising the bisect.

    git bisect start --term-new fixed --term-old broken

Now we specify our "fixed" and "broken" points. For this particular project, we know that the change happens between two
different tags, so specify those:

    git bisect fixed glibc-2.13
    git bisect broken glibc-2.12

Git will then tell you how many bisects it has to do to work out which commit is causing the issue (or, is fixing the
issue). If you're clever, you can tell it to run a script so that it can run each commit through automatically. In this
case I use the glibc_version_header.py script, which can apply all previously required patches and compiler flags.
Because we're doing the bisect "backwards", we need to invert the result of the script:

    git bisect run bash -c "! ./glibc_version_header.py"

Now all you need to do is wait for a bit for each commit to be built (git bisect uses binary search, so runs in O(log n)
time), and bingo, the commit that fixed the build problem has been found!.

A change to binutils 2.21 changes how `.ctors` and `.dtors` sections of binaries work. Apparently 2.12 hardcodes the old
behaviour, resulting in the built binaries just segfaulting when they're used as part of the build process. The full
patch for this is quite large and changes a lot of stuff we don't care about for this project, so my own patch just
stubs out the necessary blocks with `#if 0` instead.

* [PATCH: Support mixing .init_array.\* and .ctors.\* input sections](http://sourceware.org/ml/binutils/2010-12/msg00466.html)

### 2.10

    *** mixed implicit and static pattern rules. Stop.

Another minor one here. At some point in the past `make` stopped allowing mixing rules with patterns in them (`%`) and
those without. Duplicating the makefile rule was easy enough.

* [\[toolchain/eglibc\] manual/Makefile: Don't mix pattern rules with normal rules](https://gitlab.federez.net/crans/OpenWrt-ChaosCalmer-UAPPro/commit/cdc9ae57fc31e370f4f511b74360425839389bc9)

### 2.9

A linker failure on build. This seems to have been caused by a binutils change as well. Luckily the solution was quite
simple - the output of `ld` had changed slightly, so all that needed to be done was to adjust a single sed command.

* [Bug 258072 - sys-libs/glibc-2.9_p20081201-r1 compilation fails with sys-devel/binutils-2.19.51.0.2](https://bugs.gentoo.org/258072)

### 2.5

GCC5 switched its default C standard from C89 to C99. This changes how `inline` is treated and using `extern` with it is
no longer tolerated in the same way. Instead, you have to use the similar `__extern_inline` instead. Due to the
essentially unrelated error messages this generated ("redefiniton of \<function\>"), finding the patch that actually
fixed this required some extensive bisecting.

* [\* configure.in (libc_cv_gnu89_inline): Test for -fgnu89-inline.](http://sourceware.org/git/?p=glibc.git;a=commitdiff;h=b037a293a48718af30d706c2e18c929d0e69a621)

And with that, we are done! A working (or at least compiled) glibc 2.5 for modern Linux systems. Now you can relax and
enjoy all that extra compatibility afforded by all those ancient headers.

## Further reading

* [Elf Symbol Versioning](https://web.archive.org/web/20170124195801/https://www.akkadia.org/drepper/symbol-versioning)
* [GlibcBuildIssues](https://web.archive.org/web/20130306095513/http://plash.beasts.org/wiki/GlibcBuildIssues)
