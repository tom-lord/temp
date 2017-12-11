---
layout: post
title: 10 More New Features in Ruby v2.5
---

With the upcoming release of ruby v2.5 scheduled (as per tradition) for 25th December,
it's good to know what's changed in the language - so you can take advantage any new
(or refined) features.

[This popular blog post](https://blog.jetbrains.com/ruby/2017/10/10-new-features-in-ruby-2-5/)
by [Junichi Ito](https://twitter.com/jnchito) highlighted 10 new features; but since
there are [so many](https://docs.ruby-lang.org/en/trunk/NEWS.html) improvements to the language,
let's dive in and unravel _10 more_ handpicked highlights!

## 1. More public `Module` methods

(Features [#14132](https://bugs.ruby-lang.org/issues/14132) and
[#13133](https://bugs.ruby-lang.org/issues/14133)).

`Module#attr`, `attr_accessor`, `attr_reader`, `attr_writer`, `define_method`,
`alias_method`, `undef_method` and `remove_method` are now *all* public.

For example:

```ruby
# Ruby 2.4
Integer.alias_method :plus, :+
#=> NoMethodError: private method `alias_method' called for Integer:Class

# Workaround 1:
Integer.send(:alias_method, :plus, :+)

# Workaround 2:
class Integer
  alias_method :plus, :+
end

# Ruby 2.5
Integer.alias_method :plus, :+
```

This update follows nicely from similar changes back in Ruby v2.1.0, where `Module#include`
and `Module#prepend` were [made public](https://github.com/ruby/ruby/blob/v2_1_0/NEWS#core-classes-updates-outstanding-ones-only).

I think this is a great example of ruby being a community-driven language: Its design
has evolved over time, influenced by how its users _want_ it to be; not set in stone
due to the original designers' opinions.

## 2. `String#start_with?` supports regexp

(Feature [#13712](https://bugs.ruby-lang.org/issues/13712)).

This enhancement provides some nice syntactic sugar, which could help prevent a
common (and sometimes security-related) mistake in ruby:

```ruby
# Checking whether a string starts with a lower-case letter

string.match?(/^[a-z]/) # BAD!
string.match?(/\A[a-z]/) # Good
```

The mistake arrises because, unlike most other languages, the `^` anchor means "starts
of _line_" in ruby; not "start of _string_". (That's what `\A` is for.)

So, that first line of code would also return `true` when `string = "123\ntest"`.

Using the new `String#start_with?` method, such a mistake wouldn't happen:

```ruby
# Ruby v2.5
string.start_with?(/[a-z]/) # Also good
```

## 3. Improvements to `binding.irb`

(Features [#13099](https://bugs.ruby-lang.org/issues/13099) and [#14124](https://bugs.ruby-lang.org/issues/14124)).

For a few years now, the [recommended ruby toolkit](https://www.sitepoint.com/rubyists-time-pry-irb/)
for developer consoles and debuggers has been dominated by [`pry`](https://github.com/pry/pry), [`byebug`](https://github.com/deivid-rodriguez/byebug),
or even some [hybrid](https://github.com/deivid-rodriguez/pry-byebug) of the above.

...Which is a little odd, since ruby already comes with a pretty good built-in REPL: `irb`
("Interactive Ruby").

But as of ruby `2.5`, we see two enhancements to the library that help bridge the gap,
and make it a little less likely to feel the need for `pry` in every application:

* `require 'irb'` is no longer needed in your code, in order to invoke `binding.irb`.
* Show source around `binding.irb` is now shown on startup.

```ruby
# Ruby v2.4

# test.rb:
require 'irb'
def test
  binding.irb
end
test

# Running the file yields:
irb(main):001:0>

# Ruby v2.5:
# test.rb:
def test
  binding.irb
end
test

# Running the file yields:
From: test.rb @ line 2 :

    1: def test
 => 2:   binding.irb
    3: end
    4: test

irb(main):001:0>
```

## 4. `Integer.sqrt` added

(Feature [#13219](https://bugs.ruby-lang.org/issues/13219))

Quite often in algorithms, you need to find "the largest integer less than `sqrt(n)`".
For example, a simple algorithm to check whether `n` is prime is to simply try dividing
`n` by all primes up to `sqrt(n)`.

This operation can now be executed more succinctly, within the core library methods:

```ruby
# Ruby v2.4
Math.sqrt(4) #=> 2.0
Math.sqrt(4).to_i #=> 2

Math.sqrt(10) #=> 3.1622776601683795
Math.sqrt(10).to_i #=> 3

# Ruby v2.5
Integer.sqrt(4) #=> 2
Integer.sqrt(10) #=> 3
```

## 5. `String#casecmp` and `casecmp?` now return nil for non-string arguments

(...instead of raising a `TypeError`)

(Bug [#13312](https://bugs.ruby-lang.org/issues/13312)).

Fairly self-explanatory:

```ruby
# Ruby v2.4
"foo".casecmp 123
#=> TypeError: no implicit conversion of Integer into String

# Ruby v2.5
"foo".casecmp 123
#=> nil
```

Changes like this usually slip under the radar, as they are not "exciting" features.
But it just goes to show: There's still all sorts of simple room for improvement in
the language; and sometimes the change you wish to
[make may not even be very complicated](https://bugs.ruby-lang.org/attachments/6464/patch.diff)!

But all of these many incremental changes add up, and are what makes modern ruby
such a robust, stable language.

## 6. `mathn.rb` removed from stdlib

(Feature [#10169](https://bugs.ruby-lang.org/issues/10169)).

It may sound odd to announce the "removal" of a library as a new feature, but
please hear me out!...

[`mathn`](https://github.com/ruby/mathn) is an odd library. It not only adds to,
but also _changes_, the behaviour of `Integer`s!

When you `require 'mathn'`, this in turn load other libraries such as `require 'prime'`,
which adds the methods: `Integer.each_prime`, `Integer.from_prime_division`,
`Integer#prime?` and `Integer#prime?`.

However, more bizarrely, it also
[*redefines*](https://github.com/ruby/mathn/blob/93cee0b3309239748017b51688596962f43467f7/lib/mathn.rb#L67-L76)
`Integer#/` to be an alias for `Numeric#quo`!

```ruby
1/3 # => 0
(1/3).class #=> Integer

# In Ruby v2.5, this line will `raise LoadError: cannot load such file -- mathn`
# (Unless you also install the deprecated `mathn` gem.)
require 'mathn'
1/3 # => (1/3)
(1/3).class #=> Rational
```

This isn't _always_ a problem, but it certainly doesn't conform the the [principle
of least surprise](https://en.wikipedia.org/wiki/Principle_of_least_astonishment).

For example, there was [a case about 9 years ago](https://stackoverflow.com/a/3445121/1954610)
where the Rubinius VM essentially "blew up" due to some obscure code simply running
`require 'mathn'`. Even today, people are sometimes left confused by [different behaviour
on development vs production servers](https://stackoverflow.com/q/47457498/1954610),
due to some production-specific dependency loading the library!

This behaviour-changing aspect of the `mathn` library has been a known issue for
several years, now. The library
[was officially "deprecated" as of Ruby v2.2](https://bugs.ruby-lang.org/issues/10169),
but that doesn't stop people from using it of course!

However, thanks to this change in Ruby v2.5, you can now only load `mathn` *as a gem*.
This finally means:

* Developers will need to [bundle a deprecated gem](https://github.com/ruby/mathn#installation)
into their project, in order to use it. (Which means usage is much less likely!)
* ...And therefore, even if some dependency in your project _does_ require `mathn`,
you will have traceability of the root culprit in the `Gemfile.lock`!

## 7. Default template file encoding is changed from ASCII-8BIT to UTF-8 in erb command

(Bug [#14095](https://bugs.ruby-lang.org/issues/14095)).

Similar to #5 above, this just goes to show how open well the community works to improve
the language. What started out as a [discussion on Reddit](https://www.reddit.com/r/ruby/comments/7bszd5/setting_script_encoding_for_erb_file/)
was then raised with the core developers, and promptly fixed. And, once again,
the code fix was [simple](https://bugs.ruby-lang.org/projects/ruby-trunk/repository/revisions/60739/diff).

This (obscure) bug was in fact a lingering legacy of Ruby 1.9's implementation!
However, its fix could prove to be quite useful in conjunction with
[other changes](https://bugs.ruby-lang.org/projects/ruby-trunk/repository/revisions/58891/diff)
in Ruby 2.5.

## 8. `Hash#slice` added

(Feature [#8499](https://bugs.ruby-lang.org/issues/8499)).

The `ActiveSupport` library, which comes bundled with the popular Ruby on Rails
framework (but can also be used in isolation) provides many - often controversial -
extensions to Ruby's core classes.

Over the past few years, however, Ruby has gradually been [cherry-picking the "best"
features](http://mitrev.net/ruby/2015/11/13/the-operator-in-ruby/) and merging them
into the core language.
[`Hash#slice`](https://apidock.com/rails/ActiveSupport/CoreExtensions/Hash/Slice)
is one such method; continuing the trend.

```ruby
# Ruby v2.5
{a: 1, b: 2, c: 3}.slice(:a, :b)
#=> {:a=>1, :b=>2}
```

## 9. `SecureRandom.alphanumeric` added

(Feature [#10849](https://bugs.ruby-lang.org/issues/10849)).

One thing that's always struck me as a little odd in ruby, given its vastly rich
core library methods, is its lack of a clear "random string" generator.

[This StackOverflow question](https://stackoverflow.com/q/88311/1954610), for
example, demonstrates just some of the many (often surprising!) popular methods
used by developers, to generate a "random string". Here are a few of them:

```ruby
# 1.
(0...50).map { ('a'..'z').to_a[rand(26)] }.join

# 2.
('a'..'z').to_a.shuffle[0,8].join

# 3.
rand(36**length).to_s(36)

# 4.
require 'securerandom'
SecureRandom.hex
```

Each approach has various pros and cons with regard to its flexibility, readability
and entropy ("true randomness"). For instance, only option 4 above should be considered
_cryptographically secure_. Other `SecureRandom` methods include:

```ruby
SecureRandom.base64 # e.g. "/qWJPsvoxnSe17HrTlzQ7Q=="
SecureRandom.urlsafe_base64 # e.g. "IINbXW4YKjfoncJMpP-CkQ"
SecureRandom.random_bytes # e.g. "\t\x85\xB9\x9C\xD1\f\xD3\xE6t\xB4S^\v-\xFFo"
SecureRandom.random_number # e.g. 0.8993676724126768
```

Due to the common use case of needing a purely alphanumeric string, the new
[`SecureRandom.alphanumeric` method](https://github.com/ruby/ruby/blob/86a794a6c34c9412f497907736b4857739b7af3c/lib/securerandom.rb#L273-L293)
fits in nicely with this collection.

Unfortunately though, if you need to generate a _cryptographically secure_ random
string with some other specific characters, you'll need still to write something
a little clunky - e.g.

```ruby
# 20 random chars; chosen from a, b, c, d, e, v, w, x, y and z
(1..20).flat_map { [*'a'..'e', *'v'..'z'].sample(1, random: SecureRandom) }
```

... Or if you're just looking for a clean syntax for arbitrary random strings, why not
check out my [`regexp-examples`](https://github.com/tom-lord/regexp-examples#usage)
library?

## 10. `Net::HTTP::STATUS_CODES`

(Feature [#12935](https://bugs.ruby-lang.org/issues/12935)).

Ruby 2.5 has now [defined a hash of all HTTP status codes](https://github.com/ruby/ruby/blob/bd73d374715ae8ca6e53ebd4a32f3ae2d6542352/lib/net/http/status.rb#L21-L81)!

Previously, this was only recorded in [documentation](https://github.com/ruby/ruby/blob/86a794a6c34c9412f497907736b4857739b7af3c/lib/net/http.rb#L331-L387),
and it was left up to web frameworks such as Rails to [define their own](http://www.railsstatuscodes.com/)
"human-friendly" mappings.

Additionally, definitions of the following status codes have been added to ruby:

```
# Ruby v2.5
require 'net/http/status_codes'
Net::HTTP::STATUS_CODES
{
  # ...
  208 => 'Already Reported',
  308 => 'Permanent Redirect',
  421 => 'Misdirected Request',
  506 => 'Variant Also Negotiates',
  508 => 'Loop Detected',
  510 => 'Not Extended',
  # ...
}
```

## Bonus: Unicode 10.0 support

(Feature [#13685](https://bugs.ruby-lang.org/issues/13685)).

Keeping up with the new unicode standard, [released in June 2017](http://blog.unicode.org/2017/06/announcing-unicode-standard-version-100.html),
Ruby 2.5 has been updated to support the new characters.

The new characters include a [Bitcoin symbol](https://www.unicode.org/charts/PDF/Unicode-10.0/U100-20A0.pdf)
and a [56 new emojis](http://www.unicode.org/emoji/charts/emoji-released.html)!

<p align="center"><img src="https://imgs.xkcd.com/comics/vomiting_emoji_2x.png"></p>

## Conclusion

The ruby language has increasingly stabilised over the past few years. And while it's
fair to say that these changes are far less revolutionary than what we saw back in
Ruby 2.0 for instance, there is still a lot going on within this active community.

Ruby v2.5 will be released on Christmas Day, but you can
[download a pre-release version today](https://www.ruby-lang.org/en/downloads/releases/)
and try it out!
