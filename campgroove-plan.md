# CampGroove — College Event Ticket App · Architecture Plan

## Top-Level Overview

**Goal:** Build CampGroove, a responsive, lightweight web application for college event ticketing. The system targets college students (ticket buyers) and student organizer leaders (event creators), with a System Admin layer that has full override authority.

**Scope:** Two separate projects inside the `TICKET APP/` workspace folder:
- `campgroove-client` — React (Vite) SPA, Tailwind CSS for styling, React Router v6 for navigation.
- `campgroove-server` — Node.js / Express REST API, MongoDB (Mongoose ODM), JWT authentication.

**Key constraints:**
- Authentication is restricted to `.edu` email addresses only.
- OTP email verification is simulated (console-logged or stored in DB, not sent via real SMTP).
- Payment uses Stripe SDK in **test mode** (real library, no real charges).
- QR code generation is fully client-side (`qrcode.react`).
- All work lives under `TICKET APP/`.

---

## Data Model Entities

### User
| Field | Type | Notes |
|---|---|---|
| `_id` | ObjectId | |
| `name` | String | |
| `email` | String | Must end in `.edu`; unique |
| `passwordHash` | String | bcrypt |
| `role` | Enum | `student` \| `organizer` \| `admin` |
| `isVerified` | Boolean | OTP email verified |
| `isActive` | Boolean | Admin can deactivate |
| `otp` | String | Temp 6-digit code |
| `otpExpiresAt` | Date | |
| `createdAt` | Date | |

### Event
| Field | Type | Notes |
|---|---|---|
| `_id` | ObjectId | |
| `title` | String | |
| `description` | String | |
| `category` | String | e.g. Music, Sports, Academic |
| `venue` | String | |
| `date` | Date | |
| `capacity` | Number | Total seats |
| `ticketsRemaining` | Number | Decremented on reservation |
| `price` | Number | 0 = free |
| `coverImage` | String | URL or base64 |
| `organizer` | ObjectId → User | |
| `status` | Enum | `draft` \| `published` \| `cancelled` |
| `createdAt` | Date | |

### Ticket
| Field | Type | Notes |
|---|---|---|
| `_id` | ObjectId | |
| `event` | ObjectId → Event | |
| `student` | ObjectId → User | |
| `stripePaymentIntentId` | String | Test-mode PI id |
| `amountPaid` | Number | |
| `qrPayload` | String | `ticketId:eventId:studentId` composite |
| `status` | Enum | `reserved` \| `cancelled` |
| `createdAt` | Date | |

### Notification
| Field | Type | Notes |
|---|---|---|
| `_id` | ObjectId | |
| `recipient` | ObjectId → User | |
| `type` | Enum | `ticket_confirmed` \| `event_cancelled` \| `account_action` |
| `message` | String | |
| `isRead` | Boolean | |
| `createdAt` | Date | |

---

## Component Breakdown (Client)

```
campgroove-client/src/
├── main.jsx                    # Vite entry point
├── App.jsx                     # Router root, global providers
├── api/                        # Axios instance + per-resource API helpers
│   ├── axiosClient.js
│   ├── authApi.js
│   ├── eventsApi.js
│   ├── ticketsApi.js
│   └── adminApi.js
├── context/
│   ├── AuthContext.jsx          # JWT, user role, login/logout
│   └── NotificationContext.jsx  # Unread count, fetch notifications
├── routes/
│   ├── ProtectedRoute.jsx       # Redirect if not authed
│   └── RoleRoute.jsx            # Redirect if wrong role
├── pages/
│   ├── auth/
│   │   ├── RegisterPage.jsx
│   │   ├── VerifyOtpPage.jsx
│   │   └── LoginPage.jsx
│   ├── student/
│   │   ├── EventsPage.jsx       # Browse & search published events
│   │   ├── EventDetailPage.jsx  # Single event + Buy ticket CTA
│   │   ├── CheckoutPage.jsx     # Stripe Elements mock payment
│   │   ├── TicketConfirmPage.jsx# QR code + ticket details
│   │   └── MyTicketsPage.jsx    # Student's ticket history
│   ├── organizer/
│   │   ├── OrgDashboardPage.jsx # My events list
│   │   ├── CreateEventPage.jsx  # Create/Edit event form
│   │   └── EventStatsPage.jsx   # Tickets sold, capacity gauge
│   └── admin/
│       ├── AdminDashboardPage.jsx # System overview stats
│       ├── AdminUsersPage.jsx     # All users, deactivate/change role
│       ├── AdminEventsPage.jsx    # All events, CRUD override
│       └── AdminTicketsPage.jsx   # All tickets, cancel override
├── components/
│   ├── layout/
│   │   ├── Navbar.jsx
│   │   ├── Sidebar.jsx          # Organizer / Admin only
│   │   └── Footer.jsx
│   ├── ui/
│   │   ├── Button.jsx
│   │   ├── Input.jsx
│   │   ├── Modal.jsx
│   │   ├── Badge.jsx            # Event status, ticket status
│   │   ├── Spinner.jsx
│   │   └── Toast.jsx
│   ├── events/
│   │   ├── EventCard.jsx        # Grid card with cover, date, price
│   │   ├── EventGrid.jsx        # Responsive card grid
│   │   └── EventStatusBadge.jsx
│   ├── tickets/
│   │   ├── QRCodeDisplay.jsx    # wraps qrcode.react
│   │   └── TicketCard.jsx       # Mini ticket stub component
│   ├── notifications/
│   │   └── NotificationBell.jsx # Dropdown with unread items
│   └── checkout/
│       └── StripePaymentForm.jsx # Stripe Elements wrapper
```

