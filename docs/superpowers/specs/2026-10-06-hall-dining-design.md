# BUP Hall Dining — v1 Design

**Date:** 2026-10-06 · **Status:** Draft for review · **Repo:** NoboIshraque/BUP-hall-dining-web

## 1. Problem

Each night, students book next-day meals by telling a book keeper their serial number, which is written in a paper register. At mealtime they say their serial number and take their food. Nothing checks who is speaking, so for over a month people have been collecting other students' meals by saying their serial numbers.

## 2. Goals and non-goals

**Goals**
- Only the student who booked a meal can collect it. The system makes this decision automatically; counter staff never judge identity.
- Every booked meal can be served once only.
- Students manage their own bookings from their phone, with no nightly queue.
- The kitchen sees exact counts. Students and admins see monthly bills.
- Good enough to demo to the hall authority before any hosting money is spent.

**Not in v1** (see §11): payment tracking, guest meals, handing a meal to a friend, push/email alerts, offline counter mode, multiple halls, face recognition, student photos.

## 3. Roles

| Role | Who | Signs in with | Can |
|---|---|---|---|
| Student | Hall residents on the roster | `@student.bup.edu.bd` | Set meal plan, show meal pass, link or report card, see history and bill |
| Counter | Book keeper, dining staff | `@bup.edu.bd`, on the staff list | Run the counter station, see live counts and printable lists |
| Admin | Dining manager, provost office | `@bup.edu.bd`, on the staff list | Everything, including roster, slots, prices, cutoff, closed days, billing, logs, exceptions |

An email domain alone never grants access. Students must be on the roster and staff on the staff list. An admin can give a role to any allowed account (e.g. a student dining manager). A password-based superuser exists only for first-time setup.

## 4. Booking meals

- **Meal slots** are set by the admin: name, serving window, display order, active or not. Defaults: Breakfast, Lunch, Dinner. During Ramadan, e.g. Sehri and Iftar.
- **Weekly plan.** Each student ticks which slots they eat on each weekday (e.g. lunch and dinner, Sunday–Thursday). It stays in force until they change it. New students start with a default plan set by the admin (default: everything off).
- **One-off changes.** A student can turn a specific meal on or off for a specific date, up to 14 days ahead.
- **Cutoff.** All of day D's meals lock at 22:00 on the day before (time is admin-configurable). After that, only an admin can change day D.
- **Late plan edits are safe.** Plans are stored as versions with a start date. An edit made after a cutoff takes effect from the first day that is still open, so it can never change a locked day.
- **Locking writes the register.** When day D locks, the system writes one booking row per student per meal they are on. Locking runs on the first request to the site after the cutoff, and from an optional daily scheduled command. Running it twice does nothing extra.
- **Closed days** (vacations, Eid): the admin marks dates; no bookings and no charges on those dates. Closing a day that has already locked removes its unserved bookings.

## 5. Collecting a meal (fully automated)

The counter station is a phone on a stand at the serving counter, plus a USB card reader plugged in with an OTG adapter. Students identify themselves in one of two ways:

1. **Meal pass QR** in the app. It encodes a signed token (student + timestamp, signed with a server secret) that refreshes every 20 seconds and expires after 45. A screenshot or photo of someone else's pass stops working within seconds, so a thief would need the owner's signed-in phone.
2. **ID card tap.** The reader types the card chip's serial number, which the station sends to the server. The card must be linked first (§6).

For either one, the server runs the same check in a single transaction:

> identify the student → active student? → which slot is being served now? → booked for this slot today? → not served yet? → mark served → **green**

- The station shows a full-screen **green ✓** (serial number, name, slot) with a success sound, or a **red ✗** with the reason and an error sound. Staff serve on green and refuse on red. Nobody checks faces, and no photos are stored.
- Red reasons: not booked for this meal · already served at <time> · pass expired, refresh it · card not linked · card reported lost · no meal being served now · can't reach the server.
- The current slot is chosen from the serving windows. Staff can switch to another active slot if service runs late; the switch is logged.
- Marking served is a conditional update, so two scans at the same moment can never both succeed.
- Scanning the same QR token again within its lifetime repeats the original green, so a double read or a network retry doesn't show a false "already served".
- Every scan, green or red, goes into a scan log.
- **There is no manual override at the counter.** For exceptions (for example, phone dead and card forgotten), an admin can mark a booked meal served or add an extra, billed meal. A reason is required, and it shows in the student's history. A student without their phone can also sign in on a friend's phone with their own account.
- The station screen stays awake. Either the front or the rear camera can be used, so the phone can face the students (self-scan) or be held by staff.

## 6. ID cards

- **Linking:** the student taps "Link ID card" in the app, which shows a link QR. They scan it at the station, which says "tap your card now" for 10 seconds. Once they tap, the card is linked and the station shows their serial number.
- One active card per student; linking a new card replaces the old one. A card already linked to someone else is rejected, and an admin sorts it out.
- "Report lost card" in the app disables the card immediately.
- Card serial numbers are shown only to admins. They can be copied with special tools. That is acceptable for meals, given served-once, the scan log, the student's history and instant disabling. BUP cards must have a fixed serial number (see §12).

## 7. Student app

A mobile website that students can install to their home screen.

- **Home:** today's meals (booked, or served at what time); tomorrow's meals with on/off toggles and a "locks at 10:00 PM" countdown; a big **Show meal pass** button.
- **Meal pass:** full-screen QR with a countdown ring, serial number, name and current slot. Keeps the screen awake.
- **Plan:** a weekly grid (weekday × slot), plus the next 14 days for one-off changes.
- **History and bill:** meals by date with served status; the current month's running bill and past months. Each served meal has a **Not me?** button that files a report to the admin.
- **Account:** card status, report lost card, sign out of all devices (for a lost phone).

