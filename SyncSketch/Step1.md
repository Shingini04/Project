# STEP 1

This guide walks step-by-step through setting up a monorepo for a real-time collaborative whiteboard application (like Excalidraw or Miro). Each step details the underlying mechanisms, common systems-engineering trade-offs, and critical software engineering interview talking points.

---

### Step 1: Initialize a Monorepo with Turborepo

#### What is a Monorepo?
A **monorepo** (monolithic repository) is an architectural pattern where multiple distinct projects, libraries, and microservices live inside a single version-controlled repository (Git). 

* **Monorepo vs. Polyrepo (Multirepo):**
  * In a *polyrepo* setup, your client app, HTTP authentication server, and WebSocket service live in separate Git repositories. When you change a shared data type or an API schema, you must update the type, publish it to an internal registry (like npm or GitHub Packages), wait for the package to build, bump the version string in every consuming repository, and coordinate multi-PR deployments.
  * In a *monorepo*, your HTTP API, WebSocket server, client frontend, database client, and common TypeScript types all live side-by-side. A change to an interface immediately triggers type checking across all downstream consumers in the same commit.

#### What is Turborepo?
Turborepo is a high-performance build orchestration system tailored for JavaScript and TypeScript monorepos, written in Rust.

* **Directed Acyclic Graph (DAG):** Turborepo analyzes the dependencies between your internal packages and constructs a DAG to determine the optimal execution order. If your WebSocket server depends on `@repo/db` and `@repo/db` depends on `@repo/typescript-config`, Turborepo ensures upstream prerequisites build first, while executing independent tasks in parallel across CPU threads.
* **Hermetic Fingerprinting & Remote Caching:** Turborepo generates a hash fingerprint of your source code, environment variables, task arguments, and lockfiles. If the hash hasn't changed since the previous run, it avoids re-executing tasks (such as transpilation, linting, or testing) and restores build artifacts from local disk storage or cloud storage within milliseconds.

> 🔍 **Deep-Dive Search Topics:**  
> * "Directed Acyclic Graphs (DAG) in build systems"  
> * "Monorepo vs Multirepo at scale (Google and Meta build strategies)"  
> * "Turborepo Remote Caching architecture"

> 🎯 **Interview Edge:**  
> When asked *"Why use Turborepo over basic npm workspaces?"*, explain that npm workspaces only handle symlinking and dependency resolution; they do not optimize task execution. Turborepo provides DAG-aware task scheduling, parallel execution, and content-addressable build caching.

---

### Step 2: Use `pnpm` Instead of `npm`

`pnpm` stands for *Performant npm*. It is a drop-in replacement for traditional package managers (`npm`, `yarn v1`) that re-engineers package storage and resolution.

#### The Flaws of Traditional `npm`
1. **Disk Duplication:** If you maintain 10 local microservices that each depend on `next@14.x`, standard npm writes duplicate copies of those binary and source files to every single project's `node_modules`.
2. **Phantom Dependencies (Dependency Leaks):** Traditional npm flattens dependencies to prevent deeply nested file paths (due to Windows path length limits). If library $A$ installs library $B$, npm hoists $B$ directly into the top-level `node_modules`. Your code can import $B$ without declaring it in `package.json`. If $A$ removes $B$ in a minor update, your code breaks silently in production.
3. **NPM Doppelgängers:** Multiple versions of the same transitive dependency create duplicate nested tree branches, inflating installation times and disk space.

> 🔍 **Deep-Dive Search Topics:**  
> * "Flat vs Nested node_modules directory structures"  
> * "Phantom dependencies in Node.js ecosystem"

---

### Step 3: Install `pnpm` Globally and Understand Hard Links

Run:
```bash
npm install -g pnpm
```

