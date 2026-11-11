# Prompt: Kenya localization audit and plan (standalone)

Paste everything below the line into Claude Code, in the project root. This prompt stands alone. It does not depend on any earlier prompt.

---

<role>
You are a senior Laravel and Vue engineer and product localization specialist with expertise in the Kenyan market: M-Pesa (Safaricom Daraja), school fee collection, CBC school levels, Kiswahili and English user interfaces, SMS delivery, and the Kenya Data Protection Act 2019.
Your audience: a non-technical product owner who needs plain answers, plus a developer who will build from your report.
Communication style: plain, short sentences, tables where useful, no jargon without a one-line explanation. State uncertainty explicitly instead of guessing.
</role>

<task>
AUDIT the existing, working system and produce a plan to localize it for Kenyan schools, so that Kenyan school admins and parents find it natural and easy to read. This is a READ-ONLY audit followed by a written plan. Do not change, create or delete any code, migration, config or data.

Key requirements:
- Verify in code what exists today. Do not trust this description over the code.
- Mark each localization item EXISTS / PARTIAL / MISSING / CONFLICT, with file path and line as evidence.
- Design the Kenyan payment flow (M-Pesa, per term, school collects) on top of what exists, with the least change to working behaviour.
- Draft the Kenyan wording and the SMS templates in English and Kiswahili.
- Finish with a build order and questions for the owner.
</task>

<context>
## The product in short
A school-bus tracking system. The parent (same person as the guardian) has NO app. They receive SMS or WhatsApp messages, each with a link, and open a light web page in their phone browser to see the live bus. The driver has an app (the only app). The system admin and the school admin use web panels (Vue). Everything runs on one self-managed VPS: no subscriptions, no Supabase, no Firebase for new work, open-source maps (OpenStreetMap with Leaflet or MapLibre), MySQL, Redis.

## Owner decisions for the Kenyan market (treat as requirements)
- D1. **The school collects the money and uses the platform.** Parent fees go straight to the school's own M-Pesa. The platform never holds school or parent money.
- D2. **M-Pesa through Safaricom Daraja, directly, with two ways to receive:** Paybill ("Pay Bill") and Till ("Buy Goods"). Each school registers its own shortcode and credentials.
- D3. **Prompting:** a parent can be prompted to pay with an M-Pesa STK Push (the "enter your PIN" popup on their phone). The prompt can be started by the parent (from a pay link) or by the school admin from the console on the parent's behalf. [Assumption: confirm. See question Q2.]
- D4. **Billing is per term**, not per day.
- D5. **Language is chosen per parent: Kiswahili or English.** This applies to SMS, WhatsApp text and the web page. Admin panels may stay in English at first.
- D6. Currency is **KES**, in whole shillings. Time zone is **Africa/Nairobi (UTC+3, no daylight saving)**; server stays UTC and only new code converts.
- D7. The parent web page and SMS must work on cheap Android phones and low data.

## What a previous audit of THIS codebase reported (verify each; it may be out of date)
- Plans: admin creates School Plans (type 0) and Parent Plans (type 1). Fields: name, coins, price (whole number), availability (1 = buy once, 2 = buy again).
- Purchase: goes through a provider (Braintree, Stripe, Razorpay, Flutterwave, Paystack, PayTabs). On success the server creates a charge row (payer, plan, price, coins, date, copies of plan name and price) and adds coins to `users.balance`.
- Coins are prepaid trip credit: one coin = one student's trip for one day. The daily trip job takes 1 coin from the school balance. If the school has none, it takes 1 from the parent's balance. If both are empty, the student's status becomes `out_of_coins` (5), and returns to active when coins arrive.
- Dashboard money cards: Total Earning, Schools Payments, Parents Payments (all sums of charge prices), Best Selling Plans (top 5 by money). All show the platform's collections. Price is shown as `N USD` from one currency setting.
- Payments page: one row per charge (ID, Plan, Paid price, Date, Payer). Shows only purchases. No coin usage, refunds or unmatched money.
- Unused "redeem" payout code exists with no route or button.
- Gaps already noted: no provider receipt numbers stored, no per-student or per-term fees, no refunds, no part payments, no unmatched payments, parent payer identity linked to their account only, all money lands in platform totals.
- Guardians (role 5) and Parents (role 4) should be treated as the same person. The dashboard Guardians figure is parents plus guardians.

## Access rule that payment must support
The parent can see the bus link only if the student's term fee is paid. The link is sent in every message (5 minutes away, 2 minutes away, boarded, dropped off). It shows only that child's trip. How "partly paid" is treated is an open question (Q4).

