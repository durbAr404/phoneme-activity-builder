# 🎵 Phoneme Activity Builder

A full-stack, database-driven web application for **Speech Pathology educators** to create interactive **phoneme-based** Wordle and Word Search activities using HCE (Harrington, Cox, Evans) phoneme symbols for Australian English.

The project is built across four assessments in CSE3CWA. **Assessment 3** extends the earlier phases with a **data-driven operations dashboard**, **observability metrics**, **saved activity management**, **Playwright end-to-end tests**, **JMeter load testing**, and a **Lighthouse accessibility audit**.

---

## 📋 Project Overview

The Phoneme Activity Builder allows Speech Pathology teachers to:

- **Manage phoneme-based word lists** through a full CRUD interface
- Create **phoneme-based Wordle games** using HCE phoneme symbols
- Generate **phoneme-based Word Search puzzles**
- Export **standalone HTML files** that run offline in any browser
- **Save and reopen activity configurations** from the database
- **Monitor system usage** via a real-time operations dashboard

### Who is this for?

- **Speech Pathology teachers** — create engaging classroom activities
- **Speech Pathology students** — practise phoneme recognition interactively

### Why phonemes?

Australian English uses **HCE (Harrington, Cox, Evans)** broad phoneme symbols. The system stores each phoneme as a **separate database row**, so **multi-character symbols** (e.g., `/tʃ/`, `/iː/`, `/æɪ/`) are handled correctly rather than being split into individual letters.

---

## 🆕 What Assessment 3 Adds

| Feature                     | Assessment 2 | Assessment 3                                                 |
| --------------------------- | ------------ | ------------------------------------------------------------ |
| Observability               | ❌           | ✅ `PageVisit`, `GenerationEvent`, `ActivityLog` models      |
| Dashboard                   | ❌           | ✅ `/dashboard` with live metrics and alerts                 |
| Metrics API                 | ❌           | ✅ `/api/metrics` aggregating usage data                     |
| Time tracking               | ❌           | ✅ Client-side page-visit and dwell-time tracking            |
| Generation tracking         | ❌           | ✅ Success/failure recorded on every HTML export             |
| Saved activities UI         | ❌           | ✅ `/activities` page to browse and reopen configurations    |
| Database-backed healthcheck | ❌           | ✅ `/api/health` pings the DB and returns 503 if unreachable |
| Docker persistent volume    | ❌           | ✅ Named volume keeps the DB across container restarts       |
| Testing                     | ❌           | ✅ Playwright E2E tests + JMeter load tests                  |
| Accessibility               | ❌           | ✅ Lighthouse audit (Accessibility 100)                      |

---

## 🛠️ Tech Stack

| Technology                  | Purpose                                |
| --------------------------- | -------------------------------------- |
| **Next.js 16** (App Router) | Full-stack React framework             |
| **React 19**                | UI library                             |
| **TypeScript**              | Type safety                            |
| **Prisma 6 + SQLite**       | Database ORM and storage               |
| **Zod**                     | Input validation                       |
| **Tailwind CSS**            | Styling                                |
| **Docker**                  | Containerisation                       |
| **Playwright**              | End-to-end testing                     |
| **JMeter**                  | Load testing                           |
| **Lighthouse**              | Accessibility and performance auditing |

---

## 🚀 Getting Started

### Prerequisites

- **Node.js 20+**
- **npm 10+**
- **Docker Desktop** (for containerised mode)
- **Java 17+** (for JMeter, optional)

### Local development

```bash
# 1. Clone the repository
git clone https://github.com/21946247-Durbar/phoneme-activity-builder.git
cd phoneme-activity-builder

# 2. Install dependencies
npm install

# 3. Configure environment
cp .env.example .env
# Default DATABASE_URL is: file:./dev.db

# 4. Apply migrations and create the SQLite database
npx prisma migrate dev

# 5. Seed with 90 phoneme words, default settings, and sample activities
npx prisma db seed

# 6. Run the dev server
npm run dev
```

---

**Open http://localhost:3000**

### Docker

```

# Build the image (multi-stage, includes baked-in seed data)

docker build -t phoneme-activity-builder .

# Run the container with persistent storage

docker compose up -d

# Verify health (DB connectivity is checked)

curl http://localhost:3001/api/health

```

The Docker setup uses:

- A **multi-stage build** (deps → builder → runner) that bakes migrations and seed data into the image

- A **named volume** (phoneme-prisma → /app/prisma) so data survives container restarts

- A **non-root user** for security

- A **healthcheck** that verifies the database is reachable

- Default port mapping 3001:3000 (dev server can stay on 3000)

---

## 🗄️ Database Schema

Nine models under `prisma/schema.prisma`:

#### Core models

- **WordList** — a named collection of words

- **Word** — an English word with its transcription

- **WordPhoneme** — one row per phoneme (symbol + position)

- **Activity** — a saved Wordle or Word Search configuration

- **ActivityWord** — join table linking activities ↔ words

- **Setting** — global app defaults

#### Observability models

- **PageVisit** — records page path, session, and dwell time

