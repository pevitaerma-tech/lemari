# LEMARI — Architecture Overview

*Draft v2 · 1 Oct 2026 · Written before any code. Every decision here can still change. Changes get explained first and need approval.*

*LEMARI is a joint project of Erma and Pevita. "LEMARI" is a working title, stored in one config file so it can be renamed. Project accounts use the shared address pevitaerma@gmail.com.*

**Launch audience:** mainly **Indonesia and Malaysia**. The founders are in the Netherlands. The app is in **English + Bahasa Indonesia** from day one, with Malay to follow.

This document is the plan for the whole app. It covers:

1. Architecture: the big picture
2. Folder structure
3. Languages and localization
4. Database schema (summary; the full version is in `docs/SCHEMA.md`)
5. Privacy and security model
6. Privacy law: GDPR, Indonesia PDP, Malaysia PDPA
7. Server-side functions
8. AI and image pipeline
9. External providers and rough costs
10. Navigation
11. MVP vs 1.1 vs later
12. Biggest risks
13. Phased roadmap
14. Accounts and keys needed
15. Decisions

---

## 1. Architecture: the big picture

```
┌──────────────────────────────┐
│  LEMARI app (Android + iOS)  │   React Native + Expo + TypeScript
│  - screens, design tokens    │   Holds ONLY: Supabase URL + publishable key
│  - English / Bahasa Indonesia│
│  - strips photo metadata     │
└──────────────┬───────────────┘
               │ logged-in user's token
               ▼
┌──────────────────────────────────────────────────────────┐
│  Supabase (Southeast Asia, Singapore)                    │
│                                                          │
│  Auth ──── email code, Google, Sign in with Apple        │
│  Database ─ PostgreSQL + Row Level Security (RLS)        │
│             = the rules for who may see which row        │
│  Storage ── private buckets; images via short-lived      │
│             signed links; same rules as the database     │
│  Edge Functions ── small server programs for anything    │
│             that needs a secret key (AI, background      │
│             removal, account deletion, reminders)        │
└──────────────┬───────────────────────────────────────────┘
               │ secret keys live here only
               ▼
┌──────────────────────────────────────────────────────────┐
│  External providers (each behind a swappable interface)  │
│  OpenAI (vision, stylist, embeddings) · Background       │
│  removal (Photoroom) · Expo push · later: RevenueCat,    │
│  resale-market data                                      │
└──────────────────────────────────────────────────────────┘
```

**In plain English:**

- **The app** displays things and takes photos. It can't be trusted, because anyone could build a fake app and talk to our database directly. So the app never decides who sees what.
- **The database decides.** Every table has rules (RLS) like "you can read this item only if you own it, or if the owner shared it with you and you're an accepted friend." Even a hacked app gets nothing extra.
- **Server functions** do the work that needs secret keys, such as calling OpenAI or the background-removal service. Wherever possible they act *as the logged-in user*, so the same database rules apply. The all-powerful "service role" key is used only for a few admin jobs (deleting an account, scheduled reminders, payment webhooks).
- **No separate servers ("microservices").** Supabase covers everything for the MVP.
- **Singapore** is the closest Supabase region to Jakarta and Kuala Lumpur, so the app feels fast for the main users.

### Main technical choices

