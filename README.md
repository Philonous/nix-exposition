<!-- Author: Philipp Balzarek -->
<!-- License: CC BY 4.0 -->

# WTF is Nix?

This document is a very high-level overview over Nix. I wrote it because I
remember being confused about what problems Nix is trying to solve and how it is
trying to solve them.

It concentrates more on the "what" and "why" than the details of the "how".

## So what is Nix?

Nix is (more than) a package management system. In that goal it is roughly
comparable to Debian's `apt` or many other package manager. However, its
underlying principles, method of operation and implementation are radically
different.

NixOs (https://nixos.org/) is a Linux distribution based on Nix.

## What does Nix consist of?

To understand how Nix works it is necessary to understand the three components it builds upon:

* The Nix Store
* Nix Derivations
* The Nix language

### What is the Nix store?

In most Linux distributions, installing a package (say, bash) involves copying
the files that belong to that package into a certain well-known place in the
file system (e.g. `/bin/bash`). This approach works well enough, but it comes
with a downside: You can only have one version of bash installed on your system,
since there can only be one `/bin/bash`.

Instead, imagine that every package version, including libraries, is always
installed in its own, globally unique but deterministic directory. No package
would be "blessed" by being installed system-wide, so having multiple versions
of bash "installed" at the same time becomes trivial, they just reside in
different directories. They can refer to libraries by specifying which one to
load with the absolute path, again in its unique directory. Additionally, since
the same library version would always be installed in the same place, multiple
packages can find and use it, saving disk space without having to worry about
incompatibilities.

This is the Nix store.

For example, bash would not reside under /bin/bash, but instead, say `/nix/store/xadrr3l5jvkkm3g3lb2g81j5wz51zqdv-bash-interactive-4.4-p23/bin/bash`

`/nix/store` is where all "installed" Nix packages live. The next part is the
package name, prefixed by a hash that ensures that all store paths are unique
(the hash of _what_ is explained a little later). Lastly within the package
there there can be arbitrary subfolders, including bin, usr/lib etc.

To make the store work well, we require the following:
* The store path uniquely determines its contents. I.e. `/nix/store/xadrr3l5jvkkm3g3lb2g81j5wz51zqdv-bash-interactive-4.4-p23/bin/bash`
  will always contain the same bash version, compiled with the same flags and
  linked against the same libraries (in the same store paths). If any of those
  change, a different path is produced.
* It does not require that the hash part of the store path is the hash of its
  contents. The hash only needs to uniquely identify the content. (this becomes important later)
* The contents of a store paths are _never_ modified in any way.
* Files in the store are only allowed to depend on files in the same or other
  store paths, not anywhere else. This includes for example linked libraries
  and external programs that might be called. This requirement
  ensures that the store path together with all the store paths it (recursively)
  refers to are self-sufficient. Such a self-sufficient set of paths is called a
  `closure`.

The Nix store has a database that tracks which store paths need other paths to
function, so you don't accidentally delete the libreadline library that your
bash still needs. This allows Nix to determine orphaned packages that aren't
referred to by anything any longer and can be deleted. `nix-collect-garbage`
finds those orphaned store paths and deletes them. Nix also needs to know which
packages you want to keep for yourself, since otherwise it would just always
delete everything; these are called garbage-collection roots.

"Updates" to a package are performed by putting a new version into a new
path, and replacing programs with new ones that refer that new path. This
ensures that updating a library can't break other programs, since they will just
continue to the old version of the library. "Downgrading" a program becomes a
no-op; as long as the store-path isn't deleted the old program never goes away
and still uses the old libraries as before.

You can put something into the Nix store yourself using `nix-store --add
<path>`. nix-store will return the store path it created. Adding the same files
multiple times will return the same store path, since the hash will be the same.


See https://nixos.org/guides/nix-pills/garbage-collector.html for more details


For example the closure that contains the the bash executable in path could can be determiend like this:

```
> nix-store --query -R /nix/store/2jysm3dfsgby5sw5jgj43qjrb5v79ms9-bash-4.4-p23

/nix/store/czc3c1apx55s37qx4vadqhn3fhikchxi-libunistring-0.9.10
/nix/store/xim9l8hym4iga6d4azam4m0k0p1nw2rm-libidn2-2.3.0
/nix/store/9df65igwjmf2wbw0gbrrgair6piqjgmi-glibc-2.31
/nix/store/2jysm3dfsgby5sw5jgj43qjrb5v79ms9-bash-4.4-p23
/nix/store/pk73wc0x32y2bmbiqiddb084w398ab6y-ncurses-6.2
/nix/store/j2913gjd71vmym9p2smwzwd7dn0vsxq0-readline-7.0p5
/nix/store/xadrr3l5jvkkm3g3lb2g81j5wz51zqdv-bash-interactive-4.4-p23
```

### What are Derivations?

Putting files into the store manually obviously doesn't scale well, so Nix comes
with a build system. On the lowest level of that build system there are
`derivations`. Derivations are (not quite, but almost) json files that contain
the information required to produce the build artifact, for example which other
store paths to use for that build (programs like compilers, gnu make,
preprocessors etc.) and source files as well what executable (`builder`) to call
to perform the build. Most of the time the executable will be bash and it will
run a script that just calls the build tool the package requires, e.g. gnu make,
cabal or ant.

We want to know which (unique) store path the derivation builds before it is
instantiated (run), so we can check if it is already in the store or if we can
fetch it from a cache, avoiding having to perform the build again. To make this
possible, we demand that each derivation produces the same output every time it
runs. Thus we can hash the derivation itself, knowing that its hash uniquely
identifies the output it produces.

There are essentially two ways to ensure that derivations generate reproducible outputs:

* The derivation only uses input files that are in the Nix store already, by
  fixed paths, and it only uses programs that are deterministic (e.g. a compiler
  with a fixed set of flags).
* The derivation knows in advance what the hash of its output is going to be and
  checks that the output matches that hash. This is called a `fixed output
  derivation` and is usefull e.g. for downloading sources from the web.
* Some programs, like git, have consistency checks already built in; if you
  check out a specific commit by its ID, you are already guaranteed to
  always get the same data. This kind of falls in the middle between the two
  other categories.

There are more subtleties (e.g. some compilers aren't deterministic) that fall
outside the scope of this document.

Also note that Nix doesn't really have an air-tight way of restricting you from
breaking those invariants. If you do, the contents of the store path will be
inconsistent and software depending on it may fail. However, Nix does try to guide
you towards writing consistent derivations.


### What is the Nix language

Derivations are very limited and static because they must produce predicatble
outputs. Nix provides a more convenient language to generate them, which
confusingly is also called Nix. It is "purely functional" in the sense that
there are no statements, only expressions, but evaluating Nix expressions can
have certain side-effects, like putting something into the Nix store.

For details of the Nix language, see e.g. https://nixos.org/guides/nix-pills/index.html

### How do they plug together?

When you have a project that you want to build with Nix you write a `Nix
expression` (a piece of Nix code) describing a `derivation`.

Then you tell Nix to `evaluate` that expression. Nix will put all the files the
expression refers to into the store, replacing the paths in the expression
by the store path. It then produces a `derivation`. In the derivation, all dynamic
constructs are resolved and evaluated and references to local files, urls
etc. are replaced by store path, so it is truly static that produces predictable
results.

Last, you ask Nix to `instantiate` that derivation. It will then run `builder`
that produces the output and store it in the Nix store and tells you the path.


## I'm still confused

### But what about the FHS?

Linux' [Filesystem Hierarchy
Standard](https://en.wikipedia.org/wiki/Filesystem_Hierarchy_Standard) specifies
where programs, libraries and other files should be stored on a linux system.
For example system binaries should be in `/bin`. Nix breaks these expectations
by putting packages into the Nix store instead, which means that programs
written or compiled with the FHS in mind will not function properly.

To work around this problem, software programs packaged for Nix need to be
patched, reconfigured and/or recompiled. For programs where the source code is
not available the binary has to be patched to set the interpreter (the path of
ld-linux) and rpath (where it looks for dynamic libraries). Text files are often
modified with sed. Sometimes more "creative" tricks are needed like
LD_PRELOAD-ing libc to dynamically replace paths or wrapper scripts.

This sounds like a terrible hack, but it works surprisingle well in practice.

If all else fails, a FHS-compliant "shadow" system can be set up using links and
the program can be chrooted in there. This is for example necessary when the
software in question itself downloads other executables that then can't be
patched beforehand (e.g. steam)


## So what does Nix offer us?

* Strong, declarative notion of dependence between packages
* Encapsulation "for free"
* Can trivially have many different versions of a package installed
* Up-, down- and sidegrades "for free".

All of this not only makes for a fresh way of building an OS, but is desireable when developing software:

* Declarative way managing dependencies
* Each developer gets the same result, independent of environment
* Resulting output closure is self-sufficient, can easily be exported
  * When the closure is copied to another machine with Nix, it "just
    works". This is the basis of [NixOps](https://github.com/NixOS/nixops)
  * Can be wrapped as a Docker image without dependencies on base images
* Caching (of build outputs, intermediate packages etc.) "for free"

* Can easily have per-project development environment with individual versions
  of compilers, library versions, development tools etc. "installed". Especially
  in combination with [direnv](https://direnv.net/)

* A bit like Docker, but...
  * "More" declarative
  * more fine-grained, per-package instead of one base images
  * stronger; docker image tags aren't actually immutable

# How can I try Nix?

You can install Nix via your system package manager, or if you have docker just run

```
> docker run --rm -it nixos/nix
[...]
/# nix-channel --update
```

## What is nixpkgs

Nixpkgs is a collection of `Nix Expressions`, (Nix Language files) that describe
how to download and build a large number of software packages. It's the basis of
an entire operating system, NixOS.

Nixpkgs is hosted on github: https://github.com/NixOS/nixpkgs

# A (very) brief introduction to NixOS

NixOS is an entire system built on Nix, using nixpkgs as its package source.

* But how does NixOS know which package is "installed", when I can have many different versions of e.g. bash in the nix store?

Symlinks! The "current system" is a collection of symlinks to packages in the
store. That's called a "profile". Each user can have their own, this means users
can install their own software without needing root privileges, because Nix uses
hashes to ensure that packages in the Nix store don't collide.

"Updating" a package then just becomes putting the new version in the store and replacing the symlinks in the profile.

You can also change package versions "on the fly" by adding them to your PATH variable.
