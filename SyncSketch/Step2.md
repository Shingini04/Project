# STEP 2

This guide builds directly on the Turborepo monorepo from Step 1 in **8 steps**. We will establish strict trust boundaries with shared internal packages (`@repo/backend-common`, `@repo/common`, and `@repo/db`), introduce runtime schema validation with **Zod**, persist data to **PostgreSQL via Prisma**, and secure both the **Express HTTP API** and **WebSocket server** with a single shared **JWT secret**.

---

### Step 1: Create `packages/backend-common` to Share `JWT_SECRET`

#### Why Not Just Put `config.ts` in Both Backends?

Both `apps/http-backend` (which *signs* JWTs on `/signin`) and `apps/ws-backend` (which *verifies* JWTs during the WebSocket handshake) must use the exact same secret key. Duplicating `config.ts` across both apps creates **configuration drift**—rotating or editing the secret in one service silently breaks authentication in the other.

At the same time, we must **not** put `JWT_SECRET` inside a general `@repo/common` package that `apps/web` also imports. If a frontend bundler (like Next.js/Webpack) bundles a module containing server secrets, it can leak private keys into client-side JavaScript sent to the browser. Creating a dedicated `@repo/backend-common` package enforces a clean **server-only trust boundary**.

#### Create the Folder Structure

```bash
mkdir -p packages/backend-common/src

```

#### `packages/backend-common/package.json`

```json
{
  "name": "@repo/backend-common",
  "version": "1.0.0",
  "private": true,
  "type": "module",
  "main": "./dist/index.js",
  "types": "./dist/index.d.ts",
  "exports": {
    "./config": {
      "types": "./src/index.ts",
      "default": "./dist/index.js"
    }
  },
  "scripts": {
    "build": "tsc -b"
  },
  "devDependencies": {
    "@repo/typescript-config": "workspace:*",
    "typescript": "^5.6.3"
  }
}

```

#### `packages/backend-common/tsconfig.json`

```json
{
  "extends": "@repo/typescript-config/base.json",
  "compilerOptions": {
    "rootDir": "./src",
    "outDir": "./dist",
    "declaration": true,
    "declarationMap": true
  },
  "include": ["src/**/*"]
}

```

#### `packages/backend-common/src/index.ts`

```typescript
export const JWT_SECRET = process.env.JWT_SECRET || "123123";

```

> 🔍 **Deep-Dive Search Topics:**
> * "Privilege separation and trust boundaries in full-stack monorepos"
> * "12-Factor App configuration management (`process.env` vs fallback defaults)"
> 
> 

---

### Step 2: Create `packages/common` and Introduce Zod Schemas

#### What is Zod Anyway? (Compile-Time Types vs. Runtime Validation)

A common misconception is that TypeScript protects your API endpoints at runtime. It does not:

* **TypeScript is a Compile-Time Erasure System:** During `tsc` compilation, every `interface` and `type` is completely stripped out of the emitted JavaScript. At runtime, Node.js executes plain JavaScript with zero knowledge of your TypeScript types.
* **The Untrusted Network Boundary:** When a client sends `POST /signup`, `req.body` arrives over TCP as arbitrary, untrusted JSON. Even if you write `const data: SignupInput = req.body`, TypeScript blindly trusts the assertion. If a client sends `{ "username": 999 }` or omits `password`, your server crashes or inserts malformed data.

**Zod** bridges compile-time safety and runtime validation:

1. You declare a runtime schema object (`CreateUserSchema`).
2. At **runtime**, `CreateUserSchema.safeParse(req.body)` inspects the actual incoming JavaScript object, validates field types and constraints, and returns structured validation errors if anything fails.
3. At **compile time**, `z.infer<typeof CreateUserSchema>` extracts the static TypeScript type directly from that schema—giving both `apps/web` (frontend forms) and `apps/http-backend` (API handlers) a **Single Source of Truth**.

