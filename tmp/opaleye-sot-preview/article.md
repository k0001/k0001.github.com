# Opaleye's sugar on top: SQL in the type system where it belongs

_People often talk of how solutions fall naturally into place when we program in
Haskell and embrace its type system. This article walks us through that process,
serving as a gentle introduction to some practical uses of advanced features of
the GHC Haskell type system within the context of Opaleye and SQL. I invite you
to continue reading even if you are not particularly interested in Opaleye nor
SQL, as the techniques explained here will be useful in other contexts too._

Reading, writing and maintaining hand-written SQL is hard and error prone. Over
the years and in many languages, people have tried to mitigate this problem to
different extents by creating tools that convert data types back and forth
between SQL and more practical runtime representations, as well as by creating
vocabularies for expressing queries or manipulation routines over the contents
of a database in ways that are clearer and more maintainable than just plain
hand-written SQL.

The [Haskell](https://www.haskell.org/) language ecosystem has had its fair share
of solutions to this problem too. Of these, powerful libraries such as
[`opaleye`](https://hackage.haskell.org/package/opaleye) (Tom Ellis et al.) and
[`relational-record`](https://hackage.haskell.org/package/relational-record)
(Kei Hibino et al.) are of particular interest to us today. What is special
about these libraries is that, making different trade-offs, they guarantee that
they will always generate well-formed SQL. The way they achieve this, as is
usually the case in Haskell, is by choosing to build their vocabulary on top of
precise abstractions ensuring that only things that make sense can be _composed_
together. In practical terms, this means that if we try to, for example, insert
an SQL table into a left join—whatever that might mean—then our program will
fail to compile because there is no sensible way in which we can compose an SQL
table and a left join by means of inserting the former into the latter. This
predictability in what can and cannot be composed gives us new ways to reason
formally about the queries that we write, which takes us a long way from that
old problem of being unable to read, write or maintain hand-written SQL
effectively and efficiently.

However, none of this means we won't ever generate _the wrong SQL_ when using
these tools, as there is still room for making mistakes such as mistyping the
name of a column or a table, which will lead to a well-formed but undesired SQL
being generated. It is important that we understand this difference between what
is well-formed SQL and what is the right SQL (as opposed to _the wrong SQL_). Is
`SELECT user.name FROM user` well-formed? Yes it is. But is it right or wrong?
Unfortunately we can't tell without more context, as the answer depends on
whether `user.name` was really the value we wanted to select, and on whether the
`user` table exists in the database and is the actual table we were interested
in. Otherwise, it might have been a mistake on our part when typing the names,
or maybe we wanted to select `song.name` and we accidentally referred to the
wrong table. In general, libraries such as `opaleye` and `relational-record` can
guarantee that they will generate well-formed SQL for us, but they
cannot—justifiably so—judge a well-formed SQL and decide whether it is right or
wrong, because these tools can't possibly know and understand the meaning and
purpose of said SQL. Nevertheless, we can improve the situation a bit by making
it harder for users of these libraries to write what they consider to be the
wrong SQL.

Today we will focus on Opaleye, and in particular I want to talk a bit about
[opaleye-sot](https://github.com/k0001/opaleye-sot), a yet unreleased library I
have been working on for a while that brings some ideas worth sharing, as they
can be useful beyond this library and showcase some advanced uses of the type
system that have concrete practical benefits. As the name—short for “Opaleye's
sugar on top”—hopefully indicates, this is just a thin layer of new features on
top of Opaleye's standard offer, and it is intended to complement Opaleye, not
to replace it. `opaleye-sot` brings a different take on the public API that
users of Opaleye are encouraged to use; in particular, it makes extensive use of
advanced type system features offered by
[GHC, the Glasgow Haskell Compiler](http://www.haskell.org/ghc), so as to reduce
the number scenarios that could lead to Opaleye generating the wrong SQL, to
reduce the amount of boilerplate that one needs to write in order to make
effective use of Opaleye, and to increase the readability and maintainability of
queries and data types written using this API. `opaleye-sot` is an early project
with not many people depending on it yet, so hopefully it will be able to evolve
rapidly and function as a test bed for new features or approaches, some of which
may eventually make it to `opaleye` proper, without worrying too much about
backwards compatibility for now. [PostgreSQL](http://www.postgresql.org/) is the
only supported SQL backend.

For the rest of this article, I will assume familiarity with the basic usage of
Opaleye, and build on that. Familiarity with `HList`, `TypeFamilies`, `GADTs`,
`DataKinds` and `KindSignatures` will be useful too, but not required. We will
cover the topic of preventing _the wrong SQL_, but first let us worry about a
the simpler topic of boilerplate.


## Boilerplate prevention and uniform representations

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
depending on the scenario. For example, we might have `type FooHs = Foo Bool
String` and `type FooPgW = Foo (Column PGBool) (Column PGText)`, where `Hs`
stands for “Haskell” and `PgW` stands for “PostgreSQL, write”. There are a
couple of additional scenarios like these two where `a` and `b` will need to
change. Personally, I think this is a good approach to working with different
yet somewhat similar representations, but hopefully you will agree with me that
this can lead to a non-negligible amount of boilerplate, in particular when
working with rows having many columns, and we need to prevent this. Our goal is
to reduce the amount of boilerplate we need to write regarding these types,
hopefully without resorting to the powerful but huge, fragile and uncomfortable
hammer that `TemplateHaskell` can be.

`opaleye-sot` solves this problem by using a combination of
[`HList`](https://hackage.haskell.org/package/HList)
(Oleg Kiselyov, Ralf Laemmel, Keean Schupke et al.)
and type families which we will now analyze. For those unfamiliar with with
`HList`, in its traditional representation using
[GADTs](https://wiki.haskell.org/Generalised_algebraic_datatype) it looks like
this:

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
> True : 42 : [] :: [Bool, Int]
> -- This does typecheck:
> True `HCons` 42 `HCons` HNil :: HList '[Bool, Int]
```

`HList` has some properties that are very interesting to us. To start with, not
only we can store an arbitrary number of elements of different types, but we can
also know statically—that is, at compile time—both the number and types of those
elements. Moreover, that information is kept in a type-level list promoted using
`DataKinds`, which means that we can manipulate that list and see it change its
length and contents at compile time, just like we could do with a traditional
polymorphic list (i.e., `[]`) at runtime. And finally, `HList` can be given an
instance of `ProductProfunctor` that works for lists of any length, not only
making it a suitable datatype for Opaleye, but also preventing us from having to
define a new `ProductProfunctor` instance for each different Haskell
representation we would like to use for mapping SQL rows. For these reasons, we
will use `HList` as the preferred container for all our SQL row mapping needs,
which gives us an uniform way to work with different SQL rows. In particular, we
will work with `HRecord`, a variant of `HList` provided by the `HList` library
which guarantees that each element in the list is `Tagged`. Unsurprisingly, we
will tag each element in the `HRecord` with the literal name of the SQL column
it represents, which will allow us to refer to said columns without knowing
their specific position in the SQL row. And perhaps more importantly, these tags
will keep us from accidentally mistyping column names later on, making it harder
for us to write _the wrong SQL_.

Armed with `HRecord`, we can now to try map between it and an actual SQL row in
different Opaleye scenarios. We will use this contrived but comprehensive SQL as
example:

```sql
CREATE TABLE user
  ( id serial4 NOT NULL
  , name text NOT NULL
  , favoriteNumber int4 NULL DEFAULT 42
  , age int4 NULL
  )
```

Notice what is special about those 4 columns regarding the values they can take:

Column            Can contain `NULL`?  Can `DEFAULT` be written to it?
----------------  -------------------  -------------------------------
`id`              No                   Yes
`name`            No                   No
`favoriteNumber`  Yes                  Yes
`age`             Yes                  No


In `opaleye-sot` we distinguish between these combinations very explicitly, more
so than in the way Opaleye does it out of the box today—although
[there are plans to improve this](https://github.com/tomjaguarpaw/haskell-opaleye/issues/97).
We represent non nullable columns as `Kol`, nullable columns as `Koln`, and
columns that can take a `DEFAULT` value when being written to as `WDef`. We will
talk about these types in more detail later, for now we will just see them being
used together with `HRecord` in different scenarios.


## Scenario 1: HsR, Haskell values read from the database

This is the simplest scenario. Since we are only concerned about reading from
the database here, we can safely ignore the question of whether `DEFAULT` can be
written in each particular column. We must worry about whether the column can
contain `NULL`, however. We will represent that possibility as `Maybe`.

Ignoring `HRecord`, `HList` and Opaleye for a moment, we might opt for a
representation like the following one. To prevent any confusion with forthcoming
examples, we will add the `_HsR` suffix to these types, meaning that this is the
representation of “Haskell values read from the database”. HsR: Haskell, Read.

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

And of course, if we wanted to, we could manually define field accessors similar
to the ones in the example record definition for `User_HsR`, or even lenses. But
alas, we don't want to, because doing this will open a door to _the wrong SQL_.
Let's carefully think why and analyze this bit together, because it is in small
details such as this one where Haskell can go the extra mile and radically tell
itself apart from lesser programming languages when it comes to code
maintenance.

Can you tell the difference between these two types?

```haskell
type User_HsR1 = (Int32, Text, Maybe Int32, Maybe Int32)

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

data User_HsR2 = User_HsR2
  { _user_HsR2_id             :: Int32
  , _user_HsR2_name           :: Text
  , _user_HsR2_age            :: Maybe Int32
  , _user_HsR2_favoriteNumber :: Maybe Int32a SQL row,
  }
```

That's right, the order of the two last fields was changed, but of course we
didn't notice the first time because their types were the same. And neither did
the type checker. So I ask myself what keeps the result of `SELECT id, name,
favoriteNumber, age FROM user`, in that order, from being somehow converted into
any of `User_HsR1` or `User_HsR2` above? And the answer is, of course, nothing.
One day we will wake up having 12345 years of age because that was our favorite
number.

To me, falling into traps like this one is unacceptable, even more so when
working in large projects where we can't possibly keep track of all these
details, or at least where we shouldn't need to because surely there are more
important things to do. It is not our job, we have type checkers. Our job is to
teach the type checker how to tell right from wrong and move on, which is not so
hard if we restrict ourselves to small domains and leverage a type system as
expressive as GHC's.

In order to solve this, first, we need to precisely identify what we are trying
to accomplish: We would like to use names to identify the particular columns in
our Haskell representations for SQL rows, because telling apart the meaning of
_age_ from the meaning of _favorite number_ is much easier than telling apart
the meaning of a `Maybe Int32` from the meaning of another `Maybe Int32`.
Additionally, we want to be sure that every time we say _age_ in our Haskell
representation we do mean _age_ in the SQL row too, and that we make it very
hard for this property to be violated accidentally.

Having identified our goal, and having talked previously about the problem, we
can start working towards a solution. We might categorize the problem here as
one where there is more than one source of information, that is, where both our
Haskell representation and the SQL table are trying to inform us the names of
the fields. But which source do we trust when they differ? Wrong, that is the
wrong question. The right question to ask ourselves is how to _prevent_ those
two sources from ever differing. And, as it is usually the case in situations
like this one, the problem is that we have two sources of information where we
should have had just one.

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
selecting it, updating it or for showing its name in some debugging tool—we will
refer to its type; and only then, maybe, convert said type to a term if at all
needed. Additionally, we will never refer to columns by their position in a row,
only by its name.

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
strings don't have a term-level representation, so its kind is something else:
`Symbol` in this case. Conversion between a type-level string and a term-level
string is made explicitly via `symbolVal`. Here's a GHCi session demonstrating
the usage of `Symbol` and kinds.

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

And now, with this new tool we can modify our `HList` solution from before so
that not only it mentions the type of the value contained in each particular
column, but also the name of the column itself, so that if we ever write mistype
the name of column the type checker will let us now. For this, we will use the
`Tagged` data type, yet another very simple but very powerful tool for teaching
our type checker how to tell right from wrong.

```haskell
newtype Tagged t a = Tagged a
```

Remember, what is mentioned to the left of the `=` exists on the type level and
can be known statically at compile time, and what is mentioned to its right
exists on the term level and can be known at runtime. In other words, `t` needs
not have an accompanying term of type `t`, and there is nothing in the `Tagged`
constructor that restricts `t` to be some particular type. A type like `t` which
apparently serves no purpose is called a
[panthom type](http://www.haskell.org/haskellwiki/Phantom_type), and we have
seen this kind of type before when we used `Proxy` in the previous example.

```haskell
data Proxy t = Proxy
```

Look at that. `Proxy` seems to be even more useless than `Tagged`, it carries no
information whatsoever at the term-level. But the thing is, `Proxy` is a tool
for programming at the type-level, not at the term-level, so that is perfectly
fine. When we used `Proxy` before in our `symbolVal` example, we weren't
interested in any term-level value. We couldn't possibly have been, because as
we said before, type-level strings do not exist at the term-level; if they
existed their kind would have been `*`, but alas, it is `Symbol`. Nevertheless,
`symbolVal`, a class method that exists at the term-level, needs to somehow
receive as a term-level argument a type-level string so that it can dispatch to
the appropriate `KnownSymbol` instance. `Proxy` is simply a sort of bridge that
connects the type-level with the term-level.

Now that we have learned about `Proxy`, there is not much to say about `Tagged`
beyond the fact that it is isomorphic to `(Proxy t, a)`. That is, not only it
carries some type-level value `t`, but also a plain old term-level value `a`.
With `Tagged` we can write hypothetical functions such as this:

```
callFrenchPhone :: Tagged France PhoneNumber -> IO CallInformation
```

Here, the type-checker will ensure that only `PhoneNumbers` tagged with
`France` can be called using the `callFrenchPhone` function:

```
> -- This type checks:
> callFrenchPhone (... :: Tagged France PhoneNumber)
> -- This doesn't checks:
> callFrenchPhone (... :: Tagged India PhoneNumber)
```

It is worth noting that the benefits we get from `Tagged French PhoneNumber` are
not so different from the benefits we get from introducing a `newtype` like:

```haskell
newtype FrenchPhoneNumber = FrenchPhoneNumber PhoneNumber
```

Nevertheless, `Tagged t a` is at times more convenient to use than the `newtype`
solution, and it can be used generically for `t`s that may not be yet known at
compile time.

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

## Scenario 2: Maybe HsR, possibly missing Haskell values read from the database

Even while `User_HsR` is a perfectly acceptable Haskell representation for SQL
rows comming out of the database, sometimes, those rows will be empty, such as
in the case of a missing right hand side in a `LEFT JOIN`. For those cases,
`Maybe User_HsR` will be a perfectly acceptable type within `opaleye-sot`. A
variant of `User_HsR` where each column is wrapped in `Maybe` is acceptable too,
but often less practical.

## Scenario 3: HsI, Haskell values to be inserted to the database

In this scenario the types are mostly like those in the previous one, but we
also want to consider that some columns can take a `DEFAULT` value when being.
In our case, the columns `id` and `favoriteNumber`. We will simply wrap with
`WDef` the types of the elements representing those columns in our `HRecord`.
`WDef` is defined like this:

```haskell
data WDef a = WDef | WVal a
```

The idea is that if you want to write a specific value to a column you wrap it
in the `WVal` constructor, otherwise you use the `WDef` constructor and
`opaleye-sot` will replace that with `DEFAULT` in the generated SQL.

With that in place, we can proceed to define our counterpart to `User_HsR`
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
well have used `Nothing` to signify that we want to write `DEFAULT` to the
database, and `Just` otherwise. So, why didn't we do that? Why did we introduce
a whole new type just for this? We did it because, again, this is a situation
where by carefully paying attention just once, we can forever profit from
preventing more of _the wrong SQL_.

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
value in those columns means `DEFAULT` or `NULL`. But besides all this, not at
all obvious and much more interesting to us, is the fact that both `Nothing` and
`Just Nothing` are valid values for the type `Maybe (Maybe Int32)`, which means
that mistaking one for the other would go completely unnoticed by the type
checker, quite possibly leading us to _the wrong SQL_. By making the distinction
between `WDef` and `Maybe` explicit we have taught the type-checker to tell
right from wrong once again.

## The billion dollar mistake

> I call it my billion-dollar mistake. It was the invention of the null reference
> in 1965. At that time, I was designing the first comprehensive type system for
> references in an object oriented language (ALGOL W). My goal was to ensure that
> all use of references should be absolutely safe, with checking performed
> automatically by the compiler. But I couldn't resist the temptation to put in a
> null reference, simply because it was so easy to implement. This has led to
> innumerable errors, vulnerabilities, and system crashes, which have probably
> caused a billion dollars of pain and damage in the last forty years.
>
> Tony Hoare - 2009

I think a billion dollars may be an understatement, but for the sake of
discussion let's keep it that way and move on.

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
results in a `bool` value, then we need to learn how to deal with it. Or not,
because in Haskell we already know how to deal with `NULL` by means of `Maybe`,
and we can leverage that knowledge.

Most times, if we want to make something useful with a value of type `Maybe a`,
at some point we will probably need to understand what is `a` and how to use it.
For example, if we want to `fmap` some function `a -> b` over `Maybe a`, then we
need to make a choice about what that `a` is, or at least which constraints it
satisfies. We could say we expect to have a `Maybe Int`, or a `Maybe Bool` or
even a `Num a => Maybe a`. The important thing to notice is that the `a` in
`Maybe a` must be a type with a term-level representation that exists in memory
Haskell at runtime. But when using Opaleye to generate SQL queries, we are not
dealing with Haskell values that exist in memory at runtime at the very same
moment when when the SQL is being generated. In Opaleye, when you have a `Column
PGBool`, that is just a promise that if the generated SQL is ever run, wherever
it runs, the type of a specific column inside the PostgreSQL database will be
`bool` (with the caveat that it might contain a `NULL` value), but there is no
way to manipulate the actual `PGBool` directly. Much like the `t` in `Tagged t
a`, `PGBool` and similar types exist only at the type-level, they have no
term-level representation in Haskell at runtime.

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

-- | Like 'fmap' for 'Maybe'.
--
-- Apply the given function to the underlying 'Kol a' only as long as the
-- given 'Koln a' is not 'NULL', otherwise, evaluates to 'NULL'.
mapKoln :: (Kol a -> Kol b) -> Koln a -> Koln b

-- | Monadic bind like the one for 'Maybe'.
--
-- Apply the given function to the underlying 'Kol a' only as long as the
-- given 'Koln a' is not 'NULL', otherwise, evaluates to 'NULL'.
bindKoln :: Koln a -> (Kol a -> Koln b) -> Koln b

-- | Like '(<|>) :: Maybe a -> Maybe a -> Maybe a'.
--
-- Evaluates to the first argument if it is not 'NULL', otherwise
-- evaluates to the second argument.
altKoln :: Koln a -> Koln a -> Koln a
```

And with these tools we are can now reason in terms of `Maybe`, `Functor`s,
`Monad`s and `Alternative`s _in the SQL expresions themselves_, not just in the
Opaleye query language. The definitions of those functions are not important,
but you might check the source code if you are curious. What is important is to
understand the difference between `Kol a` and `Koln a`. Whereas `Koln a`
explicitly informs us that said `a` might be `NULL`, and that we must explicitly
deal with that possibility, `Kol a` tells us that `a` can't possibly be `NULL`.
Or, in another words, `Koln a` is analogous to `Maybe a` while `Kol a` is
analogous to `Identity a`, and we now that `a` and `Identity a` are isomorphic.
So, rewriting some well-known Haskell functions to use `Identity` and following
these analogies, we up with this:

```haskell
matchKoln :: Kol      b -> (Kol      a -> Kol      b) -> Koln  a -> Kol      b
-- 'Identity x' is analogous to 'Kol x', 'Maybe x' is analogous to 'Koln x'
          :: Identity b -> (Identity a -> Identity b) -> Maybe a -> Identity b
-- 'x' is isomorphic to 'Identity x'
maybe     ::          b -> (         a ->          b) -> Maybe a ->          b

mapKoln :: (Kol      a -> Kol      b) -> Koln  a -> Koln  b
-- 'Identity x' is analogous to 'Kol x', 'Maybe x' is analogous to 'Koln x'
        :: (Identity a -> Identity b) -> Maybe a -> Maybe b
-- 'x' is isomorphic to 'Identity x'
fmap    :: (         a ->          b) -> Maybe a -> Maybe b

bindKoln :: Koln  a -> (Kol      a -> Koln  b) -> Koln  b
-- 'Identity x' is analogous to 'Kol x', 'Maybe x' is analogous to 'Koln x'
         :: Maybe a -> (Identity a -> Maybe b) -> Maybe b
-- 'x' is isomorphic to 'Identity x'
(>>=)    :: Maybe a -> (         a -> Maybe b) -> Maybe b

altKoln :: Koln  a -> Koln  a -> Koln  a
-- 'Identity x' is analogous to 'Kol x', 'Maybe x' is analogous to 'Koln x'
(<|>)   :: Maybe a -> Maybe a -> Maybe a
```

Opaleye, as of today, offers `Column a` and `Column (Nullable a)` as
counterparts to `Kol a` and `Koln a`. What is so special about `Kol a` is that
it makes it explicit that said `a` may never be `Nullable`. That's all there is
to it really. Of course there are ways to convert between `Column`, `Kol` and
`Koln` by means of `kol`, `unKol`, `koln` and `unKoln` so that `opaleye-sot`
code can readily compose with `opaleye` code using `Column`. Hopefully this
explicit difference will exist in `opaleye` proper soon.


## Scenario 4: PgR, PostgreSQL values being read from the database

Now that we understand `Kol` and `Koln`, we can proceed to give a Haskell
representation to the PostgreSQL counterpart of `User_HsR`. We will call it
`User_PgR` where `PgR` stands for “PostgreSQL, read”.

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

`User_PgR` is the type that will use when writing queries using Opaleye's query
language, it is our Haskell representation of a SQL row that doesn't exists in
the Haskell runtime but in the database. `User_PgR` is to `User_HsR` what `Kol`
is to `Identity` or `Koln` is to `Maybe`.

## Scenario 4: PgRN, possibly missing PostgreSQL values being read from the database

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
make sense. The columns that were `Koln` before need to stay as `Koln`, only the
columns that were `Kol` in `User_PgR` need to be changed to `Koln`.

## Scenario 5: PgW, PostgreSQL values to be written to the database

Finally, our last scenario. Just like `User_PgR` was the PostgreSQL counterpart
to `User_HsR`, `User_PgW` here will be the PostgreSQL counterpart to `User_HsI`,
execept not only will be used when inserting new values, but also when updating
them. Following our nomenclature, “PgW” stands for “PostgreSQL, write”.

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
Opaleye using our `Record` representations. Congratulations to us. But now that
we understand, we need to teach all of this to the type system so that it can
remember it for us like `opaleye-sot` does, so that we can forget all these
details and worry about more important things such as writing the actual SQL
queries.
