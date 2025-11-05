 SlotSwapper — Starter Project

This document contains a ready-to-run starter project for the **SlotSwapper** technical challenge (Full Stack Intern — ServiceHive). It includes a backend (Node.js + Express + Prisma + SQLite), and a frontend (React + Vite). The project is intentionally minimal but complete: auth with JWT, events CRUD, swap-request logic including transactional swap acceptance, and a simple React UI scaffold.

---

 Repository layout

```
slotswapper/
├─ backend/
│  ├─ package.json
│  ├─ prisma/
│  │  └─ schema.prisma
│  ├─ .env
│  └─ src/
│     ├─ index.js
│     ├─ server.js
│     ├─ routes/
│     │  ├─ auth.js
│     │  ├─ events.js
│     │  └─ swaps.js
│     ├─ middleware/
│     │  └─ auth.js
│     └─ utils/jwt.js
└─ frontend/
   ├─ package.json
   ├─ vite.config.js
   └─ src/
      ├─ main.jsx
      ├─ App.jsx
      ├─ pages/
      │  ├─ Login.jsx
      │  ├─ Signup.jsx
      │  ├─ Dashboard.jsx
      │  ├─ Marketplace.jsx
      │  └─ Requests.jsx
      └─ lib/api.js

```

---

 Quick setup (local, one-machine)

> Backend uses SQLite for simplicity. Prisma is used as ORM.

 Prereqs

* Node.js 18+
* npm or yarn
* (Optional) Git

 Backend

1. `cd backend`
2. `npm install`
3. Copy `.env.example` to `.env` (or create `.env`) — the file contains `DATABASE_URL` and `JWT_SECRET`.
4. `npx prisma migrate dev --name init` (creates SQLite file and migrations)
5. `node src/server.js` (or `npm run dev`)

API runs on `http://localhost:4000` by default.

 Frontend

1. `cd frontend`
2. `npm install`
3. `npm run dev`

Frontend runs on `http://localhost:5173` (Vite default).

---

 Design Choices / Notes

* **Auth:** Email+password, passwords hashed with bcrypt. JWT access tokens returned at login/signup (no refresh tokens in starter).
* **DB:** Prisma + SQLite: simple, predictable, and easy to run in reviewers' machines.
* **Swap logic:** When creating a swap request, both events are set to `SWAP_PENDING`. On response, transactionally either swap `ownerId` fields (on ACCEPT) and set statuses to `BUSY`, or revert to `SWAPPABLE` on REJECT.
* **Transactions:** The ACCEPT flow uses a Prisma transaction to ensure atomicity.
* **Frontend:** Minimal React components that demonstrate flows and call APIs. Protected routes store JWT in `localStorage` and include `Authorization: Bearer <token>` header.

---

 API Reference (top-level)

| Method |                          Path | Auth? | Description                                            |           |
| ------ | ----------------------------: | :---: | ------------------------------------------------------ | --------- |
| POST   |              /api/auth/signup |   no  | Sign up. Returns token + user.                         |           |
| POST   |               /api/auth/login |   no  | Login. Returns token + user.                           |           |
| GET    |                   /api/events |  yes  | Get all events for logged-in user.                     |           |
| POST   |                   /api/events |  yes  | Create event (title, startTime, endTime, status).      |           |
| PUT    |               /api/events/:id |  yes  | Update event (including marking SWAPPABLE).            |           |
| DELETE |               /api/events/:id |  yes  | Delete event.                                          |           |
| GET    |          /api/swappable-slots |  yes  | Return SWAPPABLE slots from other users.               |           |
| POST   |             /api/swap-request |  yes  | Create swap request: body `{ mySlotId, theirSlotId }`. |           |
| POST   | /api/swap-response/:requestId |  yes  | Respond to request: body `{ accept: true               | false }`. |
| GET    |        /api/requests/incoming |  yes  | Incoming requests for logged-in user.                  |           |
| GET    |        /api/requests/outgoing |  yes  | Outgoing requests for logged-in user.                  |           |