```text
                  ┌─────────────────────────────┐
                  │       packages/common       │
                  │  (Zod Schemas & TS Types)   │
                  └──────┬───────────────┬──────┘
                         │               │
            ┌────────────▼───┐       ┌───▼────────────────────────┐
            │    apps/web    │       │  apps/http-backend &       │
            │  (Next.js UI)  │       │  apps/ws-backend           │
            └────────────────┘       └───────┬──────────────┬─────┘
                                             │              │
                              ┌──────────────▼──┐        ┌──▼─────────────────────┐
                              │   packages/db   │        │ packages/backend-common│
                              │ (Prisma Client) │        │     (JWT_SECRET)       │
                              └─────────────────┘        └────────────────────────┘

```

#### Create the Folder Structure

```bash
mkdir -p packages/common/src

```

#### `packages/common/package.json`

```json
{
  "name": "@repo/common",
  "version": "1.0.0",
  "private": true,
  "type": "module",
  "main": "./dist/types.js",
  "types": "./dist/types.d.ts",
  "exports": {
    "./types": {
      "types": "./src/types.ts",
      "default": "./dist/types.js"
    }
  },
  "scripts": {
    "build": "tsc -b"
  },
  "dependencies": {
    "zod": "^3.23.8"
  },
  "devDependencies": {
    "@repo/typescript-config": "workspace:*",
    "typescript": "^5.6.3"
  }
}

```

#### `packages/common/tsconfig.json`

```json
{
  "extends": "@repo/typescript-config/base.json",
  "compilerOptions": {
    "rootDir": "./src",
    "outDir": "./dist",
    "declaration": true,
    "declarationMap": true
  },
  "include": ["src/**/*"]
}

```

#### `packages/common/src/types.ts`

```typescript
import { z } from "zod";

export const CreateUserSchema = z.object({
  username: z.string().min(3).max(20),
  password: z.string(),
  name: z.string(),
});

export const SigninSchema = z.object({
  username: z.string().min(3).max(20),
  password: z.string(),
});

export const CreateRoomSchema = z.object({
  name: z.string().min(3).max(20),
});

// Static TypeScript types derived directly from the runtime schemas
export type CreateUserInput = z.infer<typeof CreateUserSchema>;
export type SigninInput = z.infer<typeof SigninSchema>;
export type CreateRoomInput = z.infer<typeof CreateRoomSchema>;

```

> 🔍 **Deep-Dive Search Topics:**
> * "Type erasure in TypeScript compilation"
> * "Parse, don't validate: type-driven design at network boundaries"
> 
> 

> 🎯 **Interview Edge:**
> When asked *"Why use Zod if we already have TypeScript?"*, explain that TypeScript enforces **static compile-time contracts** inside your codebase, whereas Zod enforces **dynamic runtime validation** across untrusted network boundaries (HTTP JSON bodies, WebSocket messages, environment variables) while automatically deriving static types via `z.infer`.

---

### Step 3: Initialize `packages/db` and Define the Prisma Schema

Instead of duplicating database connections and raw SQL strings across `http-backend` and `ws-backend`, we encapsulate **Prisma ORM** inside `packages/db`.

#### How Prisma Works

1. **`schema.prisma`:** Declarative data modeling file defining tables, primary keys, unique constraints, and foreign-key relationships.
2. **Prisma Migrate:** Diffs `schema.prisma` against your PostgreSQL database and generates versioned SQL migration files.
3. **Prisma Client:** Code-generates a strongly typed query builder tailored specifically to your models (`prismaClient.user.create(...)`, `prismaClient.room.create(...)`).

#### Create the Folder Structure

```bash
mkdir -p packages/db/src

```

#### `packages/db/package.json`

