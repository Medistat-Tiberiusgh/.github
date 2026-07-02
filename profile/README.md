<div align="center">

# Medistat

**A visualization tool for dispensed medicines in Sweden.**

[Live dashboard](https://medistat.tiberiusgh.com) · [API playground](https://medistat.tiberiusgh.com/graphql)

</div>

---

## What is Medistat?

Medistat is a visualization tool for dispensed medicines in Sweden. Search for a drug and see how it is prescribed. Regional differences, age and gender demographics and trends over time. Signed-in users can also keep a personal list of medications to switch between quickly.

The statistics come from open data published by **Socialstyrelsen** (the National Board of Health and Welfare), covering prescription drugs dispensed at Swedish pharmacies since 2006, enriched with narcotic classifications from **Läkemedelsverket** (the Medical Products Agency). New years are ingested automatically as they are published.

The project is split across three repositories:

| Repository                                                      | Stack                                            | Role                                                                                 |
| --------------------------------------------------------------- | ------------------------------------------------ | ------------------------------------------------------------------------------------ |
| **[`graphql`](https://github.com/Medistat-Tiberiusgh/graphql)** | NestJS · Apollo Server · PostgreSQL · TypeScript | GraphQL API serving private user data and aggregated public statistics               |
| **[`web`](https://github.com/Medistat-Tiberiusgh/web)**         | React · Vite · Tailwind · TypeScript             | Single-page dashboard with SVG and ECharts visualizations, built with the help of AI |
| **[`db-etl`](https://github.com/Medistat-Tiberiusgh/db-etl)**   | Python · uv · Docker · PostgreSQL                | ETL pipeline that seeds the prescription dataset and keeps it current                |

---

## Architecture

<img width="4200" height="2888" alt="architecture" src="https://github.com/user-attachments/assets/ef5ff4b5-5d50-48cb-bbe5-6d65ae2ddaca" />

The whole stack runs on a single home server behind a **Cloudflare tunnel**. `cloudflared` opens an outbound connection to Cloudflare's edge, so the home network has no inbound ports forwarded. Cloudflare handles TLS and basic edge protection.

Caddy fans inbound traffic out by path:

| Path       | Routed to                                     |
| ---------- | --------------------------------------------- |
| `/`        | React SPA (nginx)                             |
| `/auth`    | GraphQL API — login exchange                  |
| `/graphql` | GraphQL API — schema, playground, all queries |
| `/deploy`  | Webhook listener — gated with a symmetric key |

Everything runs as containers on one shared Docker network, so the API talks to Postgres by container name. The **webhook listener** is the exception. Webhook listener is a small daemon installed directly on the host. When CI has pushed a fresh image, it POSTs to the webhook and the listener pulls the image and restarts the container. I've choosen to install it as a deamon since it needs to run `docker pull` and `docker compose up -d` and doing that from the container needs mounting the host's Docker socket, which seemed too complex for me to fiddle with with my current knowledge. Installing it as a deamon was more straight forward.

---

## The repositories

### [`graphql`](https://github.com/Medistat-Tiberiusgh/graphql) — the API

A NestJS application for GraphQL endpoint backed by Apollo Server. Postgres is queried with raw SQL via `pg` (no ORM), so queries against the large fact table stay explicit. The schema has two surfaces: public statistics anyone can query, and JWT-protected user data (profile and saved medications). Domain errors carry typed error codes, and anything unexpected is masked before it reaches the client.

[Browse the schema in the playground →](https://medistat.tiberiusgh.com/graphql)

### [`web`](https://github.com/Medistat-Tiberiusgh/web) — the dashboard

A React + TypeScript single-page app built with Vite, styled with Tailwind and shadcn. The charts are a mix of custom SVG (d3) and ECharts. The dashboard revolves around one search bar: pick a drug, optionally narrow by region, and every chart reshapes around the selection.

The app ships as a two-stage Docker image. The first stage builds the static bundle, the second serves it with nginx.

[Open the live dashboard →](https://medistat.tiberiusgh.com)

### [`db-etl`](https://github.com/Medistat-Tiberiusgh/db-etl) — the ETL pipeline

A Python pipeline that loads the Socialstyrelsen dataset into PostgreSQL. The raw CSVs are streamed in chunks, so files larger than available RAM are no problem (my homeserver is an old laptop with limited resources). After the initial seed, an always-on scheduler container checks the Socialstyrelsen API for newly published years and loads them automatically, reporting every run to a Discord webhook. A small sample dataset ships with the repo so the API and tests can run end-to-end quickly and easly.

---

## Dataset

|                         |                                                                                 |
| ----------------------- | ------------------------------------------------------------------------------- |
| **Source**              | Socialstyrelsen — Statistikdatabasen, Läkemedel                                 |
| **Narcotic enrichment** | Läkemedelsverket — NPL (Nationellt produktregister för läkemedel)               |
| **Coverage**            | All human prescription drugs, per year, region, ATC code, gender, and age group |
| **Metrics**             | Number of dispensings, number of patients, dispensings per 1,000 inhabitants    |

> **Worth knowing (from Socialstyrelsen).** The data only covers prescriptions dispensed at pharmacies and no over-the-counter or hospital drugs. Patient counts are not additive across drugs, and one prescription can result in several dispensings. Region is the patient's registered address; age is year of dispensing minus year of birth.

---

## Authentication

Login is done through **GitHub and Google** via OAuth with PKCE. The frontend runs the authorization flow, then hands the code to the backend, which completes the exchange with the provider and issues its own JWT. The API never handles passwords, and any instance can verify a token without a shared session store.

A logged-in user can link the other provider from their profile. If that identity already belongs to another account, the two accounts are merged.

---

## CI/CD

Every push to `main` (for the API and web) builds a Docker image and ships it to production:

1. **Build & push** — the image goes to GHCR, tagged with the commit SHA and `latest`.
2. **Integration tests** — a throwaway Postgres is seeded with the sample data and the write flows run against it, so test data never touches production.
3. **Deploy** — CI POSTs to the deploy webhook and the host pulls the new image.
4. **Smoke test** — read-only queries run against the live API.
5. **Rollback** — if the smoke test fails, a second webhook swaps the previous image back in. No registry pull needed, since the previous image is still on disk.

---

## Testing

API tests are a **[Bruno](https://www.usebruno.com/)** collection stored as plain files next to the code, so they version-control cleanly and run in CI (Postman can't export GraphQL collections as files). The write flows run pre-deploy against a throwaway database, and the read-only queries double as the post-deploy smoke test against production. Every test run creates its own user and deletes it afterwards.

---
