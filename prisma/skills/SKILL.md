---
name: prisma-clean-architecture
description: Universal Prisma ORM clean architecture, schema design, type-safe queries, migration workflows, and database performance skill. Directs the creation of production-ready, type-safe, and highly performant database interactions avoiding common ORM anti-patterns (such as N+1 queries, unindexed filters, and connection pool exhaustion). Covers model definitions, relation loading, transactional isolation, pagination strategies, and Prisma Client instantiation across any Node.js, NestJS, or TypeScript backend. Triggers on "prisma", "prisma schema", "prisma client", "prisma migrate", "database query", "prisma relation", "orm".
license: MIT
metadata:
  author: lizdev
  version: "1.0.0"
---

# CORE DATABASE AND SCHEMA DESIGN PRINCIPLES

## Declarative Data Modeling Conventions
When authoring `schema.prisma`, strict naming conventions ensure architectural clarity and decouple TypeScript domain entities from physical database schemas.
1. Model Naming: Use singular PascalCase for model names (e.g., `User`, `OrderItem`, `PaymentTransaction`).
2. Field Naming: Use camelCase for field names (e.g., `emailAddress`, `createdAt`, `authorId`).
3. Database Mapping: Map entities to snake_case database tables and columns using `@@map` and `@map` directives. This preserves idiomatic JavaScript/TypeScript camelCase ergonomics while keeping SQL queries and database schemas compliant with PostgreSQL conventions.

```prisma
model UserProfile {
  id        String   @id @default(uuid()) @db.Uuid
  userId    String   @unique @map("user_id") @db.Uuid
  bio       String?  @db.Text
  createdAt DateTime @default(now()) @map("created_at") @db.Timestamptz

  user User @relation(fields: [userId], references: [id], onDelete: Cascade)

  @@map("user_profiles")
}
```

## Explicit Relation Definitions
Prisma supports three primary relational patterns. Relations must be explicitly defined with foreign keys and cascade rules.
1. One-to-One (1-1): The dependent model holds the foreign key scalar marked with `@unique`. The principal model references the optional relation.
2. One-to-Many (1-N): The child model holds the foreign key scalar and `@relation` attribute. The parent model holds a list field (e.g., `posts Post[]`).
3. Many-to-Many (N-M): Explicit join tables are strongly preferred over implicit join tables for production systems. Explicit join models (e.g., `PostTag`) allow audit fields (e.g., `assignedAt`, `assignedBy`) and explicit composite primary keys `@@id([postId, tagId])`.

```prisma
// Explicit Many-to-Many Join Table Pattern
model Post {
  id    String    @id @default(uuid()) @db.Uuid
  title String    @db.VarChar(255)
  tags  PostTag[]

  @@map("posts")
}

model Tag {
  id    String    @id @default(uuid()) @db.Uuid
  name  String    @unique @db.VarChar(50)
  posts PostTag[]

  @@map("tags")
}

model PostTag {
  postId     String   @map("post_id") @db.Uuid
  tagId      String   @map("tag_id") @db.Uuid
  assignedAt DateTime @default(now()) @map("assigned_at") @db.Timestamptz

  post Post @relation(fields: [postId], references: [id], onDelete: Cascade)
  tag  Tag  @relation(fields: [tagId], references: [id], onDelete: Cascade)

  @@id([postId, tagId])
  @@index([tagId])
  @@map("posts_tags")
}
```

## Indexing Strategies
Indexes are critical for maintaining query performance as datasets scale.
1. Foreign Keys: Always define an `@@index` on foreign key fields to optimize join execution and cascade deletions.
2. Unique Constraints: Use `@unique` for single-field uniqueness or `@@unique([tenantId, email])` for compound uniqueness across multi-tenant boundaries.
3. Composite Indexes: Group frequently filtered and sorted columns into composite indexes (`@@index([status, createdAt])`) matching query access patterns.

