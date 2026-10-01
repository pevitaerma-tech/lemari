# LEMARI — Product Specification

"Lemari" means wardrobe/closet in Bahasa Indonesia.

**Your wardrobe. Your circle. Your rules.**

LEMARI is a PRIVATE AI-powered digital wardrobe for individuals and their trusted inner circle. It is NOT a public social network: no public wardrobe browsing, no public followers, no popularity metrics, no discovery feed.

Users build a digital wardrobe from clothes they actually wear. AI then helps them organize it, style it, recreate inspiration looks, share selected parts privately with trusted friends, lend and borrow clothes, track where borrowed clothes are, privately track valuable items, and (later) understand approximate resale value.

---

## Technical stack (preferred unless there is a strong reason to change)

- **App:** React Native, Expo, TypeScript (iOS + Android from one codebase)
- **Backend:** Supabase (PostgreSQL, Auth, Storage, Row Level Security, Edge Functions)
- **AI:** OpenAI API, called only from server-side Edge Functions; structured outputs where useful; provider behind an interface
- **Subscriptions (later):** RevenueCat
- **Notifications:** Expo Notifications
- **Auth:** email + Sign in with Apple (required by Apple if any other social login is offered); Google optional
- Must be swappable: image segmentation, background removal, resale-market data and AI providers.

---

## How clothes enter LEMARI

### 1. Outfit scan
User uploads a mirror selfie, OOTD, camera photo or saved photo. AI identifies each separate item and proposes: category, subcategory, likely color, material (if reasonably identifiable), pattern, silhouette, seasonality, style/aesthetic tags, and confidence values.

Where technically possible, extract each garment as a transparent-background cutout.

Before saving, show "5 items detected" and let the user edit, delete, merge and confirm each item.

Never present an AI-reconstructed garment as the real item. If reconstruction is ever used, clearly label original extracted image vs AI-generated representation.

### 2. Add single item
User photographs a new garment, bag, shoes, jewelry etc., or uploads a product photo. AI isolates the background where possible, recognizes the item and proposes metadata. User confirms before saving.

### 3. Recreate This Look (inspiration photo)
User uploads or shares a screenshot/photo (Instagram, Pinterest, TikTok, web, camera roll). AI analyzes categories, colors, proportions, silhouette, texture, formality, aesthetic and accessories, then searches the user's OWN wardrobe for similar pieces.

Matching must not be color-only. Consider garment type, silhouette, fit, proportions, material/texture, pattern, color, formality and aesthetic.

Show **INSPIRATION vs MY VERSION**. For each source garment: Exact/very close match · Close match · Alternative · Missing. Allow "Try another match".

Example requests: "Recreate this look with my clothes." · "Make this more feminine." · "This aesthetic but suitable for work." · "Recreate this but I don't wear heels." · "Make it suitable for 12°C."

---

## Wardrobe items

Fields to support (normalize properly; this is not the final schema):

- **Visible/shareable:** title, category, subcategory, dominant color, secondary colors, material, pattern, silhouette, fit, brand (user-entered), size, style tags, season tags, images (original, cutout, AI representation — labeled), date added, favorite, borrowable, availability status
- **Owner-only (separate table):** purchase date, purchase price, currency, purchase location, notes, condition details
- **Usage:** last worn, times worn (derived from outfit history where possible)

## Duplicate detection

If the same item appears in several outfit photos, do NOT create duplicates. Show "We think this item is already in your wardrobe" with options Same item / Different item. Store visual/semantic information (e.g. image embeddings) so matching improves over time.

## AI stylist

Conversational stylist with controlled access to the user's own authorized wardrobe data.

Examples: "What should I wear for dinner tonight?" · "Quiet luxury but not overdressed." · "Style my red trousers three ways." · "Something Parisian." · "Comfortable but sexy." · "A meeting and drinks afterward." · "Pack me outfits for four days in Valencia."

Consider available garments, preferences, occasion, weather (later), items lent out or reserved. Never recommend a physically unavailable item without saying so. Results can form visual outfit boards using real wardrobe images.

---

## Private circle

Not a public network. Users manually invite trusted people; friendships require consent from both sides. Knowing a username must never grant access.

Per-friend permissions: can view shared wardrobe · can create outfits for me · can request to borrow · can view specific collections/categories.

## Item privacy (per item)

- **PRIVATE** — only the owner
- **CIRCLE** — approved friends with view permission
- **SELECTED FRIENDS** — only explicitly chosen friends
- **AI ONLY** (future) — AI uses it for the owner's styling; friends never see it