#### How `pnpm` Works Under the Hood
1. **Global Content-Addressable Storage (CAS):** All downloaded files are stored once in a centralized location on your disk (e.g., `~/.local/share/pnpm/store`). Every file is indexed by its content hash.
2. **Hard Links:** When your project requires a package, `pnpm` doesn't duplicate the bytes. It creates an operating system **hard link** pointing to the file in the global store. A hard link is an inode reference in the filesystem table that points directly to raw disk sectors without taking extra storage.
3. **Symlinked Isolation:** Direct dependencies are symlinked into `node_modules`, while transitive dependencies are isolated inside `.pnpm/`. Your project can only import packages explicitly declared in its `package.json`.

```text
Global CAS Store (~/.local/share/pnpm/store)
  └── [Hash: 9a8f4c] -> lodash/index.js (Real bytes on disk)
           ▲
           │ (Hard Link - zero disk overhead)
Your Project (/node_modules/.pnpm/lodash@4.17.21/node_modules/lodash/index.js)
           ▲
           │ (Symbolic Link)
Your App Code (/node_modules/lodash)
```

> 🔍 **Deep-Dive Search Topics:**  
> * "Inodes, Hard Links, and Symbolic Links in Linux / POSIX filesystems"  
> * "Content-Addressable Storage mechanics in Git and pnpm"

> 🎯 **Interview Edge:**  
> In systems interviews, explain that `pnpm` provides **reproducible isolation** by preventing phantom dependencies and saves disk space via filesystem hard-linking into a global Content-Addressable Store.

---

### Step 4: Create the Turborepo Scaffold

Run:
```bash
npx create-turbo@latest
```

This interactive CLI pulls down the canonical template, generating a configured root workspace with sample apps, build pipelines, and base configuration packages.

> 🔍 **Deep-Dive Search Topics:**  
> * "Boilerplating and zero-configuration scaffolding in modern toolchains"

---

### Step 5: Navigate into the Project Directory

```bash
cd draw-app
```

Changes your current working directory (CWD) to the newly created root monorepo directory.

---

### Step 6: Open the Workspace in VS Code

```bash
code .
```

Launches VS Code targeting the root directory so that code intelligence, TypeScript language servers, and Git context span the entire monorepo.

---

### Step 7: Explore the Monorepo File System

The layout follows a standardized structure:
```text
draw-app/
├── apps/                        # Deployable services and applications
│   ├── docs/                    # Auxiliary documentation app
│   └── web/                     # User-facing frontend (Next.js)
├── packages/                    # Internal shared codebases and configs
│   ├── eslint-config/           # Centralized lint rules
│   ├── typescript-config/       # Centralized tsconfig JSON files
│   └── ui/                      # Shared component library
├── package.json                 # Monorepo root manifest
├── pnpm-lock.yaml               # Deterministic dependency resolution tree
├── pnpm-workspace.yaml          # Defines monorepo inclusion patterns
└── turbo.json                   # Pipeline definitions and caching boundaries
```

* **Root `package.json`:** Declares shared development tools (Turborepo, Prettier) and orchestration scripts across the whole repository.
* **`pnpm-workspace.yaml`:** Identifies which folders contain active projects:
  ```yaml
  packages:
    - "apps/*"
    - "packages/*"
  ```

> 🔍 **Deep-Dive Search Topics:**  
> * "Monorepo project topology and boundary encapsulation"

---

### Step 8: Understand What Packages Are in a Monorepo

In Node.js, a **package** is any folder containing a valid `package.json` manifest. In a monorepo, packages are divided into:
1. **Applications (`apps/*`):** Standalone services that are built, bundled, and deployed (e.g., frontend SPAs, backend APIs).
2. **Shared Modules (`packages/*`):** Non-deployable code consumed by applications (e.g., database clients, utility functions, validation schemas).

Using internal packages eliminates copy-pasting code between your frontend and backend, ensuring shared contracts (like drawing event payloads or user shapes) stay synchronized.

> 🔍 **Deep-Dive Search Topics:**  
> * "Domain-Driven Design (DDD) shared kernel patterns"

---

### Step 9: Install Dependencies Across the Workspace

```bash
pnpm install
```

When run at the root, `pnpm`:
1. Scans `pnpm-workspace.yaml` to discover all internal packages.
2. Resolves all remote dependencies against the npm registry.
3. Builds an internal dependency graph to link packages locally using symlinks.
4. Updates `pnpm-lock.yaml` to ensure deterministic builds across development and production environments.