---

 Key files (backend)

 backend/package.json

```json
{
  "name": "slotswapper-backend",
  "version": "1.0.0",
  "main": "src/server.js",
  "scripts": {
    "dev": "nodemon src/server.js",
    "start": "node src/server.js",
    "prisma": "prisma"
  },
  "dependencies": {
    "bcrypt": "^5.1.0",
    "cors": "^2.8.5",
    "dotenv": "^16.0.0",
    "express": "^4.18.2",
    "jsonwebtoken": "^9.0.0",
    "prisma": "^5.0.0",
    "@prisma/client": "^5.0.0"
  },
  "devDependencies": {
    "nodemon": "^2.0.22"
  }
}
```
 backend/prisma/schema.prisma

```prisma
generator client {
  provider = "prisma-client-js"
}

datasource db {
  provider = "sqlite"
  url      = env("DATABASE_URL")
}

model User {
  id        Int      @id @default(autoincrement())
  name      String
  email     String   @unique
  password  String
  events    Event[]
  requestsAsRequester SwapRequest[] @relation("requester")
  requestsAsResponder SwapRequest[] @relation("responder")
}

model Event {
  id        Int      @id @default(autoincrement())
  title     String
  startTime DateTime
  endTime   DateTime
  status    EventStatus @default(BUSY)
  ownerId   Int
  owner     User     @relation(fields: [ownerId], references: [id])
  createdAt DateTime @default(now())
}

model SwapRequest {
  id           Int      @id @default(autoincrement())
  requesterId  Int
  responderId  Int
  mySlotId     Int
  theirSlotId  Int
  status       RequestStatus @default(PENDING)
  createdAt    DateTime @default(now())

  requester    User     @relation("requester", fields: [requesterId], references: [id])
  responder    User     @relation("responder", fields: [responderId], references: [id])
}

enum EventStatus {
  BUSY
  SWAPPABLE
  SWAP_PENDING
}

enum RequestStatus {
  PENDING
  ACCEPTED
  REJECTED
}
```

 backend/.env.example

```
DATABASE_URL="file:./dev.db"
JWT_SECRET="change_this_to_a_strong_secret"
PORT=4000
```

 backend/src/server.js

```js
require('dotenv').config();
const express = require('express');
const cors = require('cors');
const { PrismaClient } = require('@prisma/client');
const prisma = new PrismaClient();
const authRoutes = require('./routes/auth');
const eventsRoutes = require('./routes/events');
const swapsRoutes = require('./routes/swaps');

const app = express();
app.use(cors());
app.use(express.json());

app.use('/api/auth', authRoutes);
app.use('/api/events', eventsRoutes);
app.use('/api', swapsRoutes); // includes swappable-slots, swap-request, swap-response, requests

const port = process.env.PORT || 4000;
app.listen(port, () => console.log(`Server listening on ${port}`));
```

 backend/src/middleware/auth.js

```js
const jwt = require('jsonwebtoken');
const { PrismaClient } = require('@prisma/client');
const prisma = new PrismaClient();

module.exports = async function (req, res, next) {
  const auth = req.headers.authorization;
  if (!auth) return res.status(401).json({ error: 'Missing auth header' });
  const token = auth.split(' ')[1];
  try {
    const payload = jwt.verify(token, process.env.JWT_SECRET);
    const user = await prisma.user.findUnique({ where: { id: payload.userId } });
    if (!user) return res.status(401).json({ error: 'Invalid token' });
    req.user = user;
    next();
  } catch (err) {
    return res.status(401).json({ error: 'Invalid token' });
  }
};
```

backend/src/routes/auth.js