```json
{
  "name": "@repo/db",
  "version": "1.0.0",
  "private": true,
  "type": "module",
  "main": "./dist/index.js",
  "types": "./dist/index.d.ts",
  "exports": {
    "./client": {
      "types": "./src/index.ts",
      "default": "./dist/index.js"
    }
  },
  "scripts": {
    "build": "tsc -b",
    "db:generate": "prisma generate",
    "db:migrate": "prisma migrate dev"
  },
  "dependencies": {
    "@prisma/client": "^5.22.0"
  },
  "devDependencies": {
    "@repo/typescript-config": "workspace:*",
    "prisma": "^5.22.0",
    "typescript": "^5.6.3"
  }
}

```

#### `packages/db/tsconfig.json`

```json
{
  "extends": "@repo/typescript-config/base.json",
  "compilerOptions": {
    "rootDir": "./src",
    "outDir": "./dist",
    "declaration": true,
    "declarationMap": true
  },
  "include": ["src/**/*"]
}

```

#### `packages/db/prisma/schema.prisma`

```prisma
generator client {
  provider = "prisma-client-js"
}

datasource db {
  provider = "postgresql"
  url      = env("DATABASE_URL")
}

model User {
  id        String   @id @default(uuid())
  username  String   @unique
  password  String
  name      String
  photo     String?
  rooms     Room[]
  chats     Chat[]
  createdAt DateTime @default(now())
}

model Room {
  id        Int      @id @default(autoincrement())
  slug      String   @unique
  adminId   String
  admin     User     @relation(fields: [adminId], references: [id])
  chats     Chat[]
  createdAt DateTime @default(now())
}

model Chat {
  id        Int      @id @default(autoincrement())
  roomId    Int
  message   String
  userId    String
  room      Room     @relation(fields: [roomId], references: [id])
  user      User     @relation(fields: [userId], references: [id])
  createdAt DateTime @default(now())
}

```

#### Schema Design Rationale

* **`User.id` (`UUID`) vs. `Room.id` (`Int`):** Non-sequential UUIDs prevent user-ID enumeration attacks, while auto-incrementing integers for `Room.id` keep in-memory room lookup maps compact and fast.
* **`Room.slug` (`@unique`):** Gives every whiteboard room a unique human-readable name (e.g., `"system-design"`) so clients can join by name.
* **`Chat`:** Stores serialized drawing actions and messages tied to both `roomId` and `userId` so late joiners can reconstruct the board state.

#### Export the Client Singleton (`packages/db/src/index.ts`)

```typescript
import { PrismaClient } from "@prisma/client";

export const prismaClient = new PrismaClient();

```

#### Add `packages/db/.env` and Run the Initial Migration

```env
DATABASE_URL="postgresql://postgres:password@localhost:5432/draw_app?schema=public"

```

```bash
pnpm install
pnpm --filter @repo/db db:migrate --name init_schema
pnpm --filter @repo/db db:generate

```

> 🔍 **Deep-Dive Search Topics:**
> * "UUIDv4 vs Auto-Incrementing Integer Primary Keys in PostgreSQL B-Tree Indexes"
> * "Referential integrity and foreign key indexing in relational schemas"
> 
> 

---

### Step 4: Link Workspace Packages into `http-backend` and `ws-backend`

Now connect `@repo/backend-common`, `@repo/common`, `@repo/db`, and `jsonwebtoken` into our two backend applications using the `workspace:*` protocol.

#### `apps/http-backend/package.json`

```json
{
  "name": "http-backend",
  "version": "1.0.0",
  "private": true,
  "type": "module",
  "scripts": {
    "build": "tsc -b",
    "start": "node ./dist/index.js",
    "dev": "tsc -b && node ./dist/index.js"
  },
  "dependencies": {
    "@repo/backend-common": "workspace:*",
    "@repo/common": "workspace:*",
    "@repo/db": "workspace:*",
    "express": "^4.21.1",
    "jsonwebtoken": "^9.0.2"
  },
  "devDependencies": {
    "@repo/typescript-config": "workspace:*",
    "@types/express": "^5.0.0",
    "@types/jsonwebtoken": "^9.0.7",
    "typescript": "^5.6.3"
  }
}

```

#### `apps/ws-backend/package.json`

