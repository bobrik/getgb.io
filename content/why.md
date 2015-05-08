* Repeatable builds

I have a requirement that at any time I can fetch the entire graph of source that went into a program, feed that to a compiler and produce a program that is identical to one created in the past.

This is the requirement I have, and this is the motivation for this talk. If you don't have this requirement, that's fine.

The plethora of tools that exist in this space shows that Go programmers have multiple, sometimes overlapping requirements. Again, that is fine, this is my solution for my requirements; it is my _hope_ that I can convince you of it's utility to you, but again, if you don't share my requirements, I may not be successful in my arguments.

Out of scope

- compiler doesn't produce byte for byte comparable binaries
- archiving compiler tool chain versions

* Why I don't have a reliable builds today

OK, so now I've told you what I want; I need to explain to you why I don't feel that I have it today.

     import "github.com/pkg/sftp"  # yes, but which revision!

The most obvious reason is the import statement inside a Go package does not provide enough information for `go`get` to select from a set of revisions available in a remote code repository the specific revision to fetch.

That information simply isn't there.

## Naming things

There are two rules for successful dependency management in Go.

### Rule 1: Things that are different _must_ have different import paths.

Who has written a log or logger package, they might all be called "log", but they are not the same package.

This why we have namespaces, `github.com/you/log`, `github.com/me/log`.

### Naming things (part 2)

Rule 2: Things that are the same _must_ have the same import path.

Are these two packages the same, or are they different ?

     github.com/lib/pq
     github.com/davecheney/foo/internal/github.com/lib/pq

They are the same, this is the same code -- this is obvious to a human, not a computer.

To a compiler these are different packages.

- Type assertions and equality are broken.
- `init()` functions will run multiple times. [[http://godoc.org/database/sql#Register][database/sql.Register]]

* The import statement cannot be changed

We cannot add anything to the import syntax for two reasons

    import "github.com/pkg/term" "{hash,tag,version}"

- Imports are opaque to the language, so some external tool dictating the format of the import declaration to the compiler is not appropriate.
- More importantly, this would be a backward incompatible syntax change.

* Versions in the URL (part 1)

We cannot embed anything in the import syntax

    import "github.com/project/v7/library"

- Popular if you want to provide multiple versions of your API at the same time.
- Not accurate enough to checkout a specific revision.
- Not reproducible to ensure everyone has the _same_ revision.
- Every import statement in every file in the package *must* be identical, even using build tags, even using conditional compilation.
- Breaks the rule of naming things, two things which are the same, must have the same import path.

* Versions in the URL (part 2)

Leads to nightmarish scenarios where equality and type assertions are broken.

    import "github.com/project/v9/lib" // registers itself as a dialer
    import "github.com/project/dialer"

    err := dialer.Dial("someurl")
    fmt.Println(err == lib.ErrTimeout) => false
    fmt.Printf("%T", err) => "lib.ErrTimeout"
    fmt.Println(v7/lib.ErrTimeout == v9/lib.ErrTimeout) => false

* Competitive Analysis

* Dude, be a good Gopher, don't break users

So the first, and longest standing solution to this problem is to always have a stable API.

- Proposed solution from the Go team for several years.
- Admirable attempt to extend the Go 1 contract to all Go code.

If it worked, we wouldn't be having this conversation today

- You _want_ to change the API for your package
- Often the API and the consumer evolve in parallel
- Even putting versions in the import path only guarantees I have _a_ version of that package, not _the_ version.

* I live in the real world

If my time in system administration taught me anything, it's the unexpected failures that get you. You can plan for the big disasters, but it turns out that the little disasters can be just as disruptive.

- code.google.com closing down. Will Github still be around in 10 years ?
- codehaus :(
- companies merging, people getting married, someone dies, trademark dispute, etc.
- FoundationDB :(

These are all little disasters, you can usually find the code again, maybe it's just a quick sed rewrite and you're back again. 

But just like the big disasters, these little disasters are indistinguishable, code which built one day, won't build the next.

* Don't be this person

.image reproducible-builds/github.png

The moral of the story is, if you are responsible for delivering a product written in Go, you need to be responsible for all the source that goes into that product.

* Tools which manage $GOPATH

Tools which fixup `$GOPATH` after `go`get`

