---
layout: post
title: Reverse Engineering Regular Expressions
---
I recently published a powerful ruby gem on Github: [regexp-examples](https://github.com/tom-lord/regexp-examples).
This library allows you to generate all (or one random) strings that match any regular expression!
(With just [a few limitations](https://github.com/tom-lord/regexp-examples#impossible-features-illegal-syntax)
on what's possible.) To install it yourself and have a quick play is dead easy:

    > gem install regexp-examples
      Fetching: regexp-examples-1.3.1.gem (100%)
      Successfully installed regexp-examples-1.3.1
      Parsing documentation for regexp-examples-1.3.1
      Installing ri documentation for regexp-examples-1.3.1
      Done installing documentation for regexp-examples after 0 seconds
      1 gem installed
    > pry
    [1] pry(main)> require 'regexp-examples'
      => true
    [2] pry(main)> ls /rexex/
      CoreExtensions::Regexp::Examples#methods: examples  random_example
      Regexp#methods: ==  ===  =~  casefold?  encoding  eql?  fixed_encoding?
        hash  inspect  match  named_captures  names  options  source  to_s  ~
    [3] pry(main)> /quick|little|demo/.examples
      => ["quick", "little", "demo"]
    [4] pry(main)> /\w{10}@(hotmail|gmail)\.com <--Backref! \1/.random_example
      => "cpRe_f2vMI@gmail.com <--Backref! gmail"

In this post, I will try to explain some of the core logic and techniques that I used to build this library. What is a regular expression, really? How/why is it always possible de-construct a regex, to list all possible strings that match it? How on Earth did I get back-references to work?!

All shall be revealed...

## What Is A "Regular" Expression?

> ### Regular Expression
> *noun, computing*
>
> a sequence of symbols and characters expressing a string or pattern to
> be searched for within a longer piece of text.

Perhaps the most confusing aspect of regular expressions comes from their formal definition, and the fact that several features in the regex language are not really "regular" at all! These "irregular" pieces of syntax are, in short (and by no coincidence!), the "illegal syntax" in my regexp-examples gem.

However, rather than mysteriously telling you what a regular expression isn't, let's try to explain what it is:

There are only _really_ **four** (yes, that's right, **four!**) pieces of syntax allowed in a "true" regular expression:

1. The "empty string", usually denoted by: `ε`
2. Literal characters, e.g. `/abc123/`
3. The `*` repeater, e.g. `/a*b*c*/`
4. The `|` ("Or") operator, e.g. `/a|b|c/`

Oh, and there's also brackets - so maybe five pieces of syntax, if you want to count those as well!

Every other piece of syntax is really just a nice way to simplify writing out horrendously long, complicated combinations of the above. Let's try a few examples:

```ruby
/a+/     == /a|a*/
/a?/     == /ε|a/
/a{2,4}/ == /aa|aaa|aaaa/
/a{3,}   == /aaaa*/
/[a-d]*/ == /(a|b|c|d)*/
/\d\n?/  == /(0|1|2|3|4|5|6|7|8|9)(ε|\n)

# Note: My use of the == operator here is not to be taken
# too literally... In ruby, Regexp equality is not based
# purely on what strings they match!
# http://ruby-doc.org/core/Regexp.html#method-i-3D-3D
```

...Hopefully, you get the idea. To put this another way, any regex that _can't_ be expressed in this way is _not_ truly a "regular" expression!

An easy way to see whether or not this is the case is: Does (part of) the regex need to know its surrounding context, in order to determine a match? For example:

```ruby
/\bword\b/ # How does "\b" know whether it lies on a word boundary?
/line1\n^line2/ # How does "^" know whether it lies at the start of a line?
/irregular (?=expression)/ # Or in general, how can any "look-ahead"/"look-behind" be regular?
```

These all need to know "what came before", or "what comes next", and are therefore not _True Regular Expressions™_. Hopefully this makes the common claim that "back-references are not regular" a little more obvious to understand: You need to know what the capture group matches before you can know what the back-reference matches (i.e. knowledge of context). So of course you cannot express such patterns using only those four symbols!

One final point to make, before we move on: There is only really one type of repeater in regex; the others are all nothing more than shorthand:

```ruby
/a/  == /a{1}/
/a?/ == /a{0,1}/
/a*/ == /a{0,}/
/a+/ == /a{1,}/
```

Understanding this structure is at the very heart of my ruby gem; the whole library architecture depends on (and, for some occasional edge cases, is restricted by!) it.

All _True Regular Expressions™_ are built using this structure:

```ruby
/Group-Repeater-Group-Repeater-Group-Repeater-.../
```

Where every group can, itself, be built using that same structure.

> What?! Show me some examples!

I'm glad you asked. Consider the following:

```ruby
/(this|that)+/
== /(this|that){1,}/
== /(t{1}h{1}i{1}s{1}|t{1}h{1}a{1}t{1}){1,}/
```

(Yuck! Thankfully we don't normally need to write them out like this!...)
But this lays the foundations for the main purpose of this blog post:

## How To Parse A Regular Expression

<p align="center"><img src="/images/bunny_hat.jpg"></p>

Without getting bogged down in the nitty-gritty implementation details of parsing, let's dive straight in and look at the internal objects generated by `RegexpExamples::Parser`:

  Picture

This may look complicated, but it's essentially not much different to what I described above. There is only one key additional thing to consider: In order to avoid problems with infinity, we must restrict repeaters like * and + to have an upper limit. Taken straight from the gem's README:

  Picture

Or, in other words, the above regex has been interpreted as equivalent to the following:

```ruby
/(a{0,2}|b{1,3}){1}/
```

Like I said above: `/Group-Repeater-Group-Repeater-Group-Repeater-.../`

## How To Generate Examples From A Regular Expression

<p align="center"><img src="/images/drumroll_please.jpg"></p>

So, we have our parsed regex. All that remains is to transform this into its possible strings. The trick to this is that all groups and repeaters are given a special method: #result. These results are then built up, piece by piece, to form the full strings that match the regex. Let's take the above example, one step at a time:

* The `SingleCharGroup` (`"a"`) has one possible result: `["a"]`
* Therefore the `StarRepeater` has three possible results: `["", "a", "aa"]`
* Similarly, `SingleCharGroup` (`"b"`) has one possible result: `["b"]`
* Therefore, `PlusRepeater` has three possible results: `["b", "bb", "bbb"]`
* Next, the `OrGroup` simply concatenates these arrays of possible results,
i.e. it has six possible results: `["", "a", "aa", "b", "bb", "bbb"]`
* And finally, the top level `OneTimeRepeater` just returns these same values.

And there you have it, for a fairly simple example! Let's look at one more, to demonstrate perhaps the most important method in the whole gem:

  Picture

Once again, we make use of `PlusRepeater#result` and `SingleCharGroup#result` to build up the final answer from each "partial result".

However, in this case we end up with the following:

```ruby
[["a", "aa", "aaa"], ["b", "bb", "bbb"], ["c", "cc", "ccc"]]
```

Where each of those inner arrays is the result of each `PlusRepeater`. We need to make one more step: Find all possible results, from joining one element from each array, to form a "final result" string. Enter the magic glue that holds this whole thing together:

  Picture

<sub>\*I've actually simplified this method slightly, to avoid confusion. The real deal can be found here.</sub>

And so, after applying this method to the above array, we end up with:

```ruby
["abc", "abcc", "abccc", "abbc", "abbcc", .....]
```

This method gets used a lot, when dealing with more complicated regexes! It is the magic function that allows patterns to be made arbitrarily complicated, with unlimited nesting of groups and so on.

So, there you have it! Now you understand all about how to generate examples from regular expressions, right?...

<p align="center"><img src="/images/iceberg.jpg"></p>

Oh, but...

* How do you deal with escaped characters, like `\d`, `\W`, etc?
* What about regexp options (ignorecase, multiline, extended form)?
* What about unicode characters, control codes, named properties, and so on?
* How on earth do you correctly parse all of the possible syntax in character sets, such as:

```ruby
/[abc]/.examples
/[a-z]/.examples
/[^\d\ba-c]/.examples
/[[:alpha:]&&[a-c]]/.examples
```

...And I'm barely getting started here! There is a huge range of syntax to consider!


To cut a long story short: parsing is complicated!! However, all the basic principles discussed above still apply.

There is just one final piece of the puzzle that I have mostly avoided up until this point: back-references.

As discussed earlier, back-references are not regular, as they require knowledge of context. They are not strictly possible to fully support with this gem! (And indeed, there are some rare edge cases where my solution does not work.)

But, as promised, all shall be revealed...

<p align="center"><img src="/images/fireworks.jpg"></p>

## How to generate examples with back-references

The important thing to recognise here is that you cannot know what the back-references need to match, until after the rest of the regex example has been generated. For example, consider the following:

```ruby
/(a|b)\1/.random_example
```

You cannot possibly know whether the `\1` is meant to be an `"a"` or a `"b"`, until after the capture group's "partial example" is chosen!

The solution? We cheat, and use a place-holder - then substitute the correct pattern back in later!

The pattern I chose is: `__X__`, where `X` is the name of your back-reference (in this case, `"1"`).

There is a lot of intricate logic involved in actually keeping track of the results of these capture groups (perhaps the topic for a follow-up blog post?), so let's gloss over this detail for now. So in summary, examples for the above regex are calculated as follows:

* The `SingleCharGroup` (`"a"`) has one possible result: `["a"]`
* The `SingleCharGroup` (`"b"`) has one possible result: `["b"]`
* The `OrGroup` has two possible results: `["a", "b"]`
* The `MultiGroup` with `group_id=1` has two possible results: `["a", "b"]`
* The `BackReferenceGroup` has one possible result: `["__1__"]`
* This gives us a final array of possible results: `[["a", "b"], ["__1__"]]`
* After applying the `permutations_of_strings` method, this gives us two "final results": `["a__1__", "b__1__"]`
* We now do one final step: Apply a `#substitute_backreferences` method on each string, to reveal the true strings that match the original regex: `["aa", "bb"]`

And now finally, young Padawan, you are ready to see the actual implementation of `Regexp#examples`:

  Picture

<sub>\*Once again, I've been naughty and shown you a slightly simplified version, to avoid confusion. See the real thing over here.</sub>

I leave you with one final example, showing the true power of this gem:

**Question:** What the hell does [this ridiculous regex](http://emailregex.com/) match?! (Side note: It's usually a bad idea to validate email addresses like this! If you want to ensure the address is correct, just send a confirmation email!)

**Answer:** (On my average machine, it takes ~0.01 seconds to generate an example string!!)