## Native Database Types and Enums
Use native database type attributes to control physical storage requirements.
1. String Types: Use `@db.VarChar(length)` for bounded strings, `@db.Text` for unbounded content, and `@db.Uuid` for UUID identifiers.
2. Temporal Types: Use `@db.Timestamptz` for timezone-aware timestamps.
3. Enums: Prefer native PostgreSQL enums (`enum UserRole { ... }`) over string constants to enforce database-level data integrity.

---

# CLIENT INITIALIZATION AND CONNECTION MANAGEMENT

## Singleton Pattern for PrismaClient
Instantiating multiple `PrismaClient` instances leads to connection pool exhaustion and database instability (`P1001` errors). In Node.js environments—especially during local development with hot module reloading (HMR) or inside serverless function deployments—you must enforce a strict singleton instance attached to `globalThis`.

```typescript
// src/lib/prisma.ts
import { PrismaClient } from "@prisma/client";

const globalForPrisma = globalThis as unknown as {
  prisma: PrismaClient | undefined;
};

export const prisma =
  globalForPrisma.prisma ??
  new PrismaClient({
    log:
      process.env.NODE_ENV === "development"
        ? ["query", "error", "warn"]
        : ["error"],
  });

if (process.env.NODE_ENV !== "production") {
  globalForPrisma.prisma = prisma;
}
```

## Connection Lifecycle and Graceful Shutdown
Prisma Client connects lazily on the first query execution. However, long-running services (NestJS, Express, Fastify) must handle SIGINT and SIGTERM OS signals to gracefully disconnect the pool and flush pending queries.

```typescript
// Graceful shutdown handling
async function shutdown() {
  console.log("Shutting down database connection pool...");
  await prisma.$disconnect();
  process.exit(0);
}

process.on("SIGINT", shutdown);
process.on("SIGTERM", shutdown);
```

---

# WORKFLOW: SCHEMA EVOLUTION AND MIGRATIONS

Engineers and AI assistants must follow this sequential 4-step workflow when making database changes.

```
[Step 1: Schema Authoring] ──> [Step 2: Local Prototype/Validate] ──> [Step 3: Migration Generation] ──> [Step 4: CI/CD Deployment]
```

## Step 1: Schema Authoring
Modify `schema.prisma` with explicit model attributes, native types, and relations.

## Step 2: Local Prototyping
For initial local exploration where data loss is acceptable, run `prisma db push` to synchronize the local database schema without generating migration files. Never use `db push` in staging or production.

## Step 3: Migration Generation
Run `prisma migrate dev --name describe_change`. This command:
1. Generates a new timestamped SQL migration file in `prisma/migrations/`.
2. Applies the SQL migration to the development database.
3. Triggers `prisma generate` to update Prisma Client TypeScript definitions.

## Step 4: CI/CD Production Deployment
In continuous integration and production pipelines, execute `prisma migrate deploy`. This reads existing migration SQL files and applies pending migrations deterministically without resetting databases or generating new files.

## Safe Non-Nullable Column Additions
Adding a required non-nullable column to an existing table containing data causes migration failures. Use a 3-phase deployment strategy:
1. Phase 1: Add the field as optional (`field String?`) or provide a temporary `@default("value")` attribute. Generate and deploy the migration.
2. Phase 2: Run a backfill script to populate existing rows with valid data.
3. Phase 3: Update `schema.prisma` to make the field required (`field String`), remove the temporary default if needed, and run `prisma migrate dev`.

---

# ADVANCED QUERY PATTERNS AND CODE EXAMPLES

## a) Preventing N+1 Query Traps with Projections

### INCORRECT / ANTIPATTERN
Fetching parent items and iterating over them with nested queries creates N+1 database operations, overwhelming the connection pool.

```typescript
// INCORRECT: Executes 1 query for users + N queries for posts
const users = await prisma.user.findMany();

for (const user of users) {
  const posts = await prisma.post.findMany({
    where: { authorId: user.id },
  });
  console.log(user.name, posts.length);
}
```

### CORRECT / CLEAN
Use `include` or explicit `select` projections to fetch parents and children in a single optimized query.