- **GenerationEvent** — records every HTML generation attempt (success/fail)

- **ActivityLog** — general event log for alerts and warnings

### Key design decision - multi-character phonemes

Storing phonemes as a delimited string would break for multi-character symbols like `tʃ`, `iː`, `æɪ`. Splitting `tʃɪn` letter-by-letter gives `t`, `ʃ`, `ɪ`, `n` - wrong.

Solution: a `WordPhoneme` table where each phoneme is its own row with an explicit position field. This correctly handles 1-, 2-, and 3-character IPA symbols, preserves order deterministically, and enables clean querying.

```

model WordPhoneme {
id Int @id @default(autoincrement())
wordId Int
symbol String
position Int
word Word @relation(fields: [wordId], references: [id], onDelete: Cascade)

@@unique([wordId, position])
}

```

## 📡 API Endpoints

All endpoints return consistent JSON: `{ success: boolean, data?: T, error?: string, details?: unknown }`.

### Core CRUD

| Method             | Endpoint               |
| ------------------ | ---------------------- |
| GET                | `/api/health`          |
| GET / POST         | `/api/wordlists`       |
| GET / PUT / DELETE | `/api/wordlists/[id]`  |
| GET / POST         | `/api/words`           |
| GET / PUT / DELETE | `/api/words/[id]`      |
| GET / POST         | `/api/activities`      |
| GET / PUT / DELETE | `/api/activities/[id]` |

### Observability (A3)

| Method       | Endpoint                | Purpose                                |
| ------------ | ----------------------- | -------------------------------------- |
| GET          | `/api/metrics`          | Aggregated dashboard metrics + alerts  |
| POST / PATCH | `/api/track/page-visit` | Record page visits and dwell time      |
| POST         | `/api/track/generation` | Record HTML generation success/failure |

### Validation

Every POST/PUT request is validated with Zod (`lib/validators.ts`). Invalid input returns `400` with per-field details. Missing resources return `404`. Unique-constraint returns `409`. Unhandled errors return `500` with a generic message (details logged server-side only).

## The `/api/health` endpoint returns `200` when the database is reachable and `503` when it is not — a proper dependency check rather than a static response.

## 📊 Dashboard (`/dashboard`)

The operations dashboard presents real-time system health and usage:

- **Health status** — live green/red indicator from `/api/health`
- **Alerts** — colour-coded warnings for empty word lists, failed generations, and error logs
- **KPI cards** — word lists, words, phonemes, activities, page visits (24h), average time on page
- **HTML Generation panel** — successful count, failed count, success rate
- **Activity Usage panel** — most-used activity type
- **Top Pages (7d)** — most visited paths
- **Recent Activity** — latest log entries
- **Quick actions** — jump to Wordle, Word Search, Word Lists, or `/api/health`

Auto-refreshes every 30 seconds. All numbers are sourced from the database.

---

---

## 📁 Saved Activities (/activities)

Every time a Wordle or Word Search is generated, its configuration is saved to the database as an Activity. The /activities page lists every saved configuration with:

- **Type badge** (WORDLE / WORDSEARCH)
- **Name**, difficulty, grid size, and attempt count
- **Source word list** and word count
- **Open in builder** button linking to the correct builder
- **Delete** button for removing outdated configurations

## This closes the loop on the A2 feedback and makes the Activity model fully integrated into the frontend.

## 🎮 Features

### Wordle Activity Builder

- Fetches words from the database
- Real-time phoneme feedback (🟢 correct · 🟡 wrong position · ⚪ absent)
- Phoneme keyboard grouped by articulatory class
- Difficulty controls
- Standalone HTML export (records success/failure)

### Word Search Activity Builder

- Fetches words from the database
- Configurable grid (10–40 rows/cols)
- Mouse drag and keyboard word selection
- Show/hide solution
- Standalone HTML export

### Word List Manager (`/word-lists`)

- Full CRUD for lists and words
- Teachers can add arbitrary phoneme words
- Inline editing and modal confirmations

### Settings

- Theme: Light / Dark / System
- Difficulty defaults, hint visibility, animation speed

---

## ♿ Accessibility

- Semantic HTML with ARIA labels throughout
- Keyboard navigation: Tab, Enter, Space, Escape
- Keyboard word selection in Word Search
- Modal traps focus and closes on Escape
- WCAG 2.1 AA contrast (verified by Lighthouse — Accessibility 100)
- Responsive with a mobile hamburger menu
- No flash-of-wrong-theme on load

---

## 🧪 Testing

### Playwright (end-to-end)

Tests live in `tests/`:

| Test file                     | Purpose                                                                  |
| ----------------------------- | ------------------------------------------------------------------------ |
| `word-list-crud.spec.ts`      | Builder flow — create, read, update, delete a list and a word            |
| `activity-generation.spec.ts` | User flow — generate Wordle + Word Search HTML, verify dashboard metrics |

**Run:**

```bash
# Make sure dev server is running first
npm run dev

# In another terminal:
npx playwright test
```

View HTML report:

```bash
npx playwright show-report
```

---