> 🔍 **Deep-Dive Search Topics:**  
> * "Deterministic builds and lockfile mechanics"

---

### Step 10: Run the Development Pipeline

```bash
pnpm run dev
```

This invokes Turborepo's `dev` pipeline defined in `turbo.json`. By default, it concurrently launches:
* `apps/web` on TCP port `3000`
* `apps/docs` on TCP port `3001`

#### Why Are Two Different Ports Required?
A **TCP Port** is an unsigned 16-bit number ($0 \text{ to } 65535$) identifying a specific communication endpoint on an operating system. A single IP address and port combination can only be bound by one process listening for incoming packets at a time. Running two web servers simultaneously on the same host requires assigning a distinct port to each.

> 🔍 **Deep-Dive Search Topics:**  
> * "OSI Model Layer 4 (Transport) vs Layer 7 (Application)"  
> * "TCP Sockets and socket binding (`bind()`, `listen()` system calls)"

> 🎯 **Interview Edge:**  
> In networking questions, explain that a TCP socket pair consists of four values: `(Source IP, Source Port, Destination IP, Destination Port)`. On a local development machine, distinct destination ports allow the OS network stack to route packets to the correct application process.

---

### Step 11: Clean Up Boilerplate and Add Custom Backends

Delete `apps/docs` because we are building a collaborative drawing app, not a documentation portal:
```bash
rm -rf apps/docs
```

Our architecture requires three deployable applications:
1. `apps/web`: Next.js / React frontend providing the drawing canvas.
2. `apps/http-backend`: Express REST API for stateless operations.
3. `apps/ws-backend`: Low-latency WebSocket service for stateful real-time synchronization.

> 🔍 **Deep-Dive Search Topics:**  
> * "Separation of Concerns in distributed architectures"

---

### Step 12: Organize the `apps` Directory

Create the backend application folders:
```bash
mkdir -p apps/http-backend apps/ws-backend
```

Your `apps` directory now contains:
```text
apps/
├── http-backend/
├── ws-backend/
└── web/
```

---

### Step 13: Create Initial `package.json` Files for Backends

Every directory inside `apps/` must contain a `package.json` to be recognized by `pnpm` and `turbo`.

#### `apps/http-backend/package.json`
```json
{
  "name": "http-backend",
  "version": "1.0.0",
  "private": true,
  "type": "module"
}
```

#### `apps/ws-backend/package.json`
```json
{
  "name": "ws-backend",
  "version": "1.0.0",
  "private": true,
  "type": "module"
}
```

* `"private": true`: Prevents the package from being accidentally published to the public npm registry.
* `"type": "module"`: Enables ECMAScript Modules (`import`/`export` syntax) instead of legacy CommonJS (`require()`).

> 🔍 **Deep-Dive Search Topics:**  
> * "CommonJS (CJS) vs ECMAScript Modules (ESM) in Node.js"

---

### Step 14: System Architecture Overview

```text
                  ┌─────────────────────────────────────────────────────┐
                  │                 Client (apps/web)                   │
                  └──────────────┬────────────────────────┬─────────────┘
                                 │                        │
                  HTTP/1.1 (JSON)│                        │ Full-Duplex TCP
                    (Port 3001)  │                        │  (Port 8080)
                                 ▼                        ▼
                   ┌────────────────────────┐  ┌────────────────────────┐
                   │   apps/http-backend    │  │    apps/ws-backend     │
                   │   (Express REST API)   │  │   (Raw WebSocket WS)   │
                   └─────────────┬──────────┘  └──────────┬─────────────┘
                                 │                        │
                                 │   SQL Queries / ORM    │
                                 └───────────┬────────────┘
                                             ▼
                                ┌────────────────────────┐
                                │   PostgreSQL Database  │
                                └────────────────────────┘
```

---

### Step 15: Understand the Roles of the Frontend, HTTP, and WebSocket Layers

