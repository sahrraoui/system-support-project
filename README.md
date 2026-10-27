# 🎫 Support Ticket System API

> A lightweight, zero-setup **customer support ticket REST API** — guests open tickets, support staff reply over a threaded conversation, and both sides get automatic email notifications. Built for developers who want a drop-in support backend (Node.js + Express 5 + SQLite) with no database server to install.

---

## 📸 Screenshots / Demo



### ▶️ Live demo (curl)

Create a ticket:

```bash
curl -X POST http://localhost:3000/tickets \
  -H "Content-Type: application/json" \
  -d '{
    "subject": "I can'"'"'t log into my account",
    "guest_email": "guest@example.com",
    "message": "Every time I try, I get an error."
  }'
```

```json
{
  "ticket": {
    "id": 1,
    "subject": "I can't log into my account",
    "status": "open",
    "guest_email": "guest@example.com",
    "guest_token": "a1b2… (128-char hex token)",
    "created_at": "2026-10-27 10:30:00"
  }
}
```

Reply as admin (triggers an email to the guest):

```bash
curl -X POST http://localhost:3000/tickets/1/messages \
  -H "Content-Type: application/json" \
  -d '{ "sender_type": "admin", "body": "We reset your password — try again." }'
```

```json
{ "messageId": 2 }
```

---

## ✨ Features

- 🎫 **Ticket creation** — guests submit a subject, email, and first message in one call
- 🔐 **Guest tokens** — a secure 128-character random token is generated per ticket
- 💬 **Threaded conversations** — messages are attached to tickets with a sender type (`user` / `admin`)
- ✅ **Status workflow** — tickets move through `open → pending → closed`
- 📧 **Email notifications** — automatic mail on ticket creation and on admin replies (Nodemailer / SMTP)
- 🗄️ **Embedded database** — SQLite in WAL mode, schema auto-applied on startup, no external DB
- ⚛️ **Atomic writes** — ticket + first message inserted in a single transaction
- 🌐 **CORS enabled** — ready to be consumed by any frontend out of the box

---

## 🛠️ Tech Stack

| Layer | Technology |
|---|---|
| Runtime | Node.js (v18+) |
| Framework | Express 5 |
| Language | TypeScript (strict mode) |
| Database | SQLite via `better-sqlite3` (synchronous, WAL) |
| Email | Nodemailer |
| Dev tooling | `ts-node-dev`, `tsx`, `tsc` |

---

## 🚀 Getting Started

### Prerequisites

- **Node.js v18 or higher**
- An SMTP catcher for local development — [Mailpit](https://github.com/axllent/mailpit) or [MailHog](https://github.com/mailhog/MailHog) listening on `localhost:1025`

```bash
# macOS
brew install mailpit && mailpit
# Windows (choco)
choco install mailpit
# or run via Docker
docker run -d -p 1025:1025 -p 8025:8025 axllent/mailpit
```

Mailpit's web UI is then available at http://localhost:8025 — you can watch every notification the API sends.

### Installation

```bash
git clone https://github.com/your-username/system-support-project.git
cd system-support-project
npm install
```

### Create the data folder

SQLite needs its parent directory to exist — create it once before the first run:

```bash
# macOS / Linux
mkdir -p data

# Windows (PowerShell)
New-Item -ItemType Directory -Force data
```

### Run in development

```bash
npm run dev
```

The server starts on **http://localhost:3000** 🎉

> 💡 On first run, `data/support.db` is created automatically and `schema.sql` is applied — no manual migration needed.

### Verify it works

```bash
curl http://localhost:3000/tickets
```

```json
{ "tickets": [] }
```

### Production build

```bash
npm run build
```

> ⚠️ **Known limitation:** the `start` script (`node dist/index.js`) doesn't line up with the current `tsconfig` output (`dist/src/index.js`) and `schema.sql` isn't copied into `dist/`. Use `npm run dev` for now — fixing this is tracked in the roadmap below.

---

## ⚙️ Configuration

The app loads `.env` automatically via `dotenv` (see `src/index.ts`). Create a `.env` file in the project root:

```bash
# Server
PORT=3000

# SMTP (local Mailpit / MailHog defaults)
SMTP_HOST=localhost
SMTP_PORT=1025
```

| Variable | Default | Description |
|---|---|---|
| `PORT` | `3000` | HTTP port the API listens on |
| `SMTP_HOST` | `localhost` | SMTP server used for notifications |
| `SMTP_PORT` | `1025` | SMTP server port (no TLS in local dev) |

> 📌 **Note:** these values are currently hardcoded (`src/index.ts`, `emailService.ts`); wiring them to `process.env` is the next item on the roadmap. `PORT`, `SMTP_HOST`, and `SMTP_PORT` above are the intended variables.

No other secrets are required — the SQLite database lives in `data/support.db`.

---

## 📡 API Endpoints

Base URL: `http://localhost:3000`

| Method | Endpoint | Description | Success |
|---|---|---|---|
| `POST` | `/tickets` | Create a ticket (subject, guest_email, message) | `201` |
| `GET` | `/tickets` | List all tickets, newest first | `200` |
| `GET` | `/tickets/:id` | Get one ticket with its full conversation | `200` |
| `POST` | `/tickets/:id/messages` | Add a reply (`sender_type`: `user` \| `admin`) | `201` |
| `PATCH` | `/tickets/:id` | Update status (`open` \| `pending` \| `closed`) | `200` |

**Error responses** use a consistent shape:

```json
{ "error": "ticket not found!" }
```

| Status | Meaning |
|---|---|
| `400` | Missing or invalid field (bad body, non-integer id, invalid status) |
| `404` | Ticket does not exist |
| `500` | Database or email delivery failure |

---

## 🗄️ Database Schema

| Table | Columns | Description |
|---|---|---|
| `tickets` | `id`, `subject`, `status`, `guest_email`, `guest_token`, `created_at` | Support tickets, each with a unique guest token |
| `messages` | `id`, `ticket_id`, `sender_type`, `body`, `created_at` | Conversation messages (`user` or `admin`) |
| `attachments` | `id`, `message_id`, `file_url`, `file_type`, `size` | File attachments (schema ready, awaiting implementation) |

All foreign keys use `ON DELETE CASCADE`.

---

## 📁 Project Structure

```
system-support-project/
├── src/
│   └── index.ts          # Express app entry point (JSON, CORS, router)
├── ticket.ts             # All ticket routes / REST logic
├── db.ts                 # SQLite connection + schema initialization
├── emailService.ts       # Nodemailer transport & email helpers
├── schema.sql            # tickets, messages, attachments tables
├── data/                 # SQLite database file (created locally)
├── tsconfig.json
└── package.json
```

---

## 🔮 Future Improvements

- [ ] Attachment upload endpoint (schema already in place)
- [ ] Token-based guest authentication for viewing private tickets
- [ ] Read `PORT` / SMTP credentials from `.env`
- [ ] Fix production build output (`dist/src/index.js` + copy `schema.sql`)
- [ ] Authentication and roles for the admin panel
- [ ] Pagination and filtering for ticket listing
- [ ] Rate limiting and input sanitization for public endpoints

---

## 👤 Contact

Built as a portfolio project — feedback, ideas, and collaborations welcome!

- **GitHub:** [sahrraoui/system-support-project](https://github.com/sahrraoui/system-support-project)


