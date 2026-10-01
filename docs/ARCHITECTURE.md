# LEMARI — Architecture Overview

*Draft v1 · 1 Oct 2026 · Written before any code. Every decision here can still change. Changes get explained first and need Erma's approval.*

This document is the plan for the whole app. It covers:

1. Architecture: the big picture
2. Folder structure
3. Database schema (summary; the full version is in `docs/SCHEMA.md`)
4. Privacy and security model
5. Server-side functions
6. AI and image pipeline
7. External providers and rough costs
8. Navigation proposal
9. MVP vs 1.1 vs later
10. Biggest risks
11. Phased roadmap
12. Accounts and keys Erma needs
13. Open decisions for Erma

---

## 1. Architecture: the big picture

```
┌──────────────────────────────┐
│  LEMARI app (iOS + Android)  │   React Native + Expo + TypeScript
│  - screens, design tokens    │   Holds ONLY: Supabase URL + publishable key
│  - strips photo metadata     │
└──────────────┬───────────────┘
               │ logged-in user's token
               ▼
┌──────────────────────────────────────────────────────────┐
│  Supabase (EU, Frankfurt)                                │
│                                                          │
│  Auth ──── who you are (email code, Sign in with Apple)  │
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
│  removal (e.g. Photoroom) · Expo push · later:           │
│  RevenueCat, resale-market data                          │
└──────────────────────────────────────────────────────────┘
```

**In plain English:**

- **The app** displays things and takes photos. It can't be trusted, because anyone could build a fake app and talk to our database directly. So the app never decides who sees what.
- **The database decides.** Every table has rules (RLS) like "you can read this item only if you own it, or if the owner shared it with you and you're an accepted friend." Even a hacked app gets nothing extra.
- **Server functions** do the work that needs secret keys, such as calling OpenAI or the background-removal service. Wherever possible they act *as the logged-in user*, so the same database rules apply. The all-powerful "service role" key is used only for a few admin jobs (deleting an account, scheduled reminders, payment webhooks).
- **No separate servers ("microservices").** Supabase covers everything for the MVP.

### Main technical choices

| Area | Choice | Why |
|---|---|---|
| App framework | Expo (latest stable SDK, checked in Phase 0), TypeScript in strict mode | Your preferred stack; one codebase for iOS and Android |
| Screens and navigation | Expo Router (file-based) | Expo's official router; deep links (invite links, notifications) come almost free |
| Server data in the app | TanStack Query | Caching, loading states and retries without home-made code |
| Forms and validation | Zod (+ react-hook-form) | The same validation rules are checked in the app *and* on the server |
| Native projects | "Continuous Native Generation" (`/ios` and `/android` folders are generated, not committed) | Fewer files to maintain; standard Expo practice |
| Dev build | Expo Go for Phase 0. **Switch to a development build in Phase 1**: Sign in with Apple must run under LEMARI's own app ID, which Expo Go can't provide | I'll tell you at the switch |
| Builds and TestFlight | EAS Build + EAS Submit | Builds in the cloud, so no Xcode headaches |
| Database tests | pgTAP tests run with the Supabase CLI against a local copy of the database | Proves the privacy rules automatically |
| Vector search | `pgvector` inside Postgres | Duplicate detection and "match my wardrobe" without an extra service |

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
│   └── settings/               Profile, privacy, delete account, export
├── src/
│   ├── components/             Reusable UI (buttons, cards, image tiles)
│   ├── features/               One folder per feature: wardrobe, scan, stylist,
│   │                           recreate, circle, borrow, vault, auth
│   ├── lib/
│   │   ├── supabase.ts         The ONE place the Supabase client is created
│   │   ├── images/             Metadata stripping, resizing, uploads
│   │   └── analytics.ts        Privacy-safe event helper (allow-listed events only)
│   ├── theme/                  Design tokens: colors, type, spacing, radii
│   └── types/database.ts       Auto-generated from the database schema
├── supabase/
│   ├── config.toml             Local Supabase settings
│   ├── migrations/             Every database change, in order (never edited by hand)
│   ├── functions/
│   │   ├── _shared/
│   │   │   ├── ai/             config.ts (ALL model names live here), provider
│   │   │   │                   interface, openai implementation, prompts
│   │   │   ├── imaging/        background-removal + segmentation interfaces
│   │   │   ├── valuation/      resale-data interface (later)
│   │   │   ├── rate-limit.ts   Per-user AI quotas
│   │   │   ├── log.ts          Privacy-safe logger (no photos, prices, prompts)
│   │   │   └── auth.ts         "Act as the calling user" helper
│   │   ├── analyze-item/       One function per folder (see section 5)
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

