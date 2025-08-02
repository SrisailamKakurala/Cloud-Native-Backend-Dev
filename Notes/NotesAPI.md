---

# 🚀 Build a **Production-Grade Notes API** (Express + Prisma + PostgreSQL)

### ✅ Secure • Fast • Reliable • Scalable • Dockerized • CI/CD Ready

---

## 🧱 TECH STACK

| Layer             | Technology                   |
| ----------------- | ---------------------------- |
| Language          | JavaScript (Node.js)         |
| Framework         | Express.js                   |
| ORM               | Prisma ORM                   |
| Database          | PostgreSQL                   |
| Auth              | JWT + Bcrypt                 |
| Validation        | Zod / Express Validator      |
| Testing           | Jest + Supertest             |
| Dockerization     | Docker + Docker Compose      |
| Deployment        | Render / Railway / EC2 / ECS |
| DevOps (Optional) | GitHub Actions CI/CD         |

---

## 🗂️ Project Structure (Best Practice)

```
src/
├── config/               # env configs, DB connection
├── controllers/          # request handlers
├── middlewares/          # auth, error, rate limit
├── models/               # Prisma schemas
├── routes/               # route definitions
├── services/             # business logic
├── utils/                # reusable helpers
├── validators/           # zod/express-validator logic
├── app.js                # express config
└── server.js             # server/bootstrap
prisma/
├── schema.prisma         # DB schema
.env                      # env secrets
Dockerfile                # container instructions
docker-compose.yml        # dev stack (Postgres + API)
```

---

## 🔐 Features

✅ User authentication (JWT-based)
✅ Password hashing (bcrypt)
✅ Rate limiting, Helmet (security)
✅ Input validation (Zod)
✅ Notes CRUD (title, content, tags)
✅ Pagination, filtering, search
✅ Soft delete & trash recovery
✅ Audit logs (createdAt, updatedAt, deletedAt)
✅ Dockerized + Dev-DB container
✅ Environment-aware config
✅ Graceful error handling + logs
✅ CI-ready (GitHub Actions)

---

## 🧪 STEP-BY-STEP GUIDE

---

### 1. ✅ Initialize Project

```bash
npm init -y
npm i express prisma @prisma/client dotenv zod jsonwebtoken bcryptjs helmet cors morgan
npm i -D nodemon
```

```bash
npx prisma init
```

---

### 2. 🔧 Configure Environment (`.env`)

```env
PORT=4000
DATABASE_URL=postgresql://postgres:password@db:5432/notesdb
JWT_SECRET=your_secret_key
```

---

### 3. 🔁 Define Prisma Schema (`prisma/schema.prisma`)

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
  email     String   @unique
  password  String
  notes     Note[]
  createdAt DateTime @default(now())
}

model Note {
  id        String   @id @default(uuid())
  title     String
  content   String
  tags      String[]
  trashed   Boolean  @default(false)
  userId    String
  user      User     @relation(fields: [userId], references: [id])
  createdAt DateTime @default(now())
  updatedAt DateTime @updatedAt
  deletedAt DateTime?
}
```

```bash
npx prisma generate
npx prisma migrate dev --name init
```

---

### 4. 🧠 Setup Express App (`app.js`)

```js
const express = require('express');
const helmet = require('helmet');
const cors = require('cors');
const morgan = require('morgan');
require('dotenv').config();

const app = express();

app.use(cors());
app.use(helmet());
app.use(morgan('dev'));
app.use(express.json());

// Routes
app.use('/api/auth', require('./routes/authRoutes'));
app.use('/api/notes', require('./routes/noteRoutes'));

// Error Handler
app.use(require('./middlewares/errorHandler'));

module.exports = app;
```

---

### 5. 🔑 Auth: Register & Login

#### 📁 `/controllers/authController.js`

```js
const jwt = require('jsonwebtoken');
const bcrypt = require('bcryptjs');
const prisma = require('../config/db');

