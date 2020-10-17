# streaming-bytestring

[![Build](https://github.com/haskell-streaming/streaming-bytestring/workflows/Tests/badge.svg)](https://github.com/haskell-streaming/streaming-bytestring/actions)
[![Build Status](https://travis-ci.org/haskell-streaming/streaming-bytestring.svg?branch=master)](https://travis-ci.org/haskell-streaming/streaming-bytestring)
[![Hackage](https://img.shields.io/hackage/v/streaming-bytestring.svg)](https://hackage.haskell.org/package/streaming-bytestring)

This library enables fast and safe streaming of byte data, in either `Word8` or
`Char` form. It is a core addition to the [`streaming`
ecosystem](https://github.com/haskell-streaming/) and avoids the usual pitfalls
of combinbing lazy `ByteString`s with lazy `IO`.

This library is used by
[`streaming-attoparsec`](http://hackage.haskell.org/package/streaming-attoparsec)
to enable vanilla [Attoparsec](http://hackage.haskell.org/package/attoparsec)
parsers to work with `streaming` "for free".

## Usage

### Importing and Types

Modules from this library are intended to be imported qualified. To avoid
conflicts with both the `bytestring` library and `streaming`, we recommended `Q`
as the qualified name:

```haskell
import qualified Streaming.ByteString.Char8 as Q
```

Like the `bytestring` library, leaving off the `Char8` will expose an API based
on `Word8`. Following the philosophy of `streaming` that "the best API is the
one you already know", these APIs are based closely on `bytestring`. The core
type is `ByteStream m r`, where:

- `m`: The Monad used to fetch further chunks from the "source", usually `IO`.
- `r`: The terminal (return) value after all streaming has concluded, often `()`.

You can imagine this type to represent an infinitely-sized collection of bytes,
although internally it references a **strict** `ByteString` no larger than 32kb,
followed by monadic instructions to fetch further chunks.

### Examples

#### File Input

To open a file of any size (that can be represented as an `Int`) and count its
characters:

```haskell
import Control.Monad.Trans.Resource (MonadResource, runResourceT)
import qualified Streaming.Streaming.Char8 as Q

-- | Represents a potentially-infinite stream of `Char`.
chars :: MonadResource m => Q.ByteStream m ()
chars = Q.readFile "huge-file.txt"

main :: IO ()
main = runResourceT (Q.length_ chars) >>= print
```

Note that `readFile` specifically requires the
[`resourcet`](http://hackage.haskell.org/package/resourcet) library.
No further use of the opened stream should be made after `runResourceT`
returns, since the file will then be closed.

#### Line splitting and `Stream` interop

In the example above you may have noticed a lack of `Of` that we usually see
with `Stream`. Our old friend `lines` hints at this too:

```haskell
lines :: Monad m => ByteStream m r -> Stream (ByteStream m) m r
```

A stream-of-streams, yet no `Of` here either. The return type can't be the naïve
`Stream (Of ByteString) m r`, since the first line break might be at the very
end of a large file. Forcing that into a single strict `ByteString` could crash
your program.

The example below counts the number of lines in a file whose first letter is
`i`.  It is not the most efficient way to do this, but is rather an
opportunity to show various useful stream transformation functions:
```haskell
countOfI :: IO Int
countOfI = runResourceT
  . S.length_                   -- IO Int
  . S.filter (== 'i')           -- Stream (Of Char) IO ()
  . S.concat                    -- Stream (Of Char) IO ()
  . S.mapped Q.head             -- Stream (Of (Maybe Char)) IO ()
  . Q.lines                     -- Stream (ByteStream IO) IO ()
  $ Q.readFile "huge-file.txt"  -- ByteStream IO ()
```

Critically, there are several functions which when combined with `mapped` can
bring us back into `Of`-land:

```haskell
head     :: Monad m => ByteStream m r -> m (Of (Maybe Char) r)
last     :: Monad m => ByteStream m r -> m (Of (Maybe Char) r)
null     :: Monad m => ByteStream m r -> m (Of Bool) r)
count    :: Monad m => ByteStream m r -> m (Of Int) r)
toLazy   :: Monad m => ByteStream m r -> m (Of ByteString r) -- Be careful with this.
toStrict :: Monad m => ByteStream m r -> m (Of ByteString r) -- Be even *more* careful with this.
```

When moving in the opposite direction API-wise, consider:

```haskell
fromChunks :: Stream (Of ByteString) m r -> ByteStream m r
```

## Under the hood

The basic intuition of a `ByteStream` as a chunked stream of bytes punctuated
by monadic actions to retrieve more, can be made precise by looking at the
internal definition:
```haskell
import qualified Data.ByteString as B
data ByteStream m r =
    Empty r
    | Chunk {-# UNPACK #-} !B.ByteString (ByteStream m r)
    | Go (m (ByteStream m r))
```
which shows that a ByteStream is one of:
- An empty stream's terminal value
- An initial strict `ByteString` segment followed by the rest the stream
- A monadic effect that returns a ByteStream.

As both the `Chunk` and `Go` constructors contain continuations of the same
type, they may recursively nest or interleave.  Many *source* ByteStreams that
read data from a file or network socket consist of sequences of chunks
alternating with monadic actions that return the next chunk.

While, as mentioned above, the terminal value of a `ByteStream` is often merely `()`,
the `r` parameter of a `ByteStream m r` plays a non-trivial role in various transformations of a
basic `ByteStream`.

In order to understand some of the more subtle transformations, it is helpful to cover the `Functor`
and `Monad` declarations, as well as their non-obvious implications.

### Functor Instance:

The type `ByteStream m` is a functor in the missing parameter `r`.  The definition of `fmap`
preserves both the stream content and any intermediate
monadic actions, and merely transforms the terminal value as a delayed action once
the end of the stream is reached:
```haskell
    instance Monad m => Functor (ByteStream m) where
      fmap f x = case x of
        Empty a      -> Empty (f a)             -- Map the terminal value
        Chunk bs bss -> Chunk bs (fmap f bss)   -- Recurse on the tail
        Go mbss      -> Go (fmap (fmap f) mbss) -- Recurse on the tail
```
As the above definition suggests, each application of `fmap` can make the
thunks representing stream continuations somewhat more complex, and can
have a performance impact when multiple transformations are applied.

### Monad Instance:

The type `ByteStream m` is also a monad, whose (simplified) definition is as follows:
```haskell
    instance Monad m => Monad (ByteStream m) where
      return = Empty

      x0 >> y = loop x0 where
        loop x = case x of
          Empty _   -> y
          Chunk a b -> Chunk a (loop b)
          Go m      -> Go (fmap loop m)

      x >>= f =
        loop x where
          loop y = case y of
            Empty a      -> f a
            Chunk bs bss -> Chunk bs (loop bss)
            Go mbss      -> Go (fmap loop mbss)
```
Here a *pure* value `r` is simply an empty `ByteStream` with `r` as its terminal
value. More interestingly, sequencing two `ByteStream`s with the `(>>)` operator
concatenates their content after discarding the terminal value of the first and
replacing its `Empty` continuation with the second stream, which will ultimately
provide the terminal value of the combined stream.  Thus, while `(>>)` is stream
concatenation, it is worth noting that the two streams may have terminal values of
different types:
```haskell
    (>>) :: ByteStream m r -> ByteStream m s -> ByteStream m s
```
The bind operator `(>>=)` is rather similar, though in this case the content appended
to the first stream is a function of its terminal value. This effect is lazy: nothing
new happens until the first stream's end is reached, at which point the
function is applied and the stream continues with the function's return value.
```haskell
    (>>=) :: ByteStream m r -> (r -> ByteStream m s) -> ByteStream m s
```
As with `fmap`, it may be worth noting that multiple monadic transformations of a stream
can result in more complex thunks for its continuation.

The `ByteStream` type is also a monad transformer, that lifts monadic actions to empty
streams that encapsulate those actions:
```haskell
instance MonadIO m => MonadIO (ByteStream m) where
  liftIO io = Go (fmap Empty (liftIO io))

instance MonadTrans ByteStream where
  lift ma = Go $ fmap Empty ma
```

### Interaction with `Stream`

A `Stream` is a more general structure that is parameterised over an
arbitrary container (functor) `f` that encapsulates the tail of the stream:
```haskell
    data Stream f m r =
        Return r
        | Step !(f (Stream f m r))
        | Effect (m (Stream f m r))
```
Any non-trivial payload of the stream is determined by the functor `f`; the
stream itself only captures the sequencing of functorial and monadic layers.
Therefore, the functor `f` typically encapsulates an additional payload type.
The most common such functor in the `streaming` library is `Of`, which
is isomorphic to a 2-tuple, but is strict in its first element:
```haskell
    data Of a b = !a :> b
```
Accordingly, each `Step` of a `Stream (Of a) m r` holds data of type `a`, and a
"tail" of the type `b = Stream f m r`.  In this way, a `Stream (Of a) m r`
represents a monadic sequence of elements of type `a`, which is periodically
punctuated by monadic effects, and ultimately results in a terminal value of
type `r`.  This is structurally reminiscent of `ByteStream`, albeit far more
general; the data content is modeled via an arbitrary functor, rather than
merely chunks holding strict `ByteString`s.

The intent of this section is to examine the compound type `Stream (ByteStream m) m r`,
which appears in the signatures of several functions in this library.

This can be a tricky structure to unravel, but it should become more clear as
we go along.  This structure models a stream of streams, in which there are
intentional semantic boundaries between the end of one stream and the start of
another, that are more than just chunking to bounded buffer lengths that keep
memory utilisation modest even while processing large streams of data.

The first thing to note is that the same `m` is used in both the `ByteStream m`
and `Stream (...) m r`, because otherwise it is not possible to stitch the two
together in such a way that incremental consumption is possible for both the
overall stream of streams, and its individual component ByteStreams, each of
which might individually be too large to bring into memory all at once.

Next we expand the definition of `Stream` to take a closer look at what such
a stream might look like:
```haskell
    Stream (ByteStream m) m r ~
        Return r
        | Step !(ByteStream m (Stream (ByteStream m) m r))
        | Effect (m (Stream (ByteStream m) m r))
```
- The `Return r` case is simple, it marks the end of the overall stream.
- The `Effect` case is also simple, it represents a monadic action that returns
  the next part of the stream.
- The most interesting case is `Step`, which we'll discuess in detail below:

A `Step` encapsulates a monadic ByteStream, which can consist of an arbibtrary
(even unbounded) number of chunks interspersed with monadic effects to read
more of the ByteStream, until the inner `ByteStream` ends, exposing its
terminal value.  But here that value is a mere `()`, it is instead the next
sub-stream of the outer `Stream`!

If the entire `Stream` is finite and non-empty, its final `ByteStream` will
have `Return r` as its terminal value.

This structure makes it possible to incrementally process each component
monadic `ByteStream`in turn, one chunk at a time, and when one `ByteStream`
ends, optionally continue processing the next `ByteStream`, taking whatever
action is appropriate at sub-stream boundaries.

If you import both `Streaming.Internal` and `Streaming.ByteString.Internal` you
can write code that works directly with the `Step`, `Effect`, `Return`
constructors of `Stream` and the analogous `Chunk`, `Go` and `Empty`
constructors of `ByteStream`, but this is not typically necesary or desirable.

### Line splitting revisited (folding streams of streams)

A minor variation on `Q.lines` is `Q.lineSplit` which splits the stream into
sub-streams of up to `n` lines for some positive number `n`.  Let's take a
look at the small program below:
```haskell
import Streaming (Stream, streamFold)
import qualified Streaming.ByteString.Char8 as Q

main :: IO ()
main = do
    -- Fragment stdin into chunks of up to 10 lines
    let mbs :: Stream (ByteStream IO) IO ()
        mbs = Q.lineSplit 10 $ Q.stdin

    -- Fold back to a `ByteStream`
    let out :: ByteStream IO ()
        out = streamFold
            (const $ Q.empty) -- (r -> b)     stop at stream end
            Q.mwrap           -- (m b -> b)   wrap monadic step
            (>> Q.empty)      -- (f b -> b)   drop next sub-stream
            mbs               --              sequence of sub-streams

    Q.stdout out              -- ???
```
It first uses `Q.lineSplit` to transform the `ByteStream` obtained from
`Q.stdin` to a stream of sub-streams, where each stream except perhaps the last consists of 10 logical lines.
Unlike `Q.lines`, it does not remove the newline terminators, though the stream
is now sub-divided into groups of 10 lines, the content is otherwise unchanged.
If all the sub-streams are output in sequence, the original input is produced
verbatim.

Next, the resulting sub-streams are folded back into a single `ByteStream`
via three helper functions that transform all three variants of a `Stream` to a
common type, which becomes the result of the fold.  The functions provided will
typically iterate over the entire stream content performing a right-fold:

- The `(r -> b)` function handles the `Return r` base case,
- The `(m b -> b)` function handles the `Effect (m (Stream f m r))` case,
  where the inner stream has already been folded to a `b`.
- The `(f b -> b)` function handles the `Step (f (Stream f m r))` case,
  again with the inner stream already folded to a `b`.

But as with any right-fold, we can be lazy in the tail of the fold.  In our
example program:

- If the whole stream is empty we return an empty `ByteStream`.
- If the stream starts with a monadic step, we use `Q.mwrap` to produce a
  `ByteStream` which begins by performing the monadic action to return its
  first chunk (in some cases, perhaps yet another monadic action or empty).
- If however the stream is a `Step` (i.e. encapsulates the first sub-stream),
  we use `(>> Q.empty)` to clobber the terminal value of the first sub-stream
  (i.e. the second sub-stream) by appending an empty `ByteStream` which has
  `()` for its terminal value.

Consequently, regardless of the number of sub-streams, the right-fold
terminates after the first one, returning either just the first sub-stream, or
an empty `ByteStream` if the whole stream was empty.  So the above program is
just the unix shell `head` command with a hard-coded line count of `10`.

We are now ready to take a look at a second function that transforms our
stream of sub-streams back into a `ByteStream`, namely `Q.concat` (not
to be confused with `concat` from `Streaming`).  Its definition is essentially:
```haskell
import qualified Streaming.ByteString as Q
import Control.Monad (join)
import Streaming (Stream, streamFold)

concat :: Stream (ByteStream m) m r -> ByteStream m r
concat = streamFold return Q.mwrap join
```
- The `(r -> b)` component is now `return` which maps `r -> Empty r`, keeping
  the return value, rather than replacing it with `()`.
- The `(m b -> b)` component is again `Q.mwrap` (which is just `Go`), which
  injects a monadic step into the `ByteStream`.
- The `(f b -> b)` step however is now `join`!  This brings us back to the
  monad instance of `ByteStream`...

Our `f b` term has type `ByteStream m (Bytream m r)`, and since `ByteStream m`
is a monad we can indeed apply `join` to such a term.  Now `join ma` is just
`ma >>= id`, and when we have:
```haskell
    mbs :: ByteStream m r
    f :: r -> ByteStream m s
```
then `mbs >>= f` is just the concatenation of the chunk and effect sequence of
`mbs` with the chunk and effect sequence of `f r`, ultimately terminating with
terminal value of `f r`.  But here, `r` is the next sub-stream, and `f` is just
the identity function, so all told we're just re-constituting the original
stream.  So the name `concat` is apt.

But there's yet more to be said.  It is appropriate to ask how lazy is the
above concat?  Do we have to bring all the substreams into memory in order to
concatenate them before we can processs any of the content of the re-composed
`ByteStream`?  The answer is fortunately no.  This is a consequence of the
careful design of the libraries in the `Streaming` ecosystem.  The various
data structures are carefully strict only in the initial segment of each
stream and not its tail, and the combinators (including `streamFold`) generally
defer work that can be posponed (especially monadic steps) yielding as much
data as is available immediately, and thunks or monadic actions for the rest.

Thus, for example, `Q.concat` acting on a `Step (Chunk x (Effect action))` returns
a term of the form `Chunk x $ Go (fmap concat action))` in which we immediately
have access to the content of the first chunk, but the rest of the re-combined
stream is available as needed, possibly after reading more data from a source
file or network peer.

This gives us a second way to write `head(1)`, which brings in another useful
function from `Streaming`:
```haskell
import qualified Streaming.Prelude as S
import qualified Streaming.ByteString.Char8 as Q

main :: IO ()
main = do
    Q.stdout
    $ Q.concat
    $ S.take 1
    $ Q.lineSplit 10
    $ Q.stdin
```
The `Streaming` variant of `take` is said to be `functor-general`, which means
that it works with `Stream f m r` for any functor `f`, not just `Of`.

This new version of the `head` program is less direct than the specialised
`streamFold` version above, and perhaps marginally slower, but is an example of
a more general pattern.

### Materialising sub-streams

Once a `ByteStream` is subdivided into logical units (blocks of lines, words,
etc.) as a `Stream (ByteStream m) m r`, it is often
useful to bring these units into memory as either lazy or strict ByteStrings,
so that they can be processed via libraries that take those data types as
input.  This is done by transforming each substream into a `pure` value
encapsulated in `Of`.  The resulting stream is then `Stream (Of b) m r`, where
`b` is a strict or lazy ByteString.

The combinators that perform such transformations carry warnings about possibly
running out of memory, crashing, etc. should any of the substreams contain an
unexpectedly large quantity of data.  These warnings are apt, but are often
ignored, because the data is needed in memory, and is typically not
problematiic.  But this makes the code that ignores the warnings vulnerable
to memory exhaustion attacks.

What should be done instead is to set appropriate (configurable) resource
limits on the quantity of data to extract from each substream, and either
report an error if the limit is exceeded, or, when appropriate, process a
truncated portion of the (sub)stream.  Some ways to do that are explored below.
But first we'll look at the raw transformations that bring arbitrarily large
quantities of data into memory:
```haskell
import qualified Data.ByteString as Strict
import qualified Data.ByteString.Lazy as Lazy
import Streaming (Of)

toLazy :: Q.ByteStream m r -> m (Of Lazy.ByteString r)
toStrict :: Q.ByteStream m r -> m (Of Strict.ByteString r)

toLazy_ :: Q.ByteStream m r -> m Lazy.ByteString
-- Older releases used to require `r` to be `()`.
toStrict_ :: Q.ByteStream m r -> m Strict.ByteString
```
The are two pairs of functions, that return either strict or lazy
bytestrings, and either retain or discard the terminal value `r`.
All four potentially suffer from the same memory exhaustion issue.

Conversion to lazy bytestrings is somewhat cheaper because there's no need to
copy the entire stream into a single block of contiguous memory, but on the
other hand once it is memory, processing of strict bytestrings can be more
efficient.  The amount of memory consumed is similar, with lazy bytestrings
using a little bit more memory to encode a sequence of strict chunks.

Which you should use depends largely on whether subsequent processing of the
data wants strict or lazy bytestrings.  The lazy bytestrings generated by
these transformations don't involve any "lazy IO", the entire sequence of
chunks is precomputed.

The more important difference between these transformations is whether the
terminal value `r` is dropped, yielding a monadic action that returns just
a bytestring, or is retained, yielding a monadic action that reeturns both
the bytestring with the stream data and the terminal value.

In the context of subdivided streams, the functions
we care about are `toLazy` and `toStrict` which retain the `r` value, since
that's where the next substream is stored.  The `toLazy_` and `toStrict_`
functions are primarily useful for a single undifferentiated stream where the
`r` value often is `()`, or otherwise carries data that is no longer of
interest.

What can we do with `toLazy` and toStrict`?  If we give `ByteStream m` a
nickname of `f`, and `Of ByteString` a nickname of `g`, squinting at our
functions we see that they repackage `f r` as `m (g r)` (this is actually a
natural transformation between functors if category theory is your cup of tea).

We're interested in turning our stream of monadic substreams `Stream f m r`
into a steam of pure values `Stream (Of ByteString) m r` (with either the
strict or lazy `ByteStrings` type).  So we need a tool from the `Streaming`
ecosystem that can substitute `f` with `g`, given a (natural) transformation
with signature: `(forall r. f r -> m (g r))`.  The `forall` (one of the
ingredients of naturality) is essential here, since `r` will encode the tail of
the stream, a complicated recursive data structure.

Fortunately, the function we're looking for is `mapped` (but better in fact
`mappedPost`, see below) which is one of the four core general-purpose
functions singled out at the top of the `Streaming` module:
```haskell
maps    :: (forall x . f x -> g x)     -> Stream f m r -> Stream g m r
mapped  :: (forall x . f x -> m (g x)) -> Stream f m r -> Stream g m r
hoist   :: (forall x . m x -> n x)     -> Stream f m r -> Stream f n r
concats :: Stream (Stream f m) m r     -> Stream f m r
```
With `mapped` we can (still subject to memory exhausion issues) replace the
monadic substreams with their pure in-memory ByteString forms:
```haskell
import qualified Data.ByteString as Strict
import qualified Data.ByteString.Lazy as Lazy
import qualified Streaming.ByteString as Q
import Streaming (Stream, mapped)

loadStrict :: Stream (Q.ByteStream m) m r -> Stream (Of Strict.ByteString) m r
loadStrict = mapped Q.toStrict

loadLazy :: Stream (Q.ByteStream m) m r -> Stream (Of Lazy.ByteString) m r
loadLazy = mapped Q.toLazy
```
The `mapped` function is also available under the name `mapsM`, which avoids
a conflict with the `lens` library.

Internally, the `toStrict` or `toLazy` functions provide a monadic action that
loads a complete substream into memory, returning the data and the next
substream as a terminal value, and `mapped` is responsible for iteratively
applying the transformation to the chain of substreams, and turning all the
returned actions into `Effect` stream elements.  From this we can infer a
candidate implementation of `mapped`:
```haskell
mapped  :: (Monad m, Functor f)
        => (forall x . f x -> m (g x))
        -> Stream f m r
        -> Stream g m r
mapped f2mg = loop
  where
    loop str = case str of
        Return r -> Return r
        Effect m -> Effect $ fmap loop m
        Step s   -> Effect $ fmap Step (f2mg (fmap loop s))
```
Note that above we've not made use of `g` being a functor, and there is no
`Functor g` constraint in the type signature.  If it happens that `fmap` is
much cheaper for `g` than for `f`, an alternative implementation is available:
```haskell
mappedPost :: (Monad m, Functor g)
           => (forall x . f x -> m (g x))
           -> Stream f m r
           -> Stream g m r
mappedPost f2mg = loop
  where
    loop str = case str of
        Return r -> Return r
        Effect m -> Effect $ fmap loop m
        Step s   -> Effect $ fmap (Step . fmap loop) (f2mg s)
```

Returning to the problem at hand of using `toStrict` and `toLazy` to bring each
substream into memory, we can observe that applying `fmap` to a `ByteStream`
introduces intermediate thunks that ultimately apply the mapping to the
terminal value.  On the other hand, once we bring the stream and terminal value
into memory as a term of type `Of ByteString r`, applying a mapping to the `r`
value is cheap and immediate.  Therefore, it is in fact `mappedPost` (a.k.a.
mapsMPost) that should be used to materialise subdivided streams:
```haskell
import qualified Data.ByteString as Strict
import qualified Data.ByteString.Lazy as Lazy
import qualified Streaming.ByteString as Q
import Streaming (Stream, mappedPost)

loadStrict :: Stream (Q.ByteStream m) m r -> Stream (Of Strict.ByteString) m r
loadStrict = mappedPost Q.toStrict

loadLazy :: Stream (Q.ByteStream m) m r -> Stream (Of Lazy.ByteString) m r
loadLazy = mappedPost Q.toLazy
```

Now that we now how to blithely bring arbitrarily large stream fragments into
memory, it is time to consider how to do so safely within appropriate resource
limits.

We'll first look at truncation, which caps the quantity of data brought in from
each substream, discarding the remaining data, and then continues with the next
substream.  If the stream is divided into logical lines, this will truncate
overly long lines, without losing subsequent lines.  When using `lineSplit`
rather than `lines`, the truncation can be detected, because truncated lines
will not end with a newline ('\n') character (special care may need to be taken
with the last line if the input source does not guarantee newline termination
of the last line).

To perform the truncation we need a combinator that takes at most `n` bytes
from a stream, discards the remainder, but crucially retains the terminal
value, which in a `Stream (ByteStream m) m r` holds the next substream.  This makes the
`Q.take` function unsuitable for our purpose, because it discards the terminal
value.  The building blocks we need are `Q.splitAt` and `Q.drained`:
```haskell
import qualified Streaming.ByteString as Q
import Data.Int (Int64)

splitAt :: Int64 -> Q.ByteStream m r -> Q.ByteStream m (Q.ByteStream m r)
drained :: (Monad m, MonadTrans t, Monad (t m)) => t m (ByteStream m r) -> t m r

-- Thus, specialising to t = Q.ByteStream, we have:
drained :: Q.ByteStream m (Q.ByteStream m r) -> Q.ByteStream m r
```
- `splitAt` turns a `ByteStream` into an initial stream of at most the
  requested length with the rest of the stream as a terminal value.
- `drained` performs all the effects of the terminal value stream, discarding
  its content, and ultimately retains just the terminal value of that stream.

Their composition `\ n -> Q.drained . Q.splitAt n` gives us the means to
truncate the stream content without losing the terminal value.  With this
we can now materialise a subdivided stream without fear of memory exhaustion:
```haskell
{-# LANGUAGE BangPatterns #-}
import qualified Data.ByteString as Strict
import qualified Data.ByteString.Lazy as Lazy
import qualified Streaming.ByteString as Q
import Streaming (Stream, mappedPost)

safeStrictLines :: Int
                -> Q.ByteStream m r
                -> Stream (Of Strict.ByteString) m r
safeStrictLines n =
    mappedPost (Q.toStrict . Q.drained . Q.splitAt maxlen) $ Q.lineSplit 1
  where !maxlen = fromIntegral n
```

If instead we want to indicate an error condition when a substream is too long,
the most efficient approach is to just allow the result to be one byte longer
than the actually desired limit, and then handle lines that are exactly that
long appropriately.  This avoids the need for `Maybe`, `Either` or other
algebraic type wrappers for the content.  But if more accurate error messages
are more important than performance, we can report the length of the ignored
portion of each line along with its initial segment.  A non-zero excess length
then indicates truncation:
```haskell
import qualified Data.ByteString as Strict
import qualified Streaming.ByteString.Char8 as Q
import Streaming (Of(..), Stream, mappedPost)

strictLinesExcessLen :: Monad m 
                     => Int
                     -> Q.ByteStream m r
                     -> Stream (Of (Strict.ByteString, Int)) m r
strictLinesExcessLen maxlen = mappedPost cut . Q.lines
  where
    cut mbs = do
        (sbs :> t) <- Q.toStrict $ Q.splitAt (fromIntegral maxlen) mbs
        (len :> r) <- Q.length t
        return $ ((sbs, len) :> r)
```

## TODO: Explain `copy` and the black-magic of parallel streaming folds!
