---
layout: page
title: "REST API Tutorial"
date: 2017-05-01 10:52:00
author: Bryn Edwards
permalink: /tutorial/rest-api
hackage_base: //hackage.haskell.org/package
---

* TOC
{:toc}

# Overview

In this tutorial, we're going to use Spock to build a simple RESTful API
with a database backend. This tutorial covers:

- Setting up a project using [Stack](https://docs.haskellstack.org/en/stable/README/)
- Spock routing: GET, POST
- Send and receiving Haskell data types as JSON
- Adding a database connection pool to Spock
- Using [Persistent](//www.yesodweb.com/book/persistent)
  - Creating a database schema
  - Running queries in Spock routes

You can find the finished code
[here](//github.com/brynedwards/spock-tutorials/tree/master/rest-api). "part1"
covers up to Adding a Database; "part2" covers up to Finishing up. 

# Project Setup

Before using Spock, you'll need to install the Haskell toolchain on your
machine. We recommend using the [stack](//haskellstack.org/) tool to
quickly get started! Our guide will be `stack` based, but you can easily
translate this to `cabal`.  Next, you can prepare a directory for your first
Spock powered application:

1. Create a new project using `stack new Spock-rest`
2. Jump into the directory `cd Spock-rest`

### Dependencies

To make sure your dependencies will match those used in this tutorial, set
your project to use Stackage LTS 8.13: Open your `stack.yaml` file, find the
`resolver` key and make sure its value is `lts-8.13`.

Next, you're going to add the packages you'll be using: `Spock`, `aeson`
and `text`. To do this, open your `Spock-rest.cabal` file, find the
`executable Spock-rest-exe` section and add `aeson`, `Spock` and `text`
to the `build-depends` key. There should also be a `Spock-rest` entry which
you can remove or just ignore. The result should look something like this:

```
  build-depends:         base
                       , aeson
                       , Spock
                       , text
```

### Imports

Let's start by adding a couple of language extensions and imports. Open
`app/Main.hs` and replace the content with:

{% highlight haskell %}

{-# LANGUAGE DeriveGeneric     #-}
{-# LANGUAGE OverloadedStrings #-}

import           Web.Spock
import           Web.Spock.Config

import           Data.Aeson       hiding (json)
import           Data.Monoid      ((<>))
import           Data.Text        (Text, pack)
import           GHC.Generics

{% endhighlight %}

We'll be using the `DeriveGeneric` extension along with `GHC.Generics`
to create `FromJSON` and `ToJSON` instances of our API type.
The `Data.Aeson` library provides our JSON type conversion; you can view its
[documentation]({{ page.hackage_base }}/aeson-1.1.2.0/docs/Data-Aeson.html#g:1)
to learn more.

# API Type

Now we'll create a data type for our API. Add the following to your `Main.hs`,
below the import statements:

{% highlight haskell %}

data Person = Person
  { name :: Text
  , age  :: Int
  } deriving (Generic, Show)

instance ToJSON Person

instance FromJSON Person

{% endhighlight %}

We can try out our new Person type in GHCi: run `stack ghci` in your
project root directory to build your project and start a REPL session. Try
using `encode` and `decode` to serialise and deserialise `Person`s into
`ByteString`s like so:

```
λ> encode Person { name = "Leela", age = 25 }
"{\"age\":25,\"name\":\"Leela\"}"

-- So our string literal can inferred as a ByteString
λ> :set -XOverloadedStrings
λ> decode "{ \"name\": \"Amy\", \"age\": 30 }" :: Maybe Person
Just (Person {name = "Amy", age = 30}) 
```

Aeson's `decode` type signature is:

{% highlight haskell %}
FromJSON a => ByteString -> Maybe a
{% endhighlight %}

So we have to tell GHCi what we want type we want it to try decode our bytestring as,
otherwise it will default to (). We can also decode it as Aeson's generic `Value` type:

```
λ> decode "{ \"name\": \"Amy\", \"age\": 30 }" :: Maybe Person
Just (Object (fromList [("age",Number 30.0),("name",String "Amy")]))
```

# Serving JSON

Now we'll add a basic Spock application to serve our JSON. Add the
following to your `Main.hs`:

{% highlight haskell %}

main :: IO ()
main = do
  spockCfg <- defaultSpockCfg () PCNoDatabase ()
  runSpock 8080 (spock spockCfg app)

app :: SpockM () () () ()
app = do
  get "person" $ do
    json $ Person { name = "Fry", age = 25 }

{% endhighlight %}

Spock includes a `json` function for serving any type that implements the
`ToJSON` typeclass, which means you can pass your `Person` and it will encode
it as JSON and set the HTTP Content-Type header to `application/json` for
you. You can start the server in GHCi by first reloading the project using
the `:reload` command then running your `main` function:

```
λ> :reload
[2 of 2] Compiling Main
Ok, modules loaded: Lib, Main.

λ> main
Spock is running on port 8080 
```

Go to [localhost:8080/person](//localhost:8080/person) and you should
see your `Person` object in JSON.

REST APIs also need to serve lists of items; since `aeson` includes a `ToJSON
a => ToJSON [a]` instance, we can easily serve a list of `Person`s. Change
your `get` function to the following:

{% highlight haskell %}

  get "person" $ do
    json [Person { name = "Fry", age = 25 }, Person { name = "Bender", age = 4 }]

{% endhighlight %}

Enter `ctrl-c` in GHCi to interrupt the server and get back to the prompt. Then,
reload your project and start the server again. Refresh your browser to see
your two `Person`s in a JSON array.

# Parsing JSON

Now we'll write a second route that will attempt to parse a POST body into our
`Person` type. Add the following to the bottom of your `app` declaration:

{% highlight haskell %}

  post "person" $ do
    maybePerson <- jsonBody
    case (maybePerson :: Maybe Person) of
      Nothing -> text "Failed to parse request body as Person"
      Just thePerson  -> text $ "Parsed: " <> pack (show thePerson)
      
{% endhighlight %}

Reload your project and start the server again. We'll need to make a POST
request to try out our new code. Here's a way of doing so using `curl`:

```
$ curl -H "Content-Type: application/json" -d '{ "name": "Bart", "age": 10 }' localhost:8080/person
Parsed: Person {name = "Bart", age = 10}
```

You can also try adding extra keys to your JSON object or removing name and
age and seeing what happens.

# Adding a Database

Now that we've seen how to do some simple REST-style requests, we'll add
a database to our application so we can provide proper API functionality.
We're going to be using the Persistent library and SQLite. The [Persistent
chapter](//www.yesodweb.com/book/persistent) in the Yesod Book covers
a lot of what we'll be using Persistent for and much more, so make sure to
refer to it while following this section.

To use Persistent, we're first going to add its dependencies to our
cabal file. Add these entries to your `build-depends` key:

```
                     , monad-logger
                     , persistent
                     , persistent-sqlite
                     , persistent-template
```

You'll have to restart GHCi so it can build these new dependencies.
Exit GHCi with `:quit` and run `stack ghci` again. Stack should
rebuild your project and show you the GHCi prompt.

We'll have to add quite a few more language extensions and imports.
At the top of your `Main.hs`, add the following lines below your current
extensions:

{% highlight haskell %}

{-# LANGUAGE EmptyDataDecls             #-}
{-# LANGUAGE FlexibleContexts           #-}
{-# LANGUAGE FlexibleInstances          #-}
{-# LANGUAGE GADTs                      #-}
{-# LANGUAGE GeneralizedNewtypeDeriving #-}
{-# LANGUAGE MultiParamTypeClasses      #-}
{-# LANGUAGE QuasiQuotes                #-}
{-# LANGUAGE TemplateHaskell            #-}
{-# LANGUAGE TypeFamilies               #-}

{% endhighlight %}

And these imports below your existing ones:

{% highlight haskell %}

import           Control.Monad.Logger    (LoggingT, runStdoutLoggingT)
import           Database.Persist        hiding (get) -- To avoid a naming clash with Web.Spock.get
import qualified Database.Persist        as P         -- We'll be using P.get later for GET /person/<id>.
import           Database.Persist.Sqlite hiding (get)
import           Database.Persist.TH

{% endhighlight %}

Next, we're going to replace our existing `Person` declaration with a
Persistent-specific one. This new one will generate code at compile time
that will allow us to serialise our datatype to SQLite with ease. Find and
remove your `data Person = ...` declaration along with `instance FromJSON`
and `instance ToJSON` and replace them with the following:

{% highlight haskell %}

share [mkPersist sqlSettings, mkMigrate "migrateAll"] [persistLowerCase|
Person json -- The json keyword will make Persistent generate sensible ToJSON and FromJSON instances for us.
  name Text
  age Int
  deriving Show
|]

{% endhighlight %}

This code will (among other things) generate record names for `Person`
that are slightly different to our initial ones: `name` becomes `personName`
and `age` becomes `personAge`. If you change your `Person` definitions in
your `get` function accordingly:

{% highlight haskell %}

    json
      [ Person {personName = "Fry", personAge = 25}
      , Person {personName = "Bender", personAge = 4}
      ]
      
{% endhighlight %}

Then your code should still compile and behave just as it did before the
change. 

## Creating the Connection Pool

Spock's [configuration]({{ page.hackage_base
}}/Spock-0.12.0.0/docs/Web-Spock-Config.html#t:SpockCfg) type includes
a database helper that manages connection pools.  Let's create a SQLite
connection pool and add it to our Spock configuration.  Replace the `spockCfg`
assignment (the `spockCfg <- ...` line) in your `main` function with the
following:

{% highlight haskell %}

  pool <- runStdoutLoggingT $ createSqlitePool "api.db" 5
  spockCfg <- defaultSpockCfg () (PCPool pool) ()

{% endhighlight %}

If we look at [createSqlitePool]({{ page.hackage_base
}}/persistent-sqlite-2.6.2/docs/Database-Persist-Sqlite.html#v:createSqlitePool)'s
type signature, we can see the `MonadLogger m` constraint. This means that
`createSqlitePool` needs to be in a monad that provides logging, so we run
it inside `runStdoutLoggingT` which will print the function's log messages to
standard output, allowing us to see its debug messages in our console.

Because we've changed the structure of our application to contain a
connection pool, we'll have to change our Spock application's type to reflect
this. Change your `app` type signature to:

{% highlight haskell %}

app :: SpockM SqlBackend () () ()

{% endhighlight %}

## Creating the Schema

Our new `Person` declaration also includes functionality for Persistent to
migrate our schema.  We'll call Persistent's `runMigration` function which
will create our schema for us. Insert this line right below the `spockCfg`
definition:

{% highlight haskell %}

  runStdoutLoggingT $ runSqlPool (do runMigration migrateAll) pool

{% endhighlight %}

The [runSqlPool]({{ page.hackage_base
}}/persistent-2.6.1/docs/src/Database-Persist-Sql-Run.html#runSqlPool)
function, which also needs to be in a `MonadLogger` monad, takes two arguments:
a function that performs actions using a connection from a pool (which here
is wrapped in parentheses), and the pool itself.

# Running Queries

We'll use a small helper function for running queries in our server.
Add the following function to your `Main.hs`:

{% highlight haskell %}

runSQL
  :: (HasSpock m, SpockConn m ~ SqlBackend)
  => SqlPersistT (LoggingT IO) a -> m a
runSQL action = runQuery $ \conn -> runStdoutLoggingT $ runSqlConn action conn

{% endhighlight %}

If you compare the right side of the `runSQL` definition to the migration code in the last
section, you'll see some similarities:

{% highlight haskell %}

  runQuery $ \conn -> runStdoutLoggingT $ runSqlConn action conn                       -- runSQL
                      runStdoutLoggingT $ runSqlPool (do runMigration migrateAll) pool -- migration

{% endhighlight %}

So, our `runSQL` function really just calls `runQuery` and uses the connection it provides
to perform some database actions.

## Adding People

Now we're ready to use our database in our application. Let's change our
POST /person function to insert the parsed `Person` into our database. In your
`post "person"` function, change the `Just thePerson` pattern match to the following:

{% highlight haskell %}

      Just thePerson  -> do
        runSQL $ insert thePerson
        text "Success!"

{% endhighlight %}

Reload your project and start the server again. Try adding a couple of people:

```
$ curl -H "Content-Type: application/json" -d '{ "name": "Walter", "age": 50 }' localhost:8080/person
Success!
$ curl -H "Content-Type: application/json" -d '{ "name": "Jesse", "age": 22 }' localhost:8080/person
Success!
```

## Listing People

Now we'll change our `get "person"` to return a list of all the people in our table:

{% highlight haskell %}

  get "person" $ do
    allPeople <- runSQL $ selectList [] [Asc PersonId]
    json allPeople

{% endhighlight %}

Reload and start the server and go to
[localhost:8080/person](//localhost:8080/person) to see your list.

## Getting a Specific Person

If you look at the JSON response for
[localhost:8080/person](//localhost:8080/person), you should see that your
`Person` objects now have `id` keys.  These values are automatically inserted
by Persistent; we'll use them to get a person by their id. Add the following
route function to your `app`:

{% highlight haskell %}

  get ("person" <//> var) $ \id' -> do
    mPerson <- runSQL $ P.get id'
    case (mPerson :: Maybe Person) of
      Nothing -> text ("Could not find a person with an id of " <> pack (show id'))
      Just thePerson -> json thePerson
      
{% endhighlight %}

And again, reload and try it out: [localhost:8080/person/1](//localhost:8080/person/1).

# Finishing Up

To finish up, try adding PUT and DELETE routes yourself.  You can probably
guess all the function names. Note that we hid `Database.Persist.delete`
to avoid a naming clash with `Web.Spock.delete`, so you'll have to use the
`P` namespace qualifier.

