---
title: "Cloudflare D1: Building a REST API for Movies at the Edge"
meta_title: "Cloudflare D1: Building a REST API for Movies at the Edge"
description: "Learn how to build a fully functional movie REST API using Cloudflare Workers, D1 (edge SQLite), and Hono — from schema migrations to global deployment."
date: 2026-04-19
image: "../../assets/images/cloudflare-worker.png"
authors: ["hieupn"]
categories: ["Web Development"]
tags: ["cloudflare", "d1", "sqlite", "hono", "typescript", "edge computing"]
series: ["Cloudflare Workers", "3"]
---

With about 35 lines of TypeScript, two SQL migration files, and a single binding declaration, you can ship a fully functional REST API backed by a relational database — deployed globally, no servers to manage. This post walks through the `d1-movie` project, which uses Cloudflare Workers, **Cloudflare D1**, and the [Hono](https://hono.dev/) router to expose a small movie catalog API at the edge.

## What Is Cloudflare D1?

Cloudflare D1 is Cloudflare's serverless relational database service, built on SQLite. Unlike traditional databases tied to a single region, D1 runs alongside your Worker on Cloudflare's edge network, which means queries execute close to users rather than traversing back to a distant data center. D1 supports standard SQL syntax, numbered schema migrations managed through Wrangler, and plugs directly into the Workers binding model — no SDK installation required.

## Why Developers Use D1

- No servers to provision or connection pools to tune.
- Standard SQL syntax — no new ORM or query builder to learn.
- Schema history is tracked through numbered `.sql` files under `migrations/`.
- The binding model exposes the database on `c.env` as a typed `D1Database` object.
- The free tier is sufficient for personal projects and low-traffic internal APIs.

## Key Concepts

### D1 Database and Bindings

Just like a KV namespace, a D1 database is created via Wrangler and declared in `wrangler.jsonc` under the `d1_databases` key. The `binding` field determines the property name on the `env` object inside your Worker. In this project the binding is named `movie_example`, so the database is accessed as `c.env.movie_example` in every route handler.

### Numbered Migrations

D1 manages schema changes through plain SQL files placed in the `migrations/` directory. Each file follows the naming convention `NNNN_description.sql`. Wrangler applies them in ascending numeric order when you run `wrangler d1 migrations apply`. This keeps the schema consistent across local, staging, and production environments without any additional tooling.

### Prepared Statements

All queries go through D1's prepared-statement API. `.prepare()` accepts a SQL string, `.bind()` fills in positional `?` parameters (preventing SQL injection), and `.all()` or `.run()` executes the query. Read queries return an object whose `results` field contains the array of matching rows; write queries return metadata about affected rows.

### Hono as the HTTP Router

[Hono](https://hono.dev/) is a lightweight, TypeScript-first router that runs on any JavaScript runtime including Cloudflare Workers. Passing the `Bindings: Env` generic to `new Hono<{ Bindings: Env }>()` lets TypeScript infer the exact type of `c.env.movie_example` as `D1Database`, giving you autocomplete and compile-time checks across every route.

## How It Works

1. An HTTP request arrives at a route such as `GET /movies` or `PUT /movies/:id`.
2. Hono matches the route and invokes the corresponding handler function.
3. The handler calls `c.env.movie_example.prepare(...)` to build a SQL query.
4. For reads (`SELECT`), `.all()` returns the result rows and the Worker responds with JSON.
5. For writes (`UPDATE`), `.run()` executes the statement and the Worker returns a success message.

## Getting Started

You need Node.js 18+, a Cloudflare account, and npm.

```bash
# Scaffold a new Workers project
npm create cloudflare@latest -- d1-movie
# Choose: Hello World → Worker only → TypeScript → No deploy

cd d1-movie

# Install Hono
npm install hono

# Authenticate Wrangler with your Cloudflare account
npx wrangler login

# Create the D1 database — copy the printed database_id into wrangler.jsonc
npx wrangler d1 create movie-example

# Generate TypeScript types for your bindings
npm run cf-typegen

# Apply migrations against the local SQLite database
npx wrangler d1 migrations apply movie-example --local

# Start the local dev server at http://localhost:8787
npm run dev
```

Use `--local` so migrations run against the local SQLite file during development. When you are ready for production, drop the flag to apply them to D1 on Cloudflare's network.

## Example: D1 Binding in wrangler.jsonc

Before writing any Worker code, declare the database in `wrangler.jsonc`. Three fields are required: `binding` sets the property name on `env`, `database_name` is what you passed to `wrangler d1 create`, and `database_id` is the UUID Cloudflare generated for that database.

```jsonc
{
  "name": "d1-movie",
  "main": "src/index.ts",
  "compatibility_date": "2026-04-19",
  "compatibility_flags": ["nodejs_compat"],
  "d1_databases": [
    {
      "binding": "movie_example",
      "database_name": "movie-example",
      "database_id": "YOUR_D1_DATABASE_ID"
    }
  ]
}
```

## Example: SQL Migrations

Two migration files define the schema and seed the database. File `0001` creates the `movies` table with four columns; file `0002` inserts five classic films. Wrangler applies them in ascending order of their numeric prefix.

```sql
-- migrations/0001_create-tables.sql
DROP TABLE IF EXISTS movies;
CREATE TABLE movies (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    title TEXT NOT NULL,
    release_date TEXT NOT NULL,
    rating REAL NOT NULL
);

-- migrations/0002_insert-movies.sql
INSERT INTO movies (title, release_date, rating) VALUES
('The Shawshank Redemption', '1994-09-23', 6),
('The Godfather', '1972-03-24', 10),
('The Dark Knight', '2008-07-18', 7),
('The Godfather: Part II', '1974-12-20', 9),
('12 Angry Men', '1957-04-10', 8);
```

## Example: The Complete Worker — src/index.ts

All API logic lives in a single file. Notice how the `Env` interface maps the `movie_example` binding to the `D1Database` type, and how each route uses `.bind()` to pass parameters safely rather than interpolating values directly into the SQL string.

```typescript
import { Hono } from "hono";

export interface Env {
  movie_example: D1Database;
}

const app = new Hono<{ Bindings: Env }>();

app.get("/", (c) => {
  return c.json({ message: "Welcome to the D1 Movie API!" });
});

// Return every movie in the database
app.get("/movies", async (c) => {
  const { results } = await c.env.movie_example
    .prepare("SELECT * FROM movies")
    .all();
  return c.json(results);
});

// Return the top 3 highest-rated movies
app.get("/favorites", async (c) => {
  const { results } = await c.env.movie_example
    .prepare("SELECT * FROM movies ORDER BY rating DESC LIMIT 3")
    .all();
  return c.json(results);
});

// Update a movie's rating by ID
app.put("/movies/:id", async (c) => {
  const id = c.req.param("id");
  const { rating } = await c.req.json();
  await c.env.movie_example
    .prepare("UPDATE movies SET rating = ? WHERE id = ?")
    .bind(rating, id)
    .run();
  return c.json({ message: "Movie rating updated successfully" });
});

export default app;
```

## Deploying to Cloudflare

Before running `deploy`, apply the migrations to the production D1 database:

```bash
# Apply migrations to D1 on Cloudflare's network (no --local flag)
npx wrangler d1 migrations apply movie-example

# Deploy the Worker
npm run deploy
```

Wrangler compiles the TypeScript, bundles the output, and prints a live URL like `https://d1-movie.<your-subdomain>.workers.dev`.

## When To Use D1

D1 is a good fit when:

- You need a relational database but want to avoid the cost and complexity of a managed PostgreSQL or MySQL service.
- Your data has a clear structure and you want to query it with standard SQL.
- The workload is read-heavy — product catalogs, content listings, application configuration.
- You are already on Workers and want to keep the entire stack inside the Cloudflare ecosystem.

D1 is less suitable for very high-write workloads such as real-time logging systems, or when you need advanced SQL features that SQLite does not support (complex stored procedures, ranked full-text search, and similar).

## Final Thoughts

The `d1-movie` project demonstrates just how lean a D1-backed API can be: one TypeScript file, two SQL migrations, and a binding declaration in `wrangler.jsonc` — that is the entire footprint. No server provisioning, no connection-pool configuration, no scaling rules to write. The same pattern scales cleanly to todo lists, inventory APIs, or any application that needs structured data stored at the edge.