A common architecture mistake is handling both REST requests and WebSocket connections within a single server instance. Separating them provides clear operational boundaries:

1. **Stateless HTTP API (`http-backend`):**
   * Handles `/signup`, `/signin`, `/create-room`, and user profile endpoints.
   * **Statelessness:** The server does not maintain an active connection with the client between requests. The client sends a request, receives a response, and the connection closes or enters an idle keep-alive state.
   * Easily scalable behind a standard round-robin load balancer.

2. **Stateful WebSocket Server (`ws-backend`):**
   * Handles real-time canvas events (e.g., drawing lines, moving shapes, room broadcasts).
   * **Statefulness:** The server maintains an open, long-lived TCP socket with every connected client and holds in-memory maps of active rooms and socket connections.

3. **Frontend Client (`apps/web`):**
   * Uses HTML5 Canvas / SVG to render shapes locally.
   * Sends infrequent actions (authentication, room creation) over HTTP REST.
   * Streams cursor movements and vector paths over a long-lived WebSocket connection.

> 🔍 **Deep-Dive Search Topics:**  
> * "Stateful vs Stateless Microservices"  
> * "Real-time canvas architectures and vector path serialization"

---

### Step 16: Understand Database Interfacing

Both backends talk to a single shared **PostgreSQL database**:
* `http-backend` queries PostgreSQL to authenticate users, hash passwords, verify permissions, and store room metadata.
* `ws-backend` queries PostgreSQL to verify room membership tokens and periodically persists canvas states so rooms can be reloaded later.

> 🔍 **Deep-Dive Search Topics:**  
> * "ACID properties in Relational Database Management Systems"  
> * "Database Connection Pooling in distributed Node.js services"

> 🎯 **Interview Edge:**  
> When discussing database access from stateful servers, explain that high-throughput WebSocket servers should avoid writing every individual mouse-movement coordinate directly to the database. Instead, coordinate changes in memory, broadcast them over sockets, and write snapshots or operation logs to the database in batches.

---

### Step 17: Initialize the Backend Source Directories

Create the source directory structure for both services:
```bash
mkdir -p apps/http-backend/src apps/ws-backend/src
```

Keeping code inside `src/` separates your input TypeScript source files from the compiled JavaScript artifacts output to `dist/`.

---

### Step 18: Understand Shared TypeScript Configurations

In a monorepo, you avoid copying configuration files across services. The Turborepo template includes a shared package: `packages/typescript-config/`.

This package exposes reusable presets, such as:
* `base.json`: Foundational compiler rules (e.g., `strict: true`, `target: ES2022`).
* `nextjs.json`: Rules tailored for Next.js applications.
* `node16.json`: Compiler options optimized for Node.js runtimes.

> 🔍 **Deep-Dive Search Topics:**  
> * "TypeScript Project References and Monorepo configurations"

---

### Step 19: Extend the Base TypeScript Configuration

Create a `tsconfig.json` in both backends:
* `apps/http-backend/tsconfig.json`
* `apps/ws-backend/tsconfig.json`

Add the following configuration to both files:
```json
{
  "extends": "@repo/typescript-config/base.json",
  "compilerOptions": {
    "rootDir": "./src",
    "outDir": "./dist"
  },
  "include": ["src/**/*"]
}
```

* `"extends"`: Inherits all compiler checks (type safety, downleveling, strict null checks) from the shared package.
* `"rootDir": "./src"`: Tells the compiler to look for input TypeScript source code only in `./src`.
* `"outDir": "./dist"`: Directs the compiler to output transpiled JavaScript to `./dist`.

---

### Step 20: Link the Shared Config via Workspace Protocols

In both `apps/http-backend/package.json` and `apps/ws-backend/package.json`, add the shared package as a development dependency:

```json
{
  "devDependencies": {
    "@repo/typescript-config": "workspace:*"
  }
}
```

#### What is `workspace:*`?
Normally, package managers look up dependencies on the public npm registry. The `workspace:*` protocol informs `pnpm` that `@repo/typescript-config` is an internal repository package. `pnpm` creates a symlink directly to `packages/typescript-config`, ensuring changes take effect immediately without publishing to npm.