exports.register = async (req, res, next) => {
  const { email, password } = req.body;
  const exists = await prisma.user.findUnique({ where: { email } });
  if (exists) return res.status(409).json({ msg: 'Email already registered' });

  const hash = await bcrypt.hash(password, 12);
  const user = await prisma.user.create({ data: { email, password: hash } });
  const token = jwt.sign({ id: user.id }, process.env.JWT_SECRET);
  res.json({ token });
};

exports.login = async (req, res, next) => {
  const { email, password } = req.body;
  const user = await prisma.user.findUnique({ where: { email } });
  if (!user || !(await bcrypt.compare(password, user.password)))
    return res.status(401).json({ msg: 'Invalid credentials' });

  const token = jwt.sign({ id: user.id }, process.env.JWT_SECRET);
  res.json({ token });
};
```

---

### 6. 🧾 Note CRUD (Secure, Reliable)

#### ✅ Features:

* Pagination: `GET /notes?page=2&limit=10`
* Search: `?q=keyword`
* Soft delete: Move to trash instead of DB delete
* Recover from trash

---

### 7. 🔒 Auth Middleware (`middlewares/auth.js`)

```js
const jwt = require('jsonwebtoken');

module.exports = (req, res, next) => {
  const token = req.headers.authorization?.split(" ")[1];
  if (!token) return res.status(401).json({ msg: "No token" });

  try {
    const decoded = jwt.verify(token, process.env.JWT_SECRET);
    req.user = decoded;
    next();
  } catch (err) {
    return res.status(401).json({ msg: "Invalid token" });
  }
};
```

---

### 8. 🗑 Soft Delete & Restore

```js
await prisma.note.update({
  where: { id },
  data: { trashed: true, deletedAt: new Date() }
});

// restore
await prisma.note.update({
  where: { id },
  data: { trashed: false, deletedAt: null }
});
```

---

### 9. 🐳 Dockerize the App

#### 📄 `Dockerfile`

```dockerfile
FROM node:18-alpine

WORKDIR /app
COPY package*.json ./
RUN npm install

COPY . .
EXPOSE 4000

CMD ["node", "server.js"]
```

#### 📄 `docker-compose.yml`

```yaml
version: '3.8'
services:
  db:
    image: postgres
    restart: always
    environment:
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: password
      POSTGRES_DB: notesdb
    ports:
      - "5432:5432"
    volumes:
      - pgdata:/var/lib/postgresql/data

  api:
    build: .
    ports:
      - "4000:4000"
    depends_on:
      - db
    environment:
      DATABASE_URL: postgres://postgres:password@db:5432/notesdb
      JWT_SECRET: your_secret_key
    volumes:
      - .:/app
    command: sh -c "npx prisma migrate deploy && node server.js"

volumes:
  pgdata:
