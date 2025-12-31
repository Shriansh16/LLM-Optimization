# Database Connection Pool Management

A thread-safe, production-ready connection pool for **SQL Server** using **SQLAlchemy's pooling engine** with **pyodbc**.

---

## Table of Contents

- [Overview](#overview)
- [Why Connection Pooling?](#why-connection-pooling)
- [Architecture](#architecture)
- [How It Works](#how-it-works)
- [Configuration Parameters](#configuration-parameters)
- [Why SQLAlchemy?](#why-sqlalchemy)
- [Usage](#usage)
- [Production Considerations](#production-considerations)

---

## Overview

This module replaces individual database connections with a **managed pool of reusable connections**. Instead of creating a new connection for every database operation (which is expensive), connections are **borrowed from a pool**, used, and then **returned for reuse**.

### Key Specifications

| Specification | Value |
|--------------|-------|
| Base Pool Size | 10 connections |
| Max Overflow | 20 connections |
| Total Capacity | 30 concurrent connections |
| Health Checks | Enabled (pre-ping) |
| Connection Recycling | 1 hour (configurable) |
| Thread Safety | Yes |

---

## Why Connection Pooling?

### The Problem

Creating a database connection is expensive. Each new connection requires:

- TCP handshake with the database server
- TLS/SSL negotiation
- Authentication and authorization
- Session initialization
- Memory allocation on both client and server

**Typical cost per connection:** `~50–200 ms`

### Without Pooling

```
Request 1: [Create 150ms][Query 10ms][Destroy 5ms]
Request 2: [Create 150ms][Query 10ms][Destroy 5ms]
Request 3: [Create 150ms][Query 10ms][Destroy 5ms]

Total Time: 495ms
Overhead: ~94%
```

### With Pooling

```
Initialization (one-time): [Create 10 Connections]

Request 1: [Borrow 0.1ms][Query 10ms][Return 0.1ms]
Request 2: [Borrow 0.1ms][Query 10ms][Return 0.1ms]
Request 3: [Borrow 0.1ms][Query 10ms][Return 0.1ms]

Total Time: ~30.6ms
Overhead: ~2%
```

### Performance Comparison

| Metric | Without Pooling | With Pooling |
|------|-----------------|--------------|
| Connection acquisition | ~150ms | ~0.1ms |
| Requests per second | ~6 | 100+ |
| Memory usage | Spiky | Stable |
| Database server load | High | Low |

---

## Architecture

```
┌───────────────────────────────────────────────┐
│                 APPLICATION                   │
│  Thread 1   Thread 2   Thread 3   ... Thread N│
└──────────┬──────────┬──────────┬─────────────┘
           │          │          │
           └──────────┴──────────┴─────────────┐
                                                ▼
┌───────────────────────────────────────────────┐
│              CONNECTION POOL                  │
│        SQLAlchemy QueuePool                   │
│                                               │
│  Base Pool:   Conn1 ... Conn10                │
│  Overflow:    Conn11 ... Conn30 (on demand)   │
│                                               │
│  Features:                                    │
│   • Thread-safe checkout/checkin              │
│   • Pre-ping health checks                    │
│   • Automatic recycling                      │
└───────────────────────────────────────────────┘
                                                ▼
┌───────────────────────────────────────────────┐
│              pyodbc (ODBC Driver)              │
└───────────────────────────────────────────────┘
                                                ▼
┌───────────────────────────────────────────────┐
│                 SQL SERVER                    │
└───────────────────────────────────────────────┘
```

---

## How It Works

### Connection Lifecycle

1. **Pool Initialization** – Creates `pool_size` connections at startup
2. **Checkout** – Application borrows a connection
3. **Health Check** – `SELECT 1` validation (pre-ping)
4. **Usage** – Queries executed via pyodbc
5. **Check-in** – Connection returned to the pool (not closed)
6. **Recycling** – Old connections replaced after `pool_recycle`

### Overflow Handling

```
Normal Load:
[■■■■■□□□□□] 5 / 10 base connections used

Traffic Spike:
[■■■■■■■■■■] 10 / 10 base connections used
[■■■■■□□□□□] 5 / 20 overflow connections created

After Spike:
[■■■□□□□□□□] 3 / 10 base connections used
Overflow connections destroyed
```

### Pre-Ping Health Check

Before handing out a connection:

1. Retrieve connection from pool
2. Execute `SELECT 1`
3. If valid → return to application
4. If invalid → discard and recreate

This prevents **stale or closed connection errors**.

---

## Configuration Parameters

### `pool_size` (default: `10`)

Number of connections maintained at all times.

```python
pool_size = 10
```

**Guideline:** Set to average concurrent DB operations.

---

### `max_overflow` (default: `20`)

Extra connections allowed during traffic spikes.

```python
max_overflow = 20
```

**Total capacity:** `pool_size + max_overflow = 30`

---

### `pool_timeout` (default: `30` seconds)

Time to wait for a connection when pool is exhausted.

```python
pool_timeout = 30
```

Raises `TimeoutError` if exceeded.

---

### `pool_recycle` (default: `3600` seconds)

Replaces old connections to avoid:

- Firewall idle timeouts
- DB server connection limits
- Memory leaks

```python
pool_recycle = 3600
```

**Recommendation:** `1800` seconds (30 minutes) for cloud environments.

---

### `pool_pre_ping` (default: `True`)

Validates connections before use.

```python
pool_pre_ping = True
```

**Trade-off:** +1–5ms latency, eliminates connection failures.

---

## Why SQLAlchemy?

### Important Clarification

This module uses **SQLAlchemy only for connection pooling**, **not** its ORM.

### What SQLAlchemy Provides (Used)

- `QueuePool` (thread-safe pooling)
- Pre-ping health checks
- Automatic recycling
- Overflow handling

### What SQLAlchemy Does *Not* Provide Here

- ORM models
- Sessions
- Query builder
- Migrations

### Actual Query Execution

```python
connection = engine.raw_connection()  # pyodbc.Connection
cursor = connection.cursor()
cursor.execute("SELECT * FROM users")
results = cursor.fetchall()
```

Pure **pyodbc**, SQLAlchemy only manages the pool.

### Why Not Just pyodbc?

| Feature | pyodbc Alone | pyodbc + SQLAlchemy |
|------|--------------|---------------------|
| Connection reuse | Manual | Built-in |
| Health checks | Manual | `pool_pre_ping` |
| Thread safety | Manual locks | Built-in |
| Recycling | Manual | `pool_recycle` |
| Overflow handling | Manual | `max_overflow` |

**SQLAlchemy Reliability**:
- Released: 2006
- 50M+ downloads/month
- Used by Reddit, Dropbox, Uber, Mozilla

---

## Usage

### Basic Usage

```python
from connection_pool import ConnectionPool

pool = ConnectionPool(
    server="your-server.database.windows.net",
    database="your-database",
    user="your-user",
    password="your-password"
)

pool.initialize()

with pool.get_connection() as conn:
    cursor = conn.cursor()
    cursor.execute("SELECT * FROM users WHERE id = ?", (user_id,))
    result = cursor.fetchone()

pool.dispose()
```

---

### Using Factory Function

```python
from connection_pool import create_connection_pool_from_env

pool = create_connection_pool_from_env()
pool.initialize()
```

---

### Context Manager

```python
with ConnectionPool(server="...", database="...", user="...", password="...") as pool:
    with pool.get_connection() as conn:
        cursor = conn.cursor()
        cursor.execute("SELECT 1")
```

Pool is automatically disposed.

---

### Health Check

```python
if pool.health_check():
    print("Database connection healthy")
else:
    print("Database connection unhealthy")
```

---

### Monitor Pool Status

```python
status = pool.get_pool_status()
print(status)
```

---

## Production Considerations

### Recommended Configuration

```python
pool = ConnectionPool(
    server=config.server,
    database=config.database,
    user=config.user,
    password=config.password,
    pool_size=10,
    max_overflow=20,
    pool_timeout=30,
    pool_recycle=1800,  # 30 minutes
    echo=False          # Never enable in production
)
```

