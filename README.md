speclean
========

Say you have a complex spec file with supports for very old systems:

    %if 0%{suse_version} && 0%{suse_version} < 600
    ...do something..
    %endif

...or systems from other vendors:

    %if "%{_vendor}" == "redhat" || 0%{?rhel_version} > 100
    ..do something..
    %else
    ..do something else..
    %endif

One day you want to clean the spec file by dropping support for old
systems and other vendors or something else that your spec file checks
for. Unfortunately doing it manually can be error prone...

Speclean takes as input a spec file and some variable definitions
(int, range of ints, string, undef) and tries to simplify tests within
the spec file given the variable values (or lack there of), removing
all unreachable code as a result.

The result of the simplifying process is printed out as a unified diff
that can be applied with the usual patch utility.

So far speclean supports 4 types of variable definitions:

- undef: assume the var is always undefined (e.g. `0%{?var}` always
  expands to 0)
- int: assume the var is always defined to a specific
  integer.  (e.g. `%{build_with_foo}` always expands to 1)
- string: assume the var is always defined to a specific string (e.g
  `%{_vendor}` always expands to "suse").
- ranged integer: assume the var is always defined to an integer
  within a [min;max] range (e.g `%{suse_version}` will always be greater
  or equal to 1100).

Usage
=====

Each type can be used through command-line options:

    -u VAR, --undef VAR   assume VAR to be undefined
    -s VAR=STRING, --string VAR=STRING
                          assume VAR to defined to the string
    -i VAR=INT, --int VAR=INT
                          assume VAR to defined to an integer
    -r VAR=MIN,MAX, --range VAR=MIN,MAX
                   assume VAR to be defined to an integer in the
                   interval MIN,MAX (included). You can use "inf" or "-inf".


You can also test out an conditional expression simplification (mostly
for debugging) with `-e`:

    $ speclean      \
      -r v=-1,inf   \
      -e "((%v > -2) && (%other == 42)) || (((%v < -10) && ((%v+1)*1) >= 0))"

    PARSING ((%v > -2) && (%other == 42)) || (((%v < -10) && ((%v+1)*1) >= 0))
      assuming %v = [-1 ; inf]
    => ((%v > -2) && (%other == 42)) || ((%v < -10) && ((%v + 1) * 1) >= 0)
    => %other == 42


Implementation
==============

The spec syntax is very poorly documented. Most of the doc only
describe the most commonly used patterns and the only solution is
often to look at the rpm source code:

- macro expansion: rpm/rpmio/macro.c build/parseSpec.c
- conditionnal expression: rpm/build/expression.c

speclean tries to properly tokenize and parse conditionnal expressions
using a tokenizer/parser generator. But this is tricky because
similarly to C files, rpm spec files are pre-processed and it seems
the 2 pass are not completely independant.

That means you can define sneaky things such as:

    %define var_is_4 == 4
    %if %foo %{var_is_4}

It's hard to reason about the source before its pre-processed, and
parsing *after* the pre-processing makes little sense as no variable
are present anymore.

The approach speclean takes is to only parse the unprocessed
conditional expressions and assume its written in a sane subset of the
spec language. When speclean cannot properly parse a condition it just
leaves it as is.

For the moment, speclean just looks at every `%if`/`%else`/`%endif`
branch and simplifies them independantly. It ignores `%define` and
doesn't try to take advantage nested conditions to assume more things
about values within branches.

Example
=======

The EXAMPLE file contains an example of a speclean run.