```typescript
// CORRECT: Single SQL query with joined selection
const usersWithPosts = await prisma.user.findMany({
  select: {
    id: true,
    email: true,
    posts: {
      select: {
        id: true,
        title: true,
        createdAt: true,
      },
      where: {
        published: true,
      },
    },
  },
});
```

## b) Sequential Operations vs. Interactive Transactions

### INCORRECT / ANTIPATTERN
Executing interdependent database operations independently without transactional boundaries leaves data in an inconsistent state if an operation fails midway. Executing side effects inside `$transaction` can cause duplicate emails if the transaction retries or rolls back.

```typescript
// INCORRECT: Non-atomic writes and side effects inside transactions
async function createUserAndSendEmail(email: string, name: string) {
  // If post creation fails, user remains in DB!
  const user = await prisma.user.create({ data: { email, name } });
  const post = await prisma.post.create({
    data: { title: "Welcome", authorId: user.id },
  });

  // BAD: Email sent inside transaction context
  await prisma.$transaction(async (tx) => {
    await tx.user.update({ where: { id: user.id }, data: { verified: true } });
    await sendWelcomeEmail(email); // Will execute even if transaction rolls back!
  });
}
```

### CORRECT / CLEAN
Group atomic database mutations inside `$transaction`, return the necessary result, and perform external side effects only after the transaction successfully commits.

```typescript
// CORRECT: Atomic transaction with side effect executed post-commit
async function createUserWithInitialPost(email: string, name: string) {
  const result = await prisma.$transaction(async (tx) => {
    const user = await tx.user.create({
      data: { email, name },
    });

    const post = await tx.post.create({
      data: {
        title: "Welcome to the Platform",
        authorId: user.id,
      },
    });

    return { user, post };
  });

  // Execute external side effect strictly AFTER commit
  await sendWelcomeEmail(result.user.email);

  return result;
}
```

## c) Efficient Pagination: Cursor-based vs. Offset-based

### INCORRECT / ANTIPATTERN
Offset-based pagination using `skip` and `take` forces the database to scan and discard `skip` rows, deteriorating query performance on large tables (O(N) degradation).

```typescript
// INCORRECT: Offset pagination on deep pages
async function getPage(pageNumber: number, pageSize: number = 20) {
  return await prisma.post.findMany({
    skip: (pageNumber - 1) * pageSize, // Scanning 100,000 rows for page 5,000!
    take: pageSize,
    orderBy: { createdAt: "desc" },
  });
}
```

### CORRECT / CLEAN
Use cursor-based pagination indexed on a unique identifier to achieve constant time O(1) page fetching regardless of depth.

```typescript
// CORRECT: Cursor-based pagination
async function getCursorPage(cursorId?: string, limit: number = 20) {
  return await prisma.post.findMany({
    take: limit,
    ...(cursorId
      ? {
          skip: 1, // Skip the cursor item itself
          cursor: { id: cursorId },
        }
      : {}),
    orderBy: { id: "desc" },
  });
}
```

## d) Type Derivation with Prisma Utility Types

### INCORRECT / ANTIPATTERN
Manually defining duplicate TypeScript interfaces for payload structures returned by complex `select` or `include` queries leads to type drift and invalid assertions.

```typescript
// INCORRECT: Manual interface mirroring Prisma output
interface ManualUserWithPosts {
  id: string;
  email: string;
  posts: { id: string; title: string }[];
}

async function getUser(id: string): Promise<ManualUserWithPosts> {
  const user = await prisma.user.findUnique({
    where: { id },
    include: { posts: true },
  });
  return user as ManualUserWithPosts; // Unsafe type assertion
}
```

### CORRECT / CLEAN
Derive strict return types directly from Prisma payload definitions using `Prisma.Args` or `Prisma.UserGetPayload`.