```json
{
  "name": "ws-backend",
  "version": "1.0.0",
  "private": true,
  "type": "module",
  "scripts": {
    "build": "tsc -b",
    "start": "node ./dist/index.js",
    "dev": "tsc -b && node ./dist/index.js"
  },
  "dependencies": {
    "@repo/backend-common": "workspace:*",
    "@repo/db": "workspace:*",
    "jsonwebtoken": "^9.0.2",
    "ws": "^8.18.0"
  },
  "devDependencies": {
    "@repo/typescript-config": "workspace:*",
    "@types/jsonwebtoken": "^9.0.7",
    "@types/ws": "^8.5.13",
    "typescript": "^5.6.3"
  }
}

```

Run `pnpm install` at the repository root to wire the local workspace symlinks into `node_modules`.

---

### Step 5: Write the JWT Auth Middleware (`apps/http-backend/src/middleware.ts`)

Protected endpoints like `POST /room` must reject unauthenticated callers and identify *which* user is creating the room (`userId`). Express **middleware** intercepts incoming HTTP requests before they reach the route handler.

#### How Stateless JWT Verification Works

1. When a user signs in, `http-backend` creates a JSON payload `{ userId }` and signs it using HMAC-SHA256 (`jwt.sign({ userId }, JWT_SECRET)`).
2. The resulting token consists of `Header.Payload.Signature` (Base64Url-encoded).
3. On protected requests, the client sends the token in the `Authorization` header. `authMiddleware` runs `jwt.verify(token, JWT_SECRET)`, which recomputes the HMAC signature. If the token was forged or altered by even a single byte, `jwt.verify` throws an error—verifying identity **statelessly** without hitting PostgreSQL.

Create `apps/http-backend/src/middleware.ts`:

```typescript
import { NextFunction, Request, Response } from "express";
import jwt, { JwtPayload } from "jsonwebtoken";
import { JWT_SECRET } from "@repo/backend-common/config";

export interface AuthenticatedRequest extends Request {
  userId?: string;
}

export function authMiddleware(
  req: AuthenticatedRequest,
  res: Response,
  next: NextFunction
) {
  const authHeader = req.headers["authorization"] ?? "";

  // Support both "Bearer <token>" and raw "<token>" headers
  const token = authHeader.startsWith("Bearer ")
    ? authHeader.slice(7)
    : authHeader;

  if (!token) {
    res.status(401).json({ message: "Unauthorized: Missing token" });
    return;
  }

  try {
    const decoded = jwt.verify(token, JWT_SECRET) as JwtPayload;

    if (decoded && typeof decoded.userId === "string") {
      req.userId = decoded.userId;
      next();
    } else {
      res.status(403).json({ message: "Forbidden: Invalid token payload" });
    }
  } catch (err) {
    res.status(403).json({ message: "Forbidden: Invalid or expired token" });
  }
}

```

```text
Client Request: POST /room
Header: Authorization: Bearer eyJhbGci...
         │
         ▼
┌─────────────────────┐      Invalid / Missing      ┌────────────────────────┐
│   authMiddleware    │ ──────────────────────────► │ 401 / 403 JSON Error   │
└────────┬────────────┘                             └────────────────────────┘
         │ Valid Signature
         │ Attaches req.userId = "uuid-123"
         │ Calls next()
         ▼
┌─────────────────────┐
│  POST /room Handler │ ──► Inserts Room into PostgreSQL with adminId = req.userId
└─────────────────────┘

```

> 🔍 **Deep-Dive Search Topics:**
> * "Stateless JWT tokens vs Stateful Server-Side Sessions"
> * "JWT signature verification and HMAC-SHA256 mechanics"
> 
> 

> 🎯 **Interview Edge:**
> When asked *"What is the main limitation of stateless JWTs?"*, explain that because the server stores no session state, **you cannot immediately revoke an individual JWT** before its expiration without maintaining a stateful revocation blocklist (such as in Redis) or rotating `JWT_SECRET` (which invalidates tokens for all users).

