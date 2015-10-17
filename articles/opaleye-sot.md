<nav><a href="../">Home</a> § <a href="../">About</a></nav>

# Opaleye's sugar on top

<p class="subtitle">SQL in the type system where it belongs</p>

People often talk of how solutions fall naturally into place when we program in
Haskell and embrace its type system. This article walks us through that process,
serving as a gentle introduction to some practical uses of advanced features of
the Haskell type system within the context of Opaleye and SQL. I invite you to
continue reading even if you are not particularly interested in Opaleye nor SQL,
as the approach explained here can be used in other contexts too.

**This is not introductory material to Haskell**. For things here to truly make
sense, you are expected to be comfortable with concepts like `Functor`, `Monad`
, `Arrow` and `Lens` at least. This is a very long article that intends to teach
you how to use the GHC type system to build resilient software and reason about
any problem you might want to tackle with it. Nevertheless, even if you don't
particularly understand the technical details, you should be able to follow the
reasoning process and appreciate the practical results.


## The problem

Reading, writing and maintaining hand-written SQL is hard and error prone. Over
the years and in many languages, people have tried to mitigate this problem to
different extents by creating tools that convert data types back and forth
between SQL and more practical runtime representations, and by creating
vocabularies for expressing queries or manipulation routines over the contents
of a database in ways that are clearer and more maintainable than just plain
hand-written SQL.