```js
const express = require('express');
const router = express.Router();
const bcrypt = require('bcrypt');
const jwt = require('jsonwebtoken');
const { PrismaClient } = require('@prisma/client');
const prisma = new PrismaClient();

router.post('/signup', async (req, res) => {
  const { name, email, password } = req.body;
  if (!email || !password || !name) return res.status(400).json({ error: 'Missing fields' });
  const existing = await prisma.user.findUnique({ where: { email } });
  if (existing) return res.status(400).json({ error: 'Email in use' });
  const hash = await bcrypt.hash(password, 10);
  const user = await prisma.user.create({ data: { name, email, password: hash } });
  const token = jwt.sign({ userId: user.id }, process.env.JWT_SECRET);
  res.json({ token, user: { id: user.id, name: user.name, email: user.email } });
});

router.post('/login', async (req, res) => {
  const { email, password } = req.body;
  const user = await prisma.user.findUnique({ where: { email } });
  if (!user) return res.status(400).json({ error: 'Invalid credentials' });
  const ok = await bcrypt.compare(password, user.password);
  if (!ok) return res.status(400).json({ error: 'Invalid credentials' });
  const token = jwt.sign({ userId: user.id }, process.env.JWT_SECRET);
  res.json({ token, user: { id: user.id, name: user.name, email: user.email } });
});

module.exports = router;
```

 backend/src/routes/events.js

```js
const express = require('express');
const router = express.Router();
const auth = require('../middleware/auth');
const { PrismaClient } = require('@prisma/client');
const prisma = new PrismaClient();

router.use(auth);

router.get('/', async (req, res) => {
  const events = await prisma.event.findMany({ where: { ownerId: req.user.id } });
  res.json(events);
});

router.post('/', async (req, res) => {
  const { title, startTime, endTime, status } = req.body;
  const e = await prisma.event.create({ data: { title, startTime: new Date(startTime), endTime: new Date(endTime), status: status || 'BUSY', ownerId: req.user.id } });
  res.json(e);
});

router.put('/:id', async (req, res) => {
  const id = parseInt(req.params.id);
  const ev = await prisma.event.findUnique({ where: { id } });
  if (!ev || ev.ownerId !== req.user.id) return res.status(404).json({ error: 'Not found' });
  const data = req.body;
  if (data.startTime) data.startTime = new Date(data.startTime);
  if (data.endTime) data.endTime = new Date(data.endTime);
  const updated = await prisma.event.update({ where: { id }, data });
  res.json(updated);
});

router.delete('/:id', async (req, res) => {
  const id = parseInt(req.params.id);
  const ev = await prisma.event.findUnique({ where: { id } });
  if (!ev || ev.ownerId !== req.user.id) return res.status(404).json({ error: 'Not found' });
  await prisma.event.delete({ where: { id } });
  res.json({ success: true });
});

module.exports = router;
```

 backend/src/routes/swaps.js