## Kenyan details to design for
- **Fees and Paybill habits:** parents commonly pay a school Paybill and type the **admission number as the account number**. Match payments to students automatically using it.
- **M-Pesa receipt code** (a code like `SHK3X…`) must be stored on every payment. It is the reconciliation key and must be unique to prevent double-recording.
- **Part payments, instalments, arrears carried between terms, siblings, and a relative paying from a different phone** are normal. Payer phone may differ from parent phone.
- **Terms:** three per year, dates set by the Ministry of Education and varying by year and school. Do NOT hard-code dates. Each school enters its own term start, end, half-term and closure days. Trips must not be created, and no fee clock should run, on weekends, public holidays or closure days.
- **School levels:** support CBC (PP1, PP2, Grade 1 to 9, then senior school Grade 10 to 12) and optionally legacy Standard 1 to 8 and Form 1 to 4. **Admission number** is a first-class student field.
- **Addresses:** most homes have no street address. Stops and homes are a map pin plus a landmark text (for example "opposite Naivas, Kahawa Sukari"). Say "pick-up point" in the interface.
- **Phone numbers:** accept 07xx, 01xx, 7xx and +254 forms; store as +254XXXXXXXXX; mask for display (+254712***678). Do not rewrite stored numbers.
- **Formats:** KES 1,500 (comma thousands, no decimals), dates DD/MM/YYYY, time 12-hour like 7:05 AM, East Africa Time.
- **SMS:** keep under 160 characters, plain letters only (no emojis, no curly quotes, since they cut the limit to 70 and cost more). Use the child's first name only. Messages are service messages and need a registered sender ID. Verify with the chosen gateway.
- **SMS gateway:** no subscription. Compare a pay-per-message Kenyan provider (for example Africa's Talking) with a self-hosted SIM or modem gateway. Verify current rates and limits. Do not assume them.
- **Kenya Data Protection Act 2019:** the school is a data controller handling children's data, parent phone numbers and location. Plan for parent consent, a short privacy notice, data minimization, retention limits, and registration with the Office of the Data Protection Commissioner. Say this must be confirmed with a lawyer.
- **M-Pesa callback safety:** callbacks land on our HTTPS domain. Use a secret path token, accept only expected amounts and references for a known pending request, ignore duplicates by receipt number, and confirm uncertain payments with the Daraja transaction or STK query. Verify the current Daraja rules and Safaricom IP guidance in the official docs. Do not rely on memory.
- **Secrets:** each school's Daraja key, secret and passkey are stored encrypted on the VPS. Never in git, never in logs.

## Draft wording (to be refined; Kiswahili must be checked by a native speaker)
| English | Kiswahili |
|---|---|
| Trip credit / Term fee | Ada ya usafiri ya muhula |
| Paid / Partly paid / Not paid | Imelipwa / Imelipwa kiasi / Haijalipwa |
| Pick-up point | Kituo cha kupanda |
| Drop-off | Kushushwa |
| Bus is on its way | Basi liko njiani |
| Term 1 / 2 / 3 | Muhula wa 1 / 2 / 3 |
| Pay now | Lipa sasa |

SMS drafts (placeholders in braces; each with the link; under 160 characters):
- EN 5 min: `Bus {plate} is 5 min from {stop}. Track {child}: {link}`
- SW 5 min: `Basi {plate} liko dakika 5 kutoka {stop}. Fuatilia {child}: {link}`
- EN 2 min: `Bus {plate} is 2 min from {stop}. Get {child} ready: {link}`
- SW 2 min: `Basi {plate} liko dakika 2 kutoka {stop}. Mwandae {child}: {link}`
- EN boarded: `{child} has boarded at {time}. Track the bus: {link}`
- SW boarded: `{child} amepanda basi saa {time}. Fuatilia: {link}`
- EN dropped: `{child} was dropped at {place} at {time}. Trip ended.`
- SW dropped: `{child} ameshushwa {place} saa {time}. Safari imeisha.`
- EN pay prompt: `{school}: Term {n} transport fee KES {amount} due. Pay: {link}`
- SW pay prompt: `{school}: Ada ya usafiri muhula wa {n} KES {amount}. Lipa: {link}`
</context>

<instructions>
Work in this order. Think step by step, then report.

1. Map the project: framework and package versions, routes (`routes/api.php`), controllers, models, migrations, jobs, scheduler (`Console/Kernel.php`), queue and cache config, language files, and the Vue apps (admin panel, school console).
2. Do NOT open or print `.env` or any secrets. Read `.env.example` only.
3. Verify and report on each item, with EXISTS / PARTIAL / MISSING / CONFLICT and file:line evidence:
   a. **Currency and formatting:** where the currency symbol and code are set; every place `USD` or another currency is hard-coded; number, date and time formatting; app timezone and how timestamps are stored.
   b. **Language support:** is there any translation system (Laravel lang files, vue-i18n)? Is there any Kiswahili? How many strings are hard-coded? Is there a per-user or per-parent language field? Where would one go?
   c. **Money model:** confirm the report above. Plans, charges, `users.balance`, the daily trip job and `out_of_coins`. Which code would need to change to move from coins to per-term fees, and what is the smallest safe path? Evaluate three options and recommend one: (i) replace coins with term fee records, (ii) keep coins internally and map a term to a number of Trip Days, (iii) run both with a per-school switch.
   d. **Payment providers:** where each foreign provider is wired in (code, config key names, UI), and how to hide them per school without deleting. Is there anything M-Pesa today?
   e. **Per-school payment settings:** is there a place to store a school's own Paybill or Till number and credentials? What is the school model and settings structure?
   f. **Students:** admission number, grade or class, how grades are stored, whether Kenyan CBC levels fit, home pin and landmark, stop and pick-up point fields.
   g. **Parents and phones:** where phone numbers are stored, formats in use, how many are missing or malformed (counts only, no real numbers), whether a separate payer phone is possible.
   h. **Terms and calendar:** any term, holiday or school-day concept? How does the daily trip job decide which days to run? Would it run on holidays today?
   i. **Messaging:** any existing SMS or WhatsApp code; message templates; where the text is built; length handling; encoding.
   j. **Parent-facing web page:** does any exist? How would a public pay page and tracking page be protected (long random token, expiry, rate limit)? What must they never show?
   k. **Dashboard and Payments page wording:** list every label to rename for Kenya (for example Total Earning, Schools Payments, Parents Payments, Best Selling Plans, Coins). Propose plain Kenyan wording and cards that matter to a school (Collected this term, Outstanding, Paid / Partly / Not paid, Unmatched payments).
   l. **Data protection:** what personal data is stored, any consent field, any retention or deletion code, how location history is kept.
   m. **Run the existing test suite once** and record pass / fail / error counts as a baseline. Do not fix failures.
4. Design (as text only, no code changes):
   - **M-Pesa flow:** STK Push (Paybill and Till variants) and manual Paybill payments with the admission number as the account reference. Show who starts what, what the callback does, what is stored, how duplicates and mismatches are handled, and how a school sees "unmatched payments".
   - **Term billing model:** term fee per student (set by the school, possibly by route or zone), school-entered term dates, instalments, balance, arrears, siblings, mid-term joiners, status Paid / Partly paid / Not paid, and the rule that decides when the tracking link works.
   - **Prompting:** the pay link in SMS, the public pay page, and the school admin "Request payment" button. Include reminder messages and their per-SMS cost.
   - **Language:** where the parent picks Kiswahili or English (school admin sets it with the parent record, and the parent can switch on the web page), and how templates are chosen.
   - **Per-school Kenya settings:** currency, default language, time zone, shortcode type and number, and a switch to turn Kenya billing on or off.
5. Refine the draft wording and SMS templates above. Check every SMS fits in 160 plain characters with realistic values (a long school name, a long first name, a long stop name). Flag any template that does not, and shorten it. Flag every Kiswahili line that needs native-speaker review.
6. Recommend a build order, each phase small, safe and switchable per school, with a one-line reason for the order.
7. Ask the owner only the questions the code cannot answer.

If information is missing or uncertain, say so rather than guessing. Do not invent file names, tables, columns, fees or API behaviour. Quote real ones, and for outside facts (Daraja, SMS rates, legal duties) say they need checking against current official sources.
</instructions>

<constraints>
- READ-ONLY. No edits, no migrations, no commits, no installs, no config changes. Running the existing tests once is the only command with side effects allowed.
- Never read, print or log `.env`, API keys, tokens, shortcodes' secrets or real phone numbers.
- Do not delete or alter existing payment providers, plans, charges, balances or the daily trip job. Report and design only. New behaviour must be additive and behind a per-school switch that defaults to OFF. Turning it off must restore exactly today's behaviour.
- Everything must run on one VPS with open-source software and no subscription. M-Pesa transaction fees and SMS per-message fees are acceptable; say plainly what they are and who pays.
- The platform must never hold school or parent money. Money goes to the school's own Paybill or Till.
- Do not recommend Supabase, Firebase, Google Maps or any paid realtime or hosting service.
- Plain language, with a one-line explanation for any technical term.
</constraints>

<output>
Format: markdown, in this structure:
1. One-paragraph verdict: can the system be localized for Kenya on top of what exists, and the biggest risk.
2. Table: item | status (EXISTS / PARTIAL / MISSING / CONFLICT) | evidence (file:line) | change needed | effort (small / medium / large).
3. Money model recommendation: the chosen option (i, ii or iii) and why, in under 150 words.
4. M-Pesa flow, as a numbered list (parent-started and school-started), including duplicates and unmatched payments.
5. Term billing design: fields, statuses, tracking-link rule, edge cases (siblings, mid-term, arrears, part payments).
6. Wording table: current label | Kenyan English | Kiswahili.
7. SMS template table: event | language | text | character count. All must be 160 or fewer.
8. Kenya settings per school (list).
9. Data protection checklist (short, flagged "confirm with a lawyer").
10. Baseline test result (pass / fail / error counts).
11. Build order, as a numbered list of small phases.
12. Questions for the owner (maximum 8, each answerable in one sentence).
Length: under 1300 words in the body, plus the tables.
</output>
