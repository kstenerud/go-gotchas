Go Gotchas
==========

Go is a very nice language that eschews complexity and magic, but its design is not without its warts, sharp edges, and landmines. Many of these are due to the go tool assigning special side-effect meanings to specific subsets of already established conventions (magic), which always invites trouble. This document is intended as a rough guide to help you avoid getting cut.



Magical Directories
-------------------

The go tool treats certain directory names in magical ways. The following directories are ignored when building:

- `testdata`
- `vendor`
- directories starting with `_`
- directories starting with `.`

There are other directories that trigger special handling, but I haven't fully tracked down what side effects they have and under what circumstances they trigger:

- `builtin`
- `cmd`
- `runtime/cgo`
- `std`

### Workaround

Avoid using these names in your directory structure.



No Cyclical Imports
-------------------

Go disallows cyclical imports. Unfortunately, this means that things at the top level package of your project can't be used by things in deeper packages. This becomes problematic when dealing with interfaces that need to be exposed to the user:

in package "toplevel":

```golang
type Doer interface {...}
func (this *Something) DoStuff(myDoer Doer) {}
```

in package "toplevel/sublevel":

```golang
import "github.com/user/toplevel" // Error: import cycle not allowed
func (this *InternalStruct) DoInternalThing(myDoer toplevel.Doer) {}
```

### Workaround

Duplicate the interface in a subpackage and import the subpackage instead:

in package "toplevel":

```golang
type Doer interface {...} // duplicate code
func (this *Something) DoStuff(myDoer Doer) {}
```

in package "toplevel/common":

```golang
type Doer interface {...} // duplicate code
```

in package "toplevel/sublevel":

```golang
import "github.com/user/toplevel/common"
func (this *InternalStruct) DoInternalThing(myDoer common.Doer) {}
```

You'll need to manually ensure that the two interfaces stay in sync.



File-Local Visibility
---------------------

Go namespaces everything to the package name, making anything whose name starts with a capital visible externally, and anything starting with a lowercase letter visible within the package. There is no such thing as file-local, only package-local. This causes problems when naming local things (such as local utility functions or file-local variables or consts) because they can clash with names in other files, or cause "bleeding" of purely file-local functions (whereby they are mistakenly accessed from another file).

### Workaround

Try to name things in ways that avoid name clashes, or manually namespace file-local things with prefixes, similar to how one would do it in C.



Unused Imports, Variables, Etc
------------------------------

Go compiling fails if an import or variable is unused. This causes problems when debugging because you have to keep chasing down chains of variable assignments for every newly "unused" variable after commenting something out. For complex codebases, this can quickly degenerate into a game of whack-a-mole as commenting out the unused things causes other variables the commented out line depended on to become "unused" themselves.

### Workaround

Modify the go compiler to emit warnings instead: https://github.com/kstenerud/go



Module Renaming
---------------

Go conflates module naming with module locating, which causes problems if you change where they are hosted.

Example:

    module github.com/someuser/somemodule

What happens if you switch to Bitbucket?

### Workaround

The only solution is not very nice: Rename the module to `bitbucket.com/someuser/somemodule`. You will need to modify all imports to packages within your module. While you could get around this with relative imports, they break a lot of go tooling and shouldn't be used. All users of your module will have to rewrite every import of your packages in their code.



Shared Test Code
----------------

The go compiler assigns magic to the source code file names. Anything ending in `_test.go` will only be compiled when calling `go test`, and will not be visible to `import` (it's almost like a shadow parallel package in a lot of ways). This causes problems when you want to share test code between packages.

### Workaround

Put your shared test code in a normal `*.go` file instead of `*_test.go`. The side effect is that your shared test code will now be considered part of the "normal" codebase. It will be compiled even when not testing, and your test code will be visible from your normal code.