---

### Step 6: Implement `/signup`, `/signin`, and `/room` in `http-backend`

Now update `apps/http-backend/src/index.ts` to validate payloads with Zod (`@repo/common/types`), persist records to PostgreSQL (`@repo/db/client`), sign JWTs (`@repo/backend-common/config`), and gate `POST /room` behind `authMiddleware`.

> **Naive Step 2 Baseline Note:** To keep this step focused on wiring the monorepo layers end-to-end, passwords are stored as plain strings for now. Before deploying to production, always hash passwords using `bcrypt` or `argon2id`.

Update `apps/http-backend/src/index.ts`:

```typescript
import express from "express";
import jwt from "jsonwebtoken";
import { JWT_SECRET } from "@repo/backend-common/config";
import {
  CreateUserSchema,
  SigninSchema,
  CreateRoomSchema,
} from "@repo/common/types";
import { prismaClient } from "@repo/db/client";
import { authMiddleware, AuthenticatedRequest } from "./middleware.js";

const app = express();
const PORT = 3001;

app.use(express.json());

app.get("/health", (req, res) => {
  res.json({ status: "ok" });
});

// 1. Naive Signup Endpoint
app.post("/signup", async (req, res) => {
  const parsedData = CreateUserSchema.safeParse(req.body);

  if (!parsedData.success) {
    res.status(400).json({
      message: "Incorrect inputs",
      errors: parsedData.error.flatten(),
    });
    return;
  }

  try {
    const user = await prismaClient.user.create({
      data: {
        username: parsedData.data.username,
        password: parsedData.data.password,
        name: parsedData.data.name,
      },
    });

    res.status(201).json({
      userId: user.id,
    });
  } catch (err) {
    // Prisma throws P2002 when the @unique username constraint is violated
    res.status(409).json({
      message: "User already exists with this username",
    });
  }
});

// 2. Naive Signin Endpoint
app.post("/signin", async (req, res) => {
  const parsedData = SigninSchema.safeParse(req.body);

  if (!parsedData.success) {
    res.status(400).json({
      message: "Incorrect inputs",
      errors: parsedData.error.flatten(),
    });
    return;
  }

  try {
    const user = await prismaClient.user.findFirst({
      where: {
        username: parsedData.data.username,
        password: parsedData.data.password,
      },
    });

    if (!user) {
      res.status(403).json({
        message: "Not authorized",
      });
      return;
    }

    const token = jwt.sign({ userId: user.id }, JWT_SECRET);

    res.json({
      token,
    });
  } catch (err) {
    res.status(500).json({
      message: "Internal server error",
    });
  }
});

// 3. Protected Create-Room Endpoint
app.post("/room", authMiddleware, async (req: AuthenticatedRequest, res) => {
  const parsedData = CreateRoomSchema.safeParse(req.body);

  if (!parsedData.success) {
    res.status(400).json({
      message: "Incorrect inputs",
      errors: parsedData.error.flatten(),
    });
    return;
  }

  const userId = req.userId;
  if (!userId) {
    res.status(401).json({ message: "Unauthorized" });
    return;
  }

  try {
    const room = await prismaClient.room.create({
      data: {
        slug: parsedData.data.name,
        adminId: userId,
      },
    });

    res.status(201).json({
      roomId: room.id,
    });
  } catch (err) {
    res.status(409).json({
      message: "Room already exists with this name",
    });
  }
});

app.listen(PORT, () => {
  console.log(`HTTP server running on http://localhost:${PORT}`);
});

```

---

### Step 7: Decode and Verify the JWT in `ws-backend` via Query Params

#### Why Query Parameters for WebSockets?

Unlike HTTP `fetch()`, the browser's native `WebSocket` constructor (`new WebSocket(url)`) **does not allow custom HTTP headers** such as `Authorization: Bearer <token>` during the initial `HTTP 101 Switching Protocols` handshake.

To authenticate the socket immediately upon connection, the client appends the JWT as a URL query parameter:

```text
ws://localhost:8080?token=eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...

