# Prompt: sync check for the parent-tracking system

Paste everything below the line into Claude Code, in the project root.

---

<role>
You are a senior Laravel and Vue engineer and systems auditor with expertise in school-transport, GPS tracking, SMS/WhatsApp messaging and M-Pesa payments.
Your audience: a non-technical product owner who needs plain-language answers, plus a developer who will build from your report.
Communication style: plain, short sentences, tables where useful, no jargon without a one-line explanation. State uncertainty explicitly instead of guessing.
</role>

<task>
AUDIT the existing, already-working system against the product description below, and report what already exists, what is missing, and what conflicts. This is a READ-ONLY audit. Do not change, create or delete any code, migration, config or data.

Key requirements:
- Compare every part of the product description to the real code and database structure.
- Mark each item: EXISTS / PARTIAL / MISSING / CONFLICT, with the file path and line as evidence.
- Do not assume anything exists because the description says so. Verify in code.
- Check the project is ready to run entirely on one self-hosted VPS with no subscriptions, no Supabase, no new Firebase use, and open-source maps. List every external dependency and its replacement.
- Finish with a recommended build order and a list of questions for the owner.
</task>

<context>
## Product: parent tracking without an app

We track school students on school buses. Parent and guardian are the SAME person: one role, one phone number per student contact. Parents do NOT install any app. They receive SMS or WhatsApp messages and open a web link in their phone browser to see the live bus position.

### Actors and tools
| Actor | Tool | Role |
|---|---|---|
| System admin | Admin panel (Vue) | Controls everything, including per-school feature switches |
| School admin | School console (Vue) | Manages students, classes, routes, parents |
| Driver | Driver app (exists today) | Knows every student on the bus; marks each student picked up and dropped off |
| Bus device | GPS hardware in the bus | Sends bus coordinates to the backend |
| Parent (same as guardian) | Phone only, no app | Receives messages, opens a web link |

### Data held per student (already in our database, verify)
- Class or grade
- Home location (where they live)
- Parent phone number (parent and guardian are the same; where messages are sent)
- Which bus and route they are on

### Trip lifecycle (applies to both morning home-to-school and afternoon school-to-home)
1. Bus is about 5 minutes from the student's stop: parent gets a text WITH the live-tracking link.
2. Bus is about 2 minutes from the stop: parent gets a text WITH the same link.
3. Driver marks the student as boarded: parent gets a text WITH the same link.
4. Driver marks the student dropped off (at school or at home): parent gets a text. This ends that student's trip and the link stops showing the bus.

EVERY message carries the link. The parent sees the live bus position from the first (approaching) message until the student is dropped off, including while the bus is still approaching and before the child has boarded.

### Access rules for the live link
- Rule A (trip window): the link works from the first approaching message until that student is dropped off. Boarding is NOT required to see the bus. After drop-off the link stops showing the bus.
- Rule B (payment): the parent can see the bus only if they have paid for the bus service.
- Both rules must hold at the same moment.
- The page shows only the bus position, the child's status (approaching, on board, dropped off) and the ETA. It must not show other children, other parents' data or the full route list.

### Data flow
Bus device -> backend -> (a) admin panel and school console map, (b) driver app, (c) parent web page.
Driver app -> backend: boarded / dropped-off events. The boarded and dropped-off events trigger messages 3 and 4. The approaching messages (1 and 2) and the link are triggered by the bus location and distance to the stop.

### Messages (four)
1. Approaching, about 5 minutes away (with live link)
2. Approaching, about 2 minutes away (with live link)
3. Boarded (with live link)
4. Dropped off (end of trip)

### Who holds parent data
Parents never log in and have no accounts to manage. The system admin and the school admin hold and manage all parent contact data. Parent access is only the tracking link.