## 8. Staff and admin

- **Live counts** (counter and admin): today per slot (booked, served, remaining); tomorrow (running total until the cutoff, then final). Refreshes on its own.
- **Printable booked list** per slot, as a paper backup if the system is down.
- **Admin panel** (Django admin):
  - students, with CSV roster import (serial no., name, student ID, email, room; matched on email; nobody is deactivated automatically)
  - staff list, slots, prices, closed days, cards
  - settings: cutoff, default plan, hall name
  - bookings: read-only except for exceptions
  - scan log and reports
- **Billing:** each slot has a price with a start date. This supports a fixed price, or a month-end "meal rate" entered at month end and dated from the 1st.
  - Monthly bill = sum of booked meals × the price on that date. Students pay for booked meals even if they don't collect them, as in the current meal-on system. Served status is shown next to each meal.
  - Admins get a monthly summary of all students with CSV export. Tracking payments is out of scope.

## 9. Sign-in

- **Microsoft sign-in** (OpenID Connect via django-allauth), locked to BUP's Microsoft tenant `c28dc317-0fd2-4e74-8494-87f82948d1f4`. Both `student.bup.edu.bd` and `bup.edu.bd` belong to this tenant (checked 2026-10-06). The app checks the tenant ID, not just the email domain, so accounts from other organizations can't pose as BUP addresses. The email must then match the roster or the staff list.
- **Risk:** BUP may require IT approval before students can sign in to a new app. This gets tested early.
- **Backup** that needs nothing from IT: a 6-digit code emailed to the uni address (allauth login by code). It uses codes rather than links because email security scanners, common on Microsoft 365, can open and use up one-time links.
- Sessions last 90 days. "Sign out of all devices" ends them all.

## 10. Architecture

- **Python + Django 5.2 LTS.** Pages are rendered on the server, with one mobile-first CSS file and small plain-JavaScript pieces. There is no frontend build step; Python is the main language to maintain.
- **QR codes:** passes are drawn on the server as SVG (segno). Scanning uses the browser's built-in barcode detector where it exists, with a small bundled JS library otherwise.
- **Database:** SQLite for development and the demo; PostgreSQL in production. Timezone: Asia/Dhaka.
- **One hall per deployment.** Another hall gets another deployment of the same code.
- **HTTPS** is required for camera access and Microsoft sign-in. Demo: the laptop plus a free HTTPS tunnel, or a free host. Production hosting is decided after the demo.

### Units

Each unit can be tested on its own. Rules that depend on time take `now` as a parameter, so tests never depend on the real clock.

| Unit | Job | Interface (sketch) |
|---|---|---|
| booking | Plans, one-off changes, cutoff, locking, counts | `is_open(date, now)` · `save_plan(student, plan, now)` · `set_meal(student, date, slot, on, now)` · `lock_day(date)` · `counts(date)` |
| passes | Issue and check QR tokens | `issue(student, now) → token` · `verify(token, now) → student` (raises Expired / Invalid) |
| cards | Link, replace, report lost, look up | `link(student, uid)` · `report_lost(student)` · `find(uid)` |
| serving | The counter decision | `serve(identifier, slot, now, operator) → Result(ok, reason, student, served_at)` |
| billing | Monthly bills | `month_bill(student, year, month)` · `month_summary(year, month)` |
| accounts | Sign-in rules, roles, session reset | allauth adapter · role checks for student, counter, admin |

### Main tables

| Table | Fields |
|---|---|
| User | email, name, role |
| Student | user, serial no., student ID, room, active |
| Card | uid, student, status |
| MealSlot | code, name, order, serving window, active |
| MealPrice | slot, amount, starts on |
| Settings | single row |
| ClosedDay | date, reason |
| PlanVersion | student, starts on, plan |
| MealChange | student, date, slot, on/off |
| DayLock | date, locked at |
| Booking | student, date, slot, served at, served via, served by, note; unique on student + date + slot |
| ScanLog | time, method, result, reason, student |
| Report | booking, message, status |

### Error handling

Every rule is enforced on the server; the UI only reflects it. On any error the counter shows red, never green. Unknown cards and bad tokens are logged and rejected.

### Testing

Tests are written first for each unit, and must cover at least:
- **Booking:** cutoff edges (21:59 vs 22:00), plan versions vs late edits, one-off changes, closed days, inactive students, running a lock twice.
- **Serving:** every result, including expired or forged passes, repeat scans, lost or unlinked cards, no active slot, and two scans at once.
- **Billing:** a price change mid-month, and closed days.
- **Access:** who can open which page.

On top of the tests, a manual checklist runs on real phones: scanning speed, the card reader over OTG, installing the app, and the screen staying awake.

## 11. Later (not v1)

Push or email alert on every serve · handing a meal to a friend · guest meals · payment tracking · feast or special-meal pricing · offline counter mode · multiple halls · reading cards with the counter phone's own NFC (if BUP cards allow it, the USB reader is no longer needed).

## 12. Open items

1. **Card type.** Use the free *NFC Tools* app on an Android phone: note the tag type and serial number, and tap twice to confirm the serial stays the same. If the phone can't read the card at all, it's probably a 125 kHz card. This decides which USB reader to buy.
2. **Microsoft sign-in approval:** whether BUP's tenant lets students sign in without IT approval. Tested when sign-in is built; the email-code backup covers the demo either way.
3. **Billing:** charging for booked meals even if not collected is assumed. The hall must supply prices or the meal rate.

## 13. Rollout (non-code)

1. Demo to the hall authority with sample data.
2. Import the roster. Students sign in, set plans and link cards (a card-linking table at the counter for a few days).
3. One-week pilot alongside the paper register, comparing counts.
4. Switch over, keeping the register as an emergency backup only.