```typescript
// CORRECT: Safe type derivation using Prisma payload helpers
import { Prisma } from "@prisma/client";

const userWithPostsSelect = Prisma.validator<Prisma.UserDefaultArgs>()({
  select: {
    id: true,
    email: true,
    posts: {
      select: {
        id: true,
        title: true,
      },
    },
  },
});

export type UserWithPostsPayload = Prisma.UserGetPayload<
  typeof userWithPostsSelect
>;

export async function getUserTyped(id: string): Promise<UserWithPostsPayload | null> {
  return await prisma.user.findUnique({
    where: { id },
    ...userWithPostsSelect,
  });
}
```

---

# ANTI-PATTERNS AND PREVENTION CHECKLIST

Avoid these common database design and Prisma client anti-patterns:

1. Unrestricted `findMany` Queries: Calling `.findMany()` without explicit `take` limits or `select` projections loads entire database tables into Node.js memory.
   - Fix: Always specify `take` or page queries using cursor pagination.

2. Unindexed Filtering: Executing `.findMany({ where: { nonIndexedColumn: value } })` triggers full table scans in PostgreSQL.
   - Fix: Add `@unique` or `@@index([nonIndexedColumn])` directives in `schema.prisma`.

3. Prisma Queries inside Loops: Calling `await prisma.model.findUnique(...)` inside `for`, `map`, or `forEach` loops.
   - Fix: Batch queries using `findMany({ where: { id: { in: ids } } })` or utilize relational `.include()`.

4. Production Schema Drift via `db push`: Executing `prisma db push` in production bypasses SQL migration histories and can cause irreversible data loss.
   - Fix: Use `prisma migrate dev` locally and `prisma migrate deploy` in production pipelines.

5. Connection Pool Exhaustion: Instantiating `new PrismaClient()` inside request handlers or serverless functions creates hundreds of unmanaged database connections.
   - Fix: Implement the global singleton pattern and set `connection_limit` parameters in connection strings.

6. Side Effects inside Transactions: Invoking external APIs, payment gateways, or email dispatchers inside `$transaction` callbacks.
   - Fix: Execute external side effects strictly after the transaction commits.

---

# REUSABLE CONFIGURATION PRESETS

## Production-Ready `schema.prisma` Template

```prisma
datasource db {
  provider = "postgresql"
  url      = env("DATABASE_URL")
}

generator client {
  provider        = "prisma-client-js"
  previewFeatures = ["fullTextSearchPostgres"]
}

enum UserRole {
  ADMIN
  USER
  GUEST
}

model User {
  id        String    @id @default(uuid()) @db.Uuid
  email     String    @unique @db.VarChar(255)
  fullName  String    @map("full_name") @db.VarChar(100)
  role      UserRole  @default(USER)
  createdAt DateTime  @default(now()) @map("created_at") @db.Timestamptz
  updatedAt DateTime  @updatedAt @map("updated_at") @db.Timestamptz
  deletedAt DateTime? @map("deleted_at") @db.Timestamptz

  posts Post[]

  @@index([role])
  @@index([createdAt])
  @@map("users")
}

model Post {
  id        String    @id @default(uuid()) @db.Uuid
  title     String    @db.VarChar(255)
  content   String?   @db.Text
  published Boolean   @default(false)
  authorId  String    @map("author_id") @db.Uuid
  createdAt DateTime  @default(now()) @map("created_at") @db.Timestamptz
  updatedAt DateTime  @updatedAt @map("updated_at") @db.Timestamptz
  deletedAt DateTime? @map("deleted_at") @db.Timestamptz

  author User @relation(fields: [authorId], references: [id], onDelete: Cascade)

  @@index([authorId])
  @@index([published, createdAt])
  @@map("posts")
}
```

## Standard Workflow Scripts for `package.json`

```json
{
  "scripts": {
    "db:generate": "prisma generate",
    "db:push": "prisma db push",
    "db:migrate": "prisma migrate dev",
    "db:deploy": "prisma migrate deploy",
    "db:studio": "prisma studio",
    "db:seed": "prisma db seed"
  },
  "prisma": {
    "seed": "ts-node prisma/seed.ts"
  }
}
```