## 3. Database schema (summary)

Full table-by-table detail with access rules is in **`docs/SCHEMA.md`**. The key ideas:

- **Things friends may see** live in `items`, `item_images` and `profiles`.
- **Things only the owner may see** live in *separate* tables: `item_private_details` (price, purchase info, notes), `vault_records`, `vault_documents`, `valuations`, `item_ai_annotations` (AI confidences), outfit history, scans, stylist chats and addresses. A friend who can see your Chanel bag can never read what you paid for it, because that's a different table with an owner-only rule.
- **Being in the Vault is itself private.** The `items` table has no "is in vault" flag. Vault membership is a row in the owner-only `vault_records` table, so friends can't even learn which items are valuables.
- **The circle**: `circle_invites` (invite links), `connections` (accepted friendships, both sides consented), `friend_permissions` (what each friend may do), `item_share_grants` (for "Selected friends" items).
- **Borrowing**: `borrow_requests` with its own status lifecycle. The database itself blocks overlapping reservations. Status changes happen only through checked server functions, never by editing the row directly.
- **Availability** (available / reserved / lent out / returning) is *calculated* from active borrows, not stored separately, so it can never get out of sync.

---

## 4. Privacy and security model

### 4.1 Who can see an item

An item has one visibility setting:

| Visibility | Who can see it |
|---|---|
| `private` | Owner only (the default for new items) |
| `circle` | Owner + accepted friends whose permission includes "can view my wardrobe" |
| `selected` | Owner + specific accepted friends the owner picked |
| `ai_only` *(later)* | Owner only; the AI may use it for the owner's own styling |

This is enforced by one database function, `can_view_item(item_id)`. The `items` rules, the `item_images` rules **and the storage bucket rules** all call that same function, so the photo rules can never drift from the item rules.

### 4.2 Rules that apply everywhere