---

## API Route Breakdown (Server)

```
campgroove-server/
├── src/
│   ├── index.js                 # Express entry, MongoDB connect
│   ├── config/
│   │   └── db.js                # Mongoose connect helper
│   ├── middleware/
│   │   ├── authMiddleware.js    # Verify JWT, attach req.user
│   │   └── roleMiddleware.js    # Require specific role(s)
│   ├── models/
│   │   ├── User.js
│   │   ├── Event.js
│   │   ├── Ticket.js
│   │   └── Notification.js
│   └── routes/
│       ├── auth.js              # POST /register, /verify-otp, /login
│       ├── events.js            # CRUD events (organizer + admin)
│       ├── tickets.js           # POST /reserve, GET /my-tickets
│       ├── notifications.js     # GET, PATCH /read
│       ├── stripe.js            # POST /create-payment-intent
│       └── admin.js             # Full CRUD override routes
```

---

## Modular Development Roadmap (Sub-Tasks)

---

### Sub-Task 1 — Project Scaffolding
**Status:** `[ ] pending`

**Intent:** Create the folder structure, initialize both projects, and install all dependencies so every subsequent sub-task has a working base.

**Expected Outcomes:**
- `campgroove-server/` is an Express app that starts with `npm run dev` and responds to `GET /health`.
- `campgroove-client/` is a Vite + React app that starts with `npm run dev` and renders a placeholder page.
- Both projects have their `.env.example` files documenting required variables.
- ESLint + Prettier configured for both.

**Todo List:**
1. Create `campgroove-server/` directory; run `npm init -y`; install `express cors dotenv mongoose bcryptjs jsonwebtoken`.
2. Install dev deps: `nodemon eslint prettier`.
3. Create `src/index.js` with Express app, CORS, JSON body parser, `/health` route, and `mongoose.connect`.
4. Create `src/config/db.js` Mongoose helper.
5. Add `.env.example` with `PORT`, `MONGO_URI`, `JWT_SECRET`, `STRIPE_SECRET_KEY`.
6. Add `npm run dev` script using `nodemon`.
7. Create `campgroove-client/` using `npm create vite@latest` with React template.
8. Install `axios react-router-dom @stripe/react-stripe-js @stripe/stripe-js qrcode.react`.
9. Install Tailwind CSS with Vite plugin; configure `tailwind.config.js`.
10. Add `.env.example` with `VITE_API_URL`, `VITE_STRIPE_PUBLISHABLE_KEY`.
11. Replace default Vite placeholder with a CampGroove branded shell page.

**Relevant Context:** Both projects live under `TICKET APP/`. Server runs on port 5000, client on 5173 (Vite default).

---

### Sub-Task 2 — Data Models & Auth API
**Status:** `[ ] pending`

**Intent:** Define all Mongoose schemas and build the authentication API (register with `.edu` validation, OTP simulation, login returning JWT).