```js
const express = require('express');
const router = express.Router();
const auth = require('../middleware/auth');
const { PrismaClient, EventStatus, RequestStatus } = require('@prisma/client');
const prisma = new PrismaClient();

// GET /api/swappable-slots
router.get('/swappable-slots', auth, async (req, res) => {
  const slots = await prisma.event.findMany({ where: { status: 'SWAPPABLE', NOT: { ownerId: req.user.id } }, include: { owner: { select: { id: true, name: true, email: true } } } });
  res.json(slots);
});

// POST /api/swap-request
router.post('/swap-request', auth, async (req, res) => {
  const { mySlotId, theirSlotId } = req.body;
  if (!mySlotId || !theirSlotId) return res.status(400).json({ error: 'Missing slot ids' });

  const mySlot = await prisma.event.findUnique({ where: { id: mySlotId } });
  const theirSlot = await prisma.event.findUnique({ where: { id: theirSlotId } });
  if (!mySlot || !theirSlot) return res.status(404).json({ error: 'Slot not found' });
  if (mySlot.ownerId !== req.user.id) return res.status(403).json({ error: 'mySlot must belong to you' });
  if (mySlot.status !== 'SWAPPABLE' || theirSlot.status !== 'SWAPPABLE') return res.status(400).json({ error: 'Both slots must be SWAPPABLE' });
  if (theirSlot.ownerId === req.user.id) return res.status(400).json({ error: 'Cannot request your own slot' });

  // Create request and set both slots to SWAP_PENDING
  const swap = await prisma.swapRequest.create({ data: {
    requesterId: req.user.id,
    responderId: theirSlot.ownerId,
    mySlotId: mySlot.id,
    theirSlotId: theirSlot.id,
    status: 'PENDING'
  }});

  await prisma.event.updateMany({ where: { id: { in: [mySlot.id, theirSlot.id] } }, data: { status: 'SWAP_PENDING' } });

  res.json(swap);
});

// POST /api/swap-response/:requestId
router.post('/swap-response/:requestId', auth, async (req, res) => {
  const requestId = parseInt(req.params.requestId);
  const { accept } = req.body;
  const reqRec = await prisma.swapRequest.findUnique({ where: { id: requestId } });
  if (!reqRec) return res.status(404).json({ error: 'Request not found' });
  if (reqRec.responderId !== req.user.id) return res.status(403).json({ error: 'Not authorized' });
  if (reqRec.status !== 'PENDING') return res.status(400).json({ error: 'Request already handled' });

  if (!accept) {
    // Reject: set request to REJECTED, revert event statuses back to SWAPPABLE
    await prisma.swapRequest.update({ where: { id: requestId }, data: { status: 'REJECTED' } });
    await prisma.event.updateMany({ where: { id: { in: [reqRec.mySlotId, reqRec.theirSlotId] } }, data: { status: 'SWAPPABLE' } });
    return res.json({ ok: true, status: 'REJECTED' });
  }

  // Accept: we need an atomic swap of ownerId and set statuses to BUSY
  try {
    await prisma.$transaction(async (tx) => {
      // refetch inside transaction
      const mySlot = await tx.event.findUnique({ where: { id: reqRec.mySlotId } });
      const theirSlot = await tx.event.findUnique({ where: { id: reqRec.theirSlotId } });
      if (!mySlot || !theirSlot) throw new Error('Slot missing');
      if (mySlot.status !== 'SWAP_PENDING' || theirSlot.status !== 'SWAP_PENDING') throw new Error('Slots not pending');

      // swap owners
      const requesterId = reqRec.requesterId;
      const responderId = reqRec.responderId;

      await tx.event.update({ where: { id: mySlot.id }, data: { ownerId: responderId, status: 'BUSY' } });
      await tx.event.update({ where: { id: theirSlot.id }, data: { ownerId: requesterId, status: 'BUSY' } });

      await tx.swapRequest.update({ where: { id: requestId }, data: { status: 'ACCEPTED' } });
    });

    return res.json({ ok: true, status: 'ACCEPTED' });
  } catch (err) {
    console.error(err);
    // attempt to revert statuses if something went wrong
    await prisma.event.updateMany({ where: { id: { in: [reqRec.mySlotId, reqRec.theirSlotId] } }, data: { status: 'SWAPPABLE' } });
    await prisma.swapRequest.update({ where: { id: requestId }, data: { status: 'REJECTED' } });
    return res.status(500).json({ error: 'Swap failed' });
  }
});

// Incoming and outgoing requests
router.get('/requests/incoming', auth, async (req, res) => {
  const items = await prisma.swapRequest.findMany({ where: { responderId: req.user.id }, include: { requester: true } });
  res.json(items);
});
router.get('/requests/outgoing', auth, async (req, res) => {
  const items = await prisma.swapRequest.findMany({ where: { requesterId: req.user.id }, include: { responder: true } });
  res.json(items);
});

module.exports = router;
```

---

 Frontend (minimal)

A Vite + React app with these minimal pieces:

* `lib/api.js` — wrapper around `fetch` that adds Authorization header.
* `pages/Login.jsx` & `Signup.jsx` — forms that call `/api/auth/*` and store token.
* `pages/Dashboard.jsx` — list your events and allow creating/updating status.
* `pages/Marketplace.jsx` — fetch `/api/swappable-slots`, show slots and "Request Swap" button. Clicking opens a small chooser of your SWAPPABLE slots and hits `/api/swap-request`.
* `pages/Requests.jsx` — shows incoming and outgoing requests, accept/reject buttons call `/api/swap-response/:id`.

