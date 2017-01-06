# xeno

A fast event-based XML parser.

## Features

* It's a SAX-style/fold parser which triggers events for open/close
  tags, attributes, text, etc.
* It handles comments.
* It uses very low memory (see memory benchmarks below).
* It's very fast (see speed benchmarks below).
* It's written in pure Haskell.

## Example

Quickly dumping XML:

``` haskell
> let input = "Text<tag prop='value'>Hello, World!</tag><x><y prop=\"x\">Content!</y></x>Trailing."
> dump input
"Text"
<tag prop="value">
  "Hello, World!"
</tag>
<x>
  <y prop="x">
    "Content!"
  </y>
</x>
"Trailing."
```

Folding over XML:

``` haskell
> fold const (\m _ _ -> m + 1) const const const 0 input -- Count elements.
2
```

``` haskell
> fold (\m _ -> m + 1) (\m _ _ -> m) const const const 0 input -- Count attributes.
3
```

Most general XML processor:

``` haskell
process
  :: Monad m
  => (ByteString -> m ())               -- ^ Open tag.
  -> (ByteString -> ByteString -> m ()) -- ^ Tag attribute.
  -> (ByteString -> m ())               -- ^ End open tag.
  -> (ByteString -> m ())               -- ^ Text.
  -> (ByteString -> m ())               -- ^ Close tag.
  -> ByteString                         -- ^ Input string.
  -> m ()
```

You can use any monad you want. IO, State, etc. For example, `fold` is
implemented like this:

``` haskell
fold openF attrF endOpenF textF closeF s str =
  execState
    (process
       (\name -> modify (\s' -> openF s' name))
       (\key value -> modify (\s' -> attrF s' key value))
       (\name -> modify (\s' -> endOpenF s' name))
       (\text -> modify (\s' -> textF s' text))
       (\name -> modify (\s' -> closeF s' name))
       str)
    s
```

The `process` is marked as INLINE, which means use-sites of it will
inline, and your particular monad's type will be potentially erased
for great performance.

## Performance goals

The [hexml](https://github.com/ndmitchell/hexml) Haskell library uses
an XML parser written in C, so that is the baseline we're trying to
beat or match roughly.

It currently is faster than Hexml for simply walking the
document. Hexml actually does more work, by allocating a big vector
and writing nodes and attributes to it, which, when I tested out
implementing with Xeno, brought the speed benchmarks to the same speed
as Hexml more or less.

Memory benchmarks for Xeno:

    Case                Bytes  GCs  Check
    4kb/xeno/sax        2,376    0  OK
    31kb/xeno/sax       1,824    0  OK
    211kb/xeno/sax     56,832    0  OK
    4kb/xeno/dom       11,360    0  OK
    31kb/xeno/dom      10,352    0  OK
    211kb/xeno/dom  1,082,816    0  OK

Speed benchmarks:

    benchmarking 4KB/hexml/dom
    time                 6.317 μs   (6.279 μs .. 6.354 μs)
                         1.000 R²   (1.000 R² .. 1.000 R²)
    mean                 6.333 μs   (6.307 μs .. 6.362 μs)
    std dev              97.15 ns   (77.15 ns .. 125.3 ns)
    variance introduced by outliers: 13% (moderately inflated)

    benchmarking 4KB/xeno/sax
    time                 5.152 μs   (5.131 μs .. 5.179 μs)
                         1.000 R²   (1.000 R² .. 1.000 R²)
    mean                 5.139 μs   (5.128 μs .. 5.161 μs)
    std dev              58.02 ns   (41.25 ns .. 85.41 ns)

    benchmarking 4KB/xeno/dom
    time                 10.93 μs   (10.83 μs .. 11.14 μs)
                         0.994 R²   (0.983 R² .. 0.999 R²)
    mean                 11.35 μs   (11.12 μs .. 11.91 μs)
    std dev              1.188 μs   (458.7 ns .. 2.148 μs)
    variance introduced by outliers: 87% (severely inflated)

    benchmarking 31KB/hexml/dom
    time                 9.405 μs   (9.348 μs .. 9.480 μs)
                         0.999 R²   (0.998 R² .. 0.999 R²)
    mean                 9.745 μs   (9.599 μs .. 10.06 μs)
    std dev              745.3 ns   (598.6 ns .. 902.4 ns)
    variance introduced by outliers: 78% (severely inflated)

    benchmarking 31KB/xeno/sax
    time                 2.736 μs   (2.723 μs .. 2.753 μs)
                         1.000 R²   (1.000 R² .. 1.000 R²)
    mean                 2.757 μs   (2.742 μs .. 2.791 μs)
    std dev              76.93 ns   (43.62 ns .. 136.1 ns)
    variance introduced by outliers: 35% (moderately inflated)

    benchmarking 31KB/xeno/dom
    time                 5.767 μs   (5.735 μs .. 5.814 μs)
                         0.999 R²   (0.999 R² .. 1.000 R²)
    mean                 5.759 μs   (5.728 μs .. 5.810 μs)
    std dev              127.3 ns   (79.02 ns .. 177.2 ns)
    variance introduced by outliers: 24% (moderately inflated)

    benchmarking 211KB/hexml/dom
    time                 260.3 μs   (259.8 μs .. 260.8 μs)
                         1.000 R²   (1.000 R² .. 1.000 R²)
    mean                 259.9 μs   (259.7 μs .. 260.3 μs)
    std dev              959.9 ns   (821.8 ns .. 1.178 μs)

    benchmarking 211KB/xeno/sax
    time                 249.2 μs   (248.5 μs .. 250.1 μs)
                         1.000 R²   (1.000 R² .. 1.000 R²)
    mean                 251.5 μs   (250.6 μs .. 253.0 μs)
    std dev              3.944 μs   (3.032 μs .. 5.345 μs)

    benchmarking 211KB/xeno/dom
    time                 543.1 μs   (539.4 μs .. 547.0 μs)
                         0.999 R²   (0.999 R² .. 1.000 R²)
    mean                 550.0 μs   (545.3 μs .. 553.6 μs)
    std dev              14.39 μs   (12.45 μs .. 17.12 μs)
    variance introduced by outliers: 17% (moderately inflated)