Must be enforced in the database and storage, not only the app.

## Friend styling

A friend with permission opens the owner's shared wardrobe, builds an outfit board ("Style Befita") and sends it: "Erma styled a look for you." AI can optionally suggest three alternatives based on the friend's picks.

## Borrowing

Owner decides per item whether it is borrowable. A friend with permission requests: start date, return date, pickup or "please send it", optional note. Owner approves or declines.

Statuses to design carefully (a borrow request has its own lifecycle, separate from the item): requested → approved/reserved or declined → handed over/shipped (lent out) → returning → returned, plus cancelled. Prevent overlapping reservations for the same item.

The app knows who has the item, since when, the expected return date and pickup/shipping method. Friendly reminders: "My dress is still with Sarah. Remind her?" Borrow history is kept privately. The AI stylist understands availability.

**Delivery:** pickup coordination, or send with secure address handling, optional tracking number, mark shipped/received. No payments or shipping labels in the MVP.

## Vault

Private area for luxury bags, jewelry, watches, rare sneakers, vintage, sentimental pieces, expensive garments, collectibles.

Private fields: exact model, purchase date, original retail price, price paid, currency, purchase location, receipt, authenticity documents, serial/reference numbers, condition, insurance notes, private notes, resale estimate range, valuation date, confidence, source metadata.

Vault/financial details must NEVER become visible to friends just because the item itself is visible. Documents go in protected storage.

## Resale value (later)

Show a range, never a single exact price (e.g. "€2,800–€3,300 · updated 29 Sep 2026 · confidence Medium"). Based on real market data (comparable listings/sold prices from legal sources, model, year, color, condition, completeness, demand). AI interprets data; it never invents prices. No fake valuations in production. Possible private dashboard: purchase value, estimated resale value, value change, value by category, "What is my wardrobe worth?"

## Outfit history

Keep original OOTDs with date, items, optional occasion/location, favorites, repeats. Later insights: most/least worn, cost per wear. Not an MVP priority.

---

## Design direction

Premium fashion app: editorial, fashion-forward, elegant, minimal, tactile, modern, calm. Garment photography and cutouts take visual priority. Excellent spacing and typography.

Avoid: corporate SaaS look, childish style, techy/neon AI aesthetics, excessive gradients, huge rounded cards everywhere, clutter, cheap e-commerce feel. Respect accessibility (contrast, text sizes, screen readers). Use design tokens.

## Navigation

Candidate tabs: HOME · WARDROBE · SCAN · STYLE · CIRCLE · PROFILE. Recreate This Look, Borrow and Vault live inside these. Don't overload the tab bar; propose the best structure (likely 5 tabs max) before building final navigation.

**Home:** today's style prompt, scan an outfit, recreate a look, recently added, borrow requests, items lent out, friend styling requests, recent outfits. Useful and clean.

## Onboarding

"Don't photograph your whole wardrobe. Just wear it. We'll build it for you."

1. Add a few outfit photos. 2. LEMARI detects your clothes. 3. Your wardrobe builds itself. 4. AI styles what you own. 5. Invite only trusted friends if you want.

Communicate privacy clearly, including that photos are analyzed by an AI service.

## Monetization (later — don't build complex pricing until asked)

- **Free:** limited items or AI use
- **Premium:** full AI styling, outfit scanning, circle, borrowing, more capacity
- **Vault / Premium+:** valuation and asset features

## Analytics

Privacy-conscious events only: onboarding_completed, item_added, outfit_scanned, item_detected, duplicate_confirmed, ai_style_requested, inspiration_uploaded, inspiration_recreated, friend_invited, borrow_requested, borrow_approved, item_returned. Never include wardrobe metadata, prices, photos or personal details in analytics.

---

## Suggested phases (Claude may propose a better order with reasons)

0. Architecture and project setup
1. Authentication + onboarding
2. Wardrobe database + item CRUD — **including the full privacy/RLS model and circle tables**, even though the circle screens come later (retrofitting privacy is risky)
3. Single item photo upload (with EXIF stripping)
4. Outfit upload + AI item identification
5. Garment extraction / cutout pipeline
6. Duplicate detection
7. AI stylist using actual wardrobe data
8. Recreate This Look
9. Private circle system (screens)
10. Friend outfit creation
11. Borrow / request / reserve / return
12. Vault
13. Subscription / paywall (RevenueCat)
14. Valuation architecture, later live-market valuation

Account deletion and data export must exist before any public (TestFlight beyond yourself) release.
