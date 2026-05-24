# 🥃 Rum Hunter

A premium, highly automated, zero-cost architecture web application designed to aggregate, normalise, and track rum inventory, pricing, and historical market trends across boutique Australian retailers, independent distilleries, and rum clubs.

By bypassing fragile, site-by-site scraper engines, **Rum Hunter** utilises generalised ingestion queues orchestrated via **n8n**, backed by a local **Large Language Model (LLM)** for intelligent data extraction, alias learning, and narrative description up-cycling.

---

## 📁 Repository Architecture

This project is configured as a lightweight, performance-tuned `pnpm` monorepo workspace to isolate backend ingestion pipelines from frontend client delivery.

```text
rum-bounty/
├── .agents/               # Operational automation agent configurations
├── apps/
│   ├── backend/          # n8n workflows, ingestion automation & local scripts
│   └── frontend/         # Next.js / Nuxt customer-facing web application
├── packages/             # Shared typescript types, utilities, and configuration blocks
├── DESIGN.md             # UI/UX design tokens, layouts, and component specs (For Stitch)
├── SITELIST.md           # Managed target registry of Australian rum merchants
└── SPEC.md               # Core database DDL, LLM prompts, and sanitiser rules (For Antigravity)