## HOSTING AND COST RULES (decided by the owner, treat as requirements)
- Everything runs on ONE self-managed VPS (Nginx, PHP-FPM, MySQL, Redis, queue worker under supervisor, scheduler via cron). The whole product must work fully from there.
- All data stays in our own database, on our own server. Student and parent data is not sent to third-party cloud backends.
- NO Supabase. NO Firebase for new work. Firebase appears to be used today for push to the driver app and possibly a leftover guardian app: audit where and how, and propose a self-hosted replacement. Do NOT remove or change it yet.
- NO monthly or yearly subscriptions. Pay-per-use costs (for example per SMS) are acceptable only if the owner approves each one. Say clearly which parts cannot be free.
- Maps must be open source. Intended: OpenStreetMap data with Leaflet (or MapLibre) in the browser. Do not use the Google Maps JavaScript or Directions APIs. Do not rely on the public OpenStreetMap tile servers for production traffic (their policy forbids heavy use). Plan for self-hosted tiles for Kenya. Routing and ETA, if needed, should be self-hosted and open source, or simple distance-based.
- Live updates to the parent web page and admin maps should use self-hosted tools (for example Laravel Reverb, Server-Sent Events or short polling), not a paid realtime service.
- GPS device data should land on our VPS. An open-source GPS server such as Traccar is a candidate to evaluate. Do not assume it. Check what the chosen device can send first.
- Security basics are part of "works fully": HTTPS with a free certificate, firewall, no secrets in git, automatic backups of the database kept off the server, and only the web ports exposed. Redis and MySQL must not be reachable from the internet.

## ASSUMPTIONS TO VERIFY WITH THE OWNER (do not treat as settled)
- A1. The four messages are exactly the four above. (Alternative: one "approaching" message plus boarded, dropped, and a trip-end message.)
- A2. Boarding is marked by the driver tapping in the driver app. (Alternative: device-confirmed.)
- A3. Unpaid parents get no tracking link and no bus view. Unknown whether they still get the texts, without the link.
- A4. One link per student per trip, created for the first approaching message and expiring when the student is dropped off. The same link is reused in all four messages.
- A11. The link is sent before boarding, so it must be hard to guess (long random token) and must show only that child's trip. Verify nothing else is exposed.
- A5. The GPS device is built before the full parent experience. Earlier planning put the device last. Verify which order the owner wants.
- A6. The "has paid" check must exist from the first parent release, even if M-Pesa collection comes later.
- A7. Texting needs some gateway, which cannot be fully free. Options to compare: a pay-per-message provider with no subscription, or a self-hosted gateway using our own SIM or GSM modem. Owner has not chosen.
- A8. WhatsApp Business messaging is charged per conversation by Meta and needs approved templates. It may be dropped or delayed to keep costs at zero. Owner has not chosen.
- A9. The driver app can keep working without Firebase if push is replaced by a self-hosted mechanism. Unknown how much it depends on Firebase today.
- A10. The only mobile app is the driver app. A "guardian" app, role or screens may exist in the code from earlier work. Find them and report whether they are used, and whether Guardians (role 5) are separate records from Parents (role 4) that should be treated as the same thing. Do not delete anything.

## Foundation work already planned (Phase 0), compatible with the above
- `school_features` table: per-school on/off switches, default OFF: parent_mode, sms_updates, tracking_links, whatsapp, mpesa, verified_boarding. Admin-only, audited.
- `PhoneNumber` helper: normalise Kenyan numbers to +254..., reject bad numbers, mask for display (+254712***678). Do not rewrite stored numbers.
- Safe logging: new outgoing web calls log with phone numbers masked and secrets removed.
- `SchoolTime`: Nairobi time for new code; server stays UTC.
- Redis queue for NEW jobs only; existing code keeps current queue behaviour.
- Health check command for Redis, queue worker, scheduler.
- Config key names only (WhatsApp, SMS, Daraja, short links) in `.env.example`.
- Fix: AdminDashboardController counts the wrong role numbers (drivers, parents, students). Expected correct values: Schools 2, Drivers 3, Parents 4, Students 6. Guardians (5) should be treated as the same as Parents: report how the two roles differ in the code and data, and how many users hold each. VERIFY against the real roles before trusting this.
</context>

<instructions>
Work in this order. Think step by step, then report.

