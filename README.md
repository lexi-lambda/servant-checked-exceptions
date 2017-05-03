
Servant.Checked.Exceptions
==========================

[![Build Status](https://secure.travis-ci.org/cdepillabout/servant-checked-exceptions.svg)](http://travis-ci.org/cdepillabout/servant-checked-exceptions)
[![Hackage](https://img.shields.io/hackage/v/servant-checked-exceptions.svg)](https://hackage.haskell.org/package/servant-checked-exceptions)
[![Stackage LTS](http://stackage.org/package/servant-checked-exceptions/badge/lts)](http://stackage.org/lts/package/servant-checked-exceptions)
[![Stackage Nightly](http://stackage.org/package/servant-checked-exceptions/badge/nightly)](http://stackage.org/nightly/package/servant-checked-exceptions)
![BSD3 license](https://img.shields.io/badge/license-BSD3-blue.svg)

`servant-checked-exceptions` provides a way to specify errors thrown by a
Servant api on the type level.  It allows easy composition between different
error types.

`servant-checked-exceptions` provides the
[`Throws`](https://hackage.haskell.org/package/servant-checked-exceptions/docs/Servant-Checked-Exceptions.html#t:Throws)
data type to signify which errors can be thrown by an api.  For instance,
imagine a `getAuthor` api that returns an `Author` based on an `AuthorId`:

```haskell
-- This is a servant-compatible type describing our api.
type Api =
  "author" :>
  Capture "author-id" AuthorId :>
  Throws CouldNotConnectToDbError :>
  Throws AuthorNotFoundError :>
  Get '[JSON] Author

-- These are the two errors that can be thrown:
data CouldNotConnectToDbError = CouldNotConnectToDbError
data AuthorNotFoundError = AuthorNotFoundError
```

The corresponding handler function uses the
[`Envelope`](https://hackage.haskell.org/package/servant-checked-exceptions/docs/Servant-Checked-Exceptions.html#t:Envelope)
data type to model the possibility of returning an `Author` successfully, or
either `CouldNotConnectToDbError` or `AuthorNotFoundError` unsuccessfully.
Internally, `Envelope` is using an open sum-type to easily represent multiple
different errors:

```haskell
getAuthorHandler
  :: AuthorId
  -> Handler (Envelope '[DatabaseError, AuthorNotFoundError] Author)
getAuthorHandler authorId = ...
```

For more documentation and usage examples, see the
[documentation](https://hackage.haskell.org/package/servant-checked-exceptions) on Hackage.

## Why would I want to use this?

Using `Envelope` with its open sum-type to represent errors gives us an easy
way to reuse errors on multiple routes.

For instance, imagine that we had another api for updating an author's name,
given the author's ID.  Using `Throws` and `Envelope`, it might look like this:

```haskell
type Api =
  "update-author-name" :>
  Capture "author-id" AuthorId :>
  Capture "author-name" AuthorName :>
  Throws CouldNotConnectToDbError :>
  Throws AuthorNotFoundError :>
  Throws AuthorNameTooShort :>
  Post '[JSON] Author

data AuthorNameTooShort = AuthorNameTooShort

postChangeAuthorName
  :: AuthorId
  -> AuthorName
  -> Handler (Envelope '[DatabaseError, AuthorNotFoundError, AuthorNameTooShort] Author)
postChangeAuthorName authorId newAuthorName = ...
```

We are able to reuse the `DatabaseError` and `AuthorNotFoundError`.  If we try
to return an error that is not declared using `Throws`, GHC will give us an
error.  We get flexiblity and type-safety.

When using [servant-docs](https://hackage.haskell.org/package/servant-docs) to
create documentation, only one instance of `ToSample` needs to be created for
each error (`DatabaseError`, `AuthorNotFoundError`, and `AuthorNameTooShort`).
Multiple instances of `ToSample` do not need to be created for _every_
different `Envelope` used in a handler.

## Example

This repository contains an [example](example/) of using
`servant-checked-exceptions`.  This includes an [api](example/Api.hs),
[server](example/Server.hs), [client](example/Client.hs), and
[documentation](example/Docs.hs).

Below I show how to compile and run these examples.

### Compile

The examples can be compiled by using the `buildexample` flag:

```sh
$ stack build --flag servant-checked-exceptions:buildexample
```

This creates three executables.  A server, a client, and a documentaiton
generator.

### Run the server

The server is a small example that will take search queries and return results.
The server can be run with the following command:

```sh
$ stack exec -- servant-checked-exceptions-example-server
```

This runs the server on port 8201.  Here is an example of using `curl` to
access the server.  This will send the query `hello`:

```sh
$ curl \
    --request POST \
    --header 'Accept: application/json' \
    'http://localhost:8201/lax-search/hello'
"good"
```

If you try to send a query that is not `hello`, the server will return an error:

```sh
$ curl \
    --request POST \
    --header 'Accept: application/json' \
    'http://localhost:8201/lax-search/hello'
"BadSearchTermErr"
```

There is also a strict api, that requires `hello` to be capitalized like `Hello`:

```sh
$ curl \
    --request POST \
    --header 'Accept: application/json' \
    'http://localhost:8201/strict-search/hello'
"IncorrectCapitalization"
$ curl \
    --request POST \
    --header 'Accept: application/json' \
    'http://localhost:8201/strict-search/Hello'
"good"
```

### Run the client

The client provides a small command line application to query the server.  In
order to use the client, the server must be running.

Use the client to access the lax search api:

```sh
$ stack exec -- servant-checked-exceptions-example-client foobar
the search term was not "Hello"
$ stack exec -- servant-checked-exceptions-example-client hello
Success: good
```

Use the client to access the strict search api:

```sh
$ stack exec -- servant-checked-exceptions-example-client --strict hello
the search term was not capitalized correctly
$ stack exec -- servant-checked-exceptions-example-client --strict Hello
Success: good
```

### Run the documentation generator

The documentation generator will generate documentation for the api in Markdown:

```sh
$ stack exec -- servant-checked-exceptions-example-docs
```

Here is a small example of the documentation that will be generated for the lax
search api:

```markdown
## POST /lax-search/:query

#### Captures:

- *query*: a search string like "hello" or "bye"

#### Response:

- Status code 200
- Headers: []

- Supported content types are:

    - `application/json`

- This is a successful response.

"good"

- a completely incorrect search term was used

"BadSearchTermErr"
```

You can see that both the success and error responses are documented.
