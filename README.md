# MoviQ — Movie Ticket Booking Platform

A full-stack movie ticket booking system with login, live seat maps, and a
concurrency-safe booking engine that guarantees a seat can never be sold
twice — even when two people click "Pay" at the exact same instant.

## Quick Start

```bash
npm install
npm start
```

Then open **http://localhost:4000** in your browser. That's it — the
SQLite database is created and seeded automatically on first run
(`server/moviq.db`). Delete that file any time to reset all data.

Requires Node.js 18+.

## Admin Panel

A default admin account is seeded automatically on first run:

- **Email:** `admin@moviq.com`
- **Password:** `admin123`

Log in with these credentials and the "Admin" link appears in the top
nav. Regular signups are always normal customers — there's no way to
self-register as an admin from the UI, and every `/api/admin/*` route
is server-side gated so a non-admin (even with a valid login token)
gets a `403` if they try to call it directly.

The database also seeds ~5 demo customer accounts (e.g.
`priya.demo@moviq.dev`, password `demo1234`) who own the pre-filled
"already booked" seats you'll see scattered across each seat map. This
is what makes the Admin Dashboard's numbers add up on a fresh install —
every booked seat has a real booking record and a real user behind it,
so Total Bookings / Revenue / Seats Booked / Registered Users are all
computed from the same data, not just cosmetic.

From the Admin Dashboard you can:
- View live stats (bookings, revenue, seat occupancy, registered users)
- See every **registered user**, with their join date, how many bookings
  they've made, and total amount spent
- See **seat availability broken down per movie and per showtime** —
  booked vs. available count with an occupancy bar for each showtime
- Add new movies (title, genre, language, synopsis, poster image URL)
- Add showtimes to any movie (theater, screen, time, price) — seats are
  auto-generated for the new show
- Remove a showtime or movie, as long as it has no bookings yet (this
  guardrail stops you from accidentally deleting something a customer
  already paid for)
- See every booking made across all customers

## Modules

1. **Auth Module** — sign up / log in, session tokens, passwords hashed
   with scrypt (never stored in plain text). Browsing and booking both
   require a logged-in account.
2. **Movie Catalog Module** — browse now-showing films, search, filter by
   genre.
3. **Showtime Module** — pick a theater/screen and time for a movie.
4. **Seat Selection Module** — a live seat map (auto-refreshes every few
   seconds) showing available / selected / booked seats, grouped by tier
   (Premium / Gold / Standard).
5. **Concurrency-Safe Booking Engine** (`server/seatLockEngine.js`) — the
   core of the project, described below.
6. **Checkout & Mock Payment Module** — order summary + a demo payment
   form (no real charges are made).
7. **Booking Confirmation Module** — a digital ticket stub with a booking
   code and decorative QR-style pattern.
8. **My Bookings Module** — every logged-in user can see their own past
   bookings.
9. **Admin Dashboard Module** — admin-only login, add/remove movies and
   showtimes, live stats (bookings, revenue, occupancy), and a table of
   every booking made on the platform.

## How double-booking is prevented

Every booking request goes through `SeatLockEngine.attemptBooking()`:

- **Already booked?** If the seat is already marked `booked` in the
  database, the request is rejected immediately with a clear message —
  this is the "goes to pay, seat is taken, bounced back to seat
  selection" behaviour.
- **Booked at different times?** The first person to click "Pay" locks
  the seat; anyone who tries afterwards for that same seat is rejected
  and sent back to reselect. No seat is ever double-sold.
- **Clicked at the exact same time?** This is the interesting case. Each
  request registers itself in a short in-memory "arbitration window"
  (~0.7s) before committing. If a second request for the *same seat*
  registers while the first is still inside that window, **both**
  requests are marked as collided — neither one wins the seat. Both
  users are told "someone else tried to book this at the same moment,
  please pick again," and the seat is released back to `available` for
  anyone to try again. This was verified with an automated test that
  fires two simultaneous requests for the same seat and confirms both
  are rejected and the seat remains free.
- The final commit still happens inside a real SQLite transaction that
  re-checks seat status, so the system is safe even under heavier load
  than the in-memory arbitration window alone would guarantee.

## Tech Stack

- **Backend:** Node.js, Express, better-sqlite3 (file-based SQL database,
  zero setup required)
- **Frontend:** Vanilla HTML/CSS/JS (no build step — just open and run)
- **Auth:** scrypt password hashing + bearer-token sessions

## Project Structure

```
moviq/
├── server/
│   ├── server.js          — Express app entry point
│   ├── db.js               — schema + seed data (movies, shows, seats)
│   ├── auth.js              — password hashing & session tokens
│   ├── seatLockEngine.js    — the concurrency-safe booking engine
│   └── routes/
│       ├── auth.js          — signup / login / logout / me
│       ├── movies.js        — movie catalog
│       ├── shows.js         — showtimes & seat maps
│       └── bookings.js      — booking confirm / mine / admin list
└── public/
    ├── index.html
    ├── styles.css
    └── app.js               — SPA-style frontend, talks to the API
```

## Notes

- Movie posters are stylised placeholder art (gradients + emoji) unless
  a `poster_url` is set, either in the seed data or via the admin panel.
- The payment form is a visual mock — no card processor is integrated.
- Change the default admin password in a real deployment — it's a demo
  credential and shouldn't be used as-is in production.
