# datagen

[![npm version](https://img.shields.io/npm/v/@jircik/datagen)](https://www.npmjs.com/package/@jircik/datagen)
[![license](https://img.shields.io/npm/l/@jircik/datagen)](LICENSE)
[![GitHub release](https://img.shields.io/github/v/release/jircik/DataGen-Cli-Tool)](https://github.com/jircik/DataGen-Cli-Tool/releases)

A CLI tool for developers to populate PostgreSQL and MongoDB databases with realistic fake data — fast, schema-driven, and reproducible.

Stop writing seed scripts by hand. Define your schema once and generate hundreds of rows in seconds.

> **Claude Code plugin available** — use Claude to set up schemas, populate tables, and manage datagen from any project.
> Install it from [Datagen-Claude-Plugin](https://github.com/jircik/DataGen-Claude-Plugin).

---

## Installation

**Requires Node.js 18+**

```bash
npm install -g @jircik/datagen
```

Verify it works:

```bash
datagen --help
```

### Contributing / running from source

```bash
git clone https://github.com/jircik/DataGen-Cli-Tool.git
cd datagen
npm install

# Run during development
npx tsx src/cli/index.ts --help

# Or link globally
npm run build && npm link
```

---

## Quick Start

```bash
# 1. Connect to your database
datagen connect "postgresql://user:pass@localhost:5432/mydb"

# 2. Create a schema file
mkdir .datagen && touch .datagen/users.schema.yaml

# 3. Populate
datagen populate .datagen/users.schema.yaml --count 50
```

---

## Commands

### `datagen connect <connection_string>`

Saves the connection and tests it before storing. Detects the database type automatically from the URI prefix.

```bash
datagen connect "postgresql://user:pass@localhost:5432/mydb"
datagen connect "mongodb://user:pass@localhost:27017/mydb?authSource=admin"

# Force a specific type
datagen connect "mydb://localhost:5432/mydb" --type postgres
```

---

### `datagen disconnect`

Clears the active connection.

```bash
datagen disconnect
```

---

### `datagen status`

Shows the currently saved connection.

```bash
datagen status
# ℹ Connected to PostgreSQL at localhost:5432/mydb
```

---

### `datagen populate`

The core command. Supports four modes.

**File mode** — populate from a schema file:
```bash
datagen populate .datagen/users.schema.yaml --count 100
```

**Folder mode** — populate all schemas in a directory, resolving dependency order automatically:
```bash
datagen populate .datagen/ --count 50
```

**Inline mode** — define fields directly, no schema file needed:
```bash
datagen populate --table users \
  --field "name:person.fullName" \
  --field "email:internet.email" \
  --field "age:number.int:min=18:max=80" \
  --count 50
```

**Override mode** — use a schema file as base and override specific fields inline:
```bash
datagen populate .datagen/users.schema.yaml --field "email:internet.url" --count 20
```

**Flags**

| Flag | Description | Default |
|---|---|---|
| `--count <n>` | Number of records to generate | `10` |
| `--table <name>` | Target table/collection (required for inline mode) | — |
| `--field <spec>` | Inline field definition, repeatable | — |
| `--dry-run` | Print records to stdout, skip insert | — |

---

### `datagen schema validate <file>`

Validates a schema file and reports any errors.

```bash
datagen schema validate .datagen/users.schema.yaml
# ✔ Valid schema: "users" (6 fields)
```

---

### `datagen schema list`

Lists all schema files in the current `.datagen/` folder.

```bash
datagen schema list
# ℹ .datagen/users.schema.yaml
# ℹ .datagen/posts.schema.yaml
```

---

### `datagen list tables`

Lists all tables (Postgres) or collections (MongoDB) in the connected database.

```bash
datagen list tables
# ℹ users
# ℹ posts
```

---

## Schema Files

Schema files live in your project's `.datagen/` folder and can be committed to version control so your whole team shares the same seed data.

### PostgreSQL

```yaml
# .datagen/users.schema.yaml
target: postgres
table: users
fields:
  id:
    type: string.uuid
    primary: true
  name: person.fullName
  email: internet.email
  age:
    type: number.int
    min: 18
    max: 80
  active:
    type: datatype.boolean
  created_at: date.past
```

### Relation fields (PostgreSQL)

When a field has `type: relation`, datagen fetches real IDs from the referenced table before inserting.

```yaml
# .datagen/posts.schema.yaml
target: postgres
table: posts
fields:
  id:
    type: string.uuid
    primary: true
  title: lorem.sentence
  body: lorem.paragraphs
  user_id:
    type: relation
    table: users
    field: id
    strategy: random   # or: sequential
```

| Strategy | Behavior |
|---|---|
| `random` | Picks a random existing ID for each row |
| `sequential` | Distributes IDs evenly across generated rows |

### MongoDB — nested objects and arrays

```yaml
# .datagen/orders.schema.yaml
target: mongo
collection: orders
fields:
  status:
    type: helpers.arrayElement
    values: ['pending', 'paid', 'shipped', 'cancelled']
  amount:
    type: number.float
    min: 10
    max: 9999
    precision: 2
  data:
    type: object
    fields:
      address: location.streetAddress
      city: location.city
      coordinates:
        type: object
        fields:
          lat: location.latitude
          lng: location.longitude
  tags:
    type: array
    items: commerce.productAdjective
    length: 3
  history:
    type: array
    length: 5
    items:
      type: object
      fields:
        event:
          type: helpers.arrayElement
          values: ['created', 'updated', 'shipped']
        timestamp: date.past
```

Fields can be written as a **shorthand string** (`name: person.fullName`) or as an **object** with options (`type`, `min`, `max`, etc.).

---

## Supported Field Types

All types map directly to [Faker.js](https://fakerjs.dev/api/) methods using the format `namespace.method`.

| Type | Example output |
|---|---|
| `person.fullName` | `"John Smith"` |
| `person.firstName` | `"John"` |
| `person.lastName` | `"Smith"` |
| `internet.email` | `"john@example.com"` |
| `internet.url` | `"https://example.com"` |
| `string.uuid` | `"a1b2c3d4-..."` |
| `number.int` + `min`/`max` | `42` |
| `number.float` + `min`/`max`/`precision` | `3.14` |
| `datatype.boolean` | `true` |
| `date.past` | ISO date string |
| `date.future` | ISO date string |
| `date.recent` | ISO date string |
| `lorem.sentence` | `"Lorem ipsum..."` |
| `lorem.paragraphs` | `"Lorem ipsum..."` |
| `location.city` | `"São Paulo"` |
| `location.country` | `"Brazil"` |
| `location.streetAddress` | `"123 Main St"` |
| `commerce.productName` | `"Awesome Chair"` |
| `helpers.arrayElement` + `values` | one of the array |

**MongoDB-only types:**

| Type | Description |
|---|---|
| `type: object` + `fields` | Nested document |
| `type: array` + `items` + `length` | Array of primitives or nested objects |

**Postgres-only types:**

| Type | Description |
|---|---|
| `type: relation` | Foreign key — fetches real IDs from the referenced table |

---

## Stack

| | |
|---|---|
| Language | TypeScript |
| Runtime | Node.js |
| CLI | Commander.js |
| Fake data | @faker-js/faker |
| Schema parsing | js-yaml + zod |
| Postgres | pg |
| MongoDB | mongodb |
| Terminal output | chalk + ora |

---

## Project Structure

```
src/
├── cli/          # Commands (connect, disconnect, status, populate, schema, list)
├── core/         # Config, schema parser, generator, schema resolver
├── drivers/      # Postgres and MongoDB drivers
└── utils/        # Logger, validator
.datagen/         # Your schema files (commit these)
```

---

## Releases

See [GitHub Releases](https://github.com/jircik/DataGen-Cli-Tool/releases) for the full changelog.

---

## Future Ideas

- Multi-database support — extend driver compatibility beyond PostgreSQL and MongoDB to include MySQL, SQLite, and other popular databases.
- `--seed <n>` flag for reproducible output
- `datagen export` — generate data to CSV/JSON without a database
- `datagen reset <table>` — truncate and re-populate
- Many-to-many relation support
- `--locale` flag for Faker.js locale (`pt_BR`, `en_US`, etc.)
- Schema inference from existing table (`datagen schema infer users`)