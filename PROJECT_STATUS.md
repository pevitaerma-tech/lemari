# LEMARI — Project Status

*Last updated: 1 Oct 2026*

LEMARI (working title) is a joint project of Erma and Pevita. Launch audience: Indonesia and Malaysia. Languages: English + Bahasa Indonesia (Malay in 1.1).

## Completed
- Product spec (`docs/PRODUCT_SPEC.md`).
- Architecture overview v2 (`docs/ARCHITECTURE.md`): updated for the Singapore region, Indonesia/Malaysia audience, English + Bahasa Indonesia, GDPR + Indonesia PDP + Malaysia PDPA, Google sign-in in the MVP, Android-first testing.
- Database schema design v2 (`docs/SCHEMA.md`): adds `user_settings` (language, currency, time zone) and `consents`, language-neutral codes, money as number + currency. No tables built yet.
- Git set up; `.gitignore` blocks `.env` files and other secrets.
- GitHub remote `pevitaerma-tech/lemari` connected through a dedicated deploy key, so this repo never uses the Mac's personal GitHub login (JayrosCreative). Commit author for this repo: Erma <pevitaerma@gmail.com>.

## Currently building
- Nothing. **Waiting for Erma to approve the Phase 0 plan** and do the Phase 0 to-dos below.

## Bugs / known issues
- Nothing has been pushed to GitHub yet. That needs the deploy key added on GitHub (see "Things Erma needs to do").
- This Mac has Node 23, a short-lived version that's no longer supported. Phase 0 proposes switching to Node 24 LTS.

## Decisions
| Date | Decision | Why |
|---|---|---|
| 29 Sep 2026 | Supabase project in **Southeast Asia (Singapore)**, auto-expose tables OFF, auto-RLS ON | Closest region to Indonesia/Malaysia; nothing reachable unless explicitly granted |
| 1 Oct 2026 | Stack: Expo (SDK 57) + TypeScript + Expo Router, Supabase, OpenAI via Edge Functions only | Matches the spec; one codebase; secrets stay on the server |
| 1 Oct 2026 | English + Bahasa Indonesia from Phase 0 (i18next + expo-localization + Intl); Malay in 1.1 | Main users are in Indonesia and Malaysia |
| 1 Oct 2026 | Fixed lists stored as language-neutral codes; money as number + ISO currency | Language can change without touching data; Rp / RM / € formatted correctly |
| 1 Oct 2026 | Sensitive data (prices, vault, addresses, AI confidences, settings, consents) in separate owner-only tables; Vault membership itself is private | Database rules work per row, not per column |
| 1 Oct 2026 | One `can_view_item` rule shared by items, item images and storage | Photo access can never drift from item access |
| 1 Oct 2026 | Account deletion in Phase 1; beta-readiness phase before the circle | Store requirement; circle testing needs real people |
| 1 Oct 2026 | No profile addresses; an address exists only per borrow and is erased 14 days after it ends | Minimise stored personal data |
| 1 Oct 2026 | Consent records (privacy policy, separate AI photo consent, age) stored per version | Required in some form by GDPR, PDP and PDPA; details to confirm with an advisor |
| 1 Oct 2026 | Native `/ios` and `/android` folders are generated, not committed | Standard Expo practice |
| 1 Oct 2026 | GitHub access via a deploy key limited to `pevitaerma-tech/lemari` only | Keeps the project separate from the personal GitHub account; least access |
| 1 Oct 2026 | **D1** 5 tabs: Home · Wardrobe · + · Style · Circle | Approved by Erma |
| 1 Oct 2026 | **D2** Email code + Google + Apple in the MVP; Android testing from Phase 0 | Approved (changed by Erma: most users are on Android) |
| 1 Oct 2026 | **D3** Usage quotas in the MVP, paywall in 1.1 | Approved |
| 1 Oct 2026 | **D4** Vault screens in 1.1 (tables in Phase 2) | Approved |
| 1 Oct 2026 | **D5** Photoroom for background removal | Approved |
| 1 Oct 2026 | **D6** Stylist chats auto-deleted after 90 days | Approved |
| 1 Oct 2026 | **D7** Private GitHub repo `pevitaerma-tech/lemari` | Approved |

**Open decisions (recommendation first):**
- D8 Developer accounts (Apple/Google) personal or organisation? Organisation is exempt from Google's 12-testers-for-14-days rule but needs a company + D-U-N-S number → *decide before Phase 1; ask your advisor*
- D9 Minimum age → *18+, confirm with advisor*
- D10 Regional fashion categories (hijab/tudung, kebaya, batik, baju kurung, gamis, sarong, modest preference…) → *you and Pevita review the list in Phase 2*
- D11 Phones set to Malay before Malay exists → *show English*
- D12 Permanent app identifier → *`com.pevitaerma.lemari`* (needed in Phase 0, can't be changed after the first store release)
- D13 Install Node 24 LTS on this Mac (replaces Node 23) → *yes*

**Legal items to check with a privacy advisor** (detail in ARCHITECTURE.md section 6): consent wording and whether AI-photo consent must be separate; transfers to Singapore / US (OpenAI) / France (Photoroom); whether outfit photos with faces count as biometric data; minimum age; retention periods; breach-notification plan; privacy policy in EN / ID / MS; DPO / registration duties.

## Phases
| # | Phase | Status |
|---|---|---|
| 0 | Setup: Expo, tokens, **en + id translations**, app-name config, Supabase link, local DB, tests, CI, **first Android build** | Plan ready, awaiting approval |
| 1 | Auth (email code, Google, Apple) + onboarding with consent + delete account (dev build switch) | Not started |
| 2 | Wardrobe + full privacy model (all privacy tables, RLS, storage rules, tests) | Not started |
| 3 | Single item photo + EXIF stripping + background removal + AI suggestions | Not started |
| 4 | Outfit scan + review screen | Not started |
| 5 | Cutout pipeline (comparison test first) | Not started |
| 6 | Duplicate detection | Not started |
| 7 | AI stylist (en + id) | Not started |
| 8 | Recreate This Look | Not started |
| 9 | Beta readiness (export, push, analytics, legal texts in 3 languages, email, Supabase Pro, Google Play closed test, TestFlight) | Not started |
| 10 | Private circle | Not started |
| 11 | Friend styling | Not started |
| 12 | Borrowing | Not started |
| — | MVP release | |
| 13 | Malay + Vault (1.1) | Not started |
| 14 | Subscriptions (1.1) | Not started |
| 15 | Valuation (later) | Not started |

## Next steps
1. Erma adds the deploy key on GitHub, then I push the first commits.
2. Erma approves the Phase 0 plan and answers D11–D13 (D8 before Phase 1).
3. I build Phase 0, one step and one commit at a time.

## Things Erma needs to do
- **Now:** add the deploy key to the `pevitaerma-tech/lemari` repo on GitHub (steps in chat).
- **For Phase 0** (steps in chat when we start): install Docker Desktop; create an Expo account with pevitaerma@gmail.com; send the Supabase project URL + publishable key; run two sign-in commands I'll give you (Supabase and Expo); have an Android phone ready.
- **Before Phase 1:** decide D8 (personal or organisation) with Pevita, then join the Apple Developer Program ($99/year; approval can take a few days).
- **Before beta:** find a privacy advisor familiar with GDPR, Indonesia's PDP law and Malaysia's PDPA; arrange a native-speaker review of the Indonesian text.