| Area | Choice | Why |
|---|---|---|
| App framework | **Expo SDK 57** (current stable; re-checked at Phase 0 start), TypeScript in strict mode | Your preferred stack; one codebase for Android and iOS |
| Screens and navigation | Expo Router (file-based) | Expo's official router; deep links (invite links, notifications) come almost free |
| Server data in the app | TanStack Query | Caching, loading states and retries without home-made code |
| Forms and validation | Zod (+ react-hook-form) | The same validation rules are checked in the app *and* on the server |
| Translations | **i18next + react-i18next**, with **expo-localization** to detect the phone's language | Expo docs list i18next as stable and well maintained; handles plurals and nested text well. Details in section 3 |
| Dates, numbers, money | The built-in `Intl` formatter (supported on all platforms with Expo's Hermes engine) | "Rp 1.250.000" vs "RM 1,250.00" vs "€1.250,00" handled correctly, no extra library |
| App name | One config file (`src/config/app.ts` + `app.config.ts`) | "LEMARI" is a working title |
| Sign-in | Supabase Auth: email 6-digit code, **Google** (native, via `@react-native-google-signin/google-signin`, the library Supabase's docs recommend), Sign in with Apple | Most users are on Android, so Google matters; Apple requires its own option when Google is offered |
| Native projects | "Continuous Native Generation" (`/ios` and `/android` folders are generated, not committed) | Fewer files to maintain; standard Expo practice |
| Dev build | Expo Go for Phase 0. **Switch to a development build in Phase 1**: Google and Apple sign-in need native code and LEMARI's own app ID, which Expo Go can't provide | I'll tell you at the switch |
| Builds | EAS Build. **Android test builds (APK) from Phase 0**, installable directly on any Android phone; iOS via TestFlight | Android testing starts early, without waiting for Google Play |
| Database tests | pgTAP tests run with the Supabase CLI against a local copy of the database | Proves the privacy rules automatically |
| Vector search | `pgvector` inside Postgres | Duplicate detection and "match my wardrobe" without an extra service |

### Built for Android first

Most users will be on Android, and many on mid-range phones with limited mobile data. So:

- **Every phase is tested on Android first**, on a real phone and the emulator, then on iPhone.
- **Photos are resized and compressed before upload** (about 1600 px on the long side), so a scan doesn't eat mobile data.
- **Wardrobe grids load small thumbnails**; full images load only on the detail screen.
- **Slow networks are tested** by simulating a slow connection; uploads show progress and can be retried.

---

## 2. Folder structure

```
lemari/
├── app/                        Screens (Expo Router: file path = screen)
│   ├── (auth)/                 Sign in, verify code
│   ├── (onboarding)/           First-run flow
│   ├── (tabs)/                 Home, Wardrobe, Add, Style, Circle
│   ├── item/[id].tsx           Item detail
│   ├── scan/                   Scan review ("5 items detected")
│   └── settings/               Profile, language, privacy, delete account, export
├── src/
│   ├── config/app.ts           App name and other brand constants (rename here)
│   ├── i18n/
│   │   ├── index.ts            Translation setup + language detection
│   │   ├── format.ts           Date / number / currency helpers (Intl)
│   │   └── locales/
│   │       ├── en/             English text, split by feature (common.json, wardrobe.json…)
│   │       └── id/             Bahasa Indonesia (same files)   [ms/ = Malay later]
│   ├── components/             Reusable UI (buttons, cards, image tiles)
│   ├── features/               One folder per feature: wardrobe, scan, stylist,
│   │                           recreate, circle, borrow, vault, auth
│   ├── lib/
│   │   ├── supabase.ts         The ONE place the Supabase client is created
│   │   ├── images/             Metadata stripping, resizing, uploads
│   │   └── analytics.ts        Privacy-safe event helper (allow-listed events only)
│   ├── theme/                  Design tokens: colors, type, spacing, radii
│   └── types/database.ts       Auto-generated from the database schema
├── locales/                    Store-level text per language (app name, permission prompts)
├── supabase/
│   ├── config.toml             Local Supabase settings
│   ├── migrations/             Every database change, in order (never edited by hand)
│   ├── functions/
│   │   ├── _shared/
│   │   │   ├── ai/             config.ts (ALL model names live here), provider
│   │   │   │                   interface, openai implementation, prompts
│   │   │   ├── imaging/        background-removal + segmentation interfaces
│   │   │   ├── i18n/           Server-side text (push notifications, emails) en + id
│   │   │   ├── valuation/      resale-data interface (later)
│   │   │   ├── rate-limit.ts   Per-user AI quotas
│   │   │   ├── log.ts          Privacy-safe logger (no photos, prices, prompts)
│   │   │   └── auth.ts         "Act as the calling user" helper
│   │   ├── analyze-item/       One function per folder (see section 7)
│   │   └── …
│   ├── tests/database/         RLS / privacy tests (pgTAP)
│   └── seed.sql                Fake test users and items, local only
├── assets/                     Fonts, icons, splash
├── docs/                       PRODUCT_SPEC, ARCHITECTURE, SCHEMA
├── CLAUDE.md                   My working rules
└── PROJECT_STATUS.md           Where we are
```

One app package, not a "monorepo". It's simpler, and the Supabase folder sits beside it.

---

## 3. Languages and localization

**Goal:** every word on screen exists in English and Bahasa Indonesia from the first screen, and Malay can be added later by translating files, with no code changes.

### How it works
- **No text is typed directly into screens.** Screens use keys like `t('wardrobe.empty.title')`, and the actual sentences live in `src/i18n/locales/en/*.json` and `…/id/*.json`.
- **Language choice:** the app starts in the phone's language if it's Indonesian or English (Malaysian phones in Malay get Indonesian or English until Malay exists; decided in Phase 0). The user can change it in Settings. The choice is saved on the device and in their private settings (`user_settings.locale`), so server messages (push notifications, emails, AI replies) use the same language.
- **Automatic check:** a test fails if any key exists in English but not in Indonesian, or the other way round. A code-check rule warns when plain text is typed into a screen.
- **Translation quality:** I write the English text and a first Indonesian draft. **A native speaker (you, Pevita or a translator) must review the Indonesian text before beta.** Fashion vocabulary and tone matter, and machine-quality Indonesian would feel cheap.
- **Longer text:** Indonesian sentences are often 20–30% longer than English. Layouts are designed to grow, and both languages are checked on screen each phase.

### Dates, numbers and money
- Formatted with `Intl`, based on the chosen language and region: `Rp 1.250.000`, `RM 1,250.00`, `€1.250,00`.
- **Prices are stored as a number + an ISO currency code** (IDR, MYR, EUR, SGD, USD…), never as formatted text. Each user has a default currency (IDR for Indonesia, MYR for Malaysia, EUR for the Netherlands), which they can change per item.
- Dates are stored in UTC and shown in the phone's time zone. Indonesia has three time zones, which matters for borrow dates and reminders.

### Content from the database
- **Fixed lists** (categories, colors, patterns, materials, seasons, style tags) are stored as **language-neutral codes** (`top.blouse`, `color.navy`) and translated in the app. Switching language changes them instantly, and AI answers map onto the same codes.
- **What users type** (titles, notes, brand, messages to friends) stays exactly as written.
- **Seasons:** Indonesia and Malaysia are tropical. Besides spring/summer/autumn/winter, tags include `hot_humid`, `rainy_season`, `air_conditioned_indoors` and `travel_cold_climate`.
- **Regional fashion vocabulary** (to confirm with you): the category list should include items such as hijab/tudung, kebaya, batik, baju kurung, kaftan/gamis, sarong, plus a "modest" style preference the stylist respects.

### AI in the user's language
- Every AI function receives the user's language. The **stylist answers in the user's language** (and follows if they write in the other one). Scan results come back as codes, so the app shows them in the chosen language.
- Prompts are written in English internally (models follow them most reliably) with an explicit "reply in {language}" instruction. Answer quality in Indonesian is part of each AI phase's test.

### App store text
- App name and permission prompts ("LEMARI needs your camera to…") are localized per language through Expo's `locales` setting, because Apple and Google show these outside the app.

---

## 4. Database schema (summary)

Full table-by-table detail with access rules is in **`docs/SCHEMA.md`**. The key ideas:

- **Things friends may see** live in `items`, `item_images` and `profiles`.
- **Things only the owner may see** live in *separate* tables: `item_private_details` (price, purchase info, notes), `vault_records`, `vault_documents`, `valuations`, `item_ai_annotations` (AI confidences), outfit history, scans, stylist chats and addresses. A friend who can see your Chanel bag can never read what you paid for it, because that's a different table with an owner-only rule.
- **Being in the Vault is itself private.** The `items` table has no "is in vault" flag. Vault membership is a row in the owner-only `vault_records` table, so friends can't even learn which items are valuables.
- **The circle**: `circle_invites` (invite links), `connections` (accepted friendships, both sides consented), `friend_permissions` (what each friend may do), `item_share_grants` (for "Selected friends" items).
- **Borrowing**: `borrow_requests` with its own status lifecycle. The database itself blocks overlapping reservations. Status changes happen only through checked server functions, never by editing the row directly.
- **Availability** (available / reserved / lent out / returning) is *calculated* from active borrows, not stored separately, so it can never get out of sync.
- **Consent** (`consents`): which version of the privacy policy and AI-processing notice each user agreed to, and when. Needed for all three privacy laws.

---

## 5. Privacy and security model

### 5.1 Who can see an item

An item has one visibility setting:

| Visibility | Who can see it |
|---|---|
| `private` | Owner only (the default for new items) |
| `circle` | Owner + accepted friends whose permission includes "can view my wardrobe" |
| `selected` | Owner + specific accepted friends the owner picked |
| `ai_only` *(later)* | Owner only; the AI may use it for the owner's own styling |

This is enforced by one database function, `can_view_item(item_id)`. The `items` rules, the `item_images` rules **and the storage bucket rules** all call that same function, so the photo rules can never drift from the item rules.

### 5.2 Rules that apply everywhere

1. **Strangers see nothing.** No public profiles, no search by username. You can only connect through an invite link that you deliberately send (e.g. over WhatsApp). The link is a long random code that expires and works once. Knowing someone's name or email gives no access.
2. **Friendship = both sides agree.** Sending the invite is your consent. Accepting it is theirs.
3. **Permissions are one-directional.** "Sarah can view my wardrobe" doesn't mean "I can view Sarah's".
4. **Removing a friend** cuts all access immediately: permissions and item grants are deleted. Pending borrow requests are cancelled. An item currently lent out stays tracked until it's returned.
5. **Sensitive data lives in owner-only tables** (see section 4).
6. **Status changes go through checked functions.** For example, only the owner can approve a borrow, and only from the "requested" state.
7. **Every new table** gets RLS policies + explicit `GRANT`s (the Supabase project doesn't expose tables automatically) + privacy tests, all in the same migration.

### 5.3 Photos and files (Storage)

| Bucket | Contains | Who can read |
|---|---|---|
| `wardrobe-images` | Item photos: crop, cutout, AI representation (each labeled) | Same as the item (`can_view_item`) |
| `scan-uploads` | Original outfit photos, single-item photos, inspiration screenshots | Owner only |
| `vault-documents` | Receipts, certificates of authenticity | Owner only |
| `avatars` | Profile pictures | Owner + accepted friends |
| `exports` | Data-export files (deleted after 24 h) | Owner only |

- All buckets are **private**. The app gets **signed links valid for ~30 minutes**.
- Files are stored under the owner's ID (`{owner_id}/…`), and uploads are only allowed into your own folder.
- **Your full outfit selfie is never shown to friends.** Friends only see the item images (crop or cutout), not the original photo with your face or your room.

### 5.4 Photo metadata (EXIF / GPS)

- **In the app:** every photo is re-saved (resized, converted to JPEG) before upload. This removes the metadata, including GPS location. Phase 3 includes an automated test proving the GPS data is really gone.
- **On the server (second safety net):** the upload function checks for leftover location data and rejects the photo if any is found.

### 5.5 Addresses (borrowing by post)

- Addresses are never stored on a profile. When a borrow is set to "please send it", the person *receiving* the item enters an address for **that borrow only**.
- Only the two people in that borrow can read it, and only while it's active. A scheduled job **erases it 14 days after the item is returned or the borrow is cancelled**.

### 5.6 Logging, analytics, AI data

- **Logs** record what happened, never the content: "analyze-item ok, 2.1 s". No photos, prices, item details, addresses or AI prompts.
- **Analytics** use only the allow-listed events in the spec (`item_added`, `borrow_requested`, …), with no wardrobe details attached.
- **Data sent to AI:** for scans, the photo (which may include a face). For the stylist, item descriptions, availability and the user's language only. Never prices, Vault data, addresses or friends' details.

### 5.7 Account deletion and data export

- **Delete account** (in Settings): a server function deletes every file the user owns in every bucket, then deletes the user. Database rows are deleted automatically along with the user. Friends lose access instantly. Built in Phase 1.
- **Export my data**: a server function builds a zip (JSON + images) and gives a 24-hour download link. GDPR, Indonesia's PDP law and Malaysia's amended PDPA all give a right to get your data. Built before any beta with other people.

### 5.8 Abuse and cost protection

- Every AI function checks a **per-user quota** before calling a provider (e.g. max scans per day). Limits live in the server config.
- Inputs are validated on the server (types, sizes, image formats, max image size).
- **Block / report a friend**: Apple and Google expect this for apps where users share content with each other. Included with the circle screens.

---

## 6. Privacy law: GDPR, Indonesia PDP, Malaysia PDPA

⚠️ **This is a technical summary, not legal advice.** Everything marked **[advisor]** must be checked with a privacy lawyer who knows these laws before public launch, ideally before the Phase 9 beta.

| Law | Why it applies | Main points for LEMARI |
|---|---|---|
| **GDPR** (EU) | The founders are based in the Netherlands; any EU users | Lawful basis + consent, data subject rights (access, export, deletion), safeguards for transfers outside the EU |
| **Indonesia PDP Law** (UU 27/2022, fully in force since Oct 2024) | Main user base | Explicit consent per purpose, rights to access/delete/withdraw consent, **data breach notice within 3 × 24 hours**, rules on sending data abroad, extra protection for "specific" (sensitive) data, which includes **biometric data** and **children's data** |
| **Malaysia PDPA 2010** (amended 2024) | Second user base | Notice and choice (in Malay *and* English), consent, mandatory **breach notification** (from 2025), data portability, rules on transfers abroad, Data Protection Officer requirement above certain thresholds |

### What this means technically (built into the plan)

1. **Consent screen in onboarding** (Phase 1), in the user's language. It covers the privacy policy, plus a **separate, explicit consent for AI processing of photos** (photos may show faces and go to AI providers in the US). Consents are stored with version and date in the `consents` table, and can be withdrawn in Settings. **[advisor]**: whether AI-photo consent must be separate from accepting the policy.
2. **Data location and transfers.** The database and files are in **Singapore**, and photos go to **OpenAI (US)** and **Photoroom (France)** for processing. That means data leaves Indonesia, Malaysia and the EU. The privacy policy must list each country and provider, and we need each provider's data processing agreement (DPA). **[advisor]**: which transfer mechanism each law accepts (consent, contracts, adequacy).
3. **Faces in photos.** We use photos to recognise *clothes*, not to identify people, and we never build face templates. **[advisor]**: confirm this isn't "biometric data" under Indonesian PDP / Malaysian PDPA. If it is, stricter consent rules apply.
4. **Minimum age.** Indonesian PDP has special rules for children's data. **Recommendation:** the app is 18+ (or 17+, Indonesia's age of adulthood) with an age confirmation at sign-up. **[advisor]**
5. **Retention** (keep only as long as needed): stylist chats 90 days (D6); borrow addresses 14 days after the borrow ends; original scan photos kept only while needed (proposal: delete the full outfit photo once its items are confirmed, unless the user saves it to outfit history); exports 24 hours. **[advisor]**
6. **Deletion and export** from inside the app (section 5.7).
7. **Breach response.** A short written plan: who notices, who decides, how users and authorities are told within the deadlines (Indonesia 3 × 24 h, GDPR 72 h, Malaysia per the 2025 rules). Prepared in Phase 9. **[advisor]**
8. **Privacy policy and terms** in English, Bahasa Indonesia and (for Malaysia) Malay, hosted on the LEMARI domain. **[advisor]**
9. **Data Protection Officer / registration** duties under PDP and PDPA depend on scale. **[advisor]**
10. **Analytics** with no personal content. Analytics only start after consent if the advisor says it's required.

---

## 7. Server-side functions

**Edge Functions** (server programs; they hold the secret keys):

| Function | What it does | Phase |
|---|---|---|
| `delete-account` | Removes all files + the user | 1 |
| `analyze-item` | Single item photo → background removal → AI suggestions (category, color, … with confidence) → embedding | 3 |
| `analyze-outfit` | Outfit photo → AI detects each garment (with position boxes) → proposals with confidence → duplicate candidates | 4 |
| `process-cutouts` | Crops each detected garment → background removal → stores cutouts | 5 |
| `stylist-chat` | Conversational stylist in the user's language; reads only the user's own items; respects availability; returns outfits built from real item IDs | 7 |
| `recreate-look` | Inspiration photo → analysis → match per garment (exact / close / alternative / missing) | 8 |
| `send-reminders` | Scheduled: borrow return reminders (in the recipient's language and time zone), invite expiry, address erasure, export clean-up, chat retention | 9 / 12 |
| `export-data` | Builds the export zip | 9 |
| `suggest-alternatives` | Friend styling: three AI alternatives from the friend's picks | 1.1 |
| `revenuecat-webhook` | Updates the user's plan when they subscribe | 1.1 |
| `estimate-value` | Interprets real market data into a value range | later |

**Database functions** (run inside Postgres, so changes are all-or-nothing):

- `can_view_item`, `is_connected`, `has_friend_permission`: the privacy building blocks
- `create_invite`, `accept_invite`, `remove_connection`, `block_user`
- `confirm_scan_detection`: turns a reviewed detection into an item, or links it to an existing one ("Same item")
- `request_borrow`, `approve_borrow`, `decline_borrow`, `cancel_borrow`, `mark_handed_over`, `mark_returning`, `mark_returned`
- `item_availability`: calculated availability
- `record_consent`, `withdraw_consent`

---

## 8. AI and image pipeline

### 8.1 Swappable interfaces

```
AIProvider            → analyzeGarment, detectOutfitItems, analyzeInspiration,
                        stylistReply(locale), embedText    (first implementation: OpenAI)
BackgroundRemover     → removeBackground(image)            (first: Photoroom)
GarmentSegmenter      → segment(outfitImage) → regions     (first: AI boxes + crop)
ValuationSource       → comparables(item)                  (later)
ImageStore            → put / signedUrl / delete           (first: Supabase Storage)
```

All model names and provider choices live in **one file**: `supabase/functions/_shared/ai/config.ts`. Switching to a newer or cheaper model means editing one line.

### 8.2 How AI results reach your wardrobe

1. You upload a photo.
2. The server returns **proposals** with confidence (e.g. "Material: wool · low confidence"), as language-neutral codes the app shows in your language.
3. You review, edit, delete or merge the proposals.
4. Only what you confirm is saved. Your corrections are saved as the truth, and the original AI guess is kept privately so the prompts can be improved later.
- Brand is never stated as a fact from appearance alone. It shows as "Possibly: …" if at all, and is otherwise left for you to enter.
- AI outputs use **structured outputs** (a fixed JSON shape), so the app never has to "read" free text.

### 8.3 Background removal and cutouts: two different problems

- **Single item photos (easy):** one garment on a background. **Photoroom** handles this well (~$0.02 per image). Free alternatives built into the phones (Google ML Kit, Apple's subject lifting) need custom native code, so they're a later cost-saving option.
- **Outfit photos (hard, the #1 technical risk):** separating *each* garment from a photo of a person wearing them. Plan:
  1. The AI returns an approximate box around each garment.
  2. The app crops each box from the photo.
  3. Background removal runs on each crop.
  - **Fallback:** if the cutout is poor, the item keeps the plain crop, labeled as such.
  - **Phase 5 starts with a 2-day test** of 2–3 approaches on ~30 real outfit photos (including hijab and layered modest outfits, which are harder) before building. You'll see the results side by side and choose.
- **AI-generated "clean product shots"** of a garment: later, if ever. Always labeled "AI representation", never shown as a real photo.

### 8.4 Duplicate detection

- Each item gets an **embedding** (a numeric "fingerprint" of its description and attributes), stored with `pgvector`.
- When a new detection's fingerprint is very close to an existing item, the app asks "We think this item is already in your wardrobe" → **Same item / Different item**. Your answer is saved and used to improve matching.
- Fingerprints are stored from Phase 3 onwards, so the data already exists by Phase 6.
- **Honest limit:** OpenAI's embeddings work on text, not images. The MVP fingerprint is built from a detailed AI description of the garment. If that isn't accurate enough, a true *image* fingerprint provider can be added behind the same interface.

---

## 9. External providers and rough costs

Prices checked 1 Oct 2026. These are **estimates** that change often; we'll re-check at signup. Supabase prices are the same in every region.

### Fixed costs

| Provider | What for | Cost | When |
|---|---|---|---|
| Apple Developer Program | TestFlight, App Store, Sign in with Apple | $99 / year | Phase 1 |
| Google Cloud project | Google sign-in setup | Free | Phase 1 |
| Google Play Console | Play Store testing and release | $25 one-time | By Phase 9 at the latest. See the 14-day testing rule in section 12 |
| Supabase | Database, auth, storage, functions | Free now. **Pro $25/month** before other people use it (the free plan pauses after 1 week of inactivity and has no backups) | Pro before first beta |
| Expo / EAS | Cloud builds | Free tier is enough to start; paid from ~$19/month if builds queue too long | Phase 0 |
| Domain name | Sign-in email sender, privacy policy page | ~€15–40 / year | Before beta |
| Email sending (e.g. Resend) | Sign-in code emails. Supabase's built-in email is heavily rate-limited and only meant for testing | Free tier (~3,000 emails/month) | Before beta |
| Translation review | Native-speaker check of Indonesian (later Malay) text | Free if done by you / Pevita; otherwise freelance | Before beta |
| Privacy lawyer | GDPR + PDP + PDPA review | One-off fee, varies widely | Before public launch |
| RevenueCat | Subscriptions | Free up to $2,500 monthly revenue, then 1% | 1.1 |

### Usage costs (per action, rough)

| Action | Rough cost |
|---|---|
| Add single item (AI + background removal) | ~$0.02–0.04 |
| Outfit scan with 5 items (AI + 5 cutouts) | ~$0.10–0.15. **Cutouts are most of the cost** |
| Stylist message | ~$0.005–0.02 |
| Recreate This Look | ~$0.02–0.05 |
| Embeddings (fingerprints) | Negligible (fractions of a cent) |

**Example:** 100 active beta testers, each doing 10 outfit scans + 30 stylist messages a month ≈ **$150/month in AI + $25 Supabase**.

**Pricing note for later:** subscription prices in Indonesia and Malaysia are much lower than in Europe, but AI costs per scan are the same everywhere. That makes per-user quotas, cheaper models where good enough, and on-device background removal more important. To revisit before setting prices (decision D3).

We'll set a **monthly spending limit** in the OpenAI dashboard from day one.

---

## 10. Navigation (D1 approved)

```
 HOME  ·  WARDROBE  ·  ( + )  ·  STYLE  ·  CIRCLE
```

- **Home**: today's prompt, quick actions, recently added, borrow requests, items lent out, friend styling. Profile and settings (including language) sit behind your avatar in the top corner.
- **Wardrobe**: all items, filters, outfit history. **Vault** (1.1) is a locked section here.
- **+ (Add)**: a sheet with *Scan an outfit · Add an item · Recreate a look*.
- **Style**: AI stylist chat and outfit boards. Recreate This Look results live here too.
- **Circle**: friends, invites, permissions, styling from and for friends, **Borrowing** (requests, lent out, borrowed).

Final screen designs get proposed before building each feature.

---

## 11. MVP vs 1.1 vs later

### MVP (first release to real users)
- **English + Bahasa Indonesia** throughout, including server messages and AI replies.
- Sign in with **email code, Google and Apple**. Onboarding with clear privacy and AI consent.
- Wardrobe: add, edit, delete items manually or by photo. Per-item privacy setting. Prices in local currency.
- Single item add with background removal and AI suggestions.
- Outfit scan → "5 items detected" review → confirm.
- Garment cutouts (or labeled crops as fallback).
- Duplicate detection (Same item / Different item).
- AI stylist using real wardrobe images, respecting availability, answering in the user's language.
- Recreate This Look (Inspiration vs My version).
- Private circle: invite links (shareable via WhatsApp), permissions, Private / Circle / Selected visibility, block / report.
- Friend styling (a friend builds a board and sends it).
- Borrowing: request → approve → hand over / ship → return, with reminders and the address rules.
- Basic outfit history (from scans).
- Account deletion, data export, consent management, push notifications, privacy-safe analytics.
- **Usage quotas** instead of a paywall (D3).

### 1.1
- **Malay** (Bahasa Melayu) translation
- Vault (private records + receipts and authenticity documents)
- Subscriptions and paywall (RevenueCat), with regional pricing
- AI alternatives in friend styling
- Share-to-LEMARI from Instagram/TikTok/Pinterest (share extension; native work)
- Sharing specific collections or categories with specific friends
- Weather-aware styling and packing lists
- `ai_only` visibility

### Later
- Resale valuation from real market data (markets relevant to Indonesia/Malaysia need their own data sources); "What is my wardrobe worth?"
- Wear insights (most/least worn, cost per wear)
- True image fingerprints for duplicates; on-device background removal
- AI representation images (always labeled)

---

## 12. Biggest risks

| # | Risk | Why it matters | How we handle it |
|---|---|---|---|
| 1 | **Multi-garment cutouts look bad** (especially layered and modest outfits) | It's the "magic moment" of the app | Phase 5 test before building; labeled crop fallback; swappable provider |
| 2 | **A privacy bug** (friend sees a price, stranger sees an item) | It's the core promise of the product | Separate owner-only tables, one shared `can_view_item` function, automated privacy tests that must pass, no app-only hiding |
| 3 | **Three privacy laws** (GDPR, PDP, PDPA), with photos (faces) sent to US/EU providers | Legal and trust risk; fines; app takedown | Consent screen and records, data minimisation, retention limits, export/deletion; **privacy lawyer before launch** (section 6) |
| 4 | **AI cost per user vs local prices** | A scan costs the same everywhere, but people in Indonesia/Malaysia will pay less | Quotas from day one, spending cap, cheaper models where good enough, on-device removal later |
| 5 | **Google Play's testing rule**: new *personal* developer accounts must run a closed test with **at least 12 testers for 14 days in a row** before they can publish | Can delay launch by weeks | Start the closed test early (Phase 9), or register as an **organisation** (exempt, but needs a company and a D-U-N-S number). Decision D8 |
| 6 | **Mid-range Android phones and mobile data** | Slow, heavy app → people leave | Android-first testing, compressed uploads, thumbnails, slow-network tests |
| 7 | **Translation quality / AI quality in Indonesian** | Feels cheap or wrong | Native-speaker review; Indonesian checked in every AI phase |
| 8 | **App Store / Play review**: account deletion, Sign in with Apple, privacy labels / data-safety form, block/report | Rejection delays launch | All planned into the MVP |
| 9 | **Duplicate detection** merges two different items, or misses duplicates | Messy wardrobe | Always ask the user; never auto-merge |
| 10 | **AI is wrong** (material, brand) | Trust | Confidence shown, user confirms, no brand/authenticity claims |
| 11 | **Scope** | Burnout, half-finished features | Strict phases, MVP list above, Vault/Malay/valuation in 1.1 or later |
| 12 | **Supabase free plan** pauses and has no backups | Data loss / downtime | Upgrade to Pro before anyone else uses it |

---

## 13. Phased roadmap

Every phase is tested on **Android first**, then iPhone, in both English and Bahasa Indonesia.

| Phase | Name | What's in it | You'll see on your phone |
|---|---|---|---|
| 0 | Setup | Git + GitHub, Expo project, design tokens, **translation system (en + id)**, app-name config, Supabase link, local database, test harness, code checks, **first Android test build** | A styled welcome screen that switches between English and Indonesian, on Android (installed app) and iPhone (Expo Go) |
| 1 | Auth + onboarding | Dev-build switch, email code + **Google** + Apple sign-in, onboarding with privacy/AI consent and age confirmation, profile with language and currency, **delete account** | Sign in with Google on Android, onboarding, delete account |
| 2 | Wardrobe + full privacy model | **All** privacy-related tables (items, private details, circle, borrow, vault skeleton, consents), RLS, storage rules, privacy tests; manual add/edit/delete; wardrobe grid; local currency | Add/edit items by hand, with prices in Rp / RM / € |
| 3 | Single item photo + AI | Camera/library, EXIF stripping (tested), compression, background removal, AI suggestions with confidence, quotas, fingerprints | Photograph a garment → cutout + suggestions → confirm |
| 4 | Outfit scan | Multi-item detection, "5 items detected" review, edit/delete/merge, crops | Scan an outfit → review → items appear |
| 5 | Cutout pipeline | Comparison test first, then the chosen approach | Clean cutouts from outfit photos |
| 6 | Duplicate detection | Same item / Different item flow | Re-scan an outfit → "already in your wardrobe" |
| 7 | AI stylist | Chat in en/id, outfit boards from real images, availability-aware | Ask "Mau pakai apa untuk makan malam?" → outfit board |
| 8 | Recreate This Look | Inspiration vs My version, try another match, modifiers ("for work", "modest") | Upload an inspiration photo → your version |
| 9 | Beta readiness | Data export, push notifications, analytics, privacy policy + terms (3 languages), breach plan, custom email sender, Supabase Pro, Indonesian text review, **Google Play closed test started (12 testers × 14 days)**, TestFlight | Install via Play Store testing / TestFlight; export your data |
| 10 | Private circle | Invites (WhatsApp share), connections, permissions, visibility, block/report | Invite a friend; see each other's shared items |
| 11 | Friend styling | Friend builds and sends a board | "Pevita styled a look for you" |
| 12 | Borrowing | Full lifecycle, overlap protection, address handling, reminders in the right time zone | Request, approve, lend, return |
| — | **MVP release** | Play Store + App Store (after legal review) | |
| 13 | Malay + Vault (1.1) | Malay translation; Vault records, documents bucket, biometric lock | App in Malay; private vault |
| 14 | Subscriptions (1.1) | RevenueCat, paywall, regional prices, plan-based quotas | Upgrade screen |
| 15 | Valuation (later) | Provider interface, then a real market-data source | Value ranges with date and confidence |

Each phase ends with: tests and type checks passing, privacy tests passing, both languages complete, a commit pushed to GitHub, an update to `PROJECT_STATUS.md`, and a stop for approval.

---

## 14. Accounts and keys needed

All project accounts use **pevitaerma@gmail.com**. I'll give exact click-by-click steps when each one is needed.

| Account | Needed by | Cost | What you'll give me |
|---|---|---|---|
| GitHub repo `pevitaerma/lemari` *(exists)* | Now | Free | Add the deploy key (steps in chat). Nothing else |
| Supabase *(project exists, Singapore)* | Phase 0 | Free → $25/mo | Project URL + **publishable (anon) key**. These are safe to share and are meant to be in the app. **Never** paste the service-role key into chat. |
| Docker Desktop (installed on your Mac) | Phase 0 | Free for small businesses | Nothing. It runs a local copy of the database for privacy tests |
| Expo account | Phase 0 | Free | The Expo username |
| An Android phone for testing | Phase 0 | — | Allow "install unknown apps" (steps given then) |
| Apple Developer Program | Phase 1 | $99/yr | Team ID; you'll create a Sign in with Apple key and paste it into Supabase yourself |
| Google Cloud project | Phase 1 | Free | OAuth client IDs (not secret); the client secret goes into Supabase yourself |
| OpenAI Platform (billing + spending limit) | Phase 3 | Pay per use | You paste the API key into **Supabase Edge Function secrets** yourself, never into chat or the app |
| Photoroom API | Phase 3 | ~$0.02/image | Same: key goes into Supabase secrets |
| Google Play Console | Phase 9 at the latest | $25 once | Nothing secret. Personal vs organisation: decision D8 |
| Domain name | Phase 9 | ~€15–40/yr | Nothing; you'll add DNS records I give you |
| Resend (email sending) | Phase 9 | Free tier | Key goes into Supabase settings |
| Analytics (e.g. PostHog) | Phase 9 | Free tier | Public project key |
| RevenueCat | Phase 14 | Free to start | Public SDK keys (safe in app) + webhook secret (server only) |

---

## 15. Decisions

### Approved (1 Oct 2026)
- **D1** 5 tabs: Home · Wardrobe · + · Style · Circle; profile behind avatar.
- **D2** Email code + **Google** + Apple sign-in, all in the MVP. Android testing starts in Phase 0.
- **D3** Usage quotas in the MVP, paywall in 1.1 (revisit after beta costs).
- **D4** Vault screens in 1.1; Vault tables designed in Phase 2.
- **D5** Photoroom for background removal to start.
- **D6** Stylist chats auto-deleted after 90 days.
- **D7** Private GitHub repo `pevitaerma/lemari` as online backup.

### Open
- **D8 Who "owns" the developer accounts?** Apple and Google accounts can be personal (one person's legal name shows as the seller) or an organisation (needs a registered company + D-U-N-S number, but Google's 12-tester rule doesn't apply). This depends on how you and Pevita set up the business. *(Recommendation: decide before Phase 1, when the Apple account is created. Ask your advisor whether a Dutch company, an Indonesian PT, or starting personal-then-transferring suits you best.)*
- **D9 Minimum age**: 18+ or 17+? *(Recommendation: 18+ to stay clear of children's-data rules; confirm with advisor.)*
- **D10 Regional categories**: confirm the fashion vocabulary (hijab/tudung, kebaya, batik, baju kurung, gamis, sarong, modest preference…). *(Recommendation: you and Pevita review the category list in Phase 2.)*
- **D11 Malaysian phones in Malay** before Malay exists: show English or Indonesian? *(Recommendation: English.)*