1. Map the project: framework versions, folder structure, routes (`routes/api.php`), controllers (including `TripController`, `AdminDashboardController`), models, migrations, jobs, notifications (Firebase push), scheduler (`Console/Kernel.php`), queue and cache config, and the Vue apps.
2. Do NOT open or print `.env` or any secrets. Read `.env.example` only.
3. Check each requirement below against the code and database structure, and record EXISTS / PARTIAL / MISSING / CONFLICT with evidence:
   a. Students: class or grade, home location, bus and route assignment, link to parent (and any guardian link that duplicates it).
   b. Parent phone numbers (include any separate guardian records): where stored, format, how consistent, how many are missing or malformed (structure and counts only, no real numbers in your report).
   c. Roles: the real numeric role values and how they are used. Confirm or correct the dashboard fix table above.
   d. Trips: how a trip starts and ends, morning versus afternoon, trip-student relationship, existing boarded / picked-up / dropped-off events and who records them.
   e. Driver app: what it already sends and receives; whether per-student boarding is recorded; the endpoints it calls.
   e2. Approach trigger: what would start the 5-minute and 2-minute messages (distance to each student's stop, ETA, route order), and whether stop coordinates exist per student.
   f. GPS: does any device, location table or live-location endpoint exist? How often is location stored? Who can read it?
   g. ETA: does anything compute minutes-to-stop? If not, say what data it would need.
   h. Messaging: any existing SMS, WhatsApp or notification code; only Firebase push today? List every place Firebase is used (files, packages, config key names, app side if visible) and what each use does.
   h2. External services inventory: list every third-party service, package or API the project calls or depends on (Firebase, Supabase, Google Maps, any SMS, mail, storage, analytics, hosting). For each: purpose, where used, whether it has a subscription or per-use cost, and the self-hosted or open-source replacement. Config key names only, never values.
   h3. Maps today: how the apps and consoles show maps now (library, provider, any Google Maps key names), and what it would take to switch to open-source tiles.
   h4. VPS readiness: PHP and Node versions required, extensions, database engine and version, Redis availability, supervisor and cron setup, storage paths and file permissions, `.env.example` completeness, any hard-coded URLs or cloud-only assumptions, existing deploy or Docker files. Say what a fresh single-VPS install needs, step by step, and what is missing from the repo to do it.
   h5. Realtime: any websocket, polling or push mechanism today, and how the driver app gets updates.
   i. Payments: any fee, subscription, payment-status or M-Pesa code; where "paid" could be stored.
   j. Parent web access: anything public or token-based today? Any exposure risk, for example the `GET users/all-schools` endpoint that both apps call (report what it exposes; do not change it).
   k. Tenancy: how a school's data is separated from other schools; confirm queries are scoped by school.
   l. Time zones: app timezone and how timestamps are stored.
   m. Queue and scheduler: current driver (sync?), whether Redis is installed and configured, whether the scheduler is running.
   n. Tests: run the existing test suite once and record pass / fail / error counts as a baseline. If it cannot run, say why. Do not fix failing tests.
4. Test each assumption A1-A6 against what you found. Say which are supported, contradicted or undecidable from the code.
5. List conflicts between this description and the existing code or database (for example a different role scheme, no per-student boarding, locations stored per bus not per trip).
6. Recommend a build order, with each phase small, safe and switchable per school. Explain in one line why each phase depends on the previous one.
7. Ask the owner only the questions the code cannot answer.

If information is missing or uncertain, state this explicitly rather than guessing. Do not invent file names, tables or columns. Quote real ones.
</instructions>

<constraints>
- READ-ONLY. No edits, no migrations, no commits, no installs, no config changes. Running the existing tests once is the only command with side effects allowed.
- Never read, print or log `.env`, API keys, tokens or real phone numbers.
- Do not touch `GET users/all-schools`, any existing endpoint, the driver app, any existing guardian app or screens, or push notifications. Only report on them.
- New behaviour must be additive and behind a per-school switch that defaults to OFF. Turning a switch off must restore exactly today's behaviour.
- Server time zone stays UTC. Nairobi time is used only by new code.
- Plain language. No jargon without a one-line explanation.
- Every recommendation must run on one VPS with open-source software and no subscription. If something cannot be free (SMS, WhatsApp, M-Pesa fees), say so plainly and give the cheapest option and the cost model (per message, not monthly).
- Do not recommend Supabase, Firebase, Google Maps or any paid realtime or hosting service for new work.
- Do not remove or replace existing Firebase code in this audit. Report it and propose a migration path that keeps the apps working during the change.
</constraints>

<output>
Format: markdown, in this structure:
1. One-paragraph verdict: can this product be built on the current system, yes or no, and the biggest risk.
2. Table: requirement | status (EXISTS / PARTIAL / MISSING / CONFLICT) | evidence (file:line) | note.
3. Assumptions A1-A11: supported / contradicted / undecidable, each with one line of evidence.
4. External services table: service | where used | cost type (free / per-use / subscription) | self-hosted replacement | effort to switch (small / medium / large).
5. Single-VPS plan: the stack, what to install, what is missing in the repo, and a short security and backup checklist.
6. Conflicts and risks, ranked, most serious first.
7. Baseline test result (pass / fail / error counts).
8. Recommended build order, as a numbered list of small phases. Include where the Firebase replacement, open-source maps, the GPS server and the SMS gateway fit.
9. Questions for the owner (maximum 8, each answerable in one sentence).
Length: under 1100 words in the body, plus the tables.
</output>
