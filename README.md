# Tsickle - TypeScript to Closure Translator [![Build Status](https://github.com/angular/tsickle/actions/workflows/node.js.yml/badge.svg)](https://github.com/angular/tsickle/actions/workflows/node.js.yml)

Tsickle converts TypeScript code into a form acceptable to the [Closure
Compiler]. This allows using TypeScript to transpile your sources, and then
using Closure Compiler to bundle and optimize them, while taking advantage of
type information in Closure Compiler.

[closure compiler]: https://github.com/google/closure-compiler/

## What conversion means

A (non-exhaustive) list of the sorts of transformations Tsickle applies:

- inserts Closure-compatible JSDoc annotations on functions/classes/etc
- converts ES6 modules into `goog.module` modules
- generates externs.js from TypeScript d.ts (and `declare`, see below)
- declares types for class member variables
- translates `export * from ...` into a form Closure accepts
- converts TypeScript enums into a form Closure accepts
- reprocesses all jsdoc to strip Closure-invalid tags

In general the goal is that you write valid TypeScript and Tsickle handles
making it valid Closure Compiler code.

## Warning: work in progress

We already use tsickle within Google to minify our apps (including those using
Angular), but we have less experience using tsickle with the various JavaScript
builds that are seen outside of Google.

We would like to make tsickle usable for everyone but right now if you'd like
to try it you should expect to spend some time debugging and reporting bugs.

## Usage

Tsickle is a library, designed to be used by a larger program that interacts
with TypeScript and the Closure compiler.

Some known clients are:

1. Within Google we use tsickle inside the [Bazel build
   system](https://bazel.build/). That code is published as
   open source as part of [Bazel's nodejs/TypeScript
   build rules](https://bazelbuild.github.io/rules_nodejs/).
1. [tscc](https://github.com/theseanl/tscc) wraps tsickle and
   closure compiler, and interops with rollup.
1. We publish a simple demo program in the `demo/` subdirectory.

## Design details

### Output format

Tsickle is designed to do whatever is necessary to make the code acceptable by
Closure compiler. We view its output as a necessary intermediate form for
communicating to the Closure compiler, and not something for humans. This means
the tsickle output may be kind of ugly to read. Its only real use is to pass it
on to the compiler.

For one example, the syntax of types tsickle produces are specific to Closure.
The type `{!Foo}` means "Foo, excluding null" and a type alias becomes a `var`
statement that is tagged with `@typedef`.

Tsickle emits modules using Closure's `goog.module` module system. This system
is similar to but different from ES modules, and was supported by Closure before
the ES module system was finalized.

### Differences from TypeScript

Closure and TypeScript are not identical. Tsickle hides most of the
differences, but users must still be aware of some differences.

#### `declare`

Any declaration in a `.d.ts` file, as well as any declaration tagged with
`declare ...`, is intepreted by Tsickle as a name that should be preserved
through Closure compilation (i.e. not renamed into something shorter). Use it
any time the specific string names of your fields are significant. That would
most often happen when the object either coming from outside your program, or
being passed out of the program.

Example:

    declare interface JSONResult {
        username: string;
    }
    let r = JSON.parse(input) as JSONResult;
    console.log(r.username);

By adding `declare` to the interface (or if it were in a `.d.ts` file), Tsickle
will inform Closure that it must use exactly the field name `.username` (and not
e.g. `.a`) in the output JS. This matters for this example because the input
JSON probably uses the string `'username'` and not whatever name Closure would
invent for it. (Note: `declare` on an interface has no additional meaning in
pure TypeScript.)

#### Exporting decorators

An exporting decorator is a decorator that has `@ExportDecoratedItems` in its
JSDoc.

The names of elements that have an exporting decorator are preserved through
the Closure compilation process by applying an `@export` tag to them.

Example:

    /** @ExportDecoratedItems */
    function myDecorator() {
      // ...
    }

    @myDecorator()
    class DoNotRenameThisClass { ... }

## Development

### Dependencies

- nodejs. Install from your operating system's package manger, by following
  instructions on https://nodejs.org/en/, or by using
  [NVM](https://github.com/nvm-sh/nvm)
- yarn. Install from your operating system's package manager or by following
  [instructions on yarnpkg.com](https://yarnpkg.com/en/docs/install).

### One-time setup

Run `yarn` to install dependencies.

### Build & Test commands

- `yarn build` builds the code base.
- Run `tsc --watch` for an interactive, incremental, and continuous build.
- `yarn lint` checks for lint.
- `yarn test` runs unit tests, e2e tests and checks for lint (but make sure to
  `yarn build` first or run tsc!). Set the `TEST_FILTER` environment variable
  to filter what golden tests to run.

### TypeScript AST help

https://astexplorer.net/ and https://ts-ast-viewer.com/ are convenient tools to
visualize and inspect a TypeScript AST.

### Debugging

You can debug tests by passing `--node_options=--inspect` or
`--node_options=--inspect-brk` (to suspend execution directly after startup).

For example, to debug a specific golden test:

```shell
TEST_FILTER=my_golden_test node --inspect-brk=4332 ./node_modules/.bin/jasmine out/test/*.js
```

Then open [about:inspect] in Chrome and choose "about:inspect". Chrome will
launch a debugging session on any node process that starts with a debugger
listening on one of the listed ports. The tsickle tests and Chrome both default
to `localhost:9229`, so things should work out of the box.

The break in specific code locations you can add `debugger;` statements in the
source code.

### Updating Goldens

Run `UPDATE_GOLDENS=y yarn test` to have the test suite update the goldens in
`test_files/...`.

### Environment variables

Set the environment variable `TEST_FILTER=<REGEX>` to limit the golden tests
(found in `test_files/...`) to only run tests with a name matching the regex.

### Releasing

On a new branch, run:

```
# tsickle releases are all minor releases for now, see npm help version.
$ npm version minor
```

This will update the version in `package.json`, commit the changes, and
create a git tag.

Push the branch, open a pull request, get it reviewed, and wait for it to be merged.

Checkout and pull the latest version from master:

```
$ git checkout master && git pull
```

Check if the tag exists. If not, re-tag the commit and push the tag.

```
$ git tag
# Does this show the tag already? If not, proceed with:
$ git tag v0.32.0 && git push origin v0.32.0  # but use correct version
```

Once the versioned tag is pushed to GitHub the release (as found on
https://github.com/angular/tsickle/releases) will be implicitly created.

From the master branch run:

```
npm config set registry https://wombat-dressing-room.appspot.com
npm login
npm publish  # runs a clean build & test automatically
```
