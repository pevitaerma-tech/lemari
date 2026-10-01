# LEMARI — Project Status

*Last updated: 1 Oct 2026*

## Completed
- Product spec written (`docs/PRODUCT_SPEC.md`).
- Architecture overview written (`docs/ARCHITECTURE.md`): architecture, folder structure, privacy model, server functions, providers and costs, MVP scope, risks, roadmap, accounts needed.
- Database schema designed on paper (`docs/SCHEMA.md`). No tables built yet.
- Git set up with a `.gitignore` that blocks `.env` files and other secrets. First commit made.

## Currently building
- Nothing. **Waiting for Erma to review the architecture and answer the open decisions below.**

## Bugs / known issues
- None (no code yet).

## Decisions
| Date | Decision | Why |
|---|---|---|
| 29 Sep 2026 | Supabase project in Central EU (Frankfurt), auto-expose tables OFF, auto-RLS ON | EU data location; nothing reachable unless explicitly granted |
| 1 Oct 2026 | Stack: Expo + TypeScript + Expo Router, Supabase, OpenAI via Edge Functions only | Matches the spec; one codebase; secrets stay on the server |
| 1 Oct 2026 | Sensitive data (prices, vault, addresses, AI confidences) in separate owner-only tables; Vault membership itself is private | Database rules work per row, not per column |
| 1 Oct 2026 | One `can_view_item` rule shared by items, item images and storage | Photo access can never drift from item access |
| 1 Oct 2026 | Account deletion moved to Phase 1; beta-readiness phase added before the circle | Apple requirement; circle testing needs real people |
| 1 Oct 2026 | No profile addresses; an address exists only per borrow and is erased 14 days after it ends | Minimise stored personal data |
| 1 Oct 2026 | Native `/ios` and `/android` folders are generated, not committed | Standard Expo practice, less to maintain |

**Open decisions (need Erma's answer; recommendation first):**
- D1 Navigation: 5 tabs Home · Wardrobe · + · Style · Circle (profile behind avatar) → *recommend yes*
- D2 Sign-in: email 6-digit code + Apple for MVP; Google in 1.1 → *recommend yes*
- D3 Money: usage quotas in MVP, paywall in 1.1 → *recommend yes, revisit after beta costs*
- D4 Vault screens in 1.1 (tables designed in Phase 2) → *recommend yes*
- D5 Background removal: start with Photoroom → *recommend yes*
- D6 Stylist chat history auto-deleted after 90 days → *recommend yes*
- D7 Private GitHub repo as online backup → *recommend yes*

## Phases
| # | Phase | Status |
|---|---|---|
| 0 | Setup (Expo project, tokens, Supabase link, local DB, test harness) | Not started |
| 1 | Auth + onboarding + delete account (dev build switch) | Not started |
| 2 | Wardrobe + full privacy model (all privacy tables, RLS, storage rules, tests) | Not started |
| 3 | Single item photo + EXIF stripping + background removal + AI suggestions | Not started |
| 4 | Outfit scan + review screen | Not started |
| 5 | Cutout pipeline (comparison test first) | Not started |
| 6 | Duplicate detection | Not started |
| 7 | AI stylist | Not started |
| 8 | Recreate This Look | Not started |
| 9 | Beta readiness (export, push, analytics, privacy policy, email, Supabase Pro) | Not started |
| 10 | Private circle | Not started |
| 11 | Friend styling | Not started |
| 12 | Borrowing | Not started |
| — | MVP release | |
| 13 | Vault (1.1) | Not started |
| 14 | Subscriptions (1.1) | Not started |
| 15 | Valuation (later) | Not started |

## Next steps
1. Erma reviews `docs/ARCHITECTURE.md` and `docs/SCHEMA.md` and answers D1–D7.
2. After approval: I write the detailed Phase 0 plan, then build it.

## Things Erma needs to do
- **Now:** read the two documents and answer D1–D7. Nothing to install or sign up for yet.
- **For Phase 0** (step-by-step instructions will follow): share the Supabase project URL + publishable key, install Docker Desktop, create a free Expo account, optionally create a GitHub account.
- **For Phase 1:** join the Apple Developer Program ($99/year). It can take a day or two to be approved, so it's worth starting early.
