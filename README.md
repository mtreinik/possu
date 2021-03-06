# Possu 🐖

![CI](https://github.com/sluukkonen/possu/workflows/CI/badge.svg)

A small companion library for [node-postgres](https://node-postgres.com/).

## Features & Goals

- A Promise-based API which aims to make common operations easy
- Write raw SQL queries with tagged template strings
- Support nested queries
- Transaction handling
- First-class TypeScript support
- Not a framework. [node-postgres](https://node-postgres.com) already handles
  things like connection pooling for us, so you can integrate Possu easily to
  an existing application.

## TODO:

- More query builder features (e.g. arrays, unnesting)
- Customization of transaction modes
- Savepoints
- Automatic transaction retrying (perhaps)

## Getting started

```
$ npm install possu
```

```typescript
import { sql, queryOne } from 'possu'
import { Pool } from 'pg'

const pool = new Pool({ ... })
const pet = await queryOne(pool, sql`SELECT name FROM pet WHERE id = ${id}`)
```

## API

- Building queries
  - [sql](#sql)
  - [sql.identifier](#sql.identifier)
- Executing queries
  - [query](#query)
  - [queryOne](#queryOne)
  - [queryMaybeOne](#queryMaybeOne)
  - [execute](#execute)
- Transactions
  - [transaction](#transaction)

## sql

Create an SQL query.

This is the only way to create queries in Possu. Other Possu functions check
at runtime that the query has been created with `sql`.

```typescript
const query = sql`SELECT * FROM pet WHERE id = ${1}`
// => { text: 'SELECT * FROM pet WHERE id = $1', values: [1] }
```

Queries may also be nested. This is a powerful mechanism for code reuse.

```typescript
const query = sql`SELECT * FROM pet WHERE id = ${1}`
const exists = sql`SELECT exists(${query})`
// => { text: 'SELECT exists(SELECT * FROM pet WHERE id = $1)', values: [1] }
```

## sql.identifier

Escape an SQL
[identifier](https://www.postgresql.org/docs/current/sql-syntax-lexical.html#SQL-SYNTAX-IDENTIFIERS)
to be used in a query. This is sometimes necessary when the name of a table
or a column is a
[keyword](https://www.postgresql.org/docs/current/sql-keywords-appendix.html).
It can also be used to create queries which are parametrized by table or
column names.

```typescript
sql`SELECT * FROM ${sql.identifier('pet')}`
// => { text: 'SELECT * FROM "pet"', values: [] }
```

```typescript
sql`SELECT * FROM pet ORDER BY ${sql.identifier('name')} DESC`
// => { text: 'SELECT * FROM pet ORDER BY "name" DESC', values: [] }
```

### query

Execute a `SELECT` or other query that returns zero or more rows.

Returns all rows.

```typescript
const pets = await query(pool, sql`SELECT * FROM pet`)
// => [{ id: 1, name: 'Iiris', id: 2: name: 'Jean' }]
```

If selecting a single column, each result row is unwrapped automatically.

```typescript
const names = await query(client, sql`SELECT name FROM pet`)
// => ['Iiris', 'Jean']
```

### queryOne

Execute a `SELECT` or other query that returns exactly one row.

Returns the first row.

- Throws a `NoRowsReturnedError` if query returns no rows.
- Throws a `TooManyRowsReturnedError` if query returns more than 1 row.

```typescript
const pet = await queryOne(pool, sql`SELECT * FROM pet WHERE id = 1`)
// => { id: 1, name: 'Iiris' }
```

If selecting a single column, it is unwrapped automatically.

```typescript
const name = await queryOne(client, sql`SELECT name FROM pet WHERE id = 1`)
// => 'Iiris'
```

### queryMaybeOne

Execute a `SELECT` or other query that returns zero or one rows.

Returns the first row or `undefined`.

- Throws a `TooManyRowsReturnedError` if query returns more than 1 row.

```typescript
const pet = await queryMaybeOne(pool, sql`SELECT * FROM pet WHERE id = 1`)
// => { id: 1, name: 'Iiris' }

const nothing = await queryMaybeOne(client, sql`SELECT * FROM pet WHERE false`)
// => undefined
```

If selecting a single column, it is unwrapped automatically.

```typescript
const name = await queryMaybeOne(pool, sql`SELECT name FROM pet WHERE id = 1`)
// => 'Iiris'
```

### execute

Execute an `INSERT`, `UPDATE`, `DELETE` or other query that is not expected to return any rows.

Returns the number of rows affected.

```typescript
const name = await execute(pool, sql`INSERT INTO pet (name) VALUES ('Fae')`)
// => 1
```

### transaction

Execute a function within a transaction.

Start a transaction and execute a set of queries within it. If the function
returns a resolved promise, the transaction is committed. Returns the value
returned from the function.

If the function returns a rejected Promise or throws any kind of error, the
transaction is rolled back and the error is rethrown.

```typescript
const petCount = await transaction(pool, async (tx) => {
  await execute(tx, sql`INSERT INTO pet (name) VALUES ('Senna')`)
  const count = await queryOne(tx, sql`SELECT count(*) FROM pet`)
  if (count > 5) {
    throw new Error('You have too many pets already!')
  }
  return count
})
```

## Error handling

All errors thrown by possu are subclasses of `PossuError`, so you can detect them with `instanceof`.