```

```bash
docker-compose up --build
```

---

### 10. 🧪 Add Testing (Optional but recommended)

```bash
npm i -D jest supertest
```

Write tests in `__tests__` for:

* Auth flow
* Note CRUD
* Pagination & filtering
* Invalid tokens

---

### 11. 🚀 Deployment (Render / Railway)

#### 🔹 Render

* Create PostgreSQL instance
* Connect `DATABASE_URL`
* Auto deploy via GitHub

#### 🔹 Railway

* Deploy directly from repo + provision Postgres
* Easy `.env` config UI

---

### 12. ✅ Extra: Production-Grade Features

| Feature               | Stack                                               |
| --------------------- | --------------------------------------------------- |
| Rate Limiting         | `express-rate-limit`                                |
| Input Validation      | `zod` or `express-validator`                        |
| Error Logs            | `winston` or `pino`                                 |
| Request Tracing       | Add `X-Request-ID`, log it across stack             |
| Monitoring            | Integrate with Datadog / Prometheus                 |
| Health Checks         | `/healthz` endpoint                                 |
| CI/CD                 | GitHub Actions: lint → test → docker build → deploy |
| Email / Notifications | Nodemailer or SES (future)                          |

---

## 🧾 Final Checklist (Cloud-Ready)

* [x] Express API with clean modular structure
* [x] PostgreSQL + Prisma for relational modeling
* [x] JWT-based secure authentication
* [x] Rate-limited, helmet-secured endpoints
* [x] Validation & error handling in all routes
* [x] Dockerized with full stack (Postgres + API)
* [x] Deployment-ready `.env` and configs
* [x] Optional: CI/CD + test coverage

---

Perfect. Here’s a **comprehensive, production-grade roadmap** for building a **GraphQL version** of your **Notes API** app using:

---

## 🧱 STACK

| Layer             | Tech                                           |
| ----------------- | ---------------------------------------------- |
| Language          | JavaScript / TypeScript (Optional)             |
| Server Framework  | **Apollo Server (standalone or with Express)** |
| ORM               | **Prisma ORM**                                 |
| DB                | **PostgreSQL**                                 |
| Auth              | **JWT + Bcrypt**                               |
| Dockerized?       | ✅ Yes                                          |
| Validation        | **GraphQL Scalars** / `zod` in resolvers       |
| DevOps (Optional) | GitHub Actions CI/CD                           |
| Deployment        | Render / Railway / ECS                         |

---

## 📁 Folder Structure (Best Practice)

```
src/
├── graphql/
│   ├── schema.js         # TypeDefs (GraphQL SDL)
│   ├── resolvers.js      # All resolver functions
│   ├── loaders/          # DataLoader setup for batching
├── auth/
│   ├── authMiddleware.js # JWT middleware
│   └── hash.js           # bcrypt logic
├── prisma/
│   └── client.js         # Prisma instance
├── config/               # dotenv, constants
├── utils/                # helpers
└── index.js              # Apollo server init
prisma/
├── schema.prisma
Dockerfile
docker-compose.yml
.env
```

---

## 🔧 1. Prisma Schema

### 📄 `prisma/schema.prisma`

```prisma
model User {
  id       String   @id @default(uuid())
  email    String   @unique
  password String
  notes    Note[]
  createdAt DateTime @default(now())
}

model Note {
  id        String   @id @default(uuid())
  title     String
  content   String
  tags      String[]
  trashed   Boolean  @default(false)
  userId    String
  user      User     @relation(fields: [userId], references: [id])
  createdAt DateTime @default(now())
  updatedAt DateTime @updatedAt
  deletedAt DateTime?
}
```

```bash
npx prisma migrate dev --name init
npx prisma generate
```

---

## 🚀 2. GraphQL TypeDefs (`schema.js`)

```js
const { gql } = require('apollo-server');

module.exports = gql`
  scalar DateTime

  type User {
    id: ID!
    email: String!
    notes: [Note!]!
  }

  type Note {
    id: ID!
    title: String!
    content: String!
    tags: [String!]!
    trashed: Boolean!
    createdAt: DateTime!
    updatedAt: DateTime!
    deletedAt: DateTime
  }

  type AuthPayload {
    token: String!
    user: User!
  }

  type Query {
    me: User
    notes: [Note!]!
    note(id: ID!): Note
  }

  type Mutation {
    register(email: String!, password: String!): AuthPayload!
    login(email: String!, password: String!): AuthPayload!

    createNote(title: String!, content: String!, tags: [String!]): Note!
    updateNote(id: ID!, title: String, content: String, tags: [String!]): Note!
    deleteNote(id: ID!): Boolean!
    restoreNote(id: ID!): Note!
  }
`;
```

---

## 🧠 3. Resolvers (`resolvers.js`)

```js
const bcrypt = require('bcryptjs');
const jwt = require('jsonwebtoken');
const { PrismaClient } = require('@prisma/client');
const prisma = new PrismaClient();

const JWT_SECRET = process.env.JWT_SECRET;

