# CampGroove 🎟

> **The college event ticketing platform built for .edu communities.**  
> A fully responsive, zero-dependency single-page web application for student clubs, campus organisers, and system administrators.

---

## Table of Contents

1. [Project Overview](#1-project-overview)
2. [Live Demo & Credentials](#2-live-demo--credentials)
3. [Architecture Summary](#3-architecture-summary)
4. [Data Models](#4-data-models)
5. [Component Map](#5-component-map)
6. [Feature Reference](#6-feature-reference)
7. [GitHub Pages Deployment Guide](#7-github-pages-deployment-guide)
8. [Customisation Guide (Branding & Copy)](#8-customisation-guide-branding--copy)
9. [Maintenance Guide — Updating Event Data Models](#9-maintenance-guide--updating-event-data-models)
10. [Adding a New Category](#10-adding-a-new-category)
11. [Extending to a Real Backend](#11-extending-to-a-real-backend)
12. [Troubleshooting](#12-troubleshooting)
13. [Contributing](#13-contributing)
14. [Licence](#14-licence)

---

## 1. Project Overview

CampGroove is a **single-file static web application** (`index.html`) that delivers a full college event ticketing experience with no build step, no server, and no external runtime dependencies beyond the Tailwind CSS CDN.

### Who is it for?

| Audience | Role in the app |
|---|---|
| **Students** | Browse published events, purchase tickets (1–5 per event), view a personal ticket drawer with live QR codes |
| **Student Organisers** | Publish, manage, and cancel campus events; monitor ticket sales |
| **System Administrators** | Full CRUD override over all users, events, and tickets; audit log of every system action |

### What makes it special?

- **`.edu` email enforcement** — client-side regex blocks any non-academic email at every auth entry point
- **Zero build toolchain** — open `index.html` in any browser, or push directly to GitHub Pages
- **In-memory global state store** — a hand-rolled pub/sub `Store` module keeps all data reactive without a framework
- **Live QR code generation** — deterministic SVG QR patterns encoded with `studentId:eventId:purchaseTimestamp`
- **Slide-over My Tickets drawer** — real-time ticket inventory displayed with category-filter tabs
- **Mock payment flow** — Campus Account balance deduction or test-mode Credit Card with field formatters

---

## 2. Live Demo & Credentials

Open `index.html` in any modern browser. No installation required.

### Demo Accounts

| Role | Email | Student ID | Password |
|---|---|---|---|
| Student | `student@mit.edu` | `STU-001` | *(any value)* |
| Organiser | `org@mit.edu` | `ORG-001` | *(any value)* |
| Admin | `admin@mit.edu` | `ADM-001` | *(any value)* |

### Test Payment Card

When using Credit/Debit Card payment:

```
Card number : 4242 4242 4242 4242
Expiry      : any future date (MM/YY)
CVC         : any 3-digit number
```

> **Note:** No real charges are made. The balance deduction for Campus Account payments is simulated in memory only and resets on page reload.

---

## 3. Architecture Summary

```
index.html
├── <style>          — All CSS: Tailwind config + custom classes + drawer + ticket template
├── HTML             — Static structure: Modals, Drawer, Navbar, Hero, Sections, Footer
└── <script>
    ├── Store         — Global in-memory state (pub/sub, IIFE module)
    ├── Seed Data     — Demo users, events, audit log
    ├── Utils         — EDU_REGEX, formatDate, formatPrice, catGradient, esc(), QR generators
    ├── Toast system  — toast(), toastRich()
    ├── Auth flows    — handleLogin(), handleRegister(), handleOtp()
    ├── Purchase flow — openPurchaseModal(), purchaseStep(), processPayment(), buildTicketTemplate()
    ├── Drawer        — openTicketDrawer(), renderTicketDrawer(), filterDrawer(), drawerQRPayload()
    ├── Organiser     — publishEvent(), renderOrgEvents()
    ├── Admin         — openAdminDashboard(), renderAdminEvents/Users/Tickets(), renderAuditLog()
    ├── Renderer      — renderEventCard(), renderEventsGrid(), renderMyTickets()
    ├── Nav/UI        — updateNav(), updateTicketBadge(), showSection(), animateCount()
    └── Boot          — DOMContentLoaded → updateNav() + renderEventsGrid()
```

### State Flow

```
User action
    │
    ▼
Store mutation  (e.g. Store.purchaseOrder())
    │
    ▼
Store._emit()  →  all subscribers notified
    │
    ▼
updateNav(state)  →  re-renders navbar, sections, drawer, badges
```

### File Size Budget

| Asset | Notes |
|---|---|
| `index.html` | ~2,400 lines — single deployable file |
| `tailwindcss` CDN | Loaded from `cdn.tailwindcss.com` at runtime |
| All other code | Inline — zero npm packages, zero build step |

---

## 4. Data Models

All data lives inside the `Store` IIFE in `index.html`. The following schemas describe every object shape used at runtime.

### User

```js
{
  id:        String,   // 'u1', 'u'+Date.now()
  name:      String,   // Full name
  email:     String,   // Must match /\.edu$/i
  sid:       String,   // Student / Staff ID (min 4 chars)
  password:  String,   // Plain text in demo; hash in production
  role:      'student' | 'organizer' | 'admin',
  isActive:  Boolean,  // Admin can toggle false to deactivate
  createdAt: String    // ISO date string
}
```

### Event

```js
{
  id:               String,   // 'e1', genId()
  title:            String,
  category:         'music' | 'sports' | 'academic' | 'arts' | 'tech' | 'social',
  capacity:         Number,   // Total seats (1–50,000)
  ticketsRemaining: Number,   // Decremented atomically on purchaseOrder()
  price:            Number,   // 0 = free; positive = charged amount per ticket
  venue:            String,
  date:             String,   // datetime-local format 'YYYY-MM-DDTHH:mm'
  organizer:        String,   // User.id reference
  organizerName:    String,   // Denormalised for display
  status:           'published' | 'cancelled' | 'draft',
  description:      String,   // Promotion / hype copy
  createdAt:        String    // ISO timestamp
}
```

### Ticket

```js
{
  id:            String,   // 'TKT-XXXX-XXXX' — FNV-1a hash of (eventId:studentId:orderRef:seatIndex:now)
  orderRef:      String,   // 'ORD-XXXXXX' — groups multi-ticket purchases
  seatNum:       Number,   // 1-based within the order
  totalInOrder:  Number,   // Total tickets in same order
  eventId:       String,
  eventTitle:    String,
  category:      String,
  venue:         String,
  date:          String,
  organizer:     String,   // Organiser display name
  studentId:     String,
  studentName:   String,
  studentEmail:  String,
  studentSid:    String,
  amountPaid:    Number,   // Price at time of purchase
  paymentMethod: 'account' | 'card',
  status:        'reserved' | 'voided',
  createdAt:     String    // ISO timestamp — used in QR payload
}
```

### Notification

```js
{
  id:      String,
  userId:  String,   // Recipient's User.id
  message: String,
  type:    'success' | 'warning' | 'info',
  read:    Boolean,
  ts:      String    // ISO timestamp
}
```

### Audit Log Entry

```js
{
  ts:     String,   // 'YYYY-MM-DD HH:mm'
  actor:  String,   // User's display name or 'System'
  action: String    // Human-readable description
}
```

### QR Payload Format

Every ticket QR encodes exactly:

```
{studentId}:{eventId}:{createdAt}
```

Example: `u1:e2:2025-07-05T09:00:00.000Z`

---

## 5. Component Map

| Component / Function | File location | Purpose |
|---|---|---|
| `Store` | `index.html` ~ln 910 | Global state IIFE — all mutations go through here |
| `#ticketDrawer` | `index.html` ~ln 147 | Slide-over My Tickets panel |
| `#purchaseModal` | `index.html` ~ln 302 | 3-step buy flow (Select → Pay → Confirm) |
| `#authModal` | `index.html` ~ln 197 | Login / Register / OTP panels |
| `#orgModal` | `index.html` ~ln 257 | Publish event form |
| `#adminModal` | `index.html` ~ln 521 | Admin dashboard (4 tabs) |
| `buildTicketTemplate(t)` | `index.html` ~ln 1599 | Returns styled printable ticket DOM element |
| `renderQRInto(el, payload)` | `index.html` ~ln 1762 | Renders deterministic SVG QR into any container |
| `renderEventCard(ev, opts)` | `index.html` ~ln 1780 | Returns event grid card DOM element |
| `renderTicketDrawer()` | `index.html` ~ln 1953 | Populates drawer with filtered, sorted tickets |
| `updateNav(state)` | `index.html` ~ln 2163 | Master UI update — called on every state change |

---

## 6. Feature Reference

### Authentication

- Email validated against `/^[a-zA-Z0-9._%+\-]+@[a-zA-Z0-9.\-]+\.edu$/i` on every keystroke (live feedback)
- Student ID requires minimum 4 characters
- OTP verification step (demo code: `123456`) after registration
- Role-based UI: students see ticket drawer, organisers see "My Events", admins see "⚙ Admin"

### Ticket Purchase Flow

1. **Select** — inventory gauge, quantity stepper (1–5, capped by remaining), live order summary with 3% service fee
2. **Payment** — Campus Account (mock balance: $250) or Credit/Debit Card (Stripe test-mode UX)
3. **Confirm** — styled ticket template per seat, unique `TKT-XXXX-XXXX` hash ID, QR code, print/download buttons
4. Rich toast: `"🎟 Ticket(s) Reserved Successfully! Confirmation sent to you@university.edu · Event Name"`

### My Tickets Drawer

- Opens from navbar ticket icon, user menu, mobile nav, and purchase step 3
- Filter tabs: All · Active · Voided · per-category
- Per-card inline QR encoded as `studentId:eventId:purchaseTimestamp`
- Badge count on navbar icon reflects active (reserved) tickets

### Organiser Dashboard

- Publish events with 7-field form (title, category, capacity, date/time, price, venue, hype/description)
- Newly published events appear at the front of the public grid **without page reload**
- Cancel or delete own events; cancellation triggers notifications to all ticket holders

### System Admin Dashboard

| Tab | Capabilities |
|---|---|
| **Events** | View all events (all statuses); Cancel or Delete any event |
| **Users** | View all accounts; change role via dropdown; Deactivate/Reactivate (own account protected) |
| **Tickets** | View all tickets across all students; Void any ticket |
| **Audit Log** | Immutable timestamped trail of every system action |

---

## 7. GitHub Pages Deployment Guide

This section is written specifically for **student club officers and IT leads** who want to host CampGroove for their organisation.

### Prerequisites

- A GitHub account
- Git installed on your machine (`git --version` to check)
- Your university's `.edu` email domain(s) ready to configure

### Step 1 — Fork or create the repository

**Option A — Fork (recommended for clubs)**

1. Go to the CampGroove repository on GitHub
2. Click **Fork** in the top-right corner
3. Name it something like `yourclub-tickets` and click **Create fork**

**Option B — New repository from scratch**

```bash
# Create a new folder and initialise git
mkdir yourclub-tickets
cd yourclub-tickets
git init

# Copy index.html and README.md into this folder, then:
git add .
git commit -m "Initial CampGroove deployment"
```

### Step 2 — Push to GitHub

```bash
# Add your GitHub repository as the remote origin
git remote add origin https://github.com/YOUR-USERNAME/yourclub-tickets.git

# Push to main branch
git branch -M main
git push -u origin main
```

### Step 3 — Enable GitHub Pages

1. Open your repository on GitHub
2. Go to **Settings** → **Pages** (in the left sidebar under "Code and automation")
3. Under **Source**, select **Deploy from a branch**
4. Set **Branch** to `main` and folder to `/ (root)`
5. Click **Save**

GitHub will display a URL like:
```
https://YOUR-USERNAME.github.io/yourclub-tickets/
```

Your site is live in ~60 seconds. Every `git push` to `main` automatically redeploys.

### Step 4 — Customise for your club

Before pushing, open `index.html` and make these quick changes:

```html
<!-- Line 6: Update the page title -->
<title>YourClub Events — Ticket Portal</title>
```

```js
// Line ~936–948: Replace SEED_USERS with your club officers
const SEED_USERS = [
  { id:'u1', name:'Your Name', email:'you@youruni.edu', sid:'EXEC-001',
    password:'changeme', role:'admin', isActive:true, createdAt:'2025-01-01' },
  { id:'u2', name:'Events Lead', email:'events@youruni.edu', sid:'ORG-001',
    password:'changeme', role:'organizer', isActive:true, createdAt:'2025-01-01' },
];

// Replace SEED_EVENTS with your real upcoming events
const SEED_EVENTS = [
  {
    id:'e1', title:'Welcome Week Mixer', category:'social',
    capacity:200, ticketsRemaining:200, price:0,
    venue:'Student Union Room B', date:'2025-09-05T18:00',
    organizer:'u2', organizerName:'Events Lead', status:'published',
    description:'Kick off the year with free pizza, music, and new friends!'
  },
  // ... add more events
];
```

```js
// Line ~716 (hero section): Update the announcement banner text
Spring Semester 2025 — Events Now Live
// Change to:
Fall 2025 — Your Club Name Events
```

### Step 5 — Restrict to your .edu domain (optional hardening)

By default, `EDU_REGEX` accepts **any** `.edu` address. To restrict to your specific university:

```js
// Around line 1124 — change this:
const EDU_REGEX = /^[a-zA-Z0-9._%+\-]+@[a-zA-Z0-9.\-]+\.edu$/i;

// To this (replace youruni.edu with your domain):
const EDU_REGEX = /^[a-zA-Z0-9._%+\-]+@youruni\.edu$/i;

// Or allow multiple domains:
const ALLOWED_DOMAINS = ['youruni.edu', 'grad.youruni.edu'];
function isValidEduEmail(email){
  return ALLOWED_DOMAINS.some(d => email.toLowerCase().endsWith('@'+d));
}
const EDU_REGEX = { test: isValidEduEmail }; // drop-in replacement
```

### Step 6 — Custom domain (optional)

To use `tickets.yourclub.org` instead of the GitHub Pages URL:

1. Add a file named `CNAME` in the root of your repository containing just your domain:
   ```
   tickets.yourclub.org
   ```
2. With your DNS provider, add a `CNAME` record pointing `tickets.yourclub.org` → `YOUR-USERNAME.github.io`
3. In GitHub Pages settings, enter your custom domain and enable **Enforce HTTPS**

### Deployment Checklist

```
[ ] Repository created and index.html pushed to main branch
[ ] GitHub Pages enabled (Settings → Pages → Deploy from branch: main)
[ ] SEED_USERS updated with real club officer accounts
[ ] SEED_EVENTS updated with real upcoming events  
[ ] Page title updated in <title> tag
[ ] Hero banner text updated
[ ] .edu domain restriction tightened to your university (optional)
[ ] CNAME configured for custom domain (optional)
[ ] Test all 3 roles by logging in with each demo account
[ ] Verify purchase flow end-to-end (ticket → drawer → QR displayed)
```

---

## 8. Customisation Guide (Branding & Copy)

### Colours

All brand colours live in the Tailwind config block at the top of `index.html`:

```js
// Lines 9-19
tailwind.config = {
  theme: {
    extend: {
      colors: {
        brand: { DEFAULT:'#7c3aed', dark:'#5b21b6', light:'#ede9fe' }, // ← change these
        accent: '#f59e0b'                                                // ← amber accent
      }
    }
  }
}
```

Replace `#7c3aed` (brand purple) with your club's primary colour. The dark/light variants should be a darker shade and a tint respectively.

### Hero gradient

```css
/* Around line 23 in <style> */
.hero-gradient {
  background: linear-gradient(135deg, #7c3aed 0%, #4f46e5 50%, #0ea5e9 100%);
}
/* Replace the three hex values with your gradient colours */
```

### App name

Search for `CampGroove` (case-sensitive) — it appears in the `<title>`, the navbar brand text, the auth modal header, and the ticket template footer. Replace all occurrences with your club's name.

---

## 9. Maintenance Guide — Updating Event Data Models

This section explains exactly how to extend or modify the event data model, for future maintainers who are not the original developers.

### How the data model works

All event objects are created in two places:

1. **Seed data** (`SEED_EVENTS` array, ~line 941) — pre-loaded at page start
2. **`publishEvent()` function** (~line 1343) — creates new event objects at runtime

Both must stay in sync. If you add a new field, you must update **both locations** plus every place the field is read.

### Adding a new field to an Event

**Example: adding a `dresscode` field**

#### Step 1 — Add to the seed data

```js
// In SEED_EVENTS (~line 941), add the field to each seed event:
{ id:'e1', title:'Spring Music Fest', ..., dresscode: 'Smart casual' },
{ id:'e2', title:'Inter-College Hackathon', ..., dresscode: '' },
// Leave empty string '' for events without a dress code
```

#### Step 2 — Add to the organiser publish form

In the `#orgModal` HTML (~line 257), add a new input field:

```html
<!-- After the Venue field, around line 240 -->
<div class="md:col-span-2">
  <label class="block text-sm font-semibold text-gray-700 mb-1">
    Dress Code <span class="text-gray-400 font-normal">(optional)</span>
  </label>
  <input id="ev_dresscode" type="text" placeholder="e.g. Smart casual, Black tie, Casual"
    class="input-base" />
</div>
```

#### Step 3 — Add to `publishEvent()` (~line 1343)

```js
function publishEvent(){
  // ...existing field reads...
  const dresscode = document.getElementById('ev_dresscode').value.trim(); // ← add this

  // ...existing validation...

  const newEvent = {
    id: genId(),
    title, category:cat, capacity:cap, ticketsRemaining:cap,
    price: isNaN(price)?0:price,
    venue, date, description:desc,
    dresscode,                   // ← add this line
    organizer: currentUser.id,
    organizerName: currentUser.name,
    status:'published',
    createdAt: new Date().toISOString()
  };
  Store.addEvent(newEvent);
  // ...
}
```

#### Step 4 — Display in the event card (optional)

In `renderEventCard()` (~line 1780), add to the card body:

```js
// After the description paragraph:
${ev.dresscode ? `<p class="text-xs text-gray-400">👔 ${esc(ev.dresscode)}</p>` : ''}
```

#### Step 5 — Display in the ticket template (optional)

In `buildTicketTemplate()` (~line 1599), add a ticket-field block inside the grid:

```html
<div class="ticket-field">
  <label>Dress Code</label>
  <span>${esc(t.dresscode||'Not specified')}</span>
</div>
```

But first, pass `dresscode` through in `Store.purchaseOrder()` (~line 1001):

```js
return {
  id: tid,
  // ...existing fields...
  dresscode: ev.dresscode || '',   // ← add this
};
```

#### Step 6 — Display in the My Tickets drawer (optional)

In `renderTicketDrawer()` (~line 1953), inside the card body grid:

```js
${t.dresscode ? `
  <div>
    <p class="text-[10px] font-bold text-gray-400 uppercase tracking-wide">Dress Code</p>
    <p class="text-gray-700 font-medium">${esc(t.dresscode)}</p>
  </div>` : ''}
```

#### Step 7 — Display in Admin dashboard (optional)

In `renderAdminEvents()` (~line 2271), add a column to the `<thead>` and a `<td>` in the row template:

```js
// In the thead:
<th>Dress Code</th>

// In the tbody tr:
<td>${esc(ev.dresscode||'—')}</td>
```

---

### Changing an existing field name

If you need to rename a field (e.g. `description` → `hypeText`):

1. Find all occurrences in `index.html`:
   ```
   Ctrl+F → description
   ```
2. Replace in `SEED_EVENTS`, `publishEvent()`, `Store.purchaseOrder()`, `renderEventCard()`, `buildTicketTemplate()`, `renderAdminEvents()`, and anywhere else it appears

> **Tip:** Always search for `ev.description`, `t.description`, and `ev_desc` separately — form element IDs use the shortened name.

---

### Adding a new event category

See [Section 10](#10-adding-a-new-category) for the full category extension checklist.

---

### Changing the ticket quantity limit per student

The max tickets per student per event is controlled by a single constant:

```js
// In the PM (purchase modal) state object (~line 1371):
const PM = {
  // ...
  MAX_PER_STUDENT: 5,   // ← change this number
  // ...
};
```

This value is automatically respected by `changeQty()`, `openPurchaseModal()`, and the cap calculation in `purchaseOrder()`.

---

### Updating the mock Campus Account balance

The default mock balance is set on the `PM` object:

```js
const PM = {
  // ...
  balance: 250.00,   // ← change this to your club's default mock balance
};
```

This resets to this value on every page load. If you want per-user balances, store them on the User object in `SEED_USERS` and read `currentUser.balance` in `populatePaymentStep()`.

---

### Updating the OTP demo code

```js
// In handleOtp() (~line 1313):
if(code !== '123456'){   // ← change '123456' to another demo code
```

Also update the hint text in the `#otpPanel` HTML:
```html
<span class="font-bold text-brand">Demo OTP: 123456</span>
```

---

## 10. Adding a New Category

To add a category (e.g. `wellness`), update these 6 locations:

### 1. Tag CSS (`<style>` block, ~line 38)

```css
.tag-wellness { background: #f0fdf4; color: #166534; }
```

### 2. `catGradient()` function (~line 1849)

```js
function catGradient(cat){
  const m = {
    // ...existing entries...
    wellness: 'linear-gradient(135deg,#f0fdf4,#86efac)',  // ← add
  };
  return m[cat] || 'linear-gradient(135deg,#f3f4f6,#e5e7eb)';
}
```

### 3. `categoryEmoji()` function (~line 1128)

```js
function categoryEmoji(cat){
  const m = {
    // ...existing entries...
    wellness: '🧘',   // ← add
  };
  return m[cat] || '🎪';
}
```

### 4. Organiser publish form — category `<select>` (~line 270)

```html
<option value="wellness">🧘 Wellness</option>
```

### 5. Events grid filter `<select>` (~line 878)

```html
<option value="wellness">Wellness</option>
```

### 6. My Tickets drawer filter tabs (~line 165)

```html
<button onclick="filterDrawer('wellness',this)">🧘 Wellness</button>
```

---

## 11. Extending to a Real Backend

CampGroove is designed so that the frontend state mutations can be replaced with API calls with minimal refactoring. The architecture plan (`campgroove-plan.md`) defines the full backend specification. Here is the replacement map:

| Current (in-memory) | Future (API call) |
|---|---|
| `Store.login(user)` | `POST /api/auth/login` → JWT stored in `localStorage` |
| `Store.registerUser(data)` | `POST /api/auth/register` |
| `Store.purchaseOrder(eventId, qty, method)` | `POST /api/tickets/reserve` with Stripe PaymentIntent |
| `Store.addEvent(ev)` | `POST /api/events` |
| `Store.cancelEvent(id)` | `PATCH /api/events/:id/cancel` |
| `Store.deactivateUser(id)` | `PATCH /api/admin/users/:id` |
| `SEED_EVENTS` | `GET /api/events?status=published` on page load |

The full backend spec — including Express routes, Mongoose schemas, Stripe integration, and environment variables — is documented in [`campgroove-plan.md`](campgroove-plan.md).

---

## 12. Troubleshooting

### Events grid is empty after adding seed events

Check that every event in `SEED_EVENTS` has `status: 'published'`. Events with `status: 'draft'` or `status: 'cancelled'` are filtered out of the public grid.

### "Must be a valid .edu email address" shows even for a valid email

The regex requires the **entire domain** to end in `.edu` — not just contain `.edu` somewhere. Ensure the email is `user@something.edu`, not `user@something.edu.fake.com`. If you restricted to a specific domain (Step 5 of the deployment guide), verify the domain spelling matches exactly.

### QR codes look identical for different tickets

The SVG QR is deterministic — the same payload always produces the same pattern. If two tickets show the same QR, their payloads are identical. Check that `t.createdAt` is unique per ticket (it uses `new Date().toISOString()` at purchase time, which is millisecond-precise).

### Drawer is not opening on "My Tickets"

Ensure the user is logged in. `openTicketDrawer()` redirects to the login modal if `currentUser` is null. Also check that the user's role is `student` or `organizer` — admin accounts do not have a ticket drawer trigger.

### Purchase button shows "Sold Out" but tickets are available

The per-student cap (default 5) may be reached. The button shows "Limit reached (5/student)" in this case, not "Sold Out". If it shows "Sold Out", `ticketsRemaining` has reached 0 — check `SEED_EVENTS` initial values or whether a previous purchase in the session decremented the count.

### Admin dashboard shows $0 revenue for paid events

Revenue is calculated from `t.amountPaid` on tickets with `status: 'reserved'`. If you voided those tickets in the Admin → Tickets tab, they are excluded. Check the Tickets tab to verify ticket statuses.

---

## 13. Contributing

Contributions from student developers are welcome. The fastest way to propose a change:

1. Fork the repository
2. Create a branch: `git checkout -b feature/your-feature-name`
3. Make your changes to `index.html`
4. Test all three roles (student, organiser, admin) manually
5. Open a Pull Request with a description of what changed and why

### Code style

- All JavaScript is vanilla ES2020+ — no TypeScript, no transpilation
- Follow the existing comment block style (`/* ─────── */`) for new sections
- Keep functions small and single-purpose
- All new UI strings must pass through `esc()` to prevent XSS

---

## 14. Licence

This project is released under the **MIT Licence**.  
Free to use, modify, and redistribute for educational and non-commercial purposes.  
Student clubs may deploy this freely without attribution, though a note in the footer is appreciated.

---

<div align="center">

**Built with ❤️ for .edu communities**  
CampGroove · College Event Ticketing Platform

</div>