### JMeter (load testing)

Plan lives in `jmeter/load-test.jmx`. It runs staged thread groups of x1, x10, x100, and x1000 users through a realistic workflow (`health` → `wordlists` → `words` → `metrics` → `home`).

**Run:**

```bash
mkdir -p jmeter/results
"C:/jmeter/apache-jmeter-5.6.3/bin/jmeter.bat" -n -t jmeter/load-test.jmx -l jmeter/results/results.jtl -e -o jmeter/results/report
```

Then open `jmeter/results/report/index.html`.

**Results observed:** ~45 req/s throughput, sub-second response times up to x100, elevated latency and 0.33% errors at x1000 — indicating the system scales well for classroom use and degrades gracefully under extreme load.

---

### Lighthouse (accessibility)

Run against the production Docker container for accurate results:

1. Open `http://localhost:3001/dashboard` in an incognito window
2. Chrome DevTools → Lighthouse → Analyse page load

**Current scores:**

- **Performance:** 90
- **Accessibility:** 100
- **Best Practices:** 100
- **SEO:** 100

The initial audit flagged white text on `bg-primary-500` buttons as failing the 4.5:1 contrast requirement. Buttons were updated to `bg-primary-600`, restoring full WCAG AA compliance.

---

## 🧭 Architectural Decisions & Trade-offs

- **SQLite vs. Postgres** — SQLite keeps the container self-contained. Trade-off: no concurrent multi-user writes. Postgres can be swapped by changing one line.
- **Prisma vs. raw SQL** — type-safe queries and migrations at the cost of an extra build step.
- **Server Components vs. client fetching** — pages are client components for interactivity; slightly slower first paint but simpler state.
- **Baked-in seed vs. runtime seeding** — the image is slightly larger but the container starts instantly with data.
- **Zod at the boundary** — one source of truth for validation; small runtime cost worth the correctness.
- **Three-stage Dockerfile** — minimal, reproducible final image.
- **Normalised phoneme storage** — one extra join, but correct HCE handling is non-negotiable.
- **Observability at the edge** — page visits and generation events are recorded from the client with best-effort `fetch` / `sendBeacon`, keeping the DB schema unchanged for existing flows.

---

## 📜 Scripts Reference

| Script                 | Description                      |
| ---------------------- | -------------------------------- |
| `npm run dev`          | Start Next.js in development     |
| `npm run build`        | Production build                 |
| `npm start`            | Start production server          |
| `npm run lint`         | Run ESLint                       |
| `npm run db:generate`  | Generate Prisma Client           |
| `npm run db:migrate`   | Create and apply a dev migration |
| `npm run db:deploy`    | Apply migrations (production)    |
| `npm run db:seed`      | Seed the database                |
| `npm run db:reset`     | Reset the DB and re-seed         |
| `npm run db:studio`    | Open Prisma Studio               |
| `npm run docker:build` | Build the Docker image           |
| `npm run docker:run`   | Run the Docker container         |

---

## 🔐 Environment Variables

| Variable       | Description          | Default         |
| -------------- | -------------------- | --------------- |
| `DATABASE_URL` | SQLite database path | `file:./dev.db` |
| `NODE_ENV`     | Runtime environment  | `development`   |

In Docker, `DATABASE_URL` is set to `file:/app/prisma/dev.db` by the Dockerfile.

## 📚 References

- Cox, F. (2012). _Australian English pronunciation and transcription_. Cambridge University Press.
- Harrington, J., & Cox, F. (2008). The acoustic characteristics of Australian English vowels. _Journal of Phonetics_, 36(2), 328–344. https://doi.org/10.1016/j.wocn.2007.09.002
- Moats, L. (2020). _Speech to print: Language essentials for teachers_ (3rd ed.). Paul H. Brookes Publishing.
- Prisma. (2024). _Prisma ORM documentation_. https://www.prisma.io/docs
- W3C Web Accessibility Initiative. (2023). _Web Content Accessibility Guidelines (WCAG) 2.1_. https://www.w3.org/TR/WCAG21/
- Docker. (2024). _Docker documentation_. https://docs.docker.com/
- Zod. (2024). _Zod: TypeScript-first schema validation_. https://zod.dev/
- Playwright. (2024). _Playwright documentation_. https://playwright.dev/
- Apache Software Foundation. (2024). _Apache JMeter_. https://jmeter.apache.org/

---

## 👨‍🎓 Student Information

| Field           | Details                                             |
| --------------- | --------------------------------------------------- |
| **Name**        | Sudipta Biswas Durbar                               |
| **Student ID**  | 21946247                                            |
| **Subject**     | CSE3CWA - Cloud-based Web Application               |
| **Assessment**  | Assessment 3: Data-driven Application and Reporting |
| **Institution** | La Trobe University                                 |

---

## 🙏 Acknowledgments

- La Trobe University for the assignment brief and guidance
- Speech Pathology educators for domain guidance
- HCE Phoneme Corpus for the Australian English phoneme dataset

---

## 📄 License

MIT License. See [LICENSE](LICENSE) for details.
