# 3-Tier Architecture

## The Restaurant Analogy

Before diving into technical terms, think of a restaurant:

| Restaurant Role | Responsibility |
|---|---|
| **Captain** | Welcomes you, checks free tables, guides you in |
| **Waiter** | Takes your order, serves food, interface between you and kitchen |
| **Chef** | Only cooks — receives order, prepares food, knows nothing about the customer |
| **Food Store** | Holds all raw ingredients, accessed only by the kitchen |

A software 3-tier architecture works exactly the same way — each layer has a single responsibility and talks only to the layer next to it.

---

## The Three Tiers

```
User (Browser)
     │
     ▼
┌─────────────────────┐
│   Tier 1: Frontend  │  ← Waiter (takes request, presents response)
│   HTML, CSS, JS     │
│   Served via Nginx  │
└─────────────────────┘
     │
     ▼
┌─────────────────────┐
│   Tier 2: Backend   │  ← Chef (processes logic, knows the rules)
│   Node.js / Java /  │
│   Python / Go       │
└─────────────────────┘
     │
     ▼
┌─────────────────────┐
│  Tier 3: Database   │  ← Food Store (holds all the data)
│  MySQL / PostgreSQL │
│  MongoDB / Redis    │
└─────────────────────┘
```

### Tier 1 — Frontend (Web / HTTP Server)

- Delivers the UI to the browser: HTML, CSS, JavaScript
- **Stateless** — does not store any user data; every request is independent
- Common servers: **Nginx**, Apache, IIS
- The frontend never talks directly to the database

### Tier 2 — Backend (Application Server)

- Contains the business logic — validations, calculations, rules
- Accepts requests from the frontend, processes them, queries the database
- **Stateless** — scales horizontally; any instance can handle any request
- Common languages: **Node.js**, Java, Python, .NET, Go, PHP
- Historically ran inside application servers (JBoss, WebSphere, Tomcat); modern microservices have the server built into the code itself

### Tier 3 — Database (Database Server)

- Stores and retrieves persistent data
- **Stateful** — data must survive restarts; cannot be duplicated carelessly
- Relational: **MySQL**, PostgreSQL, MSSQL, Oracle
- Non-relational: MongoDB, Redis, Kafka, RabbitMQ, ELK

---

## Stateless vs Stateful

| | Stateless | Stateful |
|---|---|---|
| **Tiers** | Frontend, Backend | Database |
| **Scaling** | Easy — add more servers behind a load balancer | Hard — data must be consistent across nodes |
| **Restart impact** | None | Data could be lost without persistence setup |
| **Examples** | Nginx, Node.js API | MySQL, MongoDB |

---

## Client-Server Model

Every tier follows the client-server model:

```
Client ──────────► Server
(makes request)    (responds)

Browser     ──►  Frontend (Nginx)
Frontend    ──►  Backend  (Node.js)
Backend     ──►  Database (MySQL)
```

The browser is a client to Nginx. Nginx is a client to Node.js. Node.js is a client to MySQL. Each layer only knows about its immediate neighbour.

---

## Our Project — Expense Tracker on AWS

Three separate EC2 instances, one per tier:

```
Internet
   │
   ▼
┌───────────────────────────────┐
│  Frontend EC2  (Port 80)      │
│  Nginx — serves HTML/CSS/JS   │
│  Reverse proxy /api/ ──────►──┼──┐
└───────────────────────────────┘  │
                                   ▼
                    ┌──────────────────────────────┐
                    │  Backend EC2  (Port 8080)    │
                    │  Node.js — REST API          │
                    │  Validates input, CRUD ops   │
                    └──────────────────────────────┘
                                   │
                                   ▼
                    ┌──────────────────────────────┐
                    │  Database EC2  (Port 3306)   │
                    │  MySQL 8                     │
                    │  Schema: transactions        │
                    │  Table: id, amount,          │
                    │         description,category │
                    └──────────────────────────────┘
```

### Technology Stack

| Tier | Technology | EC2 Port |
|---|---|---|
| Frontend | Nginx (static files + reverse proxy) | 80 |
| Backend | Node.js + Express | 8080 |
| Database | MySQL 8 | 3306 |

### Security Group Rules

| EC2 | Allow inbound from |
|---|---|
| Frontend | Internet (0.0.0.0/0) on port 80 |
| Backend | Frontend EC2 security group on port 8080 |
| Database | Backend EC2 security group on port 3306 |

The database is never exposed to the internet. The backend port is never exposed to the internet. Only port 80 is public.

### Request Flow

```
1. User opens browser → http://<frontend-public-ip>
2. Nginx serves index.html, style.css, app.js
3. app.js calls GET /api/transaction
4. Nginx reverse proxy forwards to http://<backend-private-ip>:8080/transaction
5. Node.js queries MySQL → SELECT * FROM transactions
6. Result flows back: MySQL → Node.js → Nginx → Browser
```

---

## Why Separate Tiers?

| Reason | Explanation |
|---|---|
| **Security** | Database is never directly reachable from the internet |
| **Scalability** | Each tier scales independently — add more backend EC2s behind a load balancer without touching the DB |
| **Maintainability** | Teams own separate tiers — frontend team, backend team, DBA team |
| **Fault isolation** | A crashed backend does not take down the database |
| **Technology freedom** | Swap Nginx for Apache, or MySQL for PostgreSQL, without rewriting other tiers |