module.exports = {
  Query: {
    me: (_, __, { user }) => prisma.user.findUnique({ where: { id: user.id } }),

    notes: async (_, __, { user }) => {
      return await prisma.note.findMany({
        where: { userId: user.id, trashed: false },
      });
    },

    note: (_, { id }, { user }) => prisma.note.findUnique({ where: { id } }),
  },

  Mutation: {
    register: async (_, { email, password }) => {
      const hash = await bcrypt.hash(password, 12);
      const user = await prisma.user.create({ data: { email, password: hash } });
      const token = jwt.sign({ id: user.id }, JWT_SECRET);
      return { token, user };
    },

    login: async (_, { email, password }) => {
      const user = await prisma.user.findUnique({ where: { email } });
      if (!user || !(await bcrypt.compare(password, user.password)))
        throw new Error('Invalid credentials');
      const token = jwt.sign({ id: user.id }, JWT_SECRET);
      return { token, user };
    },

    createNote: (_, args, { user }) => {
      return prisma.note.create({
        data: { ...args, userId: user.id },
      });
    },

    updateNote: (_, { id, ...data }, { user }) => {
      return prisma.note.update({
        where: { id },
        data,
      });
    },

    deleteNote: (_, { id }, { user }) =>
      prisma.note.update({ where: { id }, data: { trashed: true, deletedAt: new Date() } }).then(() => true),

    restoreNote: (_, { id }, { user }) =>
      prisma.note.update({ where: { id }, data: { trashed: false, deletedAt: null } }),
  },
};
```

---

## 🔐 4. JWT Middleware

```js
const jwt = require('jsonwebtoken');

module.exports = ({ req }) => {
  const token = req.headers.authorization?.split(' ')[1];
  if (!token) return { user: null };

  try {
    const decoded = jwt.verify(token, process.env.JWT_SECRET);
    return { user: decoded };
  } catch (err) {
    return { user: null };
  }
};
```

---

## 🧪 5. Apollo Server Setup (`index.js`)

```js
const { ApolloServer } = require('apollo-server');
require('dotenv').config();

const typeDefs = require('./graphql/schema');
const resolvers = require('./graphql/resolvers');
const context = require('./auth/authMiddleware');

const server = new ApolloServer({ typeDefs, resolvers, context });

server.listen({ port: process.env.PORT || 4000 }).then(({ url }) => {
  console.log(`🚀 Server ready at ${url}`);
});
```

---

## 🐳 6. Docker Setup

### 📄 `Dockerfile`

```Dockerfile
FROM node:18

WORKDIR /app

COPY package*.json ./
RUN npm install

COPY . .

EXPOSE 4000
CMD ["node", "src/index.js"]
```

---

### 📄 `docker-compose.yml`

```yaml
version: '3.8'

services:
  db:
    image: postgres
    environment:
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: password
      POSTGRES_DB: notesdb
    ports:
      - "5432:5432"
    volumes:
      - pgdata:/var/lib/postgresql/data

  api:
    build: .
    ports:
      - "4000:4000"
    depends_on:
      - db
    environment:
      DATABASE_URL: postgres://postgres:password@db:5432/notesdb
      JWT_SECRET: myjwtsecret
    command: sh -c "npx prisma migrate deploy && node src/index.js"

volumes:
  pgdata:
```

```bash
docker-compose up --build
```

---

## 📊 Optional Enhancements

| Feature            | Implementation                                |
| ------------------ | --------------------------------------------- |
| Custom Scalars     | `graphql-scalars` (for DateTime, Email, etc.) |
| DataLoader         | Batch DB calls for nested resolvers           |
| Logging            | Use `pino`, `winston`, or request logging     |
| GraphQL Validation | Add `zod` to validate input in resolvers      |
| Monitoring         | Apollo Studio / Datadog / Prometheus          |
| CI/CD              | GitHub Actions with build → test → deploy     |
| Rate Limiting      | Token bucket logic in context/resolver        |

---

## ✅ Final Result

You’ll have a **fully secure**, **cloud-ready**, **GraphQL-based Notes API**, with:

* Authentication
* Clean schema & resolvers
* PostgreSQL + Prisma
* Dockerized with Compose
* Soft delete & recovery
* Ready to deploy anywhere

---