> 🔍 **Deep-Dive Search Topics:**  
> * "pnpm Workspace protocol mechanics"

---

### Step 21: Link Dependencies with `pnpm install`

Whenever you update a `package.json` with new dependencies or workspace references, run:
```bash
pnpm install
```
This updates the symlink graph in `node_modules` so the TypeScript compiler can resolve `@repo/typescript-config/base.json`.

---

### Step 22: Configure Build, Dev, and Start Scripts

Add lifecycle scripts to both `apps/http-backend/package.json` and `apps/ws-backend/package.json`:

```json
{
  "scripts": {
    "build": "tsc -b",
    "start": "node ./dist/index.js",
    "dev": "tsc -b && node ./dist/index.js"
  }
}
```

---

### Step 23: Deconstruct the Execution Commands

* `"build": "tsc -b"`:
  * Invokes the TypeScript compiler in **build mode** (`-b`). Build mode leverages project references to compile changed dependencies incrementally, caching output state to speed up monorepo build runs.
* `"start": "node ./dist/index.js"`:
  * Runs the compiled JavaScript file directly using the Node.js runtime.
* `"dev": "tsc -b && node ./dist/index.js"`:
  * Chained via the POSIX `&&` operator: compiles the source code first, and only runs the resulting output if compilation succeeds without errors.

> 🔍 **Deep-Dive Search Topics:**  
> * "TypeScript incremental compilation flags and build mode (`-b`)"  
> * "Node.js V8 execution pipeline"

---

### Step 24: Validate Path Mappings in `tsconfig.json`

Confirm that the output and root paths in `apps/http-backend/tsconfig.json` match the execution targets:
```json
{
  "compilerOptions": {
    "rootDir": "./src",
    "outDir": "./dist"
  }
}
```
If `rootDir` is `./src`, compiling `apps/http-backend/src/index.ts` outputs the build artifact directly to `apps/http-backend/dist/index.js`. A mismatch here causes `node ./dist/index.js` to fail with a `MODULE_NOT_FOUND` error.

---

### Step 25: Verify the Turborepo Task Pipeline

Inspect `turbo.json` in the project root to ensure it includes declarations for the `build` and `dev` tasks:

```json
{
  "$schema": "https://turbo.build/schema.json",
  "tasks": {
    "build": {
      "dependsOn": ["^build"],
      "outputs": ["dist/**", ".next/**", "!.next/cache/**"]
    },
    "dev": {
      "cache": false,
      "persistent": true
    }
  }
}
```

* `"dependsOn": ["^build"]`: The caret (`^`) denotes a **topological dependency**. Turborepo guarantees that all upstream dependencies finish their `build` tasks before the downstream package builds.
* `"outputs": ["dist/**"]`: Tells Turborepo which compiled directories to cache on disk.
* `"cache": false`: Prevents long-running development servers from caching outputs, ensuring hot-reloading runs on every execution.

> 🔍 **Deep-Dive Search Topics:**  
> * "Topological sorting in build graphs"  
> * "Idempotency in continuous integration pipelines"

---

### Step 26: Prepare the Backend Services

With the project structure and build configurations complete, we now install the networking libraries for each service:
* An HTTP framework for `http-backend`.
* A WebSocket library for `ws-backend`.

---

### Step 27: Understand and Install Express

#### What is Express?
Raw Node.js provides a low-level `http` module (`import http from "node:http"`). Using it directly requires manually parsing URL query strings, reading streaming raw byte buffers from the TCP socket, and formatting response headers.

**Express** provides a middleware-driven abstraction over Node's HTTP primitives:
* **Routing Table:** Maps paths and HTTP methods (`GET`, `POST`, `PUT`, `DELETE`) to handler functions.
* **Middleware Pipeline:** Processes requests sequentially (e.g., JSON body parsing, CORS headers, authentication checks).

#### Install Express and Types
Run the following from the root workspace:
```bash
pnpm --filter http-backend add express
pnpm --filter http-backend add -D @types/express typescript
```

