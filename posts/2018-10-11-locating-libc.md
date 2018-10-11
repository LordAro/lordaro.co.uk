title: Locating libc
---

One of the advantages to most Linux distributions is how all the libraries and programs are installed to common
locations in the filesystem. This means that executable programs go in /bin, libraries in /lib, user's home directories
under /home, and various others. This is the Filesystem Hierarchy Standard and is followed by most Linux distributions.

However, there's still a significant amount of variation in the FHS about where things go across different Linux
distributions. `/lib`, `usr/lib` and `/usr/local/lib` are the common directories. Moreover, when you introduce different
architectures (e.g. 32bit programs on a 64bit OS), you get things like `/lib32`, `/lib` and `/lib64`. Obviously, which
architecture goes in `/lib` depends on the distribution. Ubuntu also has `/lib/i386-linux-gnu` and
`/lib/x86_64-linux-gnu` for certain libraries, and there's also `/usr/libx32`, which I've no idea what its use is.

All this variation makes working out where the libraries are located a bit tricky to achieve in a (reasonable)
cross-platform manner.

In this particular case, we're looking for the core C runtime library, `libc.so`. Given nearly every compiled
application on the computer will need to make a system call at some point (open a file, get a random number, etc),
nearly every (dynamically linked) application depends on this library. And typically enough, different distributions
change the directory that it's stored in. To use two distributions as an example, CentOS puts it under `/usr/lib(64)`,
Ubuntu puts it under `/usr/lib/x86_64-linux-gnu` ...or `/usr/libx32`.

So, how to find where our most important library is located in the filesystem?

### Search path

First thing you might think of is to just look in each of the directories of the library search path. The environment
variables `LIBRARY_PATH` and `LD_LIBRARY_PATH` are no help here - the former is used by the linker when linking static
binaries, and the latter isn't actually necessarily set anywhere and is more of a supplemental list to the defaults.

Okay, so we'll use ld itself to tell us what the search path is - the output of `ldconfig -v` will have, among some
other output, every directory it searches for libraries. The issue here is that the search path isn't in any sort of
deterministic order and also includes the search path for different architectures, so you're just as likely to pick up
the 32bit `libc.so` instead of the 64bit version. While we could test the architecture of each `libc.so` file we find in
the resulting directory list, it's also fairly expensive computationally to call out to `ldconfig`, especially in C, and
also to then parse its output.

There must be a better way.

### dlopen

`dlopen` is a nice function call that allows a program to load libraries when they're required rather automatically when
the program starts. This can be useful for truly dynamic plugin systems or where architecture specific libraries are
available. Usefully, the handle `dlopen` returns can be queried for file information using dlinfo ...so long as it can
actually load the library - `libc.so` is often written as a "GNU LD script", for the compile time linker, which is just
text file pointing to the actual `libc.so` binary, so cannot be loaded by dlopen.

This is still the solution that some languages use. Amazingly, they use dlopen to try to open `libc.so`, parse the error
message that's returned by dlopen to determine that it's a linker script, then parse the file itself to find the actual
location of the actual libc library. This seems horrible, but
[Ruby FFI](https://github.com/aniederl/ffi/blob/0daa680/lib/ffi/library.rb#L71-L75) and
[GHC](https://github.com/ghc/ghc/blob/98daa34/rts/Linker.c#L478-L497) both use this method, so maybe it's fine?

There must be a better way.

### gnu/lib-names.h

There is a final option for getting the actual filename of the actual libc library file, though it's not properly
portable. GNU provides a header `lib-names.h` which contains constants (in the form of `#define`s) for various system
libraries, such as `libm.so`, `librt.so`, `libc.so`, and the loader itself, `ld.so`. The header is GNU specific, so it's
not exactly POSIX friendly, but there we go. Using this we can get the filename of the library - we're still lacking the
directory, but now that we know the filename, it's not infeasible to search through all the library directories for the
correct file.



So there we have it! I eventually went with the `dlopen` method, as the [TCC library](https://bellard.org/tcc/), which
was what all this was for, has the ability to parse the linker scripts anyway.