**Expected Outcomes:**
- All four Mongoose models exist with correct field types, validations, and indexes.
- `POST /api/auth/register` rejects non-`.edu` emails with a descriptive error; hashes password; generates and stores a 6-digit OTP; logs OTP to console.
- `POST /api/auth/verify-otp` verifies the code, marks `isVerified: true`, returns JWT.
- `POST /api/auth/login` verifies password + verified status, returns JWT.
- Auth middleware (`authMiddleware.js`) validates JWT and attaches `req.user`.
- Role middleware (`roleMiddleware.js`) accepts array of allowed roles.

**Todo List:**
1. Create `src/models/User.js` — schema with all fields, `pre-save` bcrypt hook, `.edu` regex validator on email.
2. Create `src/models/Event.js` with all fields and index on `status` + `date`.
3. Create `src/models/Ticket.js` with all fields; compound unique index on `{event, student}` to prevent duplicates.
4. Create `src/models/Notification.js` with all fields.
5. Create `src/middleware/authMiddleware.js` — verify `Bearer` JWT, attach `req.user`.
6. Create `src/middleware/roleMiddleware.js` — factory `requireRole(...roles)`.
7. Create `src/routes/auth.js` with register, verify-otp, login handlers.
8. Mount `auth.js` at `/api/auth` in `index.js`.
9. Write manual `curl` test cases in a `docs/api-tests.md` file for each endpoint.

**Relevant Context:** OTP is console-logged only (no real email). JWT payload: `{ id, role }`. Token expiry: `7d`.

---

### Sub-Task 3 — Events API (Organizer + Admin)
**Status:** `[ ] pending`

**Intent:** Build the full events CRUD API, gating creation/editing to organizers and admins, and public read access to published events.

**Expected Outcomes:**
- `GET /api/events` — public, returns published events, supports `?category=`, `?search=`, `?date=` query filters.
- `GET /api/events/:id` — public, single event detail.
- `POST /api/events` — organizer/admin only; creates draft event.
- `PUT /api/events/:id` — organizer (own events only) or admin (any event).
- `PATCH /api/events/:id/publish` — organizer/admin; sets status to `published`.
- `PATCH /api/events/:id/cancel` — organizer/admin; sets status to `cancelled`; triggers notification to all ticket holders.
- `DELETE /api/events/:id` — admin only.

**Todo List:**
1. Create `src/routes/events.js` with all seven route handlers.
2. Add organizer-ownership guard: non-admin organizers may only mutate their own events.
3. Implement query filter logic for GET /api/events.
4. On cancel, query all tickets for the event, create a Notification document for each ticket holder.
5. Mount at `/api/events` in `index.js`.

**Relevant Context:** `ticketsRemaining` is set to `capacity` on creation. Cancellation notification uses `type: 'event_cancelled'`.

---

### Sub-Task 4 — Ticket Reservation & Stripe Mock Payment
**Status:** `[ ] pending`

**Intent:** Build the ticket reservation flow: create a Stripe test-mode PaymentIntent on the server, confirm it on the client with Stripe Elements, then POST to reserve the ticket.

**Expected Outcomes:**
- `POST /api/stripe/create-payment-intent` — authenticated student; receives `eventId`; returns Stripe `client_secret`.
- `POST /api/tickets/reserve` — authenticated student; receives `eventId` + `paymentIntentId`; checks remaining capacity; atomically decrements `ticketsRemaining`; creates Ticket; creates `ticket_confirmed` Notification.
- `GET /api/tickets/my-tickets` — returns the calling student's tickets with populated event data.
- Duplicate ticket purchase is rejected (compound unique index enforced).

**Todo List:**
1. Install `stripe` npm package on server.
2. Create `src/routes/stripe.js` — POST handler creates PaymentIntent with `amount` from event price (min 50 cents for Stripe); returns `clientSecret`.
3. Create `src/routes/tickets.js` — reserve handler with capacity check using `findOneAndUpdate` atomic decrement; create Ticket and Notification docs.
4. Add `GET /my-tickets` with `populate('event')`.
5. Mount both routes in `index.js`.

**Relevant Context:** For free events, skip Stripe and reserve directly. Use `{ $inc: { ticketsRemaining: -1 }, $set: {} }` with `{ new: true, runValidators: true }` and a `ticketsRemaining: { $gt: 0 }` filter to atomically prevent overselling.

---

### Sub-Task 5 — Notifications API
**Status:** `[ ] pending`

**Intent:** Build a simple read/mark-read notifications endpoint for the bell icon in the UI.

**Expected Outcomes:**
- `GET /api/notifications` — returns the authenticated user's notifications, newest first.
- `PATCH /api/notifications/:id/read` — marks a single notification as read.
- `PATCH /api/notifications/read-all` — marks all of the user's notifications as read.

