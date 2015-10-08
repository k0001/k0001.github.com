# Opaleye's sugar on top: SQL in the type system where it belongs

Reading, writing and maintaining hand-written SQL is hard and error prone.
Over the years and in many languages, people have tried to mitigate this problem
to different extents by creating tools that convert data types back and forth
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
Opaleye, and build on that. Familiarity with `HList`, `TypeFamilies`, `GADTs`
and `DataKinds` will be useful too. We will cover the topic of preventing _the
wrong SQL_, but first let us worry about a the simpler topic of boilerplate.


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

`opaleye-sot` solves this problem by using a combination of `HList` and type
families which we will now analyze. For those unfamiliar with with `HList`, in
its traditional representation using
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


## Scenario 1: representation of Haskell values read from the database.

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
  , _user_HsR2_favoriteNumber :: Maybe Int32
  }
```

That's right, the order of the two last fields was changed, but of course we
didn't notice the first time because their types were the same, and neither did
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

In order to prevent this


Now, if we wanted to make these types friendlier to Opaleye and the many
scenarios where these types would be used, we would need to make all of
the fields polymorphic:

```haskell
data User a b c d = User
  { _user_id             :: a
  , _user_name           :: b
  , _user_favoriteNumber :: d
  , _user_age            :: e
  }
```

And similarly if we didn't care about the name of the columns:

```haskell
type User a b c d = (a, b, c, d)
```
