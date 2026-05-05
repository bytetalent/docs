# Deployment Standards

Cross-stack deployment principles. Actual commands, env-var lists, and CI configs are stack-specific and live in each base or product's `docs/` overlay.

Rules for environment topology, branch promotion, env-var discipline, migration-on-deploy, CI required steps, and observability defaults live in the **`infra/deploy-conventions` skill**. Hosting-platform defaults (Vercel, Cloudflare, Azure, EAS) are in the **`infra/infra-service-defaults` skill**.

---

## Stack-Specific Deploy Commands

This guide lists principles, not commands. Each base or product's `docs/guide-deploy.md` overlay spells out:

- Exact deploy command (`vercel --prod`, `wrangler pages deploy`, etc.)
- Environment-variable checklist for that stack
- Required service accounts and how to provision them
- Rollback procedure
- Migration apply timing (pre-deploy vs. post-deploy)
- Custom domain DNS setup

---

## Related

- `infra/deploy-conventions` skill — full environment topology, branch flow rules, env-var discipline, migration rules, CI steps
- `infra/infra-service-defaults` skill — hosting and observability stack defaults per stack
- `db/drizzle-migration-patterns` skill — migration authoring and apply sequence
- [`guide-arch.md`](guide-arch.md) — Vault secret storage pattern
- [`guide-testing.md`](guide-testing.md) — E2E test cadence around deploys
