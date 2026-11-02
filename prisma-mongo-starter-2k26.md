# Prisma v6 + MongoDB + Next.js Setup Guide

A step-by-step guide to integrating **Prisma ORM v6**, **MongoDB Atlas**, and **Next.js App Router** with TypeScript.

## Tech Stack

- Next.js 15+
- TypeScript
- Prisma ORM 6.19.3
- Prisma Client 6.19.3
- MongoDB Atlas

> **Important:** This guide uses Prisma v6.19.3. Keep `prisma` and `@prisma/client` on the same version. This setup uses the `prisma-client-js` generator and the default generated client location.

---

## 1. Create Next.js Project

```bash
npx create-next-app@latest my-app --typescript --tailwind --app

cd my-app
```

## 2. Install Prisma v6

```bash
npm install -D prisma@6.19.3 dotenv

npm install @prisma/client@6.19.3
```

### Package Responsibilities

| Package | Purpose |
|---|---|
| `prisma` | Prisma CLI and schema management |
| `@prisma/client` | Database queries and CRUD operations |
| `dotenv` | Loads environment variables for Prisma CLI |

## 3. Initialize Prisma

```bash
npx prisma init --datasource-provider mongodb
```

This creates:

```text
prisma/
└── schema.prisma

.env
```

For this Prisma v6 setup, you do not need `prisma.config.ts`. The database URL will remain inside `schema.prisma`.

## 4. Configure MongoDB Atlas

1. Create a free MongoDB Atlas account.
2. Create a cluster.
3. Go to Database Access and create a database user.
4. Configure Network Access and allow your IP address.
5. Select Connect → Drivers → Node.js.
6. Copy your MongoDB connection string.

### Configure `.env`

```env
DATABASE_URL="mongodb+srv://USERNAME:PASSWORD@mycluster.mongodb.net/my-database?retryWrites=true&w=majority"
```

Replace:

- `USERNAME` with your MongoDB username.
- `PASSWORD` with your MongoDB password.
- `my-database` with your database name.

> Never commit `.env` to GitHub. Add it to `.gitignore`.

## 5. Configure Prisma Schema

File: `prisma/schema.prisma`

```prisma
generator client {
  provider = "prisma-client-js"
}

datasource db {
  provider = "mongodb"
  url      = env("DATABASE_URL")
}

model User {
  id        String   @id @default(auto()) @map("_id") @db.ObjectId
  name      String
  email     String   @unique
  createdAt DateTime @default(now())
}
```

### MongoDB Schema Rules

**ObjectId**

```prisma
id String @id @default(auto()) @map("_id") @db.ObjectId
```

- `String`: Prisma field type.
- `@id`: Primary key.
- `@default(auto())`: Automatically generates ObjectId.
- `@map("_id")`: Maps the field to MongoDB's `_id`.
- `@db.ObjectId`: Uses MongoDB ObjectId.

**Unique Field**

```prisma
email String @unique
```

Ensures that each email address is unique.

**Optional Field**

```prisma
phone String?
```

The `?` makes the field optional.

**Default Value**

```prisma
createdAt DateTime @default(now())
```

Automatically sets the creation timestamp.

## 6. Push Schema to MongoDB

```bash
npx prisma db push
```

This command:

- Synchronizes the Prisma schema with MongoDB.
- Creates collections when necessary.
- Creates indexes for supported unique constraints.
- Updates the database structure.

> **Important:** Prisma Migrate is not supported for MongoDB. Use `db push` instead of `migrate dev`.

```bash
# Do not use with MongoDB
npx prisma migrate dev

# Use this instead
npx prisma db push
```

## 7. Generate Prisma Client

```bash
npx prisma generate
```

With the `prisma-client-js` generator and no custom output, Prisma Client is generated in:

```text
node_modules/
└── .prisma/
    └── client/
```

Import Prisma Client using:

```ts
import { PrismaClient } from "@prisma/client";
```

Run `npx prisma generate` again whenever you change your Prisma schema.

## 8. Create Prisma Singleton

Create `lib/prisma.ts` in the project root.

```ts
import { PrismaClient } from "@prisma/client";

const globalForPrisma = globalThis as unknown as {
  prisma: PrismaClient | undefined;
};

export const prisma =
  globalForPrisma.prisma ?? new PrismaClient();

if (process.env.NODE_ENV !== "production") {
  globalForPrisma.prisma = prisma;
}

export default prisma;
```

### Why Singleton Pattern?

Next.js uses Hot Reload during development.

Without a singleton, repeated reloads can create unnecessary PrismaClient instances and database connections.

The singleton stores the PrismaClient instance on `globalThis` and reuses it across development reloads.

In production, the module-level instance is reused through the module cache.

## 9. Use Prisma in Next.js

### A. Server Component

File: `app/users/page.tsx`

```tsx
import prisma from "@/lib/prisma";

export default async function UsersPage() {
  const users = await prisma.user.findMany();

  return (
    <div>
      <h1>Users</h1>

      {users.map((user) => (
        <div key={user.id}>
          <h2>{user.name}</h2>
          <p>{user.email}</p>
        </div>
      ))}
    </div>
  );
}
```

