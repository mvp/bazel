Project: /_project.yaml
Book: /_book.yaml

# BUILD Style Guide

{% include "_buttons.html" %}

## Prefer DAMP BUILD files over DRY {:#prefer-damp-build-files-over-dry}

The DRY principle — "Don't Repeat Yourself" — encourages uniqueness by
introducing abstractions such as variables and functions to avoid redundancy in
code.

In contrast, the DAMP principle — "Descriptive and Meaningful Phrases" —
encourages readability over uniqueness to make files easier to understand and
maintain.

`BUILD` files aren't code, they are configurations. They aren't tested like
code, but do need to be maintained by people and tools. That makes DAMP better
for them than DRY.

## BUILD.bazel file formatting {:#formatting}

`BUILD` file formatting follows the same approach as Go, where a standardized
tool takes care of most formatting issues.
[Buildifier](https://github.com/bazelbuild/buildifier){: .external} is a tool that parses and
emits the source code in a standard style. Every `BUILD` file is therefore
formatted in the same automated way, which makes formatting a non-issue during
code reviews. It also makes it easier for tools to understand, edit, and
generate `BUILD` files.

`BUILD` file formatting must match the output of `buildifier`.

### Formatting example {:#formatting-example}

```python
# Test code implementing the Foo controller.
package(default_testonly = True)

py_test(
    name = "foo_test",
    srcs = glob(["*.py"]),
    data = [
        "//data/production/foo:startfoo",
        "//foo",
        "//third_party/java/jdk:jdk-k8",
    ],
    flaky = True,
    deps = [
        ":check_bar_lib",
        ":foo_data_check",
        ":pick_foo_port",
        "//pyglib",
        "//testing/pybase",
    ],
)
```

## File structure {:#file-structure}

**Recommendation**: Use the following order (every element is optional):

*   Package description (a comment)

*   All `load()` statements

*   The `package()` function.

*   Calls to rules and macros

Buildifier makes a distinction between a standalone comment and a comment
attached to an element. If a comment is not attached to a specific element, use
an empty line after it. The distinction is important when doing automated
changes (for example, to keep or remove a comment when deleting a rule).

```python
# Standalone comment (such as to make a section in a file)

# Comment for the cc_library below
cc_library(name = "cc")
```

## References to targets in the current package {:#targets-current-package}

Files should be referred to by their paths relative to the package directory
(without ever using up-references, such as `..`). Generated files should be
prefixed with "`:`" to indicate that they are not sources. Source files
should not be prefixed with `:`. Rules should be prefixed with `:`. For
example, assuming `x.cc` is a source file:

```python
cc_library(
    name = "lib",
    srcs = ["x.cc"],
    hdrs = [":gen_header"],
)

genrule(
    name = "gen_header",
    srcs = [],
    outs = ["x.h"],
    cmd = "echo 'int x();' > $@",
)
```

## Target naming {:#target-naming}

Target names should be descriptive. If a target contains one source file,
the target should generally have a name derived from that source (for example, a
`cc_library` for `chat.cc` could be named `chat`, or a `java_library` for
`DirectMessage.java` could be named `direct_message`).

The eponymous target for a package (the target with the same name as the
containing directory) should provide the functionality described by the
directory name. If there is no such target, do not create an eponymous
target.

Prefer using the short name when referring to an eponymous target (`//x`
instead of `//x:x`). If you are in the same package, prefer the local
reference (`:x` instead of `//x`).

Avoid using "reserved" target names which have special meaning. This includes
`all`, `__pkg__`, and `__subpackages__`, these names have special
semantics and can cause confusion and unexpected behaviors when they are used.

In the absence of a prevailing team convention these are some non-binding
recommendations that are broadly used at Google:

* In general, use ["snake_case"](https://en.wikipedia.org/wiki/Snake_case){: .external}
    * For a `java_library` with one `src` this means using a name that is not
      the same as the filename without the extension
    * For Java `*_binary` and `*_test` rules, use
      ["Upper CamelCase"](https://en.wikipedia.org/wiki/Camel_case){: .external}.
      This allows for the target name to match one of the `src`s. For
      `java_test`, this makes it possible for the `test_class` attribute to be
      inferred from the name of the target.
* If there are multiple variants of a particular target then add a suffix to
  disambiguate (such as. `:foo_dev`, `:foo_prod` or `:bar_x86`, `:bar_x64`)
* Suffix `_test` targets with `_test`, `_unittest`, `Test`, or `Tests`
* Avoid meaningless suffixes like `_lib` or `_library` (unless necessary to
  avoid conflicts between a `_library` target and its corresponding `_binary`)
* For proto related targets:
    * `proto_library` targets should have names ending in `_proto`
    * Languages specific `*_proto_library` rules should match the underlying
      proto but replace `_proto` with a language specific suffix such as:
         * **`cc_proto_library`**: `_cc_proto`
         * **`java_proto_library`**: `_java_proto`
         * **`java_lite_proto_library`**: `_java_proto_lite`

## Visibility {:#visibility}

Visibility should be scoped as tightly as possible, while still allowing access
by tests and reverse dependencies. Use `__pkg__` and `__subpackages__` as
appropriate.

Avoid setting package `default_visibility` to `//visibility:public`.
`//visibility:public` should be individually set only for targets in the
project's public API. These could be libraries that are designed to be depended
on by external projects or binaries that could be used by an external project's
build process.

## Dependencies {:#dependencies}

Dependencies should be restricted to direct dependencies (dependencies
needed by the sources listed in the rule). Do not list transitive dependencies.

Package-local dependencies should be listed first and referred to in a way
compatible with the
[References to targets in the current package](#targets-current-package)
section above (not by their absolute package name).

Prefer to list dependencies directly, as a single list. Putting the "common"
dependencies of several targets into a variable reduces maintainability, makes
it impossible for tools to change the dependencies of a target, and can lead to
unused dependencies.

## Globs {:#globs}

Indicate "no targets" with `[]`. Do not use a glob that matches nothing: it
is more error-prone and less obvious than an empty list.

### Recursive {:#recursive}

Do not use recursive globs to match source files (for example,
`glob(["**/*.java"])`).

Recursive globs make `BUILD` files difficult to reason about because they skip
subdirectories containing `BUILD` files.

Recursive globs are generally less efficient than having a `BUILD` file per
directory with a dependency graph defined between them as this enables better
remote caching and parallelism.

It is good practice to author a `BUILD` file in each directory and define a
dependency graph between them.

### Non-recursive {:#non-recursive}

Non-recursive globs are generally acceptable.

## Avoid list comprehensions {:#list-comprehensions}

Avoid using list comprehensions at the top level of a `BUILD.bazel` file.
Automate repetitive calls by creating each named target with a separate
top-level rule or macro call. Give each a short `name` parameter for clarity.

List comprehension reduces the following:

*   Maintainability. It's difficult or impossible for human maintainers and
    large scale automated changes to update list comprehensions correctly.
*   Discoverability. Since the pattern doesn't have `name` parameters,
    it's hard to find the rule by name.

A common application of the list comprehension pattern is to generate tests. For
example:

```build {.bad}
[[java_test(
    name = "test_%s_%s" % (backend, count),
    srcs = [ ... ],
    deps = [ ... ],
    ...
) for backend in [
    "fake",
    "mock",
]] for count in [
    1,
    10,
]]
```

We recommend using simpler alternatives. For example, define a macro that
generates one test and invoke it for each top-level `name`:

```build
my_java_test(name = "test_fake_1",
    ...)
my_java_test(name = "test_fake_10",
    ...)
...
```

## Don't use deps variables {:#no-dep-vars}

Don't use list variables to encapsulate common dependencies:

```build {.bad}
COMMON_DEPS = [
  "//d:e",
  "//x/y:z",
]

cc_library(name = "a",
    srcs = ["a.cc"],
    deps = COMMON_DEPS + [ ... ],
)

cc_library(name = "b",
    srcs = ["b.cc"],
    deps = COMMON_DEPS + [ ... ],
)
```

Similarly, don't use a library target with
[`exports`](/reference/be/java#java_library.exports) to group dependencies.

Instead, list the dependencies separately for each target:

```build {.good}
cc_library(name = "a",
    srcs = ["a.cc"],
    deps = [
      "//a:b",
      "//x/y:z",
      ...
    ],
)

cc_library(name = "b",
    srcs = ["b.cc"],
    deps = [
      "//a:b",
      "//x/y:z",
      ...
    ],
)
```

Let [Gazelle](https://github.com/bazel-contrib/bazel-gazelle) and other tools
maintain them. There will be repetition, but you won't have to think about how
to manage the dependencies.

## Prefer literal strings {:#literal-strings}

Although Starlark provides string operators for concatenation (`+`) and
formatting (`%`), use them with caution. It is tempting to factor out common
string parts to make expressions more concise or break long lines. However,

*   It is harder to read broken-up string values at a glance.

*   Automated tools such as
    [buildozer][buildozer] and Code Search have trouble finding values and
    updating them correctly when the values broken up.

*   In `BUILD` files, readability is more important than avoiding repetition
    (see [DAMP versus DRY](#prefer-damp-build-files-over-dry)).

*   This Style Guide
    [warns against splitting label-valued strings](#other-conventions)
    and
    [explicitly permits long lines](#differences-python-style-guide).

*   Buildifier automatically fuses concatenated strings when it detects that
    they are labels.

Therefore, prefer explicit, literal strings over concatenated or formatted
strings, especially in label-type attributes such as `name` and `deps`. For
example, this `BUILD` fragment:

```build {.bad}
NAME = "foo"
PACKAGE = "//a/b"

proto_library(
  name = "%s_proto" % NAME,
  deps = [PACKAGE + ":other_proto"],
  alt_dep = "//surprisingly/long/chain/of/package/names:" +
            "extravagantly_long_target_name",
)
```

would be better rewritten as

```build {.good}
proto_library(
  name = "foo_proto",
  deps = ["//a/b:other_proto"],
  alt_dep = "//surprisingly/long/chain/of/package/names:extravagantly_long_target_name",
)
```

[buildozer]: https://github.com/bazelbuild/buildtools/blob/main/buildozer/README.md

## Limit the symbols exported by each `.bzl` file {:#limit-symbols}

Minimize the number of symbols (rules, macros, constants, functions) exported by
each public `.bzl` (Starlark) file. We recommend that a file should export
multiple symbols only if they are certain to be used together. Otherwise, split
it into multiple `.bzl` files, each with its own [bzl_library][bzl_library].

Excessive symbols can cause `.bzl` files to grow into broad "libraries" of
symbols, causing changes to single files to force Bazel to rebuild many targets.

[bzl_library]: https://github.com/bazelbuild/bazel-skylib/blob/main/README.md#bzl_library

## Other conventions {:#other-conventions}

 * Use uppercase and underscores to declare constants (such as `GLOBAL_CONSTANT`),
   use lowercase and underscores to declare variables (such as `my_variable`).

 * Labels should never be split, even if they are longer than 79 characters.
   Labels should be string literals whenever possible. *Rationale*: It makes
   find and replace easy. It also improves readability.

 * The value of the name attribute should be a literal constant string (except
   in macros). *Rationale*: External tools use the name attribute to refer a
   rule. They need to find rules without having to interpret code.

 * When setting boolean-type attributes, use boolean values, not integer values.
   For legacy reasons, rules still convert integers to booleans as needed,
   but this is discouraged. *Rationale*: `flaky = 1` could be misread as saying
   "deflake this target by rerunning it once". `flaky = True` unambiguously says
   "this test is flaky".

## Differences with Python style guide {:#differences-python-style-guide}

Although compatibility with
[Python style guide](https://www.python.org/dev/peps/pep-0008/){: .external}
is a goal, there are a few differences:

 * No strict line length limit. Long comments and long strings are often split
   to 79 columns, but it is not required. It should not be enforced in code
   reviews or presubmit scripts. *Rationale*: Labels can be long and exceed this
   limit. It is common for `BUILD` files to be generated or edited by tools,
   which does not go well with a line length limit.

 * Implicit string concatenation is not supported. Use the `+` operator.
   *Rationale*: `BUILD` files contain many string lists. It is easy to forget a
   comma, which leads to a complete different result. This has created many bugs
   in the past. [See also this discussion.](https://lwn.net/Articles/551438/){: .external}

 * Use spaces around the `=` sign for keywords arguments in rules. *Rationale*:
   Named arguments are much more frequent than in Python and are always on a
   separate line. Spaces improve readability. This convention has been around
   for a long time, and it is not worth modifying all existing `BUILD` files.

 * By default, use double quotation marks for strings. *Rationale*: This is not
   specified in the Python style guide, but it recommends consistency. So we
   decided to use only double-quoted strings. Many languages use double-quotes
   for string literals.

 * Use a single blank line between two top-level definitions. *Rationale*: The
   structure of a `BUILD` file is not like a typical Python file. It has only
   top-level statements. Using a single-blank line makes `BUILD` files shorter.
