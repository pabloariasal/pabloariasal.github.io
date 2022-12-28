---
layout: post
title: Vim - Advanced Text Editing For The Lazy
tags: [vim]
---

Life is nothing but an endless stream of challenges. The past is made out of fought battles and the future never fails to offer new ones.
But, as Marcus Aurelius once said, never let the future disturb you. You will meet it, if you have to, with the same weapons of reason which today arm you against the present.

My next challenge was going to be Google Mock's new release. In older versions of gMock the following macro was used to mock a member function:

```
MOCK_METHOD<number_of_function_parameters>(<function_name>, <return_type>(<parameter_types>))
```
For example, in the case of `bool foo(int)` this would be:

```
MOCK_METHOD1(foo, bool(int))
```

In a new release, gMock changed the format of the macro to:

```
MOCK_METHOD(<return_type>, <function_name>, (<parameter_types>))
```
Which would be equivalent to:

```
MOCK_METHOD(bool, foo, (int))
```

The most important change is that the number of parameters isn't longer hardcoded in the macro's name, `MOCK_METHOD` can be used for all functions instead of `MOCK_METHOD1`, `MOCK_METHOD2`, etc.

Additionally, the order of
arguments changed to the more natural (`<return_type>, <function_name>, (<parameters>))` style over
`(<function_name>, <return_type>(<parameters>))`.

Imagine someone has several dozens of such macros that need to be updated to the new format. How much time would a person need to accomplish this task? The answer is: it depends. Which text editor does such person use?

This post presents a step by step guide on how you can use Vim to perform these changes automatically. Editing text is your editor's job, after all, not yours.

# The Plan

Proficient coders may have immediately seen that there is only one reasonable
solution to this problem: regular expressions. This is the kind of problem
regexes are completely unbeatable at.

The solution is as simple as a global search-replace using capture groups:

1. Define a regex that matches an old-syntax `MOCK_METHOD` macro
2. Use capture groups inside the regex to capture:
    * the parameters
    * the return type
    * the function name
3. Globally search and replace each old-style macro and format it to the new syntax, changing the order of components using the capture groups.

Don't worry if you don't know what I am talking about, we'll go over the
solution step by step.

# The Problem

As explained previously, the solution builds on top of regular expressions.
But there is a problem, some of the macros in the code base looked like this:

```
MOCK_METHOD3(veryLongFunctionName,
                bool (int, float, char))
```

The macros we need to replace may expand across an multiple lines...

Given that line breaks can occur anywhere and any number of times in the pattern, writing a regular expression matching multiline macros becomes a Turing-award winning task. Is this even still a regular language?

I wasn't bothered the slightest by this, though. I can tell my editor to flatten all multiline `MOCK_METHODs` into single lines for me.

# The Solution

VIM is _powerful_. I can tell VIM to automatically convert this:

```
MOCK_METHOD2(Bar,
                bool
                    (int, float))
```

to this:

```
MOCK_METHOD2(Bar, bool(int, float))
```

Let's see how.

## 1) Find Relevant Occurrences

The first step is to find all occurrences of `MOCK_METHOD` macros in the code. This will provide us with a list of files we can work with.

This, off course, we do by with `grep`:

```
:grep -i 'mock_method\d+\('
```
`grep` will populate the quickfix list with all occurrences of `MOCK_METHOD` macros in the code.

I use [RipGrep](https://github.com/BurntSushi/ripgrep) as grep replacement:

```
set grepprg=rg\ -H\ --no-heading\ --vimgrep
```

`rg` is friendlier than GNU `grep` as it automatically looks for matches in all files in the current directory recursively (as long as they are not hidden or gitignored).

You can also provide glob patterns to restrict the files that are considered:

```
:grep -i -g "*.cpp" -g "*.h" 'mock_method\d+'
```

If you use GNU `grep`, you have to explicitly provide the files to be searched:

```
:grep -i 'mock_method\d+' **/*.cpp **/*.h
```
At this point we can open the quickfixlist with `:copen` and will have a list of all the macros we need to replace, along with the file, line and column they appear at.

## 2) Flatten A Single Macro

Now that we have conveniently created a list of all `MOCK_METHOD` macros in the code we can tell Vim to flatten them.
For this we can record a simple macro:

First we start recording a macro in register `w`:

```
qw
```

Then we type the following sequence:

```
nf(v%J
```

This macro does the following:

1. `n` jump to the next occurrence of the search pattern in the buffer
2. `f(` move the cursor to the next `(`
3. `v%` visually select everything from the opening `(` to the closing `)`
4. `J` join all selected lines into a single one

The idea here is quite simple: use Vim's `%` motion, which jumps to the closing symbol of a bracket, to select everything inside a `MOCK_METHOD` and join all the lines into one.

So, playing this macro using `@w` on the first line of a  `MOCK_METHOD`, will convert this:

```
MOCK_METHOD2(Bar,
                bool
                    (int, float))
```

to this:

```
MOCK_METHOD2(Bar, bool(int, float))
```

## 3) Repeat For All Macros

We have defined a macro that we can use to flatten a multiline `MOCK_METHOD`, but how do we play the macro on all occurrences?

Recall that when executing the macro we first jump to the next match of the search pattern, but what is the search pattern?
We haven't defined it yet! Let's do that now.

Remember that we are trying to match every instance of `MOCK_METHOD` that spans across multiple lines.
In other words, we have to match every line that contains a `MOCK_METHOD` but doesn't end with a semicolon:

```
/MOCK_METHOD[^;]*$
```
The regex will match a macro of the form:

```
MOCK_METHOD1(Foo,
        bool(int))
```
as it doesn't end with a semicolon, but will ignore this:

```
MOCK_METHOD1(Foo, bool(int));
```

We can now flatten all multiline `MOCK_METHODs` by repeatedly replaying the macro using `@w`. The macro will jump to next match in the buffer and flatten it automatically.

After we have flattened all `MOCK_METHOD`s in the current buffer we can move to the next one with `:cnfile`: the next file saved in the quickfixlist.

But we are lazy, very lazy, why not automate everything?

```
:cfdo normal 99@w | update
```

This will execute the macro stored in register `w` 99 times in every one of the buffers in the quickfixlist. Don't worry about the 99 times, this is a common way of telling Vim to keep executing the macro "infinitely"  until there are no more matches in the buffer. Infinity is approximated by a large number. It's a poor man's `while(true)`.

`| update` tells Vim to save the buffer to disk before jumping to the next one. Per default, Vim won't allow you to change buffers if there are unsaved changes in them (see `:help nohidden`).

# Regex to The Rescue

Now that all `MOCK_METHODs` have been flattened to a single line, we can do the long promised regex search and replace to change the format of mock method macros to gMock's new style.

Remember, this:

```
MOCK_METHOD<number_of_function_parameters>(<function_name>, <return_type>(<parameter_types>))
```

must be transformed into this:

```
MOCK_METHOD(<return_type>, <function_name>, (<parameter_types>))
```

This can be accomplished with the following substitution:

```
s/\vMOCK_METHOD\d+\(([^,]+),\s+([^\(]+)\(([^\)]*)\)\)/MOCK_METHOD(\2, \1, (\3))/
```

Note the usage of Vim's very magic flag `\v`, which makes characters like `(` or `+` have a special meaning instead of matching literally. This is important for the readability of the regex, without it all special symbols would have to be escaped.

The search pattern matches a string in the form: `MOCK_METHOD<number_of_parameters>(<function_name>, <return_type>(<parameter_types>))`, the old gMock style.

## Capture Groups
Note that we are putting pairs of `()` in special places: we are *capturing* certain parts of the match.

The first capture group is the function name, the second the return type, and the third the function parameters. Capture groups are very useful because we can refer to them in the replace section of the substitution using `\1` or `\2` .

For example, if you want to replace all occurrences of `ab` by `ba`, you can do:

```
%s/\v(a)(b)/\2\1/g
```
We use this me mechanism to transform an old-style `MOCK_METHOD` macro into the new style, which is basically a reordering of parts.

The substitute command we defined transforms an old-style single line(!) macro into a new-style one, but how can we replace all?

Easy, we do the same trick and run a global substitution in all files in the quickfixlist:

```
:cfdo %s/\vMOCK_METHOD\d+\(([^,]+),\s+([^\(]+)\(([^\)]*)\)\)/MOCK_METHOD(\2, \1, (\3))//
```

# That's All Folks

And we are done, with some regex and effective usage of your tools we were able to circumvent a horrible manual task.
Contrary to popular belief, learning Vim is not for people who have a lot time in their hands, is for the ones who don't.
Vim is for the lovers of the quick and easy, for the devotes of the law of the least effort. Vim is for the impatient. Vim is for the lazy.