> Prisma Client should only be used on the server, not directly inside Client Components.

### B. API Route Handler

File: `app/api/users/route.ts`

```ts
import prisma from "@/lib/prisma";
import { NextResponse } from "next/server";

export async function GET() {
  const users = await prisma.user.findMany();

  return NextResponse.json(users);
}

export async function POST(req: Request) {
  const { name, email } = await req.json();

  const user = await prisma.user.create({
    data: {
      name,
      email,
    },
  });

  return NextResponse.json(user, {
    status: 201,
  });
}
```

| Method | Endpoint | Purpose |
|---|---|---|
| GET | `/api/users` | Get all users |
| POST | `/api/users` | Create a new user |

For production APIs, add request validation, authentication, authorization, and error handling.

### C. Server Actions

File: `app/actions/userActions.ts`

```ts
"use server";

import prisma from "@/lib/prisma";

export async function createUser(name: string, email: string) {
  return prisma.user.create({
    data: {
      name,
      email,
    },
  });
}

export async function deleteUser(id: string) {
  await prisma.user.delete({
    where: { id },
  });
}
```

Server Actions allow server-side database operations to be called through Next.js actions. Add appropriate authorization and input validation before using them in production.

## 10. MongoDB Relations

Example: One User can have multiple Posts.

Update `prisma/schema.prisma`:

```prisma
generator client {
  provider = "prisma-client-js"
}

datasource db {
  provider = "mongodb"
  url      = env("DATABASE_URL")
}

model User {
  id        String   @id @default(auto()) @map("_id") @db.ObjectId
  name      String
  email     String   @unique
  createdAt DateTime @default(now())

  posts     Post[]
}

model Post {
  id        String   @id @default(auto()) @map("_id") @db.ObjectId
  title     String
  content   String?
  published Boolean  @default(false)
  createdAt DateTime @default(now())

  author    User     @relation(fields: [authorId], references: [id])
  authorId  String   @db.ObjectId
}
```

### Understanding the Relation

```prisma
posts Post[]
```

A User can have multiple Posts.

```prisma
authorId String @db.ObjectId
```

Stores the referenced User ObjectId in the Post document.

```prisma
author User @relation(fields: [authorId], references: [id])
```

Defines the relationship between Post and User.

> MongoDB relations use document references. They are not the same as SQL foreign key constraints.

After changing the schema:

```bash
npx prisma db push
npx prisma generate
```

## 11. Common MongoDB + Prisma Gotchas

| Topic | Important Note |
|---|---|
| Prisma Migrate | Not supported with MongoDB |
| `db push` | Use to synchronize schema |
| ObjectId | Use `String` with `@db.ObjectId` |
| `autoincrement()` | Not supported |
| Transactions | Require a replica set |
| Prisma Studio | MongoDB is not supported by Prisma Studio in this setup |

### Recommended Database Tools

- MongoDB Compass
- MongoDB Atlas Dashboard

## 12. Project Structure

```text
my-app/
│
├── app/
│   ├── api/
│   │   └── users/
│   │       └── route.ts
│   │
│   ├── actions/
│   │   └── userActions.ts
│   │
│   ├── users/
│   │   └── page.tsx
│   │
│   ├── layout.tsx
│   └── page.tsx
│
├── lib/
│   └── prisma.ts
│
├── prisma/
│   └── schema.prisma
│
├── .env
├── .gitignore
├── next.config.ts
├── package.json
└── tsconfig.json
```

## 13. Quick Reference — Commands

| Command | Purpose |
|---|---|
| `npx prisma init --datasource-provider mongodb` | Initialize Prisma |
| `npx prisma db push` | Synchronize schema |
| `npx prisma generate` | Generate Prisma Client |
| `npx prisma db pull` | Introspect existing database |
| `npx prisma --version` | Check Prisma version |

## 14. Final Setup Checklist

- [ ] Create Next.js project.
- [ ] Install Prisma v6.19.3.
- [ ] Create MongoDB Atlas cluster.
- [ ] Configure `DATABASE_URL` in `.env`.
- [ ] Configure `prisma/schema.prisma`.
- [ ] Run `npx prisma db push`.
- [ ] Run `npx prisma generate`.
- [ ] Create `lib/prisma.ts`.
- [ ] Test GET `/api/users`.
- [ ] Test POST `/api/users`.

---

## Official Documentation

- [Prisma MongoDB Documentation](https://www.prisma.io/docs/orm/overview/databases/mongodb)
- [Prisma Client Generation](https://www.prisma.io/docs/orm/prisma-client/setup-and-configuration/generating-prisma-client)
- [Prisma v6 Documentation](https://www.prisma.io/docs/orm/v6)
- [MongoDB Atlas](https://www.mongodb.com/atlas)
- [MongoDB Documentation](https://www.mongodb.com/docs/)

---

**Note:** This guide intentionally uses Prisma v6 with `prisma-client-js`, the default generated client location, and `@prisma/client` imports. It does not use Prisma v7 configuration or driver adapters.