```

When a client connects, `wss.on("connection", (ws, request) => ...)` exposes the underlying Node.js `http.IncomingMessage` (`request`). We parse `request.url` using `URLSearchParams`, verify the token against the shared `JWT_SECRET` from `@repo/backend-common/config`, and immediately terminate (`ws.close()`) unauthorized sockets before accepting any messages.

Update `apps/ws-backend/src/index.ts`:

```typescript
import { WebSocketServer } from "ws";
import jwt, { JwtPayload } from "jsonwebtoken";
import { JWT_SECRET } from "@repo/backend-common/config";

const wss = new WebSocketServer({ port: 8080 });

function checkUser(token: string): string | null {
  try {
    const decoded = jwt.verify(token, JWT_SECRET) as JwtPayload;

    if (!decoded || typeof decoded.userId !== "string") {
      return null;
    }

    return decoded.userId;
  } catch (err) {
    return null;
  }
}

wss.on("connection", (ws, request) => {
  const url = request.url;
  if (!url) {
    ws.close(1008, "Missing URL");
    return;
  }

  // Extract token from "/?token=..."
  const queryParams = new URLSearchParams(url.split("?")[1]);
  const token = queryParams.get("token") ?? "";
  const userId = checkUser(token);

  if (!userId) {
    // 1008 = RFC 6455 Policy Violation close code
    ws.close(1008, "Unauthorized");
    return;
  }

  console.log(`Authenticated WebSocket client connected: userId=${userId}`);

  ws.on("message", (data) => {
    console.log(`Received from ${userId}: %s`, data.toString());
    ws.send("pong");
  });
});

console.log("WebSocket server listening on ws://localhost:8080");

```

> 🔍 **Deep-Dive Search Topics:**
> * "RFC 6455 WebSocket Close Status Codes (1000 vs 1008 Policy Violation)"
> * "Security trade-offs of JWTs in URL query strings (reverse proxy access logs & browser history)"
> 
> 

> 🎯 **Interview Edge:**
> When asked *"What is the security trade-off of passing a JWT in a WebSocket URL query parameter, and how do you harden it in production?"*, explain that query strings appear in plaintext inside reverse proxy logs (NGINX, Cloudflare) and browser history. In production, either exchange the JWT over HTTPS for a **single-use, short-lived opaque ticket** (e.g., 10-second TTL in Redis) to pass in the URL, or accept the socket connection and require an **initial `auth` JSON frame** before subscribing the client to any room.

---

### Step 8: Build and Verify the End-to-End Authentication & DB Pipeline

1. **Compile all workspace packages and services in topological order:**
```bash
pnpm run build

```


Turborepo's DAG scheduler automatically builds `@repo/backend-common`, `@repo/common`, and `@repo/db` first, emitting their `.js` and `.d.ts` files before compiling `http-backend` and `ws-backend`.
2. **Start both servers concurrently:**
```bash
pnpm run dev

```


3. **Verify `/signup`, `/signin`, `/room`, and WebSocket auth:**
```bash
# 1. Create a user in PostgreSQL
curl -X POST http://localhost:3001/signup \
  -H "Content-Type: application/json" \
  -d '{"username":"alice","password":"password123","name":"Alice"}'

# 2. Sign in to receive a signed JWT
curl -X POST http://localhost:3001/signin \
  -H "Content-Type: application/json" \
  -d '{"username":"alice","password":"password123"}'

# 3. Create a protected room in PostgreSQL using the JWT
curl -X POST http://localhost:3001/room \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer <YOUR_JWT_TOKEN>" \
  -d '{"name":"design-board"}'

```


Connecting a WebSocket client to `ws://localhost:8080?token=<YOUR_JWT_TOKEN>` now stays open and echoes `"pong"`, whereas connecting without a valid token is immediately rejected with close code `1008`.