**Todo List:**
1. Create `src/routes/notifications.js` with three handlers.
2. All routes require `authMiddleware`.
3. Mount at `/api/notifications` in `index.js`.

---

### Sub-Task 6 — Admin Override API
**Status:** `[ ] pending`

**Intent:** Expose admin-only endpoints that give full CRUD authority over users, events, and tickets.

**Expected Outcomes:**
- `GET /api/admin/users` — paginated user list with role filter.
- `PATCH /api/admin/users/:id` — update role or `isActive` (deactivate/reactivate).
- `DELETE /api/admin/users/:id` — hard delete user.
- `GET /api/admin/events` — all events (any status).
- `PUT /api/admin/events/:id` — override any event field.
- `DELETE /api/admin/events/:id` — hard delete event.
- `GET /api/admin/tickets` — all tickets.
- `PATCH /api/admin/tickets/:id/cancel` — cancel any ticket; restore `ticketsRemaining`.
- `GET /api/admin/stats` — aggregate counts: total users, events, tickets, revenue.

**Todo List:**
1. Create `src/routes/admin.js` with all handlers.
2. Gate every route with `requireRole('admin')`.
3. Stats route uses `Ticket.aggregate` for revenue sum and counts.
4. Mount at `/api/admin` in `index.js`.

---

### Sub-Task 7 — Client Auth Flow (Register / OTP / Login)
**Status:** `[ ] pending`

**Intent:** Build the React authentication screens and the `AuthContext` that stores the JWT and drives role-based routing.

**Expected Outcomes:**
- `AuthContext` stores decoded user (`id`, `role`, `name`) and provides `login()`, `logout()`.
- `RegisterPage` validates `.edu` email on client before submitting; shows error toast on non-.edu.
- `VerifyOtpPage` accepts 6-digit code; redirects to appropriate dashboard on success.
- `LoginPage` logs in and redirects based on role (`student` → `/events`, `organizer` → `/organizer`, `admin` → `/admin`).
- `ProtectedRoute` redirects unauthenticated users to `/login`.
- `RoleRoute` redirects users without the required role to `/unauthorized`.

**Todo List:**
1. Create `src/api/axiosClient.js` — Axios instance with `VITE_API_URL` base URL; request interceptor attaches JWT from localStorage.
2. Create `src/api/authApi.js` — `register`, `verifyOtp`, `login` functions.
3. Create `src/context/AuthContext.jsx` — decode JWT on load; expose `user`, `login`, `logout`.
4. Create `src/routes/ProtectedRoute.jsx` and `src/routes/RoleRoute.jsx`.
5. Build `RegisterPage`, `VerifyOtpPage`, `LoginPage` with Tailwind styling.
6. Configure React Router in `App.jsx` with all route definitions.

---

### Sub-Task 8 — Student Event Browsing & Checkout Flow
**Status:** `[ ] pending`

**Intent:** Build the student-facing pages: event discovery, event detail, Stripe checkout, and ticket confirmation with QR code.

**Expected Outcomes:**
- `EventsPage` shows a responsive grid of published events; has search bar + category filter.
- `EventDetailPage` shows full event info; "Get Ticket" button is disabled when sold out.
- `CheckoutPage` renders Stripe Elements (card number, expiry, CVC); on success calls `/api/tickets/reserve`.
- `TicketConfirmPage` renders the QR code (payload: `ticketId:eventId:studentId`) using `qrcode.react`; allows download.
- `MyTicketsPage` lists the student's reserved tickets.

**Todo List:**
1. Create `src/api/eventsApi.js` and `src/api/ticketsApi.js`.
2. Build `EventCard`, `EventGrid`, `EventStatusBadge` components.
3. Build `EventsPage` with filter/search state and API call.
4. Build `EventDetailPage` with sold-out guard.
5. Build `CheckoutPage` — wrap with `<Elements stripe={...}>`, handle `stripe.confirmCardPayment`, then call reserve endpoint.
6. Build `TicketConfirmPage` with `QRCodeDisplay` component using `qrcode.react`.
7. Build `MyTicketsPage` with `TicketCard` component.

**Relevant Context:** Use Stripe test card `4242 4242 4242 4242`. Stripe publishable key from `VITE_STRIPE_PUBLISHABLE_KEY`.

---