> The full frontend code is provided in the scaffold below for main pieces.

frontend/package.json

```json
{
  "name": "slotswapper-frontend",
  "version": "1.0.0",
  "private": true,
  "scripts": {
    "dev": "vite",
    "build": "vite build",
    "preview": "vite preview"
  },
  "dependencies": {
    "react": "^18.2.0",
    "react-dom": "^18.2.0",
    "react-router-dom": "^6.14.1"
  },
  "devDependencies": {
    "vite": "^5.0.0"
  }
}
```

 frontend/src/lib/api.js

```js
const API_ROOT = import.meta.env.VITE_API_URL || 'http://localhost:4000/api';

function authHeaders() {
  const token = localStorage.getItem('token');
  return token ? { Authorization: `Bearer ${token}` } : {};
}

export async function get(path) {
  const res = await fetch(`${API_ROOT}${path}`, { headers: { 'Content-Type': 'application/json', ...authHeaders() } });
  return res.json();
}
export async function post(path, body) {
  const res = await fetch(`${API_ROOT}${path}`, { method: 'POST', headers: { 'Content-Type': 'application/json', ...authHeaders() }, body: JSON.stringify(body) });
  return res.json();
}
export async function put(path, body) {
  const res = await fetch(`${API_ROOT}${path}`, { method: 'PUT', headers: { 'Content-Type': 'application/json', ...authHeaders() }, body: JSON.stringify(body) });
  return res.json();
}
```

 frontend/src/pages/Marketplace.jsx (high-level)

```jsx
import React, { useEffect, useState } from 'react';
import { get, post } from '../lib/api';

export default function Marketplace() {
  const [slots, setSlots] = useState([]);
  const [mySwappables, setMySwappables] = useState([]);

  useEffect(() => { fetchSlots(); fetchMyEvents(); }, []);
  async function fetchSlots() { const s = await get('/swappable-slots'); setSlots(s); }
  async function fetchMyEvents() { const e = await get('/events'); setMySwappables(e.filter(x => x.status === 'SWAPPABLE')); }

  async function requestSwap(theirSlotId) {
    // naive prompt to pick first my swappable slot
    if (!mySwappables.length) return alert('No SWAPPABLE slot to offer');
    const mySlotId = mySwappables[0].id;
    await post('/swap-request', { mySlotId, theirSlotId });
    fetchSlots(); fetchMyEvents();
  }

  return (
    <div>
      <h2>Marketplace</h2>
      <ul>
        {slots.map(s => (
          <li key={s.id}>{s.title} — {new Date(s.startTime).toLocaleString()} — owner: {s.owner.name}
            <button onClick={() => requestSwap(s.id)}>Request Swap</button>
          </li>
        ))}
      </ul>
    </div>
  );
}
```

---

Tests / Bonus

* You can add tests for `swap-response` using Jest and supertest — focus on Accept path transactionality.
* Real-time notifications: add a simple WebSocket (Socket.IO) event emitted when a swap-request is created or when accepted.

---

 Assumptions made

* Timezones: `startTime` and `endTime` are stored as native JS Date / ISO strings in DB. In production, you'd normalize to UTC and store timezone info or use a calendar provider.
* No conflict checks: The starter does not check for overlapping events when accepting swaps. You can add validations that swapped-in event doesn't conflict with a user's other events.
* Authentication: Short-lived JWT only. For a production app you'd add refresh tokens and better session handling.

---


Challenges Faced
Transactional Complexity: Ensuring the atomicity of the swap acceptance and rejection logic was the primary challenge. I leveraged PostgreSQL and Prisma's transaction capabilities to guarantee that the critical ownership exchange and status updates are completed reliably.

Author

Yashwanth Royal Ande