* `--filter http-backend`: Instructs `pnpm` to install packages exclusively into `apps/http-backend`.
* `@types/express`: Ambient type definitions providing autocompletion and type checking for Express APIs.

> 🔍 **Deep-Dive Search Topics:**  
> * "Middleware pattern in web frameworks"  
> * "DefinitelyTyped (`@types`) and ambient type declarations"

---

### Step 28: Implement the Express HTTP Server

Create `apps/http-backend/src/index.ts`:

```typescript
import express from "express";

const app = express();
const PORT = 3001;

app.use(express.json());

app.get("/health", (req, res) => {
  res.json({ status: "ok" });
});

app.listen(PORT, () => {
  console.log(`HTTP server running on http://localhost:${PORT}`);
});
```

* `app.use(express.json())`: A built-in middleware that intercepts incoming requests with a `Content-Type: application/json` header, reads the streaming chunks, parses the payload, and attaches the resulting object to `req.body`.
* `app.listen(PORT, ...)`: Binds the process to the specified TCP port and begins accepting inbound network connections.

---

### Step 29: Implement the WebSocket Server

#### What is the `ws` Library?
Unlike Socket.io, which adds extra abstractions (like custom heartbeat protocols, fallback HTTP long-polling, and packet wrappers), **`ws`** is a fast, lightweight, standards-compliant implementation of the RFC 6455 WebSocket specification.

#### The WebSocket Handshake Mechanism
1. The client initiates a standard HTTP request with upgrade headers:
   ```http
   GET /chat HTTP/1.1
   Host: server.example.com
   Upgrade: websocket
   Connection: Upgrade
   Sec-WebSocket-Key: dGhlIHNhbXBsZSBub25jZQ==
   Sec-WebSocket-Version: 13
   ```
2. The server computes a SHA-1 hash of the key, appends a standard GUID, and responds with:
   ```http
   HTTP/1.1 101 Switching Protocols
   Upgrade: websocket
   Connection: Upgrade
   Sec-WebSocket-Accept: s3pPLMBiTxaQ9kYGzzhZRbK+xOo=
   ```
3. The underlying TCP socket remains open, switching to full-duplex framing where either party can transmit messages at any time with minimal overhead (as little as 2 to 10 bytes of framing headers per message).

#### Install `ws` and Types
From the root workspace:
```bash
pnpm --filter ws-backend add ws
pnpm --filter ws-backend add -D @types/ws typescript
```

#### Minimal WebSocket Starter (`apps/ws-backend/src/index.ts`)
```typescript
import { WebSocketServer } from "ws";

const wss = new WebSocketServer({ port: 8080 });

wss.on("connection", (ws) => {
  ws.send("connected to ws-backend");

  ws.on("message", (data) => {
    console.log("received: %s", data);
  });
});

console.log("WebSocket server listening on ws://localhost:8080");
```

* `new WebSocketServer({ port: 8080 })`: Creates an HTTP server under the hood that automatically upgrades incoming requests to WebSocket connections on port `8080`.
* `wss.on("connection", (ws) => { ... })`: An event listener that fires whenever a client completes the protocol handshake. The `ws` instance represents the direct, full-duplex connection to that specific client.

> 🔍 **Deep-Dive Search Topics:**  
> * "RFC 6455 WebSocket Protocol Specification"  
> * "TCP Frame Masking and Framing bit mechanics in WebSockets"  
> * "WebSockets vs Server-Sent Events (SSE) vs HTTP/2 WebTransport"

> 🎯 **Interview Edge:**  
> When comparing WebSockets to HTTP polling:
> * **Short Polling:** Repeated HTTP requests at set intervals waste bandwidth due to repeated TCP handshakes and HTTP header overhead.
> * **Long Polling:** The server holds the HTTP request open until data arrives; better latency than short polling, but re-establishing the connection after each response still adds overhead.
> * **WebSockets:** A single persistent TCP connection with minimal frame headers (2–10 bytes), ideal for high-frequency, bidirectional data exchange like real-time canvas coordination.