### Sub-Task 9 — Organizer Dashboard
**Status:** `[ ] pending`

**Intent:** Build the organizer-facing pages for creating, editing, publishing, and monitoring their events.

**Expected Outcomes:**
- `OrgDashboardPage` lists all events belonging to the logged-in organizer with status badges and action buttons (edit, publish, cancel).
- `CreateEventPage` is a form with all Event fields; submits to `POST /api/events`; re-used for editing via `PUT /api/events/:id`.
- `EventStatsPage` shows capacity gauge (tickets sold vs total) and list of ticket holders.

**Todo List:**
1. Build `OrgDashboardPage` with event list, status badges, and inline publish/cancel actions.
2. Build `CreateEventPage` with controlled form; image URL input (no file upload needed).
3. Build `EventStatsPage` with a simple visual capacity bar and ticket holder table.
4. Protect all organizer routes with `RoleRoute` requiring `organizer` or `admin`.

---

### Sub-Task 10 — Notification Bell
**Status:** `[ ] pending`

**Intent:** Wire up the `NotificationBell` component to poll the notifications API and allow students/organizers to view and dismiss notifications.

**Expected Outcomes:**
- Bell icon in Navbar shows unread count badge.
- Clicking opens a dropdown listing the 10 most recent notifications.
- Clicking a notification marks it as read; "Mark all read" button available.
- Polling every 30 seconds (simple `setInterval` approach).

**Todo List:**
1. Create `src/context/NotificationContext.jsx` — fetch on mount and every 30 s; expose `notifications`, `unreadCount`, `markRead`, `markAllRead`.
2. Build `NotificationBell.jsx` with badge and dropdown.
3. Add `NotificationContext` provider in `App.jsx`.
4. Wire `markRead` call on notification item click.

---

### Sub-Task 11 — Admin Dashboard
**Status:** `[ ] pending`

**Intent:** Build the admin-facing pages giving full system oversight and override controls.

**Expected Outcomes:**
- `AdminDashboardPage` shows aggregate stats: total users, published events, tickets sold, total revenue.
- `AdminUsersPage` lists all users in a table with role badges; has "Deactivate", "Change Role", "Delete" actions per row.
- `AdminEventsPage` lists all events (all statuses); has "Edit", "Cancel", "Delete" actions; admin can also create events.
- `AdminTicketsPage` lists all tickets with "Cancel Ticket" override action.
- All admin pages protected by `RoleRoute` requiring `admin`.

**Todo List:**
1. Create `src/api/adminApi.js` — all admin API helper functions.
2. Build `AdminDashboardPage` with stats cards.
3. Build `AdminUsersPage` with data table, role selector dropdown, deactivate toggle, delete button.
4. Build `AdminEventsPage` re-using `CreateEventPage` form in a modal for admin edits.
5. Build `AdminTicketsPage` with cancel override.
6. Add admin navigation links to `Sidebar` (admin-only visible).

---

### Sub-Task 12 — UI Polish, Responsiveness & Final Wiring
**Status:** `[ ] pending`

**Intent:** Ensure the entire app is responsive, visually consistent, and all pages are correctly linked and protected. Final end-to-end smoke test.

**Expected Outcomes:**
- All pages are mobile-responsive (Tailwind responsive prefixes used throughout).
- `Navbar` shows correct links per role; hides organizer/admin links from students.
- Toast notifications appear for all async operations (success + error).
- 404 page exists for unmatched routes.
- `README.md` files in both projects document setup steps and environment variables.

**Todo List:**
1. Build `Toast.jsx` with auto-dismiss; integrate into global layout.
2. Audit all pages for mobile breakpoints; fix layout issues.
3. Build role-aware `Navbar` with mobile hamburger menu.
4. Add `NotFoundPage.jsx` and wire to React Router catch-all.
5. Write `campgroove-client/README.md` and `campgroove-server/README.md`.
6. End-to-end walkthrough: register → verify → browse → checkout → view QR → organizer creates event → admin overrides.

---

## Environment Variables Reference

### `campgroove-server/.env`
```
PORT=5000
MONGO_URI=mongodb://localhost:27017/campgroove
JWT_SECRET=your_jwt_secret_here
STRIPE_SECRET_KEY=sk_test_...
```

### `campgroove-client/.env`
```
VITE_API_URL=http://localhost:5000/api
VITE_STRIPE_PUBLISHABLE_KEY=pk_test_...
```