The [Haskell](https://www.haskell.org/) language ecosystem has had a number of
solutions to this problem too. Of these, powerful libraries such as Tom Ellis'
[`opaleye`](https://hackage.haskell.org/package/opaleye) and Kei Hibino's
[`relational-record`](https://hackage.haskell.org/package/relational-record) are
of particular interest to us today. What is special about these libraries is
that, making different trade-offs, they guarantee that they will always generate
well-formed SQL. The way they achieve this, as is usually the case in Haskell,
is by choosing to build their vocabulary on top of precise abstractions ensuring
that only things that make sense can be _composed_ together. In practical terms,
this means that if we try to, for example, insert an SQL table into a left
join, then our program will fail to compile because there is no sensible way in
which we can compose an SQL table and a left join by means of inserting the
former into the latter. This predictability in what can and cannot be composed
gives us new ways to reason formally about the queries that we write, which
takes us a long way from that old problem of being unable to read, write or
maintain hand-written SQL effectively and efficiently.

However, none of this means we won't ever generate _the wrong SQL_ when using
these tools, as there is still room for making mistakes such as mistyping the
name of a column or a table, which will lead to a well-formed but undesired SQL
being generated. It is important that we understand this difference between what
is well-formed SQL and what is the right SQL (as opposed to _the wrong SQL_). Is
`SELECT user.name FROM user` well-formed? Yes it is. But is it right or wrong?
Unfortunately we can't tell without more context, as the answer depends on
whether `user.name` was really the value we wanted to select, and on whether the
`user` table exists in the database and is the actual table we were interested
in. Otherwise, it might have been a mistake on our part when typing the column
name, or maybe we wanted to select `song.name` but accidentally referred to the
wrong table. In general, libraries such as `opaleye` and `relational-record` can
guarantee that they will generate well-formed SQL for us, but they
cannot—justifiably so—judge a well-formed SQL and decide whether it is right or
wrong, because these tools can't possibly know and understand the meaning and
purpose of said SQL. Nevertheless, we can improve the situation a bit by making
it harder for users of these libraries to write what they consider to be _the
wrong SQL_.

Today we will focus on Opaleye, and in particular I want to talk about
[opaleye-sot](https://github.com/k0001/opaleye-sot), a yet unreleased library I
have been working on for a while that brings some ideas worth sharing, as they
showcase some advanced uses of the type system that have concrete practical
benefits and can be useful for solving other problems too. As the name—short for
“Opaleye's sugar on top”—hopefully indicates, this is just a thin layer of new
features on top of Opaleye's standard offer, and it is intended to complement
Opaleye, not to replace it. `opaleye-sot` brings a different take on the public
API that users of Opaleye are encouraged to use; in particular, it makes
extensive use of advanced type system features offered by
[GHC, the Glasgow Haskell Compiler](http://www.haskell.org/ghc), so as to reduce
the number scenarios that could lead to Opaleye generating _the wrong SQL_, to
reduce the amount of boilerplate that one needs to write in order to make
effective use of Opaleye, and to increase the readability and maintainability of
queries and data types written using this API. `opaleye-sot` is an early project
with not many people depending on it yet, so hopefully it will be able to evolve
rapidly and function as a test bed for new features or approaches, some of which
may eventually make it to `opaleye` proper, without worrying too much about
backwards compatibility for now. For now, and the foreseeable future,
[PostgreSQL](http://www.postgresql.org/) is the only supported SQL backend.

For the rest of this article, I will assume familiarity with the basic usage of
Opaleye, and build on that. Familiarity with `HList`, `TypeFamilies`, `GADTs`,
`Arrows`, `DataKinds`, `PolyKinds` and `KindSignatures` will be useful too, but
not required. We will cover the topic of preventing _the wrong SQL_, but first
let us worry about the simpler topic of boilerplate. That is, about things we
would rather not write.


## Boilerplate and representations

Opaleye, quite wisely, doesn't force itself into any particular data type for
the Haskell representation of SQL rows, instead it allows us to pick any
representation we want as long as we can provide a
[`ProductProfunctor`](https://hackage.haskell.org/package/product-profunctors-0.6.3.1/docs/Data-Profunctor-Product.html#t:ProductProfunctor)
instance for it. This means, for example, that if we have a row with two fields,
we can choose to represent that as any product type with two elements: be it
`(a, b)`, a custom `Foo a b`, or similar. Those `a` and `b`, however, will
change depending on what we are trying to accomplish. For example, assuming a
SQL row with a `bool` column and a `text` column, when giving that SQL row a
concrete Haskell representation we may use the type `Foo Bool String`, but when
writing said SQL row to the database we will need to use `Foo (Column PGBool)
(Column PGText)`. For this reason, Opaleye recommends leaving `a` and `b`
polymorphic, and using type synonyms to fix `a` and `b` to particular types
depending on the scenario. There is only a limited number of scenarios like
these two where `a` and `b` will need to change. Personally, I think this is a
good approach to working with different yet somewhat similar representations,
but hopefully we all can agree that this can lead to a non-negligible amount of
boilerplate, in particular when working with rows having many columns, and we
need to prevent this. Our goal is to reduce the amount of boilerplate we need to
write regarding these types, hopefully without resorting to the powerful but
huge, fragile and uncomfortable hammer that `TemplateHaskell` can be.

`opaleye-sot` solves this problem by using a combination of
[`HList`](https://hackage.haskell.org/package/HList) (Oleg Kiselyov, Ralf
Laemmel, Keean Schupke et al.) and type families which we will now analyze in
detail. For those unfamiliar with with `HList`, we will start by mentioning its
traditional representation using
[GADTs](https://wiki.haskell.org/Generalised_algebraic_datatype):

```haskell
data HList :: [*] -> * where
  HNil  :: HList '[]
  HCons :: x -> HList xs -> HList (x ': xs)
```

That is, an `HList :: [*] -> *` is analogous to `[] :: *` where the `HNil` is
analogous to `[] :: []` and `HCons` is analogous to `(:) :: x -> [x] -> [x]`,
but where each of the elements inside the successive `HCons` constructors can be
of different types.

```
> -- This doesn't typecheck:
> (:) True ((:) 42 []) :: [Bool, Int]
> -- This does typecheck:
> HCons True (HCons 42 HNil) :: HList '[Bool, Int]
```

`HList` has some properties that are very interesting to us. To start with, not
only we can store an arbitrary number of elements of different types, but we can
also know statically—that is, at compile time—both the number and types of those
elements. Moreover, that information is kept in a type-level list promoted using
`DataKinds`, which means that we can manipulate that list and see it change its
length and contents at compile time, just like we could do with a traditional
term-level polymorphic list at runtime. And finally, `HList` can be given an
instance of `ProductProfunctor` that works for lists of any length, not only
making it a suitable datatype for Opaleye, but also preventing us from having to
define a new `ProductProfunctor` instance for each different Haskell
representation we would like to use for mapping SQL rows. For these reasons, we
will use `HList` as the preferred container for all our SQL row mapping needs,
which gives us a uniform way to work with different SQL rows.

Armed with `HList`, we can now to try map between it and an actual SQL row in
different Opaleye scenarios. We will use this contrived but comprehensive SQL as
example:

```sql
CREATE TABLE "public"."user"
  ( "id" serial4 NOT NULL
  , "name" text NOT NULL
  , "favoriteNumber" int4 NULL DEFAULT 42
  , "age" int4 NULL
  )
```

Notice what is special about those 4 columns regarding the values they can take:

<div class="table-wrapper" style="overflow-x:visible;">

Column            Can it contain `NULL`?  Do we want `DEFAULT` to be written to it?
----------------  ----------------------  -----------------------------------------
`id`              No                      Yes
`name`            No                      No
`favoriteNumber`  Yes                     Yes
`age`             Yes                     No

</div>


Notice how we worded our question: _Do we want `DEFAULT` to be written to it?_
We put it this way because in PostgreSQL, unfortunately, even if you do not ask
for a column to have a default value, `DEFAULT` can be written to it, and by
default `DEFAULT` means `NULL`. That is, `DEFAULT` is at times just a different
spelling for `NULL`. Later we will talk about why `NULL` is, was, and will ever
be a terrible idea, but for the time being we have to acknowledge the fact that
it exists and that we have to deal with it, which we will do by hiding it under
a type-safe interface that will prevent this accidental meaning for `DEFAULT`.
This will force us to say `NULL` if we want to say `NULL`, or to explicitly
acknowledge, per column, that `DEFAULT` is an acceptable value to write to it
even when it could mean `NULL`, even in columns marked `NOT NULL`. So, from now
on we will ignore the fact that `DEFAULT` can be written to any column if done
manually from SQL, because from within `opaleye-sot`'s DSL `DEFAULT` will only
be available if we asked for it. This situation, as unfortunate as it might be,
serves as a good example as what software many times is: Layers of abstraction
over layers of abstraction, where each layer fixes some problems found in the
layer below, allowing upper layers to be oblivious of it. Haskell, with its
precise type system and functional nature, can do a great job in such scenarios.

In `opaleye-sot` we will also explicitly distinguish between the combinations of
whether `NULL` and `DFAULT` are possible, even more so than how Opaleye does it
out of the box today—although
[there are plans to improve this](https://github.com/tomjaguarpaw/haskell-opaleye/issues/97).
We will now talk about this in more detail as we try to understand all the
different Opaleye scenarios in which we will be working.


## Scenario 1: HsR

<p class="subtitle">Haskell values read from the database</p>

Since we are only concerned about reading from the database here, we can safely
ignore the question of whether `DEFAULT` can be written to each particular
column. We must worry about whether the column can contain `NULL`, however. We
will represent that possibility as `Maybe`.

Ignoring `HList` and Opaleye for a moment, we might opt for a
representation like the following one. To prevent any confusion with forthcoming
examples, we will add the `_HsR` suffix to these types, meaning that this is the
representation of “Haskell values read from the database”, that is, “HsR” stands
for “Haskell, Read”.

```haskell
data User_HsR = User_HsR
  { _user_HsR_id             :: Int32
  , _user_HsR_name           :: Text
  , _user_HsR_favoriteNumber :: Maybe Int32
  , _user_HsR_age            :: Maybe Int32
  }
```

Or even something like this if we are not particularly interested in the
names of the columns:

```haskell
type User_HsR = (Int32, Text, Maybe Int32, Maybe Int32)
```

With `HList` we can express this very same thing:


```haskell
type User_HsR = HList '[Int32, Text, Maybe Int32, Maybe Int32]
```

If we wanted, we could manually define field accessors similar to the ones in
the example record definition for `User_HsR`, or even lenses. But alas, we don't
want to, because doing this will open a door to _the wrong SQL_. Let's carefully
think about why and analyze this bit together, because it is in small details
such as this one where Haskell can go the extra mile and radically tell itself
apart from other programming languages when it comes to code maintenance.

Can you tell the difference between these two types?

```haskell
type User_HsR1 = (Int32, Text, Maybe Int32, Maybe Int32)
```

```haskell
type User_HsR2 = (Int32, Text, Maybe Int32, Maybe Int32)
```

What about these two?

```haskell
data User_HsR1 = User_HsR1
  { _user_HsR1_id             :: Int32
  , _user_HsR1_name           :: Text
  , _user_HsR1_favoriteNumber :: Maybe Int32
  , _user_HsR1_age            :: Maybe Int32
  }
```

```haskell
data User_HsR2 = User_HsR2
  { _user_HsR2_id             :: Int32
  , _user_HsR2_name           :: Text
  , _user_HsR2_age            :: Maybe Int32
  , _user_HsR2_favoriteNumber :: Maybe Int32
  }
```

That's right, the order of the two last fields was changed, but of course we
didn't notice the first time because their types were the same. And neither did
the type checker. So we should ask ourselves what keeps the result of `SELECT
id, name, favoriteNumber, age FROM user`, in that order, from being somehow
converted into any of `User_HsR1` or `User_HsR2` above? And the answer is, of
course, nothing. One day we will wake up having 12345 years of age because that
was our favorite number.

To me, falling into traps like this one is unacceptable, even more so when
working in large projects where we can't possibly keep track of all these
details, or at least where we shouldn't need to because surely there are more
important things to do. It is not our job, we have type checkers. Our job is to
teach the type checker how to tell right from wrong and move on, which is not so
hard if we restrict ourselves to small domains and leverage a type system as
expressive as GHC's.

In order to solve this, first, we need to precisely identify what we are trying
to accomplish: We would like to use names to identify the particular columns in
our Haskell representations for SQL rows because telling apart the meaning of
_age_ from the meaning of _favorite number_ is much easier than telling apart
the meaning of a `Maybe Int32` from the meaning of another `Maybe Int32`.
Additionally, we want to be sure that every time we say _age_ in our Haskell
representation we do mean _age_ in the SQL row too, and that we make it very
hard for this property to be violated accidentally.

Having identified our goal, we can start working towards a solution. We might
categorize the problem here as one where there is more than one source of
information, that is, where both our Haskell representation and the SQL table
are trying to inform us the names of the fields. But which source do we trust
when they differ? Wrong, that is the wrong question. The right question to ask
ourselves is how to _prevent_ those two sources from ever differing. And, as it
is usually the case in situations like this one, the problem is that we have two
sources of information where we should have had just one.

If you have been building software for a while you already know that it is not
truly possible for data representations existing in different realms to map
perfectly with one another. That is, as an example in our case, we can't expect
that a Haskell data type that today maps perfectly to a SQL row will continue
to do so after any of the two is modified. Nevertheless, if we are careful
when we design said representations and their usage, we can ensure that our
software is resilient to future compatible changes, and that incompatible
changes don't go unnoticed.

What we will do is define the names of the columns just once _in the type
system_, and then, every time we need to refer to a particular column—be it for
selecting it, updating it, showing its name in some debugging tool or something
else—we will refer to its type. And only then, maybe, convert said type to a
term if at all needed. Additionally, we will never refer to columns by their
position in a row, only by its name.

The names of PostgreSQL tables can be expressed as Haskell values of type
`String`. In GHC, values of type `String` can have a type-level representation
as a type of kind
[`Symbol`](https://hackage.haskell.org/package/base-4.8.1.0/docs/GHC-TypeLits.html#t:Symbol)s,
which can trivially be converted back to `String`s at runtime—and
with some caveats, also `String`s can be converted to `Symbol`s, but we are not
particularly interested in this last conversion today.

If you are unfamiliar with kinds, can think of them as “the types of types”.
Every type that has a term level representation (e.g., `Int`, `Bool`, `Maybe
Double`, etc.) has a kind named `*` (yes, `*`, that is the name). Type-level
strings don't have a term-level representation, so its kind can be something
else: `Symbol` in this case. Conversion between a type-level string and a
term-level string is made explicitly via `symbolVal`. Here's a GHCi session
demonstrating the usage of `Symbol` and kinds.

```
> :set -XDataKinds
> :set -XKindSignatures
> import Data.Proxy
> import GHC.TypeLits
> :type ("hello" :: (String :: *))
("hello" :: (String :: *)) :: String
> :kind ("hello" :: Symbol)
("hello" :: Symbol) :: Symbol
> :type (symbolVal (Proxy :: Proxy ("hello" :: Symbol)) :: (String :: *))
(symbolVal (Proxy :: Proxy ("hello" :: Symbol)) :: (String :: *))
> symbolVal (Proxy :: Proxy ("hello" :: Symbol)) :: (String :: *)
"hello"
```

Terms, types and kinds: Make sure you understand the difference between them
[at least for a while](https://typesandkinds.wordpress.com/2015/08/19/planned-change-to-ghc-merging-types-and-kinds/).

With this new tool we can modify our `HList` solution from before so that not
only it mentions the type of the value contained in each particular column, but
also the name of the column itself, so that if we ever mistype the name of a
column the type checker will let us now. For this, we will use the `Tagged` data
type, yet another very simple but very powerful tool for teaching our type
checker how to tell right from wrong.

```haskell
newtype Tagged (t :: k) a = Tagged a
```

Remember, what is mentioned to the left of the `=` exists on the type level and
can be known statically at compile time, and what is mentioned to its right
exists on the term level and can be known at runtime. In other words, `t` needs
not have an accompanying term of type `t`, and there is nothing in the `Tagged`
constructor that restricts `t` to be some particular type nor of a particular
kind (as witnessed by the polymorphic kind `k`). A type like `t` which
apparently serves no purpose is called a
[panthom type](http://www.haskell.org/haskellwiki/Phantom_type), and we have
seen this kind of type before when we used `Proxy` in the previous example.

```haskell
data Proxy (t :: k) = Proxy
```

Look at that. `Proxy` seems to be even more useless than `Tagged`, it carries no
information whatsoever at the term-level. But the thing is, `Proxy` is a tool
for programming at the type-level, not at the term-level, so that is perfectly
fine. When we used `Proxy` before in our `symbolVal` example, we weren't
interested in any term-level value. We couldn't possibly have been, because as
we said before, type-level strings do not exist at the term-level; if they
existed their kind would have been `*`, but alas, it is `Symbol`. Nevertheless,
[`symbolVal`](https://hackage.haskell.org/package/base-4.8.1.0/docs/GHC-TypeLits.html#v:symbolVal),
a class method that exists at the term-level, needs to somehow receive as a
term-level argument a type-level string so that it can dispatch to the
appropriate `KnownSymbol` instance. `Proxy` is simply a bridge that connects the
type-level world with the term-level world.

Now that we have learned about `Proxy`, there is not much to say about `Tagged`
beyond the fact that it is isomorphic to `(Proxy t, a)`. That is, not only it
carries some type-level value `t`, but also a plain old term-level value `a`.
With `Tagged` we could write hypothetical functions such as this:

```
callFrenchPhone :: Tagged France PhoneNumber -> IO CallInformation
```

Here, the type-checker will ensure that only `PhoneNumbers` tagged with
`France` can be called using the `callFrenchPhone` function:

```
> -- This typechecks:
> callFrenchPhone (... :: Tagged France PhoneNumber)
> -- This doesn't typecheck:
> callFrenchPhone (... :: Tagged India PhoneNumber)
```

It is worth noting that the benefits we get from `Tagged French PhoneNumber` are
not so different from the benefits we get from introducing a `newtype` like:

```haskell
newtype FrenchPhoneNumber = FrenchPhoneNumber PhoneNumber
```

Nevertheless, `Tagged t a` is at times more convenient and uniform to use than
the `newtype` solution, and it can also be used for `t` s that may not be yet
known at compile time.

Putting together our new knowledge of `Tagged`, `Symbol` and we learned about
`HList`, this is what we end up with:

```haskell
type User_HsR = HList
  '[ Tagged "id" Int32
   , Tagged "name" Text
   , Tagged "favoriteNumber" (Maybe Int32)
   , Tagged "age" (Maybe Int32)
   ]
```

Perfect. Now the type of the Haskell representation for a value in a
column (e.g., `Text`) will always be accompanied by the name of its column
at the type-level (e.g., `"name"`). Just one last detail: The `HList` library we
are using provides a variant to `HList` named
[`Record`](https://hackage.haskell.org/package/HList-0.4.1.0/docs/Data-HList-Record.html)
which has an API specially designed for working with `HList`s whose elements are
`Tagged` values, just like here. From here on we will use `Record` instead
of `HList` due to the convenience of this API, but other than that, there is no
fundamental difference between an `HList` and a `Record`. So, our final
`User_HsR`, at least for now, is as follows:


```haskell
type User_HsR = Record
  '[ Tagged "id" Int32
   , Tagged "name" Text
   , Tagged "favoriteNumber" (Maybe Int32)
   , Tagged "age" (Maybe Int32)
   ]
```

## Scenario 2: Maybe HsR

<p class="subtitle">Possibly missing Haskell values read from the database</p>

Even while `User_HsR` is a perfectly acceptable Haskell representation for SQL
rows coming out of the database, sometimes those rows will be empty, such as in
the case of a missing right hand side in a `LEFT JOIN`. For those cases, `Maybe
User_HsR` will be a perfectly acceptable type within `opaleye-sot`. A variant of
`User_HsR` where each column is wrapped in `Maybe` is acceptable too, but often
less practical.

## Scenario 3: HsI, Haskell values to be inserted to the database

<p class="subtitle">Haskell values to be inserted to the database</p>

In this scenario the types are mostly like those in the HsR scenario, but we
also want to consider that some columns can take a `DEFAULT` value when being
written to. In our case, the columns `id` and `favoriteNumber`. We will simply
wrap with `WDef` the types of the elements representing those columns in our
`Record`. `WDef` is defined like this:

```haskell
data WDef a = WDef | WVal a
```

The idea is that if you want to write a specific value to a column you wrap it
in the `WVal` constructor, otherwise you use the `WDef` constructor and
`opaleye-sot` will replace that with `DEFAULT` in the generated SQL.

With this in place, we can proceed to define our counterpart to `User_HsR`
containing Haskell values to be inserted to the database in a particular SQL
row, which we will call `User_HsI`, short for “Haskell, insert”.

```haskell
type User_HsI = Record
  '[ Tagged "id" (WDef Int32)
   , Tagged "name" Text
   , Tagged "favoriteNumber" (WDef (Maybe Int32))
   , Tagged "age" (Maybe Int32)
   ]
```

You may have noticed that `WDef` is isomorphic to `Maybe`, and that we might as
well have used `Nothing` to signify that we want to write `DEFAULT` to a column,
and `Just` otherwise. So, why didn't we do that? Why did we introduce a whole
new type just for this? We did it because, again, this is a situation where by
carefully paying attention just once, we can forever profit from preventing more
of _the wrong SQL_.

Let's pay attention to the `favoriteNumber` column. Not only it can take a
`DEFAULT` value when being written to, but it can also contain a Haskell
representation for `NULL` as `Nothing`. If we had used `Maybe` instead of
`WDef`, then the type of the column—ignoring the `Tagged` wrapper—would have
been `Maybe (Maybe Int32)`, and there are important problems with that. The
first one is that it is not obvious what each of the `Maybe`s mean anymore: Was
it the outer or the inner `Maybe` the one signifying `DEFAULT`? And which one
signified `NULL`? The answer is not obvious. Moreover, if we had used `Maybe`
instead of `WDef`, the meaning of the `Maybe`s in the columns `id` and `age`
would have been completely ambiguous; we couldn't possibly know if a `Nothing`
value in those columns meant `DEFAULT` or `NULL`. But besides all this, and not
at all obvious and much more interesting to us, is the fact that both `Nothing`
and `Just Nothing` are valid values for the type `Maybe (Maybe Int32)`, which
means that mistaking one for the other would go completely unnoticed by the type
checker, quite possibly leading us to _the wrong SQL_. By making the distinction
between `WDef` and `Maybe` explicit we have taught the type-checker to tell
right from wrong once again.

## The billion dollar mistake

<div class="epigraph">
> I call it my billion-dollar mistake. It was the invention of the null reference
> in 1965. At that time, I was designing the first comprehensive type system for
> references in an object oriented language (ALGOL W). My goal was to ensure that
> all use of references should be absolutely safe, with checking performed
> automatically by the compiler. But I couldn't resist the temptation to put in a
> null reference, simply because it was so easy to implement. This has led to
> innumerable errors, vulnerabilities, and system crashes, which have probably
> caused a billion dollars of pain and damage in the last forty years.
>
> <footer>Tony Hoare, 2009.</footer>
</div>

SQL, as most other programming languages out there, is happy to take `NULL`
anywhere an expression of a specific type is expected, effectively
subverting any type-system guarantees—much like `undefined` in Haskell, which
one does not use, recommend, nor talk about. In SQL, `NULL` is particularly
noteworthy because we don't always find the world modeled in
[Sixth Normal Form](https://en.wikipedia.org/wiki/Sixth_normal_form). Instead,
tables are bound to have missing data, which is most often represented as
`NULL`.

Once we acknowledge the fact that `NULL` will stay in `SQL` for a while, and
that we have no idea whether `SELECT (x :: bool) AND (y :: bool) FROM t` even
results in a `bool` value, then we need to learn how to deal with it. But not
from scratch, because in Haskell we already know how to deal with `NULL` by
means of `Maybe` and we can leverage that knowledge.

Most times, if we want to make something useful with a value of type `Maybe a`,
at some point we will probably need to understand what is `a` and how to use it.
For example, if we want to `fmap` some function `a -> b` over `Maybe a`, then we
need to make a choice about what that `a` is, or at least which constraints it
satisfies. We could say we expect to have a `Maybe Int`, or a `Maybe Bool` or
even a `Num a => Maybe a`. The important thing to notice is that the `a` in
`Maybe a` must be a type with a term-level representation that exists in memory
Haskell at runtime. But when using Opaleye to generate SQL queries, we are not
dealing with Haskell values that exist in memory in the Haskell runtime at the
very same moment when when the SQL is being generated. In Opaleye, when you have
a `Column PGBool`, that is just a promise that if the generated SQL is ever run,
wherever it runs, the type of a specific column inside the PostgreSQL database
will be of type `bool` (with the caveat that it might contain a `NULL` value),
but there is no way to directly manipulate a term-level representation for said
`PGBool` in Haskell like we could manipulate an `Int`, a `Bool` or `()`. Much
like the `t` in `Tagged t a`, `PGBool` and similar types exist only at the
type-level, they have no term-level representation in Haskell at runtime. In
fact, since these types have no term-level representation, they could have had a
kind different from `*`, but for reasons beyond the scope of this article `*`
works better when you have to assume an open world, so you will find that
`PGBool` and similar have kind `*`.

What we need, and what `opaleye-sot` offers, is a `Maybe`-like type that can be
used in Opaleye's query language and can't possibly be mixed with a
non-`Maybe`-like type. Much like how the type-checker keeps us from multiplying
an `Int` by a `Maybe Int`, we want the type-checker to prevent us from
multiplying a `PGInt4` by a `PGInt4` that may be `NULL` unless we explicitly
deal with the fact that said possibility exists. In `opaleye-sot`, we call this
type `Koln`, short for “Column, nullable”, but with a “K” instead of a “C” to
avoid any potential confusion with Opaleye's `Column`. A type like `Koln PGBool`
is basically Opaleye's `Column (Nullable PGBool)`. As I said before,
[there are plans](https://github.com/tomjaguarpaw/haskell-opaleye/issues/97) to
implement this idea in Opaleye proper, but for the time being this is only
available in `opaleye-sot`.

Now that we have a precise Haskell representation for the description of a
possible `NULL` value in a PostgreSQL column, we can deploy the whole `Maybe`
toolset we Haskellers have grown to be so fond of:

```haskell
-- | Like 'maybe'. Case analysis for 'Koln'.
--
-- If 'Koln a' is 'NULL', then evaluate to the first argument,
-- otherwise it applies the given function to the underlying 'Kol a'.
matchKoln :: Kol b -> (Kol a -> Kol b) -> Koln a -> Kol b
```

```haskell
-- | Like 'fmap' for 'Maybe'.
--
-- Apply the given function to the underlying 'Kol a' only as long as the
-- given 'Koln a' is not 'NULL', otherwise, evaluates to 'NULL'.
mapKoln :: (Kol a -> Kol b) -> Koln a -> Koln b
```

```haskell
-- | Monadic bind like the one for 'Maybe'.
--
-- Apply the given function to the underlying 'Kol a' only as long as the
-- given 'Koln a' is not 'NULL', otherwise, evaluates to 'NULL'.
bindKoln :: Koln a -> (Kol a -> Koln b) -> Koln b
```

```haskell
-- | Like '(<|>) :: Maybe a -> Maybe a -> Maybe a'.
--
-- Evaluates to the first argument if it is not 'NULL', otherwise
-- evaluates to the second argument.
altKoln :: Koln a -> Koln a -> Koln a
```

With these tools we are can now reason in terms of `Maybe`, `Functor`s, `Monad`s
and `Alternative`s _in the SQL expresions themselves_, not just in the Opaleye
query language, and suddenly SQL doesn't look so bad anymore. The definitions of
those functions are not important, but you might check the source code if you
are curious. What is important is to understand the difference between `Kol a`
and `Koln a`. Whereas `Koln a` explicitly informs us that said `a` might be
`NULL`, and that we must explicitly deal with that possibility, `Kol a` tells us
that `a` can't possibly be `NULL`. Or, in another words, `Koln a` is analogous
to `Maybe a` while `Kol a` is analogous to `Identity a`, and we now that `a` and
`Identity a` are isomorphic. So, rewriting some well-known Haskell functions to
use `Identity` and following these analogies, we end up with this:

```haskell
matchKoln
  :: Kol b -> (Kol a -> Kol b) -> Koln a -> Kol b
-- 'Identity x' is analogous to 'Kol x', 'Maybe x' is analogous to 'Koln x'
  :: Identity b -> (Identity a -> Identity b) -> Maybe a -> Identity b
-- 'x' is isomorphic to 'Identity x'
  :: b -> (a -> b) -> Maybe a -> b
-- We reached the type of `maybe`
```

```haskell
mapKoln
  :: (Kol a -> Kol b) -> Koln a -> Koln b
-- 'Identity x' is analogous to 'Kol x', 'Maybe x' is analogous to 'Koln x'
  :: (Identity a -> Identity b) -> Maybe a -> Maybe b
-- 'x' is isomorphic to 'Identity x'
  :: (a -> b) -> Maybe a -> Maybe b
-- We reached the type of `fmap` for 'Maybe'
```

```haskell
bindKoln
  :: Koln a -> (Kol a -> Koln b) -> Koln b
-- 'Identity x' is analogous to 'Kol x', 'Maybe x' is analogous to 'Koln x'
  :: Maybe a -> (Identity a -> Maybe b) -> Maybe b
-- 'x' is isomorphic to 'Identity x'
  :: Maybe a -> (a -> Maybe b) -> Maybe b
-- We reached the type of '(>>=)' for 'Maybe'
```

```haskell
altKoln
  :: Koln a -> Koln a -> Koln a
-- 'Identity x' is analogous to 'Kol x', 'Maybe x' is analogous to 'Koln x'
  :: Maybe a -> Maybe a -> Maybe a
-- We reached the type of '(<|>)' for 'Maybe'
```

Opaleye, as of today, offers `Column a` and `Column (Nullable a)` as
counterparts to `Kol a` and `Koln a`. What is so special about `Kol a` is that
it makes it explicit that said `a` may never be `Nullable`. That's all there is
to it really. Of course there are ways to convert between `Column`, `Kol` and
`Koln` by means of `kol`, `unKol`, `koln` and `unKoln` so that `opaleye-sot`
code can readily compose with `opaleye` code using `Column`. Hopefully this
explicit difference will exist in `opaleye` proper soon.


## Scenario 4: PgR

<p class="subtitle">PostgreSQL values read in the database</p>

Now that we understand `Kol` and `Koln`, we can proceed to give a Haskell
representation to the PostgreSQL counterpart of `User_HsR`. We will call it
`User_PgR` where “PgR” stands for “PostgreSQL, read”.

```haskell
type User_PgR = Record
  '[ Tagged "id" (Kol PGInt4)
   , Tagged "name" (Kol PGText)
   , Tagged "favoriteNumber" (Koln PGInt4)
   , Tagged "age" (Koln PGInt4)
   ]
```

There is nothing fundamentally new here. We simply started from `User_HsR`,
replaced `Int32` and `Text` for their PostgreSQL counterparts `PGInt4` and
`PGText`, and then applied the same isomorphisms and analogies we applied before
when comparing `maybe` to `matchKoln` or `fmap` to `mapKoln`.

`User_PgR` is the type that we will use when writing queries using Opaleye's
query language, it is our Haskell representation of a SQL row that doesn't
exists in the Haskell runtime but in the PostgreSQL database. `User_PgR` is to
`User_HsR` what `Kol` is to `Identity` or `Koln` is to `Maybe`.

## Scenario 5: PgRN

<p class="subtitle">Possibly missing PostgreSQL values read in the database</p>

Just like we needed `Maybe User_HsR` to interpret the result of the missing
right hand side of a `LEFT JOIN` in Haskell, we need something similar for the
PostgreSQL side of things.

Like we said, a valid alternative to `Maybe User_HsR` is a variant of `User_HsR`
where each of the column types is wrapped in `Maybe`, signifying as expected
that each column can be `NULL`. So, it shouldn't be surprising by now that what
we need is an analogy to that representation but using `Koln` instead of `Maybe`.

Without further ado, we take `User_PgR`, wrap each column type in `Koln`, and
name the result `User_PgRN`, meaning “PostgreSQL, read, nullable”.

```haskell
type User_PgRN = Record
  '[ Tagged "id" (Koln PGInt4)
   , Tagged "name" (Koln PGText)
   , Tagged "favoriteNumber" (Koln PGInt4)
   , Tagged "age" (Koln PGInt4)
   ]
```

Notice that if we would have blindly “wrapped” the last two columns with `Koln`
we would have ended with `Koln (Koln PGInt4)` in each of them, which doesn't
work as `Koln` expects that its argument is one of `PGInt4`, `PGBool`, etc. So we
simply flattened the two redundant `Koln` layers into one, which is semantically
the same. In other words, only columns that were `Kol` in `User_PgR` need to be
changed to `Koln` in `User_PgRN`, the others remain the same.

## Scenario 6: PgW

<p class="subtitle">PostgreSQL values to be written to the database</p>

Finally, our last scenario. Just like `User_PgR` was the PostgreSQL counterpart
to `User_HsR`, `User_PgW` here will be the PostgreSQL counterpart to `User_HsI`,
except not only will it be used when inserting new values, but also when
updating them. Following our nomenclature, “PgW” stands for “PostgreSQL, write”.

Just like we did for `User_HsI`, where we started from `User_HsR` and added the
`WDef` wrappers where necessary, here we will start from `User_PgR` and do the
same, meaning that when inserting or updating a row in the database we can set
its value to `DEFAULT`.


```haskell
type User_PgW = Record
  '[ Tagged "id" (WDef (Kol PGInt4))
   , Tagged "name" (Kol PGText)
   , Tagged "favoriteNumber" (WDef (Koln PGInt4))
   , Tagged "age" (Koln PGInt4)
   ]
```

And with this we complete our understanding of what is needed to fully leverage
Opaleye using our `Record` representations. Next we will learn how to delegate
that knowledge to the type system so that we can forget about the details and
concentrate on more important things such as writing the actual SQL queries.


## Single representation for all scenarios

A while ago I talked about how Opaleye encouraged us to define polymorphic types
such as `data Foo a b = Foo a b` and then fixing `a` and `b` to be concrete
types depending on the scenario, as opposed to fixing `a` and `b` beforehand as
we did for `User_HsR`, `User_HsI`, etc. We will now move towards that
representation, except we will not be specifying the `a`s and `b`s manually each
time.

First, let's try to understand precisely the problem we are trying to solve.
Notice how if we put all of our scenarios side by side, we can easily recognize
a shape demanding to be made polymorphic:

```haskell
type User_HsR = HList
  '[ Tagged "id"             Int32
   , Tagged "name"           Text
   , Tagged "favoriteNumber" (Maybe Int32)
   , Tagged "age"            (Maybe Int32)
   ]
```

```haskell
type User_HsI = Record
  '[ Tagged "id"             (WDef Int32)
   , Tagged "name"           Text
   , Tagged "favoriteNumber" (WDef (Maybe Int32))
   , Tagged "age"            (Maybe Int32)
   ]
```

```haskell
type User_PgR = Record
  '[ Tagged "id"             (Kol PGInt4)
   , Tagged "name"           (Kol PGText)
   , Tagged "favoriteNumber" (Koln PGInt4)
   , Tagged "age"            (Koln PGInt4)
   ]
```

```haskell
type User_PgRN = Record
  '[ Tagged "id"             (Koln PGInt4)
   , Tagged "name"           (Koln PGText)
   , Tagged "favoriteNumber" (Koln PGInt4)
   , Tagged "age"            (Koln PGInt4)
   ]
```

```haskell
type User_PgW = Record
  '[ Tagged "id"             (WDef (Kol PGInt4))
   , Tagged "name"           (Kol PGText)
   , Tagged "favoriteNumber" (WDef (Koln PGInt4))
   , Tagged "age"            (Koln PGInt4)
   ]
```

For now, we will work with the least polymorphic shape that can serve all such
types:

```haskell
type User a b c d = Record
  '[ Tagged "id"             a
   , Tagged "name"           b
   , Tagged "favoriteNumber" c
   , Tagged "age"            d
   ]
```

Our goal is to somehow have `a`, `b`, `c` and `d` fixed to the expected types
depending on the scenario, but we definitely do not want to do that by hand. For
this we will rely on the `TypeFamilies` GHC language extension, which among other
things, allows us to write functions on types akin to functions on terms. That
is, if a term-level function `f :: a -> b` takes a term of type `a` to a term of
type `b`, then its counterpart type-level function `F :: a -> b` takes a type of
kind `a` to a type of kind `b`. Take as example the function `fst`, which gives
us the first element in a pair of terms:

```haskell
fst :: (a, b) -> a
fst (a, b) = a
```

We can promote the term-level function `fst :: (a, b) -> a` to a type-level
function `Fst :: (a, b) -> a` which gives us the first element in a pair of
types.

```haskell
type family Fst (x :: (a, b)) :: a where
  Fst '(a, b) = a
```

Other that the syntax here being a bit noisier, `Fst` has the expected kind and
behavior, as we can see in GHCi:

```
> :kind Fst
Fst :: (a, b) -> a
> :kind '(Int, "hello")
'(Int, "hello") :: (*, Symbol)
> :kind! Fst '(Int, "hello")
Fst '(Int, "hello") :: *
= Int
> :type 4 :: Fst '(Int, "hello")
4 :: Fst '(Int, "hello") :: Int
> 4 :: Fst '(Int, "hello")
4
```

There are two non obvious additional properties of type-level functions that are
worth mentioning and further tell them apart from type synonyms. First, just
like in normal term-level functions, there can be more than one pattern on the
left hand side of the function definition, and second, the expression on the
right hand side need not be present on the left hand side. Take for example the
following type-level function:

```haskell
type family Bar (x :: *) :: * where
  Bar Int  = Bool
  Bar Char = Double
```

`Bool` and `Double`, seemingly, come out of the blue. The kind of `Bar` is `* ->
*`, and it is defined for both `Int` and `Char`.

```haskell
> :kind Bar
Bar :: * -> *
> :kind! Bar Int
Bar Int :: *
= Bool
> :kind! Bar Char
Bar Char :: *
= Double
```

For other types, it is not:

```haskell
> :kind! Bar Bool
Bar Bool :: *
= Bar Bool
```

Unfortunately GHCi doesn't complain here that this type-level function is
undefined, presumably because we are not applying its type to any term-level
expression so there is nothing to type-check yet, but in any case we get a hint
that this type won't get us anywhere by noticing that `Bar Bool` is not being
reduced to another type, such as it did previously in GHCi when `Bar Int` was
reduced to `Bool` and `Bar Char` was reduced to `Double`. In any case, the
type-checker will fail to compile a program that tries to assign the type `Bar
Bool` to an expression. It can be useful to keep these things in mind when
working with type-level functions like these.

If we look at type-level functions more generally, we should notice that only
few aspects of them are new to us here, as from day one when programming in
Haskell we are exposed to type-level functions in the form of type constructors
like `Maybe :: * -> *`, `[] :: * -> *` or `Either :: * -> * -> *`. Everything we
learned about using type-level functions applies to type constructors too. Of
particular interest to us now are the type constructors `Tagged :: * -> * -> *`
and `Record :: [*] -> *` which we are using in our polymorphic definition of
`User`. Let's try to decompose our polymorphic `User` a bit more.

When we came up with our last definition of `User`, we said that it was the
least polymorphic version of the general shape of all our `User` variants,
somewhat implying that there existed more polymorphic versions of this shape,
and indeed they do. Gently, we will start to decompose this even further so as
to understand how to build a `Record` like this one out of the essential parts
that describe a row in the `user` SQL table. Which are those essential parts?
They are the ones mattered until now, and not one more. We can be certain of
this, because guided by the types, we carefully analyzed each of the Opaleye
scenarios we needed to cover, and only made as few changes as necessary in order
to satisfy our precise requirements. Namely, for each column in our SQL row we
need to know:

1. The name of the column.

2. Whether `DEFAULT` can be written to the column.

3. Whether the column can contain `NULL`.

4. The type of the Opaleye query language representation for value in the column
   when in the PostgreSQL runtime (e.g., `PGInt4`, `PGBool`).

5. The type of a value in the column when in the Haskell runtime (e.g., `Int`, `Bool`).


We can prove that these are our essentials by showing that each of them is
necessary to derive at least one of the `User` variants we need for our Opaleye
scenarios, and by noting that not one of them can be derived from the others. To
recapitulate: All of our scenarios need to know about the names of the columns,
that is, about the essential we numbered 1 above; apart from that, the `HsR`
scenario needs to know about essentials 3 and 5; the `HsI` scenario needs to
know about essentials 2, 3 and 5; the `PgR` and `PgRN` scenarios need to know
about essentials 3 and 4, and the `PgW` scenario needs to know about essentials
2, 3 and 4.

With this new certainty in mind, let's create a type that will hold this
essential information for us at the type level.

```haskell
data Col name wd rn pg hs
   = Col name wd rn pg hs
```

`Col` might look a bit funny, different to what you would be expecting to see if
this was a type intended to be used at the term-level. We will use `Col` as a
promoted datatype, meaning that everything to the left of the `=` will exist as
kinds, and everything to the right of the `=` will exist as types.
Unfortunately, we have to repeat on the kind side each of the type variables
present in the type because otherwise we can't fix these type variables to be
of a particular kind due to limitations of the `data` construct. That is, we can
write something like `data Foo = Foo String Int`, but we can't write `data Bar =
Bar * Symbol *`, for example. We will expect a well-formed `Col` to have the
kind `Col Symbol WD RN * *`, soon we will see what that means. Luckily for the
users of `opaleye-sot` this verbose kind is mostly used internally, and it is
not something they really have to worry about.

It is worth mentioning that some of these uncomfortable details that arise when
using kinds will improve as Richard Eisenberg's work towards a
[dependently typed Haskell](https://ghc.haskell.org/trac/ghc/wiki/DependentHaskell)
advances, but we are not there yet.

`Col Symbol WD RN * *` simply mentions the five essentials about a column which
we enumerated before. The first argument to `Col`, `Symbol`, mentions its name.
The second argument, `WD`, is isomorphic to `Bool` and describes whether
`DEFAULT` can be written to this column.

```haskell
data WD = W  -- ^ Write a specific value.
        | WD -- ^ Possibly write 'DEFAULT'. See 'WDef'.
```

We can justify rolling our own type instead of reusing `Bool` by reviewing some
of the reasons that led us to use `WDef` instead of `Maybe` a while ago: We are
trying to have the type-checker do as much work for us as possible, and we are
trying to make it obvious for users of `WD` to understand its meaning. These
days, the meaning of `Bool` is overrated, and in most cases it can be considered
a sign of poor programming. Think about it this way: We reach a road junction
and have to decide whether to take the road going north or the road going east.
We ask a group of people passing by which road to take and they answer `False`.
They tell us that they clearly remember `False` being the answer because a while
ago they heard a wise man say it, but unfortunately, they didn't hear the
question well, so they don't know if `False` was the answer to “should I go
north?” or to “should I go east?”. And now we are stuck there not knowing where
to go. In most situations, `Bool` doesn't carry any meaning with it, and we
can't possibly learn about that meaning without first knowing the question being
asked. In our `WD` type we fix this issue by giving a precise meaning to each of
its constructors: wherever you see the `WD` constructor you know it means that
`DEFAULT` can be written, and wherever you see the `W` constructor it means
otherwise. Furthermore, as the question is now embedded in the answer, we are
now safe from accidentally answering the wrong question.

The next argument to `Col` is `RN`, which describes whether a `NULL` value may
be present in our column or not. `RN` follows the same reasoning as `WD`. Notice
how we can't possibly mistake `RN` for `WD` even if we forget the order in which
they are presented as arguments to `Col`, which is something we couldn't have
prevented had we used `Bool` for both fields.

```haskell
data RN = R  -- ^ Read plain value.
        | RN -- ^ Possibly read @NULL@.
```

There is not much to say about the next two arguments. One of them tells the
type that the value in this column will take when treated as a Haskell value,
and the other tells the type it will have when treated as a placeholder, in
Opaleye queries, for a value existing inside the PostgreSQL database. Since
both of them share the same kind `*`, at fist it seems that we could
accidentally mix and the other and it would go unnoticed. And that is true.
However, `Col` will not stand on its own, instead it will be present in a larger
structure we haven't learned about yet, and this structure will further
constraint `Col` so that we get an early type-checker complaint if we make this
mistake. Nevertheless, even without that additional check, we know that the
type we will use as a representation for the PostgreSQL type of this column
will exists only at the type-level, while the type for the Haskell
representation of the value in this column will be used both at the type-level
and the term-level. That is, sooner or later, the type-checker will complain if
we use the wrong type at the wrong place within our expected scenarios.

Having understood `Col`, we can proceed to demonstrate how it can be used to
describe the columns in our `user` table:

```haskell
'Col "id" 'WD 'R PGInt4 Int32
```

```haskell
'Col "name" 'W 'R PGText Text
```

```haskell
'Col "favoriteNumber" 'WD 'RN PGInt4 Int32
```

```haskell
'Col "age" 'W 'RN PGInt4 Int32
```

Notice the `'` before some of the constructors. This is important when using
`DataKinds`, as it helps differentiate a term-level constructor from a
type-level constructor.

And as we know, the four of these `Col`s share the same kind, so they might as
well be put together in some polymorphic container such as a type-level list.

```haskell
'[ 'Col "id" 'WD 'R PGInt4 Int32
 , 'Col "name" 'W 'R PGText Text
 , 'Col "favoriteNumber" 'WD 'RN PGInt4 Int32
 , 'Col "age" 'W 'RN PGInt4 Int32
 ]
```

At last we have collected in a single type all the information that is essential
to describe the group of SQL columns we are interested in, and not one bit more.
We are on the right track.

Let's recall once again what our least polymorphic shape for `User` looked like
so that we can finally break it apart and move on:

```haskell
type User a b c d = Record
  '[ Tagged "id"             a
   , Tagged "name"           b
   , Tagged "favoriteNumber" c
   , Tagged "age"            d
   ]
```

The first thing to notice is that `Record :: [*] -> *` is a type-constructor
expecting a monomorphic type-level list as an argument. That is, we can postpone
worrying about `Record` until we figure out how to derive suitable monomorphic
lists from our list of `Col`s. The second thing to notice, similarly, is that
`Tagged :: Symbol -> * -> *` is another type-constructor we apply to types that
we know are present in our `Col` descriptions, which means that we can
generalize this a bit and say that to each `Col` we will apply a type-level
function `f :: Col Symbol WD RN * * -> *` leading to a type looking like a fully
applied `Tagged :: Symbol -> * -> *`, where the second argument to said `Tagged`
will vary depending on the Opaleye scenario.

The first thing we need to do is figure out how convert each of our `Col`s into
the type expected for this column by our `User_HsR` from before. That is, `Tagged
"id" Int32`, `Tagged "age" (Maybe Int32)`, etc. We already know how to do this
using type-level functions, so without further ado I present to you the answer:

```haskell
type family Col_HsR (col :: Col Symbol WD RN * *) :: * where
  Col_HsR ('Col n w 'R  p h) = Tagged n h
  Col_HsR ('Col n w 'RN p h) = Tagged n (Maybe h)
```

The type `Col_HsR ('Col "id" 'WD 'R PGInt4 Int32)` reduces to `Tagged "id"
Int32` as expected, and similarly for the other columns. Hopefully by now you
are not surprised that this works. The rest of the scenarios are similar and
don't introduce any new ideas:

```haskell
type family Col_HsI (col :: Col Symbol WD RN * *) :: * where
  Col_HsI ('Col n 'W  'R  p h) = Tagged n h
  Col_HsI ('Col n 'W  'RN p h) = Tagged n (Maybe h)
  Col_HsI ('Col n 'WD 'R  p h) = Tagged n WDef h
  Col_HsI ('Col n 'WD 'RN p h) = Tagged n (WDef (Maybe h))
```

```haskell
type family Col_PgR (col :: Col Symbol WD RN * *) :: * where
  Col_PgR ('Col n w 'R  p h) = Tagged n (Kol p)
  Col_PgR ('Col n w 'RN p h) = Tagged n (Koln p)
```

```haskell
type family Col_PgRN (col :: Col Symbol WD RN * *) :: * where
  Col_PgRN ('Col n w r p h) = Tagged n (Koln p)
```

```haskell
type family Col_PgW (col :: Col Symbol WD RN * *) :: * where
  Col_PgW ('Col n 'W  'R  p h) = Tagged n (Kol p)
  Col_PgW ('Col n 'W  'RN p h) = Tagged n (Koln p)
  Col_PgW ('Col n 'WD 'R  p h) = Tagged n WDef (Kol p)
  Col_PgW ('Col n 'WD 'RN p h) = Tagged n (WDef (Koln p))
```

From looking at the right hand side of these equations it is clear that the
`Tagged n` part could be abstracted away. Nevertheless, we wouldn't gain much by
doing so at this point, so we will leave it at that and move on.

The last piece of the puzzle is selecting one of these functions depending on
the Opaleye scenario we are working on, and mapping it over the type-level list
of `Col`s so as to finally get the desired monomorphic type-level list `Record`
expects. Mapping a type-level function over a type-level list is perfectly
doable, just like mapping term-level functions over term-level lists, so we
already know that is our answer. However, even if we had a high level function
`Map :: (a -> b) -> [a] -> [b]` for applying the given function to every element
of the list, that wouldn't be sufficient. The problem is that, currently, GHC
doesn't allow us to pass a partially applied type-level function around; it
requires that all our type-level functions be fully saturated, which leaves out
the possibility of ever writing `Map Col_PgR`, for example. Have we reached a
dead end? Fortunately, no. There exists a technique called defunctionalization
that, apparently by taking the fun out of function application, allows you to
work with and pass around partially applied type-level functions. We are not
particularly interested in the details of this technique today, it should
suffice to say that you may already be familiar with the principles of it if you
have ever used the `ApplyAB` class in the `HList` library. In our case, we will
use the defunctionalization support offered by Richard Eisenberg's
[`singletons`](https://hackage.haskell.org/package/singletons) library, which
among many other things, will allow us to map the type-level functions on `Col`
we just wrote over a list of `Col`s. So, doing a bit of wishful thinking
regarding the implementation, we will assume we got to write functions like
these:

```haskell
-- Takes a list of 'Col's and applies 'Col_HsR' to each of them.
type Cols_HsR :: [Col Symbol WD RN * *] -> [*]
```

```haskell
-- Takes a list of 'Col's and applies 'Col_HsI' to each of them.
type Cols_HsI :: [Col Symbol WD RN * *] -> [*]
```

```haskell
-- Takes a list of 'Col's and applies 'Col_PgR' to each of them.
type Cols_PgR :: [Col Symbol WD RN * *] -> [*]
```

```haskell
-- Takes a list of 'Col's and applies 'Col_PgRN' to each of them.
type Cols_PgRN :: [Col Symbol WD RN * *] -> [*]
```

```haskell
-- Takes a list of 'Col's and applies 'Col_PgW' to each of them.
type Cols_PgW :: [Col Symbol WD RN * *] -> [*]
```

If you are interested in the details of how these are implemented you can take a
look at the code in `opaleye-sot`. It can be useful to understand this technique
if you are going to be spending more time in the type system.

With these in place, we can start worrying about where to keep all this
information about our rows.

## Tables

So far we have talked about SQL rows but not particularly in the context of SQL
tables. This was a deliberate choice meant to reinforce the idea that in most
cases you can think of an SQL row as just a list of columns resulting from an
SQL query, perhaps across many tables. Still, making the connection between SQL
rows and SQL tables is necessary in many cases, so we will work on that.

We want to treat each table as a completely different entity and have the
type-checker help us enforce it so that not even tables that have the same
shape can be accidentally confused. But, as different as these tables might be,
we want to be able to work with all of them in a uniform and lightweight
manner. Accomplishing all of this will be our goal.

In PostgreSQL each table can be uniquely identified by its name and the name of
the schema it belongs to. These are just strings, and we already know that
strings can exists at the type-level in the form of `Symbol`. For our `user`
table we can say that `"user" :: Symbol` is its name, and that `"public" ::
Symbol` is the name of its schema. With this in mind we can restrict the
type-level list of `Col`s from before so that it is tied to this particular
`user` table and not to any other table that might share its shape. As before,
we accomplish this by simply tagging our list with additional information:

```haskell
Tagged '("public", "user")
  '[ 'Col "id" 'WD 'R PGInt4 Int32
   , 'Col "name" 'W 'R PGText Text
   , 'Col "favoriteNumber" 'WD 'RN PGInt4 Int32
   , 'Col "age" 'W 'RN PGInt4 Int32
   ]
```

But still, where do we define this? Where do we actually write this type in our
code? Of course we could assign a type synonym to this type expression and move
on, but that would mean introducing a new type synonym for every table that we
wanted to support, and we wouldn't be able to enforce some constraints we would
like to enforce on these types at their definition site. We will instead rely
on _open type families_, or more precisely, _associated open type families_.
These will allow us to achieve all of our goals.

The type-level functions we have been using until know are officially known as
_closed type families_, and what is so special about them is that, akin to
normal term-level functions, all of their patterns must be defined on the very
same place within the body of its `where` clause. On the other hand, _open type
families_ are defined more like typeclasses and their instances: we declare
the type family just once in a single place and then give instances for that
type family possibly across different modules. This _open world assumption_
makes open type families fundamentally necessary if we don't know at compile
time all of the types on which our type-level function should work, like in our
case. The differences in syntax between open and closed type families are
minimal. For example, let's convert our `Bar` closed type family from before
into an open type family.

From a closed type family:

```haskell
type family Bar (x :: *) :: * where
  Bar Int  = Bool
  Bar Char = Double
```

To a single open type family declaration:

```haskell
type family Bar (x :: *) :: *
```
And to zero or more open type family instances, possibly defined across different
modules:

```haskell
type instance Bar Int  = Bool
type instance Bar Char = Double
```

Having this new knowledge, we can now create a new type family named `Cols` that
when applied to the names of a schema and a table, it gives us our tagged list
of `Col`s:

```haskell
type family Cols (schema :: Symbol) (table :: Symbol)
  :: [Col Symbol WD RN * *]
```

Assuming `Cols` is being provided by some hypotetical library, users of this type
family simply have to provide new instances for it:

```haskell
type instance Cols "public" "user" =
  '[ 'Col "id" 'WD 'R PGInt4 Int32
   , 'Col "name" 'W 'R PGText Text
   , 'Col "favoriteNumber" 'WD 'RN PGInt4 Int32
   , 'Col "age" 'W 'RN PGInt4 Int32
   ]
```

And if there also existed a type-level function like the `TaggedCols` below,
`TaggedCols "public" "user"` would equal our tagged list of `Col`s from before:

```haskell
type TaggedCols (schema :: Symbol) (table :: Symbol)
  = Tagged '(schema, table) (Cols schema table)
```

However, this is still unsatisfactory. The problem is that maybe in our codebase
we happen to need to interact with two or more different databases at the same
time, where two or more of them might have tables named `"user"` in a schema
named `"public"` with shapes different from the one we just laid out. However,
our current approach with `Cols` requires that there exists only one combination
of schema name and table name. In order to fix this, `Cols` needs to take a
third type parameter that uniquely identifies the database where the table
exists. We are not interested in what this type is, we are only interested in
the meaning we decide to give to its role: This type parameter will be used as a
unique identifier for a database, and as long as it is equal to another type
used in a similar way we will say that we are talking about a same database,
otherwise we will say that we are talking about different ones. It is up to the
user to decide which type to provide, we the library authors are not interested
in the type per se but in its role. So, our definition of `Cols` will look like
this:

```haskell
type family Cols (database :: k) (schema :: Symbol) (table :: Symbol)
  :: [Col Symbol WD RN * *]
```

And then an instance for one database could look like this:

```haskell
-- | This is only ever used at the type level to uniquely identify a specific
-- database. All the tables belonging to this database should use this same
-- identifier.
data Db1

type instance Cols Db1 "public" "user" =
  '[ 'Col "id" 'WD 'R PGInt4 Int32
   , 'Col "name" 'W 'R PGText Text
   , 'Col "favoriteNumber" 'WD 'RN PGInt4 Int32
   , 'Col "age" 'W 'RN PGInt4 Int32
   ]
```

And for another database something like this:

```haskell
data Db2

type instance Cols Db2 "public" "user" = -- ... not important
```

One last detail: Since we now have incoming information about the database where
the table exists, we should keep it instead of just throwing it away as we do in
`Cols`. The reason for this is that this new information might come handy later,
for example, to prevent any attempts to join tables across different databases.
Indeed, `opaleye-sot` does just that. So let's update `TaggedCols` to take this
into account:

```haskell
type TaggedCols (database :: k) (schema :: Symbol) (table :: Symbol)
  = Tagged '(database, schema, table) (Cols database schema table)
```

This is a fine representation, but it is not entirely satisfying to us because
we can't add any constraints to the types that show up in it, so we will try
something else. There are two reasons why we want to add constraints: First,
because we want these constraint to give us early compile time errors if we get
something wrong when enumerating our columns (for example, when mistaking the
purpose of the last two `* *` arguments to `Col`), and second, because working
with `Record` and `HList` will require that our list of `Col`s satisfies various
boring constraints, and we would rather satisfy those constraints right away
when defining our list instead of doing it later on each use site. Our solution,
which we briefly talked about before, is _associated_ open type families.

The difference between associated open type families and normal open type
families is that associated type families are declared inside typeclasses, and
instances for said type family are given within the instances for the typeclass
itself. This enables us, among other things, to mention the associated type
family in the class or instance context where we can further constraint them.
Let's take for example `Bar`, our open type family from before:

```haskell
type family Bar (x :: *) :: *
```

If we wanted to convert `Bar` to an associated open type family within some
class `Qux`, we would do it like this:

```haskell
class Qux (x :: *) where
  type Bar x :: *
```

And instances for this typeclass and associated type family would be specified
in the following way, possibly in different modules:

```haskell
instance Qux Int where
  type Bar Int = Bool
```

```haskell
instance Qux Char where
  type Bar Char = Double
```

This new `Bar` still has the kind `* -> *`, and it is defined for every `x` that
is an instance of `Qux`. And as we wanted, now, we can further constraint `Bar
x` so that it satisfies some particular constraints. For example, we could
enforce that `Bar x` be an instance of `Num`:

```haskell
class Num (Bar x) => Qux (x :: *) where
  type Bar x :: *
```

With this new definition of `Qux`, the previously given instance for `Qux Int`
would fail to compile because `Bar Int` is `Bool`, which is not an instance of
`Num`, effectively giving us early compile time errors at the definition site
for things that would probably have been constrained on their use site anyway,
leading to compile errors there later.

One last minor remark: Just like when using a method `f` of a class `X` on a
term `a :: t` requires that there is an instance of `X` in scope that defines
that method `f` in a way that it can accept `a :: t` as its argument, using a
associated type family such as `Bar x` in our example will require that there
exists an instance of `Qux x` in scope, meaning that this constraint will be
propagated upstream until it is satisfied.

Let's now convert our `Cols` to an associated open type family within a type
class that we will call `Tisch`, which is the German word for “table”—a good
name because it doesn't clash with Opaleye's `Table`, a type we will use later
on.


```haskell
class ITisch d s t
   => Tisch (database :: k) (schemaName :: Symbol) (tableName :: Symbol)
 where
   type Cols database schemaName tableName
     :: [Col Symbol WD RN * *]
```

We will not pay attention at all to the internals of the `ITisch` superclass; as
we suggested before, it just constraints `Cols` as we did in our `Qux` and `Bar`
example, and it ensures `Cols` satisfies many constraints that will be required
by `HList` and `Record`. The implementation of `ITisch` however, doesn't teach
us anything fundamentally new, so let's ignore it. Also, the `TaggedCols` from
before would continue to work just fine because `Cols` still takes the same
three type arguments.

`Tisch` as defined above is fine, but hopefully you can see that passing those
three arguments everywhere will get cumbersome, even more so if we keep in mind
that at times we will need to pass these values as phantom types in a `Proxy` or
similar type. For this seemingly cosmetic reason we will change our `Tisch` so
that it takes a single type parameter, and from it derive `database`,
`schemaName` and `tableName` just like we did with `Cols`. Beyond the looks,
however, this change will impact substantially on the removal of much noise that
would otherwise exist in our query language—which we haven't talked about yet.

```haskell
class ITisch t => Tisch (t :: k) where
  type Database t :: *
  type SchemaName t :: Symbol
  type SchemaName = "public"
  type TableName t :: Symbol
  type Cols t :: [Col Symbol WD RN * *]
```

Look at that, we were even able to give a default value to `SchemaName t` this
time. This value can of course be overridden, but `"public"`, just like in
PostgreSQL, serves as a good default for the schema name. Now users of `Tisch`
can define their tables like this—assuming the table exists in the database we
called `Db1` before:

```haskell
-- | Unique type-level identifier for the '"public"."user"' PosgreSQL table.
data User

instance Tisch User where
  type Database User = Db1
  type TableName User = "user"
  type Cols User =
    '[ 'Col "id" 'WD 'R PGInt4 Int32
     , 'Col "name" 'W 'R PGText Text
     , 'Col "favoriteNumber" 'WD 'RN PGInt4 Int32
     , 'Col "age" 'W 'RN PGInt4 Int32
     ]
```

The `Tisch` described here is exactly the same `Tisch` that you will find today
in `opaleye-sot`.

All that is left is to forget about our previous `TaggedCols`, as now we have a
better way of representing a table. The thing to keep in mind is that as long as
we have a type `t` for which there exists an instance of `Tisch`, we can always
derive from it any of `Database`, `SchemaName`, `TableName` and `Cols`, which
effectively allows us to express things like `Tagged t (Cols t)` if ever needed.


## The elephant in the room

An minor detail to notice, or perhaps to ignore, is that in our implementation
of `Tisch` we have fixed the kind of `Database t` to `*` instead of keeping it
polykinded as before. This will ease the implementation on some features related
to preventing the accidental comparison of columns across different databases.
But, how do we know that?

At this point I think we should discuss _the elephant in the room_: “How do we
come up with these judgments? How do we _foresee_ which of `*` or `k` will be
better? How will we ever know these things for ourselves? Who are you to tell me
what to write?”. And essentially, it boils down to this: Working with the type
system is a very interactive and enriching activity, we learn from it, and we
constantly go back to improve things, some of which we will understand and
remember the next time we find ourselves solving a similar problem. The type
checker is an excellent teacher and companion; we try one approach and fail, we
try another approach and fail again but this time understanding why, then we
succeed but later realize that maybe if we had done something a bit differently
we could have covered a larger space and solved more problems, so we go back and
fail again until eventually we don't, and then we have learned something.
Throughout this process we gain some understanding about which things are worth
doing and which aren't, and from there we make informed choices.

Up until now we have walked through this reasoning process together, but maybe
this article is becoming to long and we should speed up things a bit by saving
us all this back and forth with the type checker—an experience best lived
personally. So when I say that the kind of `Database t` will be `*` because that
will simplify some things later, or when I tell you that the `ITisch` constraint
on `Tisch` is there to keep us from harm and reduce some noise related to the
usage of `HList` and `Record`, what I am really saying is that for reasons that
should become apparent later, these choices serve our ultimate purpose well. The
type-checker told me so, as it would have told you had you two been talking.

Regarding these interactive sessions with the type-checker, I am sure most have
their own story. For me it was years ago when I spent many hours failing to
write a `Functor` instance for something like `(a, b)` where `fmap` applied the
given function to both `a` and `b`: It took me a lot of time to understand that
it wasn't possible, but eventually I did, and never again I had trouble
reasoning about parametric polymorphism in typeclasses and related concepts.
_Peer-programming_ with the type system, if you will, is a beautiful and
enriching exercise no matter how silly the things we are trying to understand
might seem at first. It is worth trying.


## Every type we will ever need

From a type `t` that is an instance of `Tisch` we can derive every type we will
ever need for working with the Opaleye scenarios we want to support. This
shouldn't be a surprise to us, after all we have already achieved this once
before with our definitions of `Cols_HsR`, `Cols_HsI`, etc. We already know that
we can say `Cols_HsR t` and move on. Nevertheless, similarly to how we did with
`TaggedCols` before, we would like to tag the resulting `Record` with some
identifier for our table so as to keep us from accidentally attempting to, say,
interpret the result of one table as the Haskell representation intended for a
different one. This is no secret to us anymore: We know why telling apart
apples from oranges is useful; it teaches the type system to tell right from
wrong, and we value that. What we want is a type-level function like the
following one, which takes as argument a `t` that is an instance of `Tisch`:


```haskell
type HsR t = Tagged t (Cols_HsR (Cols t))
```

And similarly for the `HsI`, `PgR`, `PgRN` and `PgW` scenarios. Of course, if we
start putting `Tagged` values in datatypes like this one, expected to play well
with Opaleye, we have to ensure `ProductProfunctor` instances can exist for
them. But they can, and they are exported by `opaleye-sot`, so let's not worry
about it.

Fundamentally, there is not much more to this, but from a practical point of
view there is one last thing to do: Just like we created `WDef`, `WD` and `RN`
to help us reason about meaning, we will create a new type isomorphic to `Proxy`
that we will use to exclusively tag types that are one of our `Tisch` instances.

```haskell
-- “T” stands for table, of course.
data Tisch t => T (t :: k) = T
```

This will prevent us from accidentally confusing, say, a `Tagged` list of
columns for a table from a `Tagged` single column, because we know that we will
only ever use `T` to tag things that concern the entire table. As a bouns, this
will the improve the clarity and readability of queries and compiler errors when
using `T` within Opaleye's query language. Hopefully you will agree that
writing `T::T User`—which we will frequently need to write in our query
language—can lead to code that is more readable than if we used the usual `Proxy
:: Proxy User`, and that compilation errors talking about `T` will guide us in
the right direction. So, we will change the `HsR` function we just introduced,
as well as its siblings, to use `T`. Later on, when paying attention to the
query language, we will see more sophisticated uses for `T`.

```haskell
type HsR t = Tagged (T t) (Cols_HsR (Cols t))
```

Actually, this is a very good idea; let's introduce a similar type for tagging
columns belonging to a particular table, reproducing the gains from `T` in yet
another scenario:

```haskell
-- “TC” stands for table column, of course. 'c' carries the name of the column.
data Tisch t => TC (t :: k) (c :: Symbol) = TC
```

We will see sophisticated use cases for `TC` later when we look at the query
language. For now we will just change our previous implementation of `Col_HsR`
and its siblings to tag each column using `TC`, instead of just the name of the
column:

```haskell
type family Col_HsR (t :: k) (col :: Col Symbol WD RN * *) :: * where
  Col_HsR t ('Col n w 'R  p h) = Tagged (TC t n) h
  Col_HsR t ('Col n w 'RN p h) = Tagged (TC t n) (Maybe h)
```

And finally, after a long journey of understanding, `HsR` and its siblings have
the expected behavior. This is, for example, the type `PgW User` reduces to:


```haskell
Tagged
  (T User)
  (Record
    '[ Tagged (TC User "id")             (WDef (Kol PGInt4))
     , Tagged (TC User "name")           (Kol PGText)
     , Tagged (TC User "favoriteNumber") (WDef (Koln PGInt4))
     , Tagged (TC User "age")            (Koln PGInt4)
     ])
```

To appreciate the reach of what we have accomplished let's contemplate the fact
that `Table (PgW User) (PgR User)` is now a perfectly acceptable type for the
argument to Opaleye's
[`queryTable`](https://hackage.haskell.org/package/opaleye-0.4.2.0/docs/Opaleye-Table.html#v:queryTable)
or any of the
[manipulation functions](https://hackage.haskell.org/package/opaleye-0.4.2.0/docs/Opaleye-Manipulation.html)
in Opaleye, that `Query (PgR User)` is an acceptable type for the return type of
`queryTable`, that `PgRN User` can be one of the types resulting from a
[`leftJoin`](https://hackage.haskell.org/package/opaleye-0.4.2.0/docs/Opaleye-Join.html#v:leftJoin)
that a function `PgR User -> PgW User` can be passed as a callback
argument to
[`runUpdate`](https://hackage.haskell.org/package/opaleye-0.4.2.0/docs/Opaleye-Manipulation.html#v:runUpdate),
and that a list of `HsR User` can be obtained from
[`runQuery`](https://hackage.haskell.org/package/opaleye-0.4.2.0/docs/Opaleye-RunQuery.html#v:runQuery).
All of this without specifying the details for the `user` table more than once,
and without relying in `TemplateHaskell` for it. We have achieved a lot, we
distanced ourselves from _the wrong SQL_ a bit more, and we understand
everything there is to understand about how we did it.

We have completed our understanding of how to represent SQL tables in the type
system. If you would like to make a pause before continuing, this would be a
good time. Next we will learn how to use the tables we have defined here.

## Tisch befriends Opaleye

The simplest useful thing we can do with a SQL table is to query it. Opaleye
exports an function named
[`queryTable`](https://hackage.haskell.org/package/opaleye-0.4.2.0/docs/Opaleye-Table.html#v:queryTable)
that does just that:

```haskell
queryTable :: Default ColumnMaker r r => Table w r -> QueryArr () r
```

The `Default ColumnMaker` bit is not important right now, you can ignore it.
First, as we usually do when reading function types, we will pay attention to
the return type. `QueryArr () r` is the type of an Opaleye query that, through
an `Arrow` interface, takes an argument of type `()` and returns a value of type
`r`. Notice that just like we can't tell from the type `Maybe Int` whether it
contains `Just 5` or `Nothing`, we can't tell from the type what this `QueryArr
() r` does. However, if we were to look inside it we would find that it
generates a query analogous to `SELECT a.x, a.y FROM s.t a`, and for now you
will have to just trust me on this. The result of this query, that is, the `r` in
`QueryArr () r`, will be a Haskell representation of the values in the selected
PostgreSQL columns. The `()`, as you can probably imagine, is superfluous, yet
it needs to be passed around to satisfy the needs of the `Arrow` interface.
Thus, we conclude that any knowledge about `s`, `t`, `x` and `y` must come from
the passed in `Table w r`.

We know that a `Table w r` must specify at least its name and the name of its
schema, none of which show up in the type, as well as the Haskell representation
for PostgreSQL values in table columns showing up as `r` in the type. We
already know this much. Additionally, we can learn from Opaleye's documentation
that `w` indicates the Haskell representation of PostgreSQL values when these
are being written to the table columns. From this we can hypothesize that `Table
(PgW User) (PgR User)`, for example, should be an acceptable type for the
description of our `user` table. And indeed it is. But how do we construct a
`Table (PgW User) (PgR User)`? The `Opaleye` documentation points us to the
`TableWithSchema` constructor for `Table` and explains how to build one by
hand. But of course, we don't want to do this by hand because we know that
manual processes are error prone and their results are hard to maintain.
Instead, knowing that within `Tisch` we already have, at the type-level, all the
information that the `TableWithSchema` constructor wants, we will write a
function that given a `t` that is an instance of `Tisch`, it returns a `Table
(PgW t) (PgR t)`. In other words, we want to implement this:

```haskell
table :: T t -> Table (PgW t) (PgR t)
```

We could have used `Proxy` instead of `T` to pass around the type `t`, but as we
said before, in `opaleye-sot` we use `T` throughout the API whenever we want to
tag something that has a `Tisch` instance, as this leads to more readable code
in the query language and to more helpful error messages.

Even if quite illuminating, I won't go into the details of how `table` is
implemented because it is not fundamentally important and it would take a long
time to explain; you are invited read the source code for it in `opaleye-sot` if
you are curious. It should suffice to say that given a `t`, we convert
`SchemaName t` and `TableName t` to `String`s using `symbolVal`, then seemingly
out of thin air we make a term-level value of type `TableProperties (PgW t) (PgR
t)`, and finally we just give those three arguments to `TableWithSchema`. I
promise this is not a magical process, this is just profiting at the term-level
from information that is already present at the type-level. You can always go
from types to terms in Haskell—what's hard is going in the other direction, but
we are not trying to accomplish that today.

With `table` implemented, we finally have a way to derive a `Table (PgW t) (PgR
t)` for any `Tisch t`. For example, we can write:

```haskell
queryUsersTable :: QueryArr () (PgR User)
queryUsersTable = queryTable (table (T::T User))
```

Or, assuming we had another table described by `Tisch Drawing` that for some
users provided details about their drawings, we could even use `table` to do a
left join between them, using
the `leftJoin` exported from `opaleye-sot`, which is basically `opaleye`'s [`leftJoin`](https://hackage.haskell.org/package/opaleye-0.4.2.0/docs/Opaleye-Join.html#v:leftJoin) made compatible with `Kol`:

```haskell
queryUsersAndDrawings :: QueryArr () (PgR User, PgRN Drawing)
queryUsersAndDrawings =
  leftJoin (queryTable (table (T::T User)))
           (queryTable (table (T::T Drawing)))
           (\(user, drawing) ->
               {- the implementation here is not important -})
```

In general, we can write `table (T::T t)` wherever a `Table w r` is expected.
And actually, since a lot of times `t` can be inferred, such as in the
`queryUsersTable` example above, `opaleye-sot` exports a variant of `table` that
doesn't take `t` explicitly but instead infers it from the context. This doesn't
always work, of course, but when it does it reduces some of the noise in Opaleye
queries.

```haskell
table' :: Tisch t => Table (PgW t) (PgR t)
```

In `opaleye-sot` you will often have variants of the same function, like `table`
and `table'` here, where the one whose name ends with `'` is the one whose type
is expected to be inferred from its context at the use site. We can now rewrite
`queryUsersTable` to use `table'`:


```haskell
queryUsersTable :: QueryArr () (PgR User)
queryUsersTable = queryTable table'
```

In fact, this is so common that `opaleye-sot` exports these two functions for
us:

```haskell
queryTisch' :: Tisch t => Query (PgR t)
queryTisch' = queryTable table'

queryTisch :: Tisch t => T t -> Query (PgR t)
queryTisch _ = queryTisch'
```

`Query`, which you haven't seen before, is simply a synonym for `QueryArr ()`.
That is, `Query (PgR t)` means `QueryArr () (PgR t)`. This comes handy since most
of the times the first argument to `QueryArr` will be `()` anyway.

Now we can rewrite `queryUsersTable` once again:

```haskell
queryUsersTable :: Query (PgR User)
queryUsersTable = queryTisch'
```

But then, why bother? We have simplified the implementation of `queryUsersTable`
so much that maybe it is not worth giving its definition a top-level name
anymore; we could just inline `queryTisch'` or `queryTisch (T::T User)` wherever
needed. One of the goals of `opaleye-sot` is exactly this: Making the
things that are used in the query language feel so lightweight that we can't
justify using them anywhere but inline within a query, effectively reducing the
maintenance costs over time, as well as the reading and writing overhead.

## Interpreting results

Let's now run `queryTisch (T::T User)` and interpret the results as a Haskell
datatype. Opaleye exports a function named
[`runQuery`](https://hackage.haskell.org/package/opaleye-0.4.2.0/docs/Opaleye-RunQuery.html#v:runQuery)
which we will use in this example:

```haskell
runQuery :: Default QueryRunner r h => Connection -> Query r -> IO [h]
```

We will not worry too much about the `Default QueryRunner r h` constraint, it
only says that it should be possible to convert from the representation as type `r`
of a PostgreSQL query result to a term of type `h` representing the values in
the columns of that result in the actual Haskell runtime. We know from before
that when querying just a single table such as `User`, `r` will be `PgR
User` and `h` will be `HsR User` which we designed specifically for this
purpose:

```haskell
getUsers :: Connection -> IO [HsR User]
getUsers conn = runQuery conn (queryTisch (T::T User))
```

In the case of results where all of the columns are possibly `NULL`, like when
there is a missing right hand side on a `LEFT JOIN`, you may use `Maybe (HsR t)`
as the result type. For example, running `queryUsersAndDrawings` from before:

```haskell
getUsersAndDrawings :: Connection -> IO [HsR User, Maybe (HsR Drawing)]
getUsersAndDrawings conn = runQuery conn queryUsersAndDrawings
```

At some point we may want to convert these `HsR User` and `HsR Drawing` to other
internal representations of our own. This is not necessary though, if `HsR User`
suffices for us we can definitely work with that, and then we would have the
entire `Record` API at our disposal out of the box. But maybe we need to
interface with other piece of our codebase or a third party library that expects
a different representation for a user or a drawing, and for those cases you will
want to abandon the `HsR User` representation. A simple homemade function
`HsR User -> Maybe MyUser` would work just fine, but for
your convenience `opaleye-sot` exports a typeclass `Tisch t => UnHsR t a` with a
single method `unHsR' :: HsR t -> Either SomeException a`. This is mostly so
that we can treat conversions from any `HsR t` uniformly and that we don't have
to think of names for these conversions functions, which is very very hard. The
monomorphic `Either SomeException` wrapper around `a` makes the `Applicative` and
`Monad`ic composition of `unHsR'` results very practical, a desirable property
when trying to combine results from different queries; and of course, the
`SomeException` is there so that you can be very precise about why converting
from `HsR User` to `MyUser` failed, for example.

As a general recommendation: Remember that as our application evolves `HsR User`
will probably change, so we should build resilient code here. The best way to do
this, of course, is by always preferring to use the most precise types
available, but sometimes that won't be practical; for example, you wouldn't
create an `PersonAgeInYears` type that contains an integer number between 0 and
whatever the optimal life expectancy is, instead you would just use an
unsigned integer. If we had favorite colors, for example, and we knew that the
only acceptable favorite colors within `MyUser` would be red, black or purple,
then we should check this explicitly when converting from `HsR User` to `MyUser`
and fail with a precise exception if it is not the case. We may for example
fail with a `MyUser_BadFavoriteColour` exception, which will be very easy to
recognize if it ever happens in the future. We have a powerful language at our
disposal, let's use it wisely.

## Inserting rows

We will now go in the other direction: From Haskell into the PostgreSQL
database. We know we have thought about this scenario before when we implemented
`HsI` and `PgW`, so we should be covered. Let's just worry about how to actually
perform an insert using the
[`runInsert`](https://hackage.haskell.org/package/opaleye-0.4.2.0/docs/Opaleye-Manipulation.html#v:runInsert)
function from `opaleye`:

```haskell
runInsert :: Connection -> Table w r -> w -> IO Int64
```

There's nothing fundamentally new for us here: We already know what `Table w r`
means and we know that `Table (PgW t) (PgR t)` is a suitable candidate for it.
The returned `Int64` is of no particular concern to us at this point, it just
mentions the number of rows that were actually inserted to the database. We know
how to build a `Table (PgW t) (PgR t)` by means of `table`, but how do we build
the `PgW t` we need to pass as a second argument? Well, `PgW t` is just a
`Record` with nothing more than `Tagged`s, `Kol`s, `Koln`s and `WDef`s in it, so
in principle we could build it by hand. In practice, however, it is easier to
build an `HsI t` first and then have that automatically converted to a `PgW t`.

An `HsI t`, as we already know, will be a `Record` containing plain old Haskell
values like `Maybe`, `Int`, etc. That is, there is nothing secret about it and
we can use any of the tools exported by the `HList` library to build one. As a
convenience, `opaleye-sot` exports a function named `mkHsI` for constructing an
`HsI t` in a resilient and practical way. Similarly to the
[`hEndR`](https://hackage.haskell.org/package/HList-0.4.1.0/docs/Data-HList-HList.html#g:8)
function for building `Record`s, `mkHsI` is intended to be used together with
[`hBuild`](https://hackage.haskell.org/package/HList-0.4.1.0/docs/Data-HList-HList.html#v:hBuild).
For example, using `mkHsI` you could write the following function that
constructs an `HsI User`:

```haskell
toHsI_User
  :: Text                -- ^ Name.
  -> WDef (Maybe Int32)  -- ^ Favorite number.
  -> Maybe Int32         -- ^ Age.
  -> HsI User
toHsI_User name favoriteNumber age =
   mkHsI $ \set_ -> hBuild
      (set_ (C::C "id") WDef)
      (set_ (C::C "name") name)
      (set_ (C::C "favoriteNumber") favoriteNumber)
      (set_ (C::C "age") age)
```

`mkHsI` takes as its only argument a function `f` that will return an `HList` of
all the `Tagged (TC t c) a` elements `HsI t` expects, but not necessarily in its
canonical order. This function `f` is passed as its only argument yet another
function, here bound to the name `set_`, that will allow us to construct one of those
`Tagged (TC t c) a` values effectively associating a value `a` to a column
named `c` in the table uniquely identified by `t`. Perhaps this is easier to
understand if you think of `set_` just an assignment operator, here named `.=`:

```haskell
mkHsI $ \(.=) -> hBuild
   ((C::C "id")             .= WDef)
   ((C::C "name")           .= name)
   ((C::C "favoriteNumber") .= favoriteNumber)
   ((C::C "age")            .= age)
```

This is no magic. What we are seeing here, for the first time, is an usage of
the type-level names we gave to columns: `C "id"` uniquely identifies a
column named `"id"` in some table at the type-level. We already knew that we
would be using these as sort of references to our columns, so this shouldn't
come as a surprise. `C` is comparable to the `Proxy`-like types `T` and `TC`
what we saw before, and it exists for similar reasons and serves related
purposes:

```haskell
data C (c :: Symbol) = C
```

`C` simply carries type-level information about the name of a column. Notice
that the type `TC t c` is isomorphic to `(T t, C c)`. That is, `C c` is “the
other half” to `T t` which we have seen before. The `set_` function we just saw
enriches a `C` with knowledge about which table we are talking about, which of
course `mkHsI` already knows about. That is, it flattens a `T t` and a `C c`
into a single `TC t c` which it then uses to tag a value of type `a`. For
example, if we were trying to build an `HsI User` using `mkHsI`, then `set_`
would have the following type and implementation:

```haskell
set_ :: C c -> a -> Tagged (TC User c) a
set_ _ = Tagged
```

And as expected, the whole `mkHsI` expression _will fail to compile_ if the
names of the columns do not match the names first defined in the `Tisch`
instance for `User`. This is what we wanted from the beginning: The type-checker
doing an excellent job keeping _the wrong SQL_ away from us.

It is worth mentioning that writing `C::C` and `T::T`, instead of the more
_whitespacey_ and traditional `C :: C` and `T :: T` is just a personal syntactic
preference of mine so as to reduce any accidental whitespace noise. You may
choose to do otherwise. Maybe this aspect of things will improve
once the [`TypeApplications`](https://phabricator.haskell.org/D1138) language
extension lands in GHC.

Now that we have an `HsI User` all we need is to do is convert it to a `PgW
User` that we can pass as an argument to `runQuery`, which we can do with a
function called `toPgW'` exported by `opaleye-sot` for such purpose. And now, at
last we can insert a row into the `User` table, here from within an example
function named `insertUser`:

```haskell
insertUser
  :: Connection          -- ^ PostgreSQL Connection.
  -> Text                -- ^ User name.
  -> WDef (Maybe Int32)  -- ^ User favorite number.
  -> Maybe Int32         -- ^ User age.
  -> IO Int64
insertUser conn name favNum age =
   runInsert conn table'
      (toPgW' (toHsI_User name favNum age))
```

One last thing: Just like we had the `UnHsR` typeclass saving us from the need
to name functions such as `toHsI_User` and allowing us to treat this kind of
conversion uniformly across our codebase, there exists a similar typeclass named
`ToHsI` serving a similar purpose. The `toPgW'` function is polymorphic enough
that it will happily accept an `HsI t` or any `a` that can be converted to an
`HsI t` through the `ToHsI` typeclass.


## Querying

`opaleye-sot` encourages you to use exactly the same querying infrastructure as
`opaleye`, that is, the `Arrow` interface for `QueryArr` or `Query`. We don't
change anything there, but we do add new features. The first feature we added
was `queryTisch`, which removed a lot of noise from our queries by allowing us
to specify few or no types. The second feature is that, by having a uniform
representations for all our tables, `opaleye-sot` can provide tools for working
with them generically: You will learn to use these tools just once, and you will
carry that knowledge everywhere within the composable Opaleye query language.
Present in most of these tools is `C` which we saw before; we will use `C`
every time we want to refer to a column in a query. For example, consider this
query selecting all the users whose age equals their favorite number:


```haskell
q1 :: Query (PgR User)
q1 = proc () -> do
   u <- queryTisch' -< ()
   restrict <<< nullFalse -< eq
      (u^.col (C::C "age"))
      (u^.col (C::C "favoriteNumber"))
   returnA -< u
```

So much happened there. Let's walk step by step. First, by giving a type to this
query we allowed `queryTisch'` to infer its return type. That is, the
type-inferer knows that within this query `u` must be of type `PgR User`.
Second, we see this strange `restrict <<< nullFalse` thing: the `restrict` here
is the one exported from `opaleye-sot`, which is just like the `restrict` from
`opaleye` except it works on `Kol PGBool` instead of `Column PGBool` for the
reasons we explained before when we talked about `Kol` and `Koln`. Now, even if
we don't yet understand how the input `eq` expression works, we must acknowledge
one thing: Comparing two `NULL`able columns like `"age"` and `"favoriteNumber"`
for equality might result in a `NULL` value in SQL, so `opaleye-sot` forces us
to deal with that fact. `eq` is a very polymorphic function that works for any
combination of `Kol` or `Koln` arguments you may apply it to. Its return type,
on the other hand, is always fully determined by the input arguments, and it
will always be `Koln` if there was one among the passed in arguments like in
our case. So we have `restrict` expecting a `Kol PGBool` and `eq` returning a
`Koln PGBool`. `nullFalse` is simply an arrow function that converts the latter
to the former, transforming a possible `NULL` value to `FALSE`. After combining
all of this, we simply return `u`. That is, this query will accomplish the same
as the following SQL:

```SQL
SELECT "id", "name", "favoriteNumber", "age"
  FROM "public"."user"
 WHERE "age" IS NOT NULL
   AND "favoriteNumber" IS NOT NULL
   AND "age" = "favoriteNumber"
```

Of course the SQL generated by Opaleye will be uglier, but we are not here for
the looks. If we try to compile this query, however, we will fail; we will get
a type-checker error saying that the following instance is missing:

```haskell
instance Comparable User "age" User "favoriteNumber"
```

In `opaleye-sot` every binary operation between two columns will require that a
`Comparable` instance exists between them. This is designed so as to prevent
accidentally comparing columns whose types are the same but whose meanings are
completely different and not intended to be compared. By forcing us to
explicitly ask the compiler for permission to compare across columns we can
prevent more of _the wrong SQL_. Defining `Comparable` instances is very cheap
as we only have to specify, in the instance head, the names of the columns we
want to compare as well as the tables where each of them belongs. No instance
methods need to be defined. Of course, the type-checker will reject instances
for tables or columns that do not exist, or for columns whose types are
different. For example, both `Comparable User "age" User "falseColumn"` and
`Comparable User "age" User "name"` will fail to compile. I won't explain here
how this is achieved, but you are invited to learn more it about in
`opaleye-sot`'s source code.

Next we need to pay attention to the expressions `u^.col (C::C "age")` and
`u^.col (C::C "favoriteNumber")`. We already know that `u` is a `PgR User`, and
that `C::C "age"` and `C::C "favoriteNumber"` is what we use to reference two
columns uniquely identified by those names. There are only two new pieces here:
`^.` and `col`. The first one, `^.`, comes from Edward Kmett's
[`lens`](https://hackage.haskell.org/package/lens) library and there's not much
to say about it that hasn't been said before: Given a term `s :: s` and a term
`l :: Lens' s a`, then `s ^. l :: a`. In our case, `u` is that `s`, and `col
(C::C "age")` and `col (C::C "favoriteNumber")` are our `l`s.

What is `col` then? It is just a `Lens` to the value of a column, and its type
is something like the following—here simplified and restricted to `HsR t`:

```haskell
col :: C c -> Lens' (HsR t) (Tagged (TC t c) a)
```

It doesn't show up in this simplified type signature, but `a` is fully
determined by `t` and `c`: It is the very same `a` that we expect `PgR t` to
have at the column named `c`. If we recall the type `PgR User` reduces to, then
we can see what `col` does in practice. Here is `PgR User`:

```haskell
Tagged
  (T User)
  (Record
    '[ Tagged (TC User "id")             (Kol PGInt4)
     , Tagged (TC User "name")           (Kol PGText)
     , Tagged (TC User "favoriteNumber") (Koln PGInt4)
     , Tagged (TC User "age")            (Koln PGInt4)
     ])
```

And here are the types that `col` can take, simplified:

```haskell
col (C::C "id")
  :: Lens' (PgR User) (Tagged (TC User "id") (Kol PGInt4))
```

```haskell
col (C::C "name")
  :: Lens' (PgR User) (Tagged (TC User "name") (Kol PGText))
```

```haskell
col (C::C "favoriteNumber")
  :: Lens' (PgR User) (Tagged (TC User "favoriteNumber") (Koln PGInt4))
```

```haskell
col (C::C "age")
  :: Lens' (PgR User) (Tagged (TC User "age") (Koln PGInt4))
```

In reality `col` is not just a `Lens' s a` like we see here, but a full blown
`Lens s s' a a'` where `s` and `s'` can be any of `HsR t`, `HsI t`, `PgR t`,
`PgRN t` or `PgW t`, and `a` and `a'` are fully determined by `c` and `s` or
`s'` respectively. Which means that `col` will be useful in many other scenarios
such as extracting data from an `HsR t` result or updating the value in a
column as well.

Now that we know what `nullFalse` does, and that `u^.col (C::C "age")` is of
type `Tagged (TC User "age") (Koln PGInt4)` and that `u^.col (C::C
"favoriteNumber")` is of type `Tagged (TC User "favoriteNumber") (Koln PGInt4)`,
we can conclude that `eq` must satisfy this type:

```haskell
eq :: Tagged (TC User "age") (Koln PGInt4)
   -> Tagged (TC User "favoriteNumber") (Koln PGInt4)
   -> Koln PGBool
```

And indeed it does, but there's much more to it than this monomorphic type. If
we look at the type of `eq` in `opaleye-sot` we find this appalling type:

```haskell
eq :: Op_eq x a b c => a -> b -> c
```

`Op_eq` is a constraint synonym, and if we were to expand it and follow it we
would find that, like clockwork, it has many little pieces yet it is very
precise, leading to good type inference even in light of such seemingly abstract
nonsense. We will not do that exercise today though, instead we will just read
the documentation for `eq` to understand how it is supposed to be used:

<div style="font-style:italic">

> Documentation for `eq :: Op_eq x a b c => a -> b -> c`
>
> `opaleye-sot`'s polymorphic counterpart to `opaleye`'s `.==`.
>
> Mnemonic reminder: EQual.
>
> ```haskell
> eq :: Kol  x ->                  Kol  x  -> Kol  PGBool
> eq :: Kol  x ->                  Koln x  -> Koln PGBool
> eq :: Kol  x -> Tagged (TC t c) (Kol  x) -> Kol  PGBool
> eq :: Kol  x -> Tagged (TC t c) (Koln x) -> Koln PGBool
> eq :: Koln x ->                  Kol  x  -> Koln PGBool
> eq :: Koln x ->                  Koln x  -> Koln PGBool
> eq :: Koln x -> Tagged (TC t c) (Kol  x) -> Koln PGBool
> eq :: Koln x -> Tagged (TC t c) (Koln x) -> Koln PGBool
> ```
>
> Any of the above combinations with the arguments fliped is accepted too.
> Additionally, a `Comparable` constraint will be required if you try to
> compare two `Tisch`-aware columns directly; that is, a `Kol` or a `Koln`
> tagged with `TC t c`, such as those obtained with `col`. Here are some
> simplified example type signatures of this scenario just so that you get an
> idea:
>
> ```haskell
> eq :: Comparable t1 c1 t2 c2
>    => Tagged (TC t1 c1) a
>    -> Tagged (TC t2 c2) b
>    -> Kol  PGBool
> eq :: Comparable t1 c1 t2 c2
>    => Tagged (TC t1 c1) a
>    -> Tagged (TC t2 c2) b
>    -> Koln PGBool
> ```
>
> _Important_: `opaleye`'s `Column` is deliberately not supported. Use `kol`
> or `koln` to convert a `Column` to a `Kol` or `Koln` respectively.
>
> _Debugging hint_: If the combination of `a` and `b` that you give to `eq` is
> unacceptable, you will get an error from the type-checker saying that an
> `Op2` instance is missing. Do not try to add a new instance for `Op2`, it
> is an internal class that already supports all the possible combinations of
> `x`, `a`, `b`, and `c`. Instead, make sure your are not trying to do
> something funny such as comparing two `Koln`s for equality and expecting a
> `Kol` as a result. That is, you would be trying to compare two nullable
> columns and ignoring the possibilty that one of the arguments might be
> `NULL`, leading to a `NULL` result.

</div>

And then, without going into the details of `Op_eq`, its documentation sheds
some light about its purpose:

<div style="font-style:italic">

> Documentation `Op_eq x a b c`
>
> Constraints on arguments to `eq`.
>
> Given as `a` and `b` any combination of `Kol x`, `Koln x` or their
> respective wrappings in `Tagged (TC t c)`, get `c` as result, which
> will be `Koln PGBool` if there was a `Koln x` among the given
> arguments, otherwise it will be `Kol PGBool`.
>
> This type synonym is exported for two reasons: First, it increases the
> readability of the type of `eq` and any type errors resulting from its misuse.
> Second, if you are taking any of `a` or `b` as arguments to a function where
> `eq` is used, then you will need to ensure that some constraints are
> satisfied by those arguments. Adding `Op_eq` as a constraint to that
> function will solve the problem.
>
> _To keep in mind_: The type `c` is fully determined by `x`, `a`, and `b`. This
> has the practical implication that when both `Kol z` and `Koln z`
> would be suitable types for `c`, we make a choice and prefer to only support
> `Kol z`, leaving you to use `koln` on the return type if you want to
> convert it to `Koln z`. This little inconvenience, however, significantly
> improves type inference when using `eq`.

</div>

Functions like `eq` are the other big thing that `opaleye-sot` brings to the
`opaleye` query language. And, as complicated as their implementations might be,
the net benefit of `eq` for the end-user writing a query in the Opaleye query
language is huge: It is _the single entry point_ to comparing for equality any
any two comparable things, no matter where there are contained; and as many
constraints will be required on those comparable things in order to ensure that
we don't accidentally fall into _the wrong SQL_ trap. Of course, highly
polymorphic functions like `eq` wouldn't be desirable if they were to completely
abandon type inference and result in incomprehensible error messages when
used wrong, but with some careful design and experimentation, the implementor
of functions like `eq` can make the proper trade-offs and come up with something
that works well in most scenarios while leaving room for some manual calibration
of the types in the rare cases when it is needed. In our case, the
implementation of `eq` and similar functions relies on the
`FunctionalDependencies` language extension to achieve this.

Just like `eq` is used for comparing for equality, there are similarly typed
functions for comparing for order, logical conjunction and disjunction, etc.
Additionally, unary operators like SQL's `NOT` are upgraded to work over a
similar range of argument types in `opaleye-sot`. In general, any unary or
binary function operating on `Column` can be upgraded to be as polymorphic as
`eq` using some combination of `op1`, `op2`, `liftKol1`, `liftKol2`, and other
similar functions exported by `opaleye-sot`.

## Updating rows

The last topic we will cover today is updating rows. `opaleye-sot` exports
a function named `runUpdate` which behaves just like `opaleye`'s own
`runUpdate`, except it is a bit more polymorphic and it is intended to work with
`Kol`s and `Koln`s instead of `Column`s. Its type, simplified for our example,
looks like this:

```haskell
runUpdate
  :: Connection
  -> Table w r
  -> (r -> w)
  -> (r -> Kol PGBool)
  -> IO Int64
```

Skipping the obvious `Connection`, let's try to fill one by one the arguments.
We will be inserting a row to our `User` table, which we know we can obtain from
`table (T::T User)`. Now, with `w` fixed to `PgW User` and `r` fixed to `PgR
User`, we could proceed to fill the missing functions. However, let's pay
attention to the `r -> w` function that we will need to provide first, which
will be of type `PgR User -> PgW User` when specialized to our particular use
case: What is the difference between `PgR User` and `PgW User`? As we saw
before, the only difference is that the columns that can take a `DEFAULT` value
when being written need to be wrapped in `WDef`. The practical implication of
such difference in the types is that even if we just wanted update single value
in a single column in the `User` table, we would still need to manually wrap
all the columns that can take a `DEFAULT` value in `WDef`. _Manually_, a word by
now we abhor, even more so when we know that this process is entirely
mechanical. But similarly to how we demonstrated that it was possible to go from
`HsI t` to `PgW t` by means of `toPgW'`, it is also possible to go from `PgR t`
to `PgW t` by means of `update'`, so we could just precompose `update'` with a
function of type `PgW t -> PgW t` that only worries about updating the columns
that will change. Knowing that this is such a common scenario, `opaleye-sot`
exports a function `runUpdateTisch` specially designed to work with `Tisch`
instances, which we will use instead of `runUpdate`—its type here simplified for
our explanatory purposes:

```haskell
runUpdateTisch
  :: Connection
  -> T t
  -> (PgW t -> PgW t)
  -> (PgR t -> Kol PGBool)
  -> IO Int64
```

For example, let's say last week it was my birthday and I just realized my new
age is also my new favorite number, and I want to reflect that in the `User`
table:

```haskell
runUpdateTisch conn (T::T User)
  (\u -> set (cola (C::C "favoriteNumber"))
             (WVal (u^.cola (C::C "age"))) u)
  (\u -> eq (kol ("Renzo" :: Text))
            (u^.cola (C::C "name")))
```

This code will generate an SQL comparable to this

```SQL
UPDATE "public"."user"
   SET "favoriteNumber" = "age"
 WHERE "name" = 'Renzo'
```

Of course, I'm being optimistic and assuming I am the only Renzo in the
database, but hopefully you get the idea. The first function that we passed to
`runUpdate` changes the values in the columns of the row as needed, in our case
updating the `"favoriteNumber"` column as planed using the
[`set`](https://hackage.haskell.org/package/lens-4.13/docs/Control-Lens-Setter.html#v:set)
function from the `lens` library. The other function we passed is the boolean
predicate where we select which columns we want to update—notice how we used
`eq` again here. The only new thing here is `cola`, which behaves just like the
`col` lens we saw before, except it removes the `Tagged (TC t c)` wrapper around
the value in the representation for the column that `col` would otherwise keep.

##  Conclusion

This is our stop. We have learned how `opaleye-sot` befriends the Haskell type
system to keep us safe and free from boilerplate, we learned how to reduce a
problem to its essentials, we have seen how it is possible to clean up a public
API so that it covers as many scenarios as possible while staying correct and
predictable, we learned how to help users accomplish their goals with as little
friction and as much safety as possible, we have given a practical perspective
to advanced type feature systems, and maybe we have saved the next billion
dollars as well.

We did embrace Opaleye and the generation of well-formed and _not wrong_ SQL as
our scenario, but it is important to realize that these techniques and
approaches are still applicable in other scenarios. It is up to us, as people
who understand a problem at hand, to try and teach the type system as much as
we can about that problem. And even if we don't yet understand the problem:
Let's talk to the type system about it; it will help us understand. Remember,
the type system is not magic, it is a logical reasoning tool.


## Is it on Hackage yet?

Not yet. Up until now I have been building `opaleye-sot` as a side-effect of a
another project I have been trying to bootstrap, which means that the features
currently present in `opaleye-sot` are the ones I have happened to need and have
had time to complete. But of course, there are a lot of things missing such as
more binary operators, support for composite columns, a story for aggregation
and perhaps a way of checking at runtime that the `Tisch` you compiled has the
expected shape in the database. So in a sense, this article is also an
invitation for you to contribute to `opaleye-sot`, even if just by asking for
some feature to exist or some documentation to improve. `opaleye-sot` will be on
Hackage once it can be considered _production ready_. Meanwhile, I invite you to
go and explore its [source code](https://github.com/k0001/opaleye-sot), as well
its [documentation](). In the source code you will find a tutorial in the works
too, with some additional examples. All of it free and open source as you would
justifiably expect.

## Discussion

You can discuss this comment at [reddit]().

<div style="margin-left:1em; padding-left:1em; border-left: solid #111 1px;">
_By <a href="../">Renzo Carbonara</a>. First published in October 2015.<br/>
This work is licensed under a <a rel="license"
href="http://creativecommons.org/licenses/by/4.0/">Creative Commons Attribution
4.0 International License</a>._
</div>
