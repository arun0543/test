# 📋 Infinite Scroll + Edit-Back-Sync Architecture

> **Stack:** Angular (Frontend) · Node.js (Backend) · PostgreSQL (Database)

---

## 🧩 Problem Statement

In SaaS listing pages with **infinite scroll**, when a user:
1. Scrolls through a list (40 items loaded at a time)
2. Clicks a record → navigates to detail/edit page
3. Makes an edit
4. **Goes back** to the listing page

→ The API reloads from offset `0`, showing the first 40 items — the **edited record may not be visible**, causing poor UX.

---

## 📁 Document Index

| File | Description |
|------|-------------|
| [01-problem-deep-dive.md](./01-problem-deep-dive.md) | Detailed breakdown of the problem |
| [02-solution-strategies.md](./02-solution-strategies.md) | All standard strategies to solve this |
| [03-frontend-angular.md](./03-frontend-angular.md) | Angular implementation details |
| [04-backend-nodejs.md](./04-backend-nodejs.md) | Node.js API design & patterns |
| [05-database-postgres.md](./05-database-postgres.md) | PostgreSQL query strategies |
| [06-architecture-diagram.md](./06-architecture-diagram.md) | Full architecture diagrams (Mermaid) |
| [07-decision-guide.md](./07-decision-guide.md) | When to use which strategy |

---

## 🏗️ High-Level Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                        ANGULAR (Frontend)                    │
│                                                             │
│  ┌──────────────┐    ┌──────────────┐    ┌──────────────┐  │
│  │ Listing Page │◄──►│  State Store │◄──►│  Edit Page   │  │
│  │ (Inf. Scroll)│    │ (NgRx/Signal)│    │ (Detail View)│  │
│  └──────────────┘    └──────────────┘    └──────────────┘  │
│         │                   │                               │
└───────��─┼───────────────────┼───────────────────────────────┘
          │ HTTP              │ Cache Update
          ▼                   ▼
┌─────────────────────────────────────────────────────────────┐
│                      NODE.JS (Backend)                       │
│                                                             │
│  ┌──────────────┐    ┌──────────────┐    ┌──────────────┐  │
│  │  List API    │    │  Cursor API  │    │  Edit API    │  │
│  │  /GET items  │    │  (keyset pg) │    │  PUT /item   │  │
│  └──────────────┘    └──────────────┘    └──────────────┘  │
│         │                   │                   │           │
└─────────┼───────────────────┼───────────────────┼───────────┘
          │                   │                   │
          ▼                   ▼                   ▼
┌─────────────────────────────────────────────────────────────┐
│                     POSTGRESQL (Database)                    │
│                                                             │
│  ┌──────────────┐    ┌──────────────┐    ┌──────────────┐  │
│  │ Header Table │    │  Keyset Idx  │    │ updated_at   │  │
│  │ (summary)    │    │  (cursor pg) │    │  timestamp   │  │
│  └──────────────┘    └──────────────┘    └──────────────┘  │
└─────────────────────────────────────────────────────────────┘
```

---

## ⚡ Recommended Strategy (TL;DR)

| Scenario | Strategy |
|----------|----------|
| Simple list, no heavy filtering | **Client-side cache patch** + scroll position restore |
| Large dataset, sorted by date | **Keyset/Cursor pagination** + updated_at pinning |
| Real-time collaborative edits | **WebSocket / SSE** push updates |
| Complex filters + edits | **Optimistic UI** + background re-fetch of only edited record |

---

## 🔑 Core Principle

> **Never re-fetch the whole list. Patch the in-memory cache with the single edited record, restore scroll position, and use cursor-based pagination to maintain stable ordering.**
