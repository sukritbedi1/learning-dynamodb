# Learning DynamoDB

Comprehensive guide to Amazon DynamoDB: from fundamentals to production patterns.

**Modules:** 16 lectures covering fundamentals, architecture, operations, and advanced topics.

## Structure

```
├── docs/         16 college-level lectures
├── tools/        DynamoDB Local setup (ministack + docker-compose)
└── projects/     Go SDK implementation examples (coming soon)
```

## Quick Start

### 1. Read Lectures

Start with [docs/INDEX.md](docs/INDEX.md) for the full curriculum.

- **Module 1 (Fundamentals):** What is DynamoDB, mental model shift, core concepts, data types
- **Module 2 (Architecture):** Partition keys, sort keys, access patterns, secondary indexes
- **Module 3 (Operations):** Consistency, billing, batching, transactions
- **Module 4 (Advanced):** Streams, global tables, monitoring, anti-patterns

### 2. Set Up Local Environment

```bash
cd tools/ministack
docker-compose up -d
```

DynamoDB Local running on `http://localhost:4566`

### 3. Implement with Go

(Coming soon: `projects/` folder with Go SDK examples)

---

**Status:** Lectures complete. Tools configured. Projects in progress.

**Target Audience:** Senior engineers learning DynamoDB (assumes familiarity with SQL databases, Go, AWS basics).

---

## Repo Structure for Reuse

This repo is part of a learning ecosystem:

```
~/Documents/learning/
├── dynamodb/     ← This repo
├── redis/        (future)
└── kafka/        (future)
```

Each topic has: docs/, tools/, projects/. Independent, self-contained, GitHub-ready.

---

**Last Updated:** 2026-09-27