1. **Strangers see nothing.** No public profiles, no search by username. You can only connect through an invite link that you deliberately send. The link is a long random code that expires and works once. Knowing someone's name or email gives no access.
2. **Friendship = both sides agree.** Sending the invite is your consent. Accepting it is theirs.
3. **Permissions are one-directional.** "Sarah can view my wardrobe" doesn't mean "I can view Sarah's".
4. **Removing a friend** cuts all access immediately: permissions and item grants are deleted. Pending borrow requests are cancelled. An item currently lent out stays tracked until it's returned.
5. **Sensitive data lives in owner-only tables** (see section 3).
6. **Status changes go through checked functions.** For example, only the owner can approve a borrow, and only from the "requested" state.
7. **Every new table** gets RLS policies + explicit `GRANT`s (your Supabase project doesn't expose tables automatically) + privacy tests, all in the same migration.

### 4.3 Photos and files (Storage)

| Bucket | Contains | Who can read |
|---|---|---|
| `wardrobe-images` | Item photos: crop, cutout, AI representation (each labeled) | Same as the item (`can_view_item`) |
| `scan-uploads` | Original outfit photos, single-item photos, inspiration screenshots | Owner only |
| `vault-documents` | Receipts, certificates of authenticity | Owner only |
| `avatars` | Profile pictures | Owner + accepted friends |
| `exports` | Data-export files (deleted after 24 h) | Owner only |

- All buckets are **private**. The app gets **signed links valid for ~30 minutes**.
- Files are stored under the owner's ID (`{owner_id}/…`), and uploads are only allowed into your own folder.
- **Your full outfit selfie is never shown to friends.** Friends only see the item images (crop or cutout), not the original photo with your face or your bedroom.

### 4.4 Photo metadata (EXIF / GPS)

- **In the app:** every photo is re-saved (resized, converted to JPEG) before upload. This removes the metadata, including GPS location. Phase 3 includes an automated test proving the GPS data is really gone.
- **On the server (second safety net):** the upload function checks for leftover location data and rejects the photo if any is found.

### 4.5 Addresses (borrowing by post)

- Addresses are never stored on a profile. When a borrow is set to "please send it", the person *receiving* the item enters (or picks) an address for **that borrow only**.
- Only the two people in that borrow can read it, and only while it's active. A scheduled job **erases it 14 days after the item is returned or the borrow is cancelled**.

### 4.6 Logging, analytics, AI data

- **Logs** record what happened, never the content: "analyze-item ok, 2.1 s". No photos, prices, item details, addresses or AI prompts.
- **Analytics** use only the allow-listed events in the spec (`item_added`, `borrow_requested`, …), with no wardrobe details attached.
- **Data sent to AI:** for scans, the photo (which may include a face). For the stylist, item descriptions and availability only. Never prices, Vault data, addresses or friends' details. Everything that goes to AI providers must be listed in the privacy policy (see section 10).

### 4.7 Account deletion and data export

- **Delete account** (in Settings): a server function deletes every file the user owns in every bucket, then deletes the user. Database rows are deleted automatically along with the user. Friends lose access instantly. Built in Phase 1, because it's easy while the app is small and Apple requires it.
- **Export my data** (GDPR): a server function builds a zip (JSON + images) and gives a 24-hour download link. Built before any beta with other people.

### 4.8 Abuse and cost protection

- Every AI function checks a **per-user quota** before calling a provider, e.g. max scans per day. Limits live in the server config.
- Inputs are validated on the server (types, sizes, image formats, max image size).
- **Block / report a friend**: Apple expects this for apps where users share content with each other. Included with the circle screens.

---

## 5. Server-side functions

**Edge Functions** (server programs; they hold the secret keys):

| Function | What it does | Phase |
|---|---|---|
| `delete-account` | Removes all files + the user | 1 |
| `analyze-item` | Single item photo → background removal → AI suggestions (category, color, … with confidence) → embedding | 3 |
| `analyze-outfit` | Outfit photo → AI detects each garment (with position boxes) → proposals with confidence → duplicate candidates | 4 |
| `process-cutouts` | Crops each detected garment → background removal → stores cutouts | 5 |
| `stylist-chat` | Conversational stylist; reads only the user's own available items; returns outfits built from real item IDs | 7 |
| `recreate-look` | Inspiration photo → analysis → match per garment (exact / close / alternative / missing) | 8 |
| `suggest-alternatives` | Friend styling: three AI alternatives from the friend's picks (within what the friend may see) | 1.1 |
| `send-reminders` | Scheduled: borrow return reminders, invite expiry, address erasure, export clean-up | 11 |
| `export-data` | Builds the GDPR export zip | before beta |
| `revenuecat-webhook` | Updates the user's plan when they subscribe | 1.1 |
| `estimate-value` | Interprets real market data into a value range | later |

**Database functions** (run inside Postgres, so changes are all-or-nothing):

- `can_view_item`, `is_connected`, `has_friend_permission`: the privacy building blocks
- `create_invite`, `accept_invite`, `remove_connection`
- `confirm_scan_detection`: turns a reviewed detection into an item, or links it to an existing one ("Same item")
- `request_borrow`, `approve_borrow`, `decline_borrow`, `cancel_borrow`, `mark_handed_over`, `mark_returning`, `mark_returned`
- `item_availability`: calculated availability

---

## 6. AI and image pipeline

### 6.1 Swappable interfaces (CLAUDE.md rule 8)

```
AIProvider            → analyzeGarment, detectOutfitItems, analyzeInspiration,
                        stylistReply, embedText          (first implementation: OpenAI)
BackgroundRemover     → removeBackground(image)          (first: Photoroom, see 6.3)
GarmentSegmenter      → segment(outfitImage) → regions   (first: AI boxes + crop)
ValuationSource       → comparables(item)                (later)
ImageStore            → put / signedUrl / delete         (first: Supabase Storage)
```

All model names and provider choices live in **one file**: `supabase/functions/_shared/ai/config.ts`. Switching to a newer or cheaper model means editing one line.

### 6.2 How AI results reach your wardrobe

1. You upload a photo.
2. The server returns **proposals** with confidence (e.g. "Material: wool · low confidence").
3. You review, edit, delete or merge the proposals.
4. Only what you confirm is saved. Your corrections are saved as the truth, and the original AI guess is kept privately so the prompts can be improved later.
- Brand is never stated as a fact from appearance alone. It shows as "Possibly: …" if at all, and is otherwise left for you to enter.
- AI outputs use **structured outputs** (a fixed JSON shape), so the app never has to "read" free text.

### 6.3 Background removal and cutouts: two different problems

- **Single item photos (easy):** one garment on a background. A background-removal service handles this well. **Plan:** Photoroom API (~$0.02 per image, EU company). Free alternatives built into the phones exist (Apple's subject lifting, Google ML Kit), but they need custom native code, so they're an option to evaluate later to cut costs.
- **Outfit photos (hard, the #1 technical risk):** separating *each* garment from a photo of a person wearing them. Plan:
  1. The AI returns an approximate box around each garment.
  2. The app crops each box from the photo.
  3. Background removal runs on each crop.
  - **Fallback:** if the cutout is poor, the item keeps the plain crop, labeled as such.
  - **Phase 5 starts with a 2-day test** of 2–3 approaches on ~30 real outfit photos *before* building. You'll see the results side by side and choose.
- **AI-generated "clean product shots"** of a garment: later, if ever. Always labeled "AI representation", never shown as a real photo.

### 6.4 Duplicate detection

- Each item gets an **embedding** (a numeric "fingerprint" of its description and attributes), stored with `pgvector`.
- When a new detection's fingerprint is very close to an existing item, the app asks "We think this item is already in your wardrobe" → **Same item / Different item**. Your answer is saved and used to improve matching.
- Fingerprints are stored from Phase 3 onwards, so the data already exists by Phase 6.
- **Honest limit:** OpenAI's embeddings work on text, not images. The MVP fingerprint is built from a detailed AI description of the garment. If that isn't accurate enough, a true *image* fingerprint provider can be added behind the same interface.

---

## 7. External providers and rough costs

Prices checked 1 Oct 2026. These are **estimates** that change often; we'll re-check at signup.

### Fixed costs

| Provider | What for | Cost | When |
|---|---|---|---|
| Apple Developer Program | TestFlight, App Store, Sign in with Apple | $99 / year | Phase 1 (Apple sign-in), definitely by Phase 3 |
| Google Play Console | Android testing and release | $25 one-time | When we test on Android |
| Supabase | Database, auth, storage, functions | Free now. **Pro $25/month** before other people use it (the free plan pauses after 1 week of inactivity and has no backups) | Pro before first beta |
| Expo / EAS | Cloud builds | Free tier is enough to start; paid from ~$19/month if builds queue too long | Phase 0 |
| Domain name (e.g. lemari.app) | Sign-in email sender, privacy policy page | ~€15–40 / year | Before beta |
| Email sending (e.g. Resend) | Sign-in code emails. Supabase's built-in email is heavily rate-limited and only meant for testing | Free tier (~3,000 emails/month) | Before beta |
| RevenueCat | Subscriptions | Free up to $2,500 monthly revenue, then 1% | 1.1 |

### Usage costs (per action, rough)

| Action | Rough cost |
|---|---|
| Add single item (AI + background removal) | ~$0.02–0.04 |
| Outfit scan with 5 items (AI + 5 cutouts) | ~$0.10–0.15. **Cutouts are most of the cost** |
| Stylist message | ~$0.005–0.02 |
| Recreate This Look | ~$0.02–0.05 |
| Embeddings (fingerprints) | Negligible (fractions of a cent) |

**Example:** 100 active beta testers, each doing 10 outfit scans + 30 stylist messages a month ≈ **$150/month in AI + $25 Supabase**. That's why per-user quotas are built in from the first AI feature, and why paid plans matter before a wide launch.

We'll set a **monthly spending limit** in the OpenAI dashboard from day one.

---

## 8. Navigation proposal (5 tabs)

```
 HOME  ·  WARDROBE  ·  ( + )  ·  STYLE  ·  CIRCLE
```

- **Home**: today's prompt, quick actions, recently added, borrow requests, items lent out, friend styling. Your profile and settings sit behind your avatar in the top corner (Profile doesn't need a tab).
- **Wardrobe**: all items, filters, outfit history. **Vault** is a locked section here (Face ID optional).
- **+ (Add)**: a sheet with *Scan an outfit · Add an item · Recreate a look*.
- **Style**: AI stylist chat and outfit boards. Recreate This Look results live here too.
- **Circle**: friends, invites, permissions, styling from and for friends, **Borrowing** (requests, lent out, borrowed).

Final screen designs get proposed before building each feature.

---

## 9. MVP vs 1.1 vs later

### MVP (first release to real users)
- Sign in with email code + Apple. Onboarding with a clear privacy explanation.
- Wardrobe: add, edit, delete items manually or by photo. Per-item privacy setting.
- Single item add with background removal and AI suggestions.
- Outfit scan → "5 items detected" review → confirm.
- Garment cutouts (or labeled crops as fallback).
- Duplicate detection (Same item / Different item).
- AI stylist using real wardrobe images, respecting availability.
- Recreate This Look (Inspiration vs My version).
- Private circle: invite links, permissions, Private / Circle / Selected visibility, block / report.
- Friend styling (a friend builds a board and sends it).
- Borrowing: request → approve → hand over / ship → return, with reminders and the address rules.
- Basic outfit history (from scans).
- Account deletion, data export, push notifications, privacy-safe analytics.
- **Usage quotas** instead of a paywall (see decision D3).

### 1.1
- Vault (private records + receipts and authenticity documents)
- Subscriptions and paywall (RevenueCat)
- AI alternatives in friend styling
- Share-to-LEMARI from Instagram/Pinterest (a share extension; native work)
- Sharing specific collections or categories with specific friends
- Weather-aware styling and packing lists
- Google sign-in
- `ai_only` visibility

### Later
- Resale valuation from real market data; "What is my wardrobe worth?"
- Wear insights (most/least worn, cost per wear)
- True image fingerprints for duplicates; on-device background removal
- AI representation images (always labeled)

---

## 10. Biggest risks

| # | Risk | Why it matters | How we handle it |
|---|---|---|---|
| 1 | **Multi-garment cutouts look bad** | It's the "magic moment" of the app | Phase 5 test before building; labeled crop fallback; swappable provider |
| 2 | **A privacy bug** (friend sees a price, stranger sees an item) | It's the core promise of the product | Separate owner-only tables, one shared `can_view_item` function, automated privacy tests that must pass, no app-only hiding |
| 3 | **AI cost per user** grows faster than revenue | Scans with cutouts cost ~10–15 cents | Quotas from day one, spending cap, cheap model where good enough, on-device removal later |
| 4 | **GDPR and faces**: outfit selfies (faces) and other people in inspiration photos go to US-based AI providers | Legal and trust risk in the EU | Clear consent in onboarding; privacy policy lists every provider; OpenAI's API doesn't train on your data by default (check their EU data-residency option); only the needed image is sent; get a **legal review of the privacy policy before public launch** |
| 5 | **App Store review**: account deletion, Sign in with Apple, privacy labels, block/report for shared content | Rejection delays launch | All planned into the MVP |
| 6 | **Duplicate detection** merges two different items, or misses duplicates | Messy wardrobe | Always ask the user; never auto-merge |
| 7 | **AI is wrong** (material, brand) | Trust | Confidence shown, user confirms, no brand/authenticity claims |
| 8 | **Scope**: this is a big app for a small team | Burnout, half-finished features | Strict phases, MVP list above, Vault and valuation pushed to 1.1/later |
| 9 | **Supabase free plan** pauses and has no backups | Data loss / downtime | Upgrade to Pro before anyone else uses it |

---

## 11. Phased roadmap

I kept the order from the spec, with three changes:

- **Account deletion moves to Phase 1** (cheap now, mandatory later).
- **AI on single items comes before outfit scans**, because it builds the AI plumbing on the easy case first.
- **A "beta readiness" phase is added before the circle**, since testing the circle means inviting real friends.

| Phase | Name | What's in it | You'll see on your phone |
|---|---|---|---|
| 0 | Setup | Git, Expo project, design tokens, Supabase link, local database, test harness, code checks | A styled "Hello LEMARI" screen |
| 1 | Auth + onboarding | Dev-build switch, email code + Apple sign-in, onboarding screens, profile, **delete account** | Sign in, onboarding, delete account |
| 2 | Wardrobe + full privacy model | **All** privacy-related tables (items, private details, circle, borrow, vault skeleton), RLS, storage rules, privacy tests; manual add/edit/delete; wardrobe grid | Add/edit items by hand |
| 3 | Single item photo + AI | Camera/library, EXIF stripping (tested), background removal, AI suggestions with confidence, quotas, fingerprints | Photograph a garment → cutout + suggestions → confirm |
| 4 | Outfit scan | Multi-item detection, "5 items detected" review, edit/delete/merge, crops | Scan an outfit → review → items appear |
| 5 | Cutout pipeline | Comparison test first, then the chosen approach | Clean cutouts from outfit photos |
| 6 | Duplicate detection | Same item / Different item flow | Re-scan an outfit → "already in your wardrobe" |
| 7 | AI stylist | Chat, outfit boards from real images, availability-aware | Ask "dinner tonight?" → outfit board |
| 8 | Recreate This Look | Inspiration vs My version, try another match, modifiers ("for work", "12°C") | Upload an inspiration photo → your version |
| 9 | Beta readiness | Data export, push notifications, analytics, privacy policy page, custom email sender, Supabase Pro, TestFlight for others | Install via TestFlight; export your data |
| 10 | Private circle | Invites, connections, permissions, visibility, block/report | Invite a friend; see each other's shared items |
| 11 | Friend styling | Friend builds and sends a board | "Erma styled a look for you" |
| 12 | Borrowing | Full lifecycle, overlap protection, address handling, reminders | Request, approve, lend, return |
| — | **MVP release** | App Store / Play Store | |
| 13 | Vault (1.1) | Vault records, documents bucket, Face ID lock | Private vault |
| 14 | Subscriptions (1.1) | RevenueCat, paywall, plan-based quotas | Upgrade screen |
| 15 | Valuation (later) | Provider interface, then a real market-data source | Value ranges with date and confidence |

Each phase ends with: tests and type checks passing, privacy tests passing, a commit, an update to `PROJECT_STATUS.md`, and a stop for your approval.

---

## 12. Accounts and keys you need

**Nothing needs to be done today.** I'll give exact click-by-click steps when each one is needed.

| Account | Needed by | Cost | What you'll give me |
|---|---|---|---|
| Supabase *(you already have the project)* | Phase 0 | Free → $25/mo | Project URL + **publishable (anon) key**. These are safe to share and are meant to be in the app. **Never** paste the service-role key into chat. |
| Docker Desktop (installed on your Mac) | Phase 0 | Free for personal/small business | Nothing. It runs a local copy of the database for privacy tests |
| Expo account | Phase 0 | Free | Your Expo username |
| GitHub account + private repository *(recommended)* | Phase 0 | Free | Nothing secret. It's an online backup of the code |
| Apple Developer Program | Phase 1 (Apple sign-in) | $99/yr | Team ID; you'll create a Sign in with Apple key yourself and paste it into Supabase |
| OpenAI Platform (billing + spending limit) | Phase 3 | Pay per use | You paste the API key into **Supabase Edge Function secrets** yourself, never into chat or the app |
| Photoroom API *(or the chosen background remover)* | Phase 3 | ~$0.02/image | Same: key goes into Supabase secrets |
| Domain name | Phase 9 | ~€15–40/yr | Nothing; you'll add DNS records I give you |
| Resend (email sending) | Phase 9 | Free tier | Key goes into Supabase settings |
| Google Play Console | Phase 9 (Android testers) | $25 once | Nothing secret |
| RevenueCat | Phase 14 | Free to start | Public SDK keys (safe in app) + webhook secret (server only) |
| Analytics (e.g. PostHog EU) | Phase 9 | Free tier | Public project key |

---

## 13. Open decisions for Erma

These are recorded in `PROJECT_STATUS.md` too. My recommendation comes first.

- **D1 Navigation:** 5 tabs as in section 8? *(Recommended: yes)*
- **D2 Sign-in:** email with a 6-digit code (no passwords) + Apple; Google in 1.1? *(Recommended: yes)*
- **D3 Money before launch:** MVP with free usage quotas, paywall in 1.1, *or* paywall in the MVP? *(Recommended: quotas in the MVP. Revisit before the public launch based on beta costs.)*
- **D4 Vault:** 1.1 rather than MVP? The *tables and rules* are still designed in Phase 2. *(Recommended: 1.1)*
- **D5 Background removal:** start with Photoroom (paid, simple), evaluate free on-device options later? *(Recommended: yes)*
- **D6 Stylist chat history:** keep forever, or auto-delete after e.g. 90 days? *(Recommended: 90 days, user can delete anytime)*
- **D7 GitHub:** create a private GitHub repo as an online backup? *(Recommended: yes)*
