# I18nema

Fast I18n backend to keep things running smoothly.

I18nema is a drop-in replacement for I18n::Backend::Simple, for faster
lookups (2x!) and quicker GC runs (ymmv). Translations are stored
outside of the ruby heap, and lookups happen in C (rather than the usual
inject on nested ruby hashes).

## How do I use it?

    gem 'i18nema'

and then do something like this in an initializer:

    I18n.backend = I18nema::Backend.new

You can pull in additional features, e.g.

    I18nema::Backend.send(:include, I18n::Backend::Fallbacks)

As with regular I18n, you should probably load translations before you
fork, so that all processes can use the same translations in memory. In
an initializer, just do `I18n.backend.init_translations`.

## What sort of improvements will I see?

### Faster Startup

Loading all the translations into memory is dramatically faster with
I18nema (about 4x). While this is just a one time hit, it's pretty
noticeable when you're waiting on it (e.g. console, specs). In
[canvas-lms](https://github/com/instructure/canvas-lms), I18nema brings
it down to just over half a second (from almost 2.5).

### Faster GC Runs

Because there are fewer ruby objects, the periodic GC runs will be
proportionally faster. How much faster is a question of how many
translations you have versus how many other ruby objects. Applications
that are localized in more languages should see a bigger boost (since
the translations make up a bigger share of the original ObjectSpace).

For example, [canvas-lms](https://github/com/instructure/canvas-lms) is
translated into seven other languages, and I18nema reduces (startup)
ObjectSpace by about 18% and GC runtime by about 11%.

I18nema also moves I18n's normalized_key_cache into C structs. This key
cache grows over time (it eventually holds a key/value for every
translation key used in the app), so that's another area where I18nema
is nicer on ObjectSpace than vanilla I18n.

### Faster Translation Lookups

Simple lookups (i.e. no options or interpolation) take about 55% less
time, and that holds true no matter how deeply scoped your keys are
(e.g. `foo.bar.baz.lol`).

Lookups with options are also faster (about 35% less time). The gains
aren't quite as big there yet, since proportionally more of the time is
spent in unoptimized places like `I18n#interpolate`.

## Show me the benchmarks

Here are some basic ones done with `Benchmark.bmbm` (edited for brevity)
We run `I18n.translate` 100000 times on 4 different translation keys.
The `n` in `translate(n)` denotes how many parts there are in the key,
e.g. `I18n.t('foo') -> 1`, `I18n.t('foo.bar') -> 2`

### simple `translate` (no options)

#### I18nema

                        user     system      total        real
    translate(1):   0.960000   0.010000   0.970000 (  0.963384)
    translate(2):   1.070000   0.010000   1.080000 (  1.068789)
    translate(3):   1.080000   0.000000   1.080000 (  1.083826)
    translate(4):   1.260000   0.010000   1.270000 (  1.263967)

#### I18n

                        user     system      total        real
    translate(1):   2.140000   0.000000   2.140000 (  2.137579)
    translate(2):   2.380000   0.000000   2.380000 (  2.382720)
    translate(3):   2.420000   0.000000   2.420000 (  2.428245)
    translate(4):   2.510000   0.000000   2.510000 (  2.514133)

### `translate` with options (locale, interpolation)

#### I18nema

                        user     system      total        real
    translate(1):   1.770000   0.000000   1.770000 (  1.771934)
    translate(2):   1.870000   0.010000   1.880000 (  1.880586)
    translate(3):   1.890000   0.000000   1.890000 (  1.893065)
    translate(4):   2.080000   0.000000   2.080000 (  2.083093)

#### I18n

                        user     system      total        real
    translate(1):   2.720000   0.000000   2.720000 (  2.720541)
    translate(2):   2.950000   0.010000   2.960000 (  2.953175)
    translate(3):   3.010000   0.010000   3.020000 (  3.028288)
    translate(4):   3.120000   0.010000   3.130000 (  3.117135)


## OK, so what's the catch?

I18nema is still a work in progress, so there are some compatibility
notes you should be aware of:

I18nema requires ruby 1.9.3 or later.

I18nema only supports `.yml` translation files (no `.rb`).

I18nema requires UTF-8 `.yml` files. That means that your translations
should actually be in their UTF-8 form (e.g. "Contraseña"), not some
escaped representation. I18nema uses a simplified syck implementation
and does not support many optional yml types (e.g. `binary`).

I18nema doesn't yet support symbols as translation *values* (note that
symbol [keys](http://guides.rubyonrails.org/i18n.html#basic-lookup-scopes-and-nested-keys)
and [defaults](http://guides.rubyonrails.org/i18n.html#defaults) work
just fine). Symbol values in your `.yml` file can be used in the same
way that symbol defaults can, i.e. they tell I18n to find the
translation under some other key.