- godeps (plural, canonical)
- glock
- gvp

Problems

- not reproducible, the upstream can still disappear
- must adjust your `$GOPATH` manually when moving between projects
- near universal dislike for a .lock file in the package
- universal disagreement on the format and layout of a .lock file

* Tools which vendor packages 

Copying, vendoring, rewriting the source, is the _new_ position from the Go team.

- godep 

Problems

- requires source rewriting; many uncomfortable with this
- possibly breaks the naming rules; the same package can exist in the dependency graph under multiple names
- concerns about losing track of the upstream
- ugly long import lines

* Tools which give you one $GOPATH per project

Virtual env all the things!

- gpm https://github.com/pote/gpm
- gvm 
- govm
- glide
- /usr/bin/direnv (old skool)

Problems

- not isolated from upstream going away
- hard to use, terminal or shell session becomes magic

* Proposal

* Stop working around go get

.image reproducible-builds/goget.jpg _ 400

Every one of the existing solutions is hamstrung by the fact it is working around the limitations of the `go` tool.

Stop using `go`get`. Don't use the `go` tool at all.

* Requirements

So, we're talking about writing a new build tool for Go, not a wrapper around an existing tool.

* Project based

A new build tool should be project based.

- A project is where main packages live.
- A project is effectively a single $GOPATH.
- Automatic detection, you don't need 'enter' a project.
- The owner of the project decides on the version of a particular import path in use.
- Never more than one copy of a single import per project.

* No configuration files

This one I find hard to accept, but Go developers do not want to have any sort of configuration file to build their code.

I find this hard to rationalize because most repos that I look have had dozens of turds in them, Gruntfiles, Dockerfiles, Werker configs, etc. 

- Projects are detected from the path, anything that has a `src` in the path is a project.
- Works well in practice, and is backwards compatible with `$GOPATH`.

* Respect the canonical import path

     package pdf // import "rsc.io/pdf"

- rsc added this, it's clear what he thinks about the possibility of duplicates in the binary
- Import rewriting has to rewrite the import comment as well, and that sounds like deliberately disabling the safety interlock on a handgun.

* Leaves source untouched

- Check the whole source in
- Use submodules or subtrees, svn externals, etc, if you prefer
- Don't touch the source, then you stand a chance of hashing it / diffing it / signing it

* Annoying things

If we're going to go the extreme of divorcing ourselves from the `go` tool then maybe we can fix a few other annoyances along the way

- `-tags`something` now just works
- deleting a file from a package, causes a rebuild
- deleting a package's source, we won't use the stale .a in ~/pkg
- not restricted to `go`get` ideas of correct DVCS usage, ie, can use ssh://github.com/...

* Introducing gb

.image reproducible-builds/gb.jpg _ 350

    % /usr/bin/gb

- Proof of concept
- Project based, project is automatically detected
- Dependencies are a property of the project, one copy of any package per project
- Supports vendoring without rewriting via multiple src/ directories
- Supports plugins (git style) 

* Demo time

# * This only works for projects, what about packages ?

# Yes, this solution works for projects, it encourages you to build larger projects.

# Personally, I think this is what Go needs at the moment.

# - Projects should think hard about each dependency they bring in -- they aren't free

# - Packages developed and thrown out there, hoping that someone else will use and popularise them. As Peter Bourgon noted at FOSDEM this year, does Go really need another http mux ?

# What I want to see is large projects being built with Go, then, when they are proven, lets circle back and peal off parts of those projects that are reusable. But that happens second, you can't have a stable complete package without an anchor tenant, and it makes sense to me that those libraries should be incubated inside their host, not along side them -- it worked for django, it worked for rails, I think it's a message that should be studied.

* Take aways

The problem is `go`get`, not the import statement.

The `go` tool doesn't define the language, we can build a replacement.

* Try it out

    go get github.com/constabulary/gb/...

- 100 % compatible with existing Go source.
- Don't even need to change `$GOPATH`.
- Upgrade to a gb project if you like.
- Reusable library for building Go packages, no more shelling out to `go`build`
- Write a plugin, please write a plugin.

