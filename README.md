[![Gitter](https://badges.gitter.im/rust-mysql/community.svg)](https://gitter.im/rust-mysql/community?utm_source=badge&utm_medium=badge&utm_campaign=pr-badge)

[![Build Status](https://dev.azure.com/aikorsky/mysql%20Rust/_apis/build/status/blackbeam.mysql_async?branchName=master)](https://dev.azure.com/aikorsky/mysql%20Rust/_build/latest?definitionId=2&branchName=master)
[![](https://meritbadge.herokuapp.com/mysql_async)](https://crates.io/crates/mysql_async)
[![](https://img.shields.io/crates/d/mysql_async.svg)](https://crates.io/crates/mysql_async)
[![API Documentation on docs.rs](https://docs.rs/mysql_async/badge.svg)](https://docs.rs/mysql_async)

# mysql_async

Tokio based asynchronous MySql client library for The Rust Programming Language.

## Installation

The library is hosted on [crates.io](https://crates.io/crates/mysql_async/).

```toml
[dependencies]
mysql_async = "<desired version>"
```

## Crate Features

Default feature set is wide – it includes all default [`mysql_common` features][myslqcommonfeatures]
as well as `native-tls`-based TLS support.

### List Of Features

*   `minimal` – enables only necessary features (at the moment the only necessary feature
    is `flate2` backend). Enables:

    -   `flate2/zlib"

    **Example:**

    ```toml
    [dependencies]
    mysql_async = { version = "*", default-features = false, features = ["minimal"]}
    ```

    **Note:* it is possible to use another `flate2` backend by directly choosing it:

    ```toml
    [dependencies]
    mysql_async = { version = "*", default-features = false }
    flate2 = { version = "*", default-features = false, features = ["rust_backend"] }
    ```

*   `default` – enables the following set of crate's and dependencies' features:

    -   `native-tls-tls`
    -   `flate2/zlib"
    -   `mysql_common/bigdecimal03`
    -   `mysql_common/rust_decimal`
    -   `mysql_common/time03`
    -   `mysql_common/uuid`
    -   `mysql_common/frunk`

*   `default-rustls` – same as default but with `rustls-tls` instead of `native-tls-tls`.

    **Example:**

    ```toml
    [dependencies]
    mysql_async = { version = "*", default-features = false, features = ["default-rustls"] }
    ```

*   `native-tls-tls` – enables `native-tls`-based TLS support _(conflicts with `rustls-tls`)_

    **Example:**

    ```toml
    [dependencies]
    mysql_async = { version = "*", default-features = false, features = ["native-tls-tls"] }

*   `rustls-tls` – enables `native-tls`-based TLS support _(conflicts with `native-tls-tls`)_

    **Example:**

    ```toml
    [dependencies]
    mysql_async = { version = "*", default-features = false, features = ["rustls-tls"] }

[myslqcommonfeatures]: https://github.com/blackbeam/rust_mysql_common#crate-features

## Example

```rust
use mysql_async::prelude::*;

#[derive(Debug, PartialEq, Eq, Clone)]
struct Payment {
    customer_id: i32,
    amount: i32,
    account_name: Option<String>,
}

#[tokio::main]
async fn main() -> Result<()> {
    let payments = vec![
        Payment { customer_id: 1, amount: 2, account_name: None },
        Payment { customer_id: 3, amount: 4, account_name: Some("foo".into()) },
        Payment { customer_id: 5, amount: 6, account_name: None },
        Payment { customer_id: 7, amount: 8, account_name: None },
        Payment { customer_id: 9, amount: 10, account_name: Some("bar".into()) },
    ];

    let database_url = /* ... */
    # get_opts();

    let pool = mysql_async::Pool::new(database_url);
    let mut conn = pool.get_conn().await?;

    // Create a temporary table
    r"CREATE TEMPORARY TABLE payment (
        customer_id int not null,
        amount int not null,
        account_name text
    )".ignore(&mut conn).await?;

    // Save payments
    r"INSERT INTO payment (customer_id, amount, account_name)
      VALUES (:customer_id, :amount, :account_name)"
        .with(payments.iter().map(|payment| params! {
            "customer_id" => payment.customer_id,
            "amount" => payment.amount,
            "account_name" => payment.account_name.as_ref(),
        }))
        .batch(&mut conn)
        .await?;

    // Load payments from the database. Type inference will work here.
    let loaded_payments = "SELECT customer_id, amount, account_name FROM payment"
        .with(())
        .map(&mut conn, |(customer_id, amount, account_name)| Payment { customer_id, amount, account_name })
        .await?;

    // Dropped connection will go to the pool
    drop(conn);

    // The Pool must be disconnected explicitly because
    // it's an asynchronous operation.
    pool.disconnect().await?;

    assert_eq!(loaded_payments, payments);

    // the async fn returns Result, so
    Ok(())
}
```

## Pool

The [`Pool`] structure is an asynchronous connection pool.

Please note:

* [`Pool`] is a smart pointer – each clone will point to the same pool instance.
* [`Pool`] is `Send + Sync + 'static` – feel free to pass it around.
* use [`Pool::disconnect`] to gracefuly close the pool.
* [`Pool::new`] is lazy and won't assert server availability.

## Transaction

[`Conn::start_transaction`] is a wrapper, that starts with `START TRANSACTION`
and ends with `COMMIT` or `ROLLBACK`.

Dropped transaction will be implicitly rolled back if it wasn't explicitly
committed or rolled back. Note that this behaviour will be triggered by a pool
(on conn drop) or by the next query, i.e. may be delayed.

API won't allow you to run nested transactions because some statements causes
an implicit commit (`START TRANSACTION` is one of them), so this behavior
is chosen as less error prone.

## `Value`

This enumeration represents the raw value of a MySql cell. Library offers conversion between
`Value` and different rust types via `FromValue` trait described below.

### `FromValue` trait

This trait is reexported from **mysql_common** create. Please refer to its
[crate docs](https://docs.rs/mysql_common) for the list of supported conversions.

Trait offers conversion in two flavours:

*   `from_value(Value) -> T` - convenient, but panicking conversion.

    Note, that for any variant of `Value` there exist a type, that fully covers its domain,
    i.e. for any variant of `Value` there exist `T: FromValue` such that `from_value` will never
    panic. This means, that if your database schema is known, than it's possible to write your
    application using only `from_value` with no fear of runtime panic.

    Also note, that some convertions may fail even though the type seem sufficient,
    e.g. in case of invalid dates (see [sql mode](https://dev.mysql.com/doc/refman/8.0/en/sql-mode.html)).

*   `from_value_opt(Value) -> Option<T>` - non-panicking, but less convenient conversion.

    This function is useful to probe conversion in cases, where source database schema
    is unknown.

## MySql query protocols

### Text protocol

MySql text protocol is implemented in the set of `Queryable::query*` methods
and in the [`prelude::Query`] trait if query is [`prelude::AsQuery`].
It's useful when your query doesn't have parameters.

**Note:** All values of a text protocol result set will be encoded as strings by the server,
so `from_value` conversion may lead to additional parsing costs.

### Binary protocol and prepared statements.

MySql binary protocol is implemented in the set of `exec*` methods,
defined on the [`prelude::Queryable`] trait and in the [`prelude::Query`]
trait if query is [`QueryWithParams`]. Prepared statements is the only way to
pass rust value to the MySql server. MySql uses `?` symbol as a parameter placeholder.

**Note:** it's only possible to use parameters where a single MySql value
is expected, i.e. you can't execute something like `SELECT ... WHERE id IN ?`
with a vector as a parameter. You'll need to build a query that looks like
`SELECT ... WHERE id IN (?, ?, ...)` and to pass each vector element as
a parameter.

## Named parameters

MySql itself doesn't have named parameters support, so it's implemented on the client side.
One should use `:name` as a placeholder syntax for a named parameter. Named parameters uses
the following naming convention:

* parameter name must start with either `_` or `a..z`
* parameter name may continue with `_`, `a..z` and `0..9`

**Note:** this rules mean that, say, the statment `SELECT :fooBar` will be translated
to `SELECT ?Bar` so please be careful.

Named parameters may be repeated within the statement, e.g `SELECT :foo, :foo` will require
a single named parameter `foo` that will be repeated on the corresponding positions during
statement execution.

One should use the `params!` macro to build parameters for execution.

**Note:** Positional and named parameters can't be mixed within the single statement.

## Statements

In MySql each prepared statement belongs to a particular connection and can't be executed
on another connection. Trying to do so will lead to an error. The driver won't tie statement
to its connection in any way, but one can look on to the connection id, contained
in the [`Statement`] structure.

## LOCAL INFILE Handlers

**Warning:** You should be aware of [Security Considerations for LOAD DATA LOCAL][1].

There are two flavors of LOCAL INFILE handlers – _global_ and _local_.

I case of a LOCAL INFILE request from the server the driver will try to find a handler for it:

1.  It'll try to use _local_ handler installed on the connection, if any;
2.  It'll try to use _global_ handler, specified via [`OptsBuilder::local_infile_handler`],
    if any;
3.  It will emit [`LocalInfileError::NoHandler`] if no handlers found.

The purpose of a handler (_local_ or _global_) is to return [`InfileData`].

### _Global_ LOCAL INFILE handler

See [`prelude::GlobalHandler`].

Simply speaking the _global_ handler is an async function that takes a file name (as `&[u8]`)
and returns `Result<InfileData>`.

You can set it up using [`OptsBuilder::local_infile_handler`]. Server will use it if there is no
_local_ handler installed for the connection. This handler might be called multiple times.

Examles:

1.  [`WhiteListFsHandler`] is a _global_ handler.
2.  Every `T: Fn(&[u8]) -> BoxFuture<'static, Result<InfileData, LocalInfileError>>`
    is a _global_ handler.

### _Local_ LOCAL INFILE handler.

Simply speaking the _local_ handler is a future, that returns `Result<InfileData>`.

This is a one-time handler – it's consumed after use. You can set it up using
[`Conn::set_infile_handler`]. This handler have priority over _global_ handler.

Worth noting:

1.  `impl Drop for Conn` will clear _local_ handler, i.e. handler will be removed when
    connection is returned to a `Pool`.
2.  [`Conn::reset`] will clear _local_ handler.

Example:

```rust
#
let pool = mysql_async::Pool::new(database_url);

let mut conn = pool.get_conn().await?;
"CREATE TEMPORARY TABLE tmp (id INT, val TEXT)".ignore(&mut conn).await?;

// We are going to call `LOAD DATA LOCAL` so let's setup a one-time handler.
conn.set_infile_handler(async move {
    // We need to return a stream of `io::Result<Bytes>`
    Ok(stream::iter([Bytes::from("1,a\r\n"), Bytes::from("2,b\r\n3,c")]).map(Ok).boxed())
});

let result = r#"LOAD DATA LOCAL INFILE 'whatever'
    INTO TABLE `tmp`
    FIELDS TERMINATED BY ',' ENCLOSED BY '\"'
    LINES TERMINATED BY '\r\n'"#.ignore(&mut conn).await;

match result {
    Ok(()) => (),
    Err(Error::Server(ref err)) if err.code == 1148 => {
        // The used command is not allowed with this MySQL version
        return Ok(());
    },
    Err(Error::Server(ref err)) if err.code == 3948 => {
        // Loading local data is disabled;
        // this must be enabled on both the client and the server
        return Ok(());
    }
    e @ Err(_) => e.unwrap(),
}

// Now let's verify the result
let result: Vec<(u32, String)> = conn.query("SELECT * FROM tmp ORDER BY id ASC").await?;
assert_eq!(
    result,
    vec![(1, "a".into()), (2, "b".into()), (3, "c".into())]
);

drop(conn);
pool.disconnect().await?;
```

[1]: https://dev.mysql.com/doc/refman/8.0/en/load-data-local-security.html

## Testing

Tests uses followin environment variables:
* `DATABASE_URL` – defaults to `mysql://root:password@127.0.0.1:3307/mysql`
* `COMPRESS` – set to `1` or `true` to enable compression for tests
* `SSL` – set to `1` or `true` to enable TLS for tests

You can run a test server using doker. Please note that params related
to max allowed packet, local-infile and binary logging are required
to properly run tests (please refer to `azure-pipelines.yml`):

```sh
docker run -d --name container \
    -v `pwd`:/root \
    -p 3307:3306 \
    -e MYSQL_ROOT_PASSWORD=password \
    mysql:8.0 \
    --max-allowed-packet=36700160 \
    --local-infile \
    --log-bin=mysql-bin \
    --log-slave-updates \
    --gtid_mode=ON \
    --enforce_gtid_consistency=ON \
    --server-id=1
```


## Change log

Available [here](https://github.com/blackbeam/mysql_async/releases)

## License

Licensed under either of

* Apache License, Version 2.0, ([LICENSE-APACHE](LICENSE-APACHE) or https://www.apache.org/licenses/LICENSE-2.0)
* MIT license ([LICENSE-MIT](LICENSE-MIT) or https://opensource.org/licenses/MIT)

at your option.

### Contribution

Unless you explicitly state otherwise, any contribution intentionally
submitted for inclusion in the work by you, as defined in the Apache-2.0
license, shall be dual licensed as above, without any additional terms or
conditions.
