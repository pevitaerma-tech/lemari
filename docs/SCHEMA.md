# LEMARI — Database Schema

*Status: **DESIGN DRAFT** · 1 Oct 2026 · No tables exist yet. Most tables are created in Phase 2 (privacy model first); the rest are added in the phase noted. This file must be updated with every migration.*

## How to read this

- **Who can see / change** describes the Row Level Security (RLS) rules in plain English. These rules are enforced by the database itself, not by the app.
- **"Owner"** = the user who created the row. **"Friend"** = a user with an *accepted* connection to the owner.
- Every table has RLS enabled. Your Supabase project doesn't expose new tables automatically, so every migration also `GRANT`s access to the `authenticated` role explicitly. The `anon` role (not logged in) gets **nothing** on any table.
- 🔒 = owner-only table. Friends can never read it, even if they can see the related item.
- Timestamps (`created_at`, `updated_at`) exist on every table and aren't repeated below.

---

## Privacy building blocks (database functions)

| Function | Returns true when… |
|---|---|
| `is_connected(a, b)` | a and b have an accepted connection |
| `has_friend_permission(owner, friend, perm)` | connected **and** the owner granted `perm` (`view_wardrobe`, `style_for_me`, `request_borrow`) to that friend |
| `can_view_item(item_id)` | caller is the owner, **or** item is `circle` and caller has `view_wardrobe`, **or** item is `selected` and caller is connected and has a share grant for it |
| `item_availability(item_id)` | `available` / `reserved` / `lent_out` / `returning`, calculated from active borrows |

`items`, `item_images` **and** the `wardrobe-images` storage bucket all use `can_view_item`, so photo access always matches item access.

---

## Accounts and profiles

### `profiles` (Phase 1)
One row per user. Created automatically at sign-up.
- Fields: `id` (= auth user id), `display_name`, `avatar_path`, `onboarding_completed_at`
- **See:** yourself; your accepted friends (name + avatar only).
- **Change:** yourself only.
- Deliberately **no** public username and no search, so nobody can be looked up.

### `push_tokens` 🔒 (Phase 9)
- Fields: `user_id`, `expo_push_token`, `platform`
- **See / change:** yourself only. Used by server functions to send notifications.

### `ai_usage_events` 🔒 (Phase 3)
Counts AI calls for quotas and cost tracking.
- Fields: `user_id`, `kind` (`analyze_item`, `analyze_outfit`, `stylist`, …), `units`
- **See:** yourself (so the app can show "3 scans left today"). **Change:** server only.

---

## Wardrobe

### `items` (Phase 2)
The shareable description of a garment, bag, pair of shoes or piece of jewelry.
- Fields: `id`, `owner_id`, `title`, `category`, `subcategory`, `dominant_color`, `secondary_colors[]`, `material`, `pattern`, `silhouette`, `fit`, `brand` (user-entered only), `size`, `style_tags[]`, `season_tags[]`, `is_favorite`, `is_borrowable`, `visibility` (`private` · `circle` · `selected` · `ai_only`), `status` (`active` · `archived`)
- **Default visibility:** `private`
- **See:** anyone for whom `can_view_item` is true.
- **Change:** owner only.
- No "in vault" flag, no prices, no AI confidences. Those live in owner-only tables.

### `item_share_grants` (Phase 2)
"Selected friends" list for an item.
- Fields: `item_id`, `owner_id`, `friend_id`
- **See:** owner; the friend can see grants made to them.
- **Change:** owner only, and only for accepted friends.

### `item_images` (Phase 2)
- Fields: `id`, `item_id`, `owner_id`, `kind` (`crop` · `cutout` · `product_photo` · `ai_representation`), `storage_path`, `width`, `height`, `is_primary`, `source_scan_id`
- `kind` is always shown in the app. An `ai_representation` is never presented as a real photo.
- **See:** same as the item. **Change:** owner only.

### `item_private_details` 🔒 (Phase 2)
- Fields: `item_id`, `owner_id`, `purchase_date`, `purchase_price`, `currency`, `purchase_location`, `condition_notes`, `notes`
- **See / change:** owner only.

### `item_ai_annotations` 🔒 (Phase 3)
What the AI originally proposed, kept so prompts can be improved.
- Fields: `item_id`, `owner_id`, `proposed` (JSON), `confidences` (JSON), `user_corrected_fields[]`, `ai_config_version`
- **See:** owner only. **Change:** server writes; owner can delete.

### `item_embeddings` 🔒 (Phase 3)
Numeric "fingerprint" used for duplicate detection and matching (`pgvector`).
- Fields: `item_id`, `owner_id`, `embedding`, `source` (`text_description` · later `image`), `model_version`
- **See:** owner only. **Change:** server only.

### `item_events` 🔒 (Phase 6)
Logs "same item / different item" decisions to improve duplicate matching.
- Fields: `item_id`, `owner_id`, `event` (`duplicate_confirmed`, `duplicate_rejected`), `other_item_id`
- **See / change:** owner only.

---

## Adding clothes (scans)

### `scans` 🔒 (Phase 3–4, reused in Phase 8)
One photo upload and its processing status.
- Fields: `id`, `owner_id`, `kind` (`single_item` · `outfit` · `inspiration`), `status` (`uploaded` · `processing` · `needs_review` · `confirmed` · `failed`), `source_image_path` (in the owner-only `scan-uploads` bucket), `error_code`
- **See / change:** owner only.

### `scan_detections` 🔒 (Phase 4)
Each garment the AI found in a scan, waiting for your review.
- Fields: `id`, `scan_id`, `owner_id`, `proposed` (JSON attributes), `confidences` (JSON), `bbox`, `crop_path`, `cutout_path`, `duplicate_candidate_item_id`, `duplicate_score`, `decision` (`pending` · `new_item` · `same_as_existing` · `discarded` · `merged`), `resulting_item_id`
- **See:** owner only. **Change:** only through `confirm_scan_detection`.

---

## Outfits and styling

### `outfits` 🔒 for MVP (Phase 4 / 7)
Outfit history (worn looks) and saved boards.
- Fields: `id`, `owner_id`, `kind` (`worn` · `ai_board` · `recreated`), `title`, `occasion`, `worn_on`, `photo_scan_id`, `is_favorite`
- **See / change:** owner only. *(Sharing outfits with friends may come later.)*

### `outfit_items` 🔒 (Phase 4)
- Fields: `outfit_id`, `item_id`, `position`
- **See / change:** owner only.

### `style_boards` (Phase 11)
A look a friend builds from your wardrobe and sends to you.
- Fields: `id`, `wardrobe_owner_id`, `author_id`, `message`, `status` (`sent` · `seen` · `saved` · `dismissed`)
- **See:** the wardrobe owner and the author.
- **Create:** author only, and only with the `style_for_me` permission.
- **Change status:** owner only.
- If the owner later makes an item private, the author sees "item no longer shared" instead of the item.

### `style_board_items` (Phase 11)
- Fields: `board_id`, `item_id`, `position`
- **See:** same as the board, but each item still respects `can_view_item`.

### `stylist_conversations` / `stylist_messages` 🔒 (Phase 7)
- Fields: `id`, `owner_id`, `role` (`user` · `assistant`), `content`, `suggested_outfit` (JSON of item IDs)
- **See / change:** owner only. Auto-deleted after the retention period (decision D6).

### `inspirations` 🔒 / `inspiration_matches` 🔒 (Phase 8)
- Fields (inspirations): `id`, `owner_id`, `scan_id`, `analysis` (JSON), `modifiers` (e.g. "for work", "12°C")
- Fields (matches): `inspiration_id`, `garment_index`, `item_id`, `match_level` (`exact` · `close` · `alternative` · `missing`), `score`, `reason`
- **See / change:** owner only.

---

## Private circle

### `circle_invites` 🔒 (Phase 2 tables, Phase 10 screens)
- Fields: `id`, `inviter_id`, `token_hash` (the link's secret is never stored, only its hash), `expires_at`, `used_at`, `used_by`
- **See:** inviter only.
- **Create:** via `create_invite`. **Accept:** via `accept_invite(token)` only. Invites are single-use and expire after 7 days.

### `connections` (Phase 2 tables, Phase 10 screens)
One row per accepted friendship.
- Fields: `user_a`, `user_b` (stored in a fixed order so a pair appears only once), `status` (`accepted` · `blocked`), `blocked_by`
- **See:** the two people involved.
- **Change:** only via `accept_invite`, `remove_connection`, `block_user`.

### `friend_permissions` (Phase 2 tables, Phase 10 screens)
What one friend may do with *your* things. One direction only.
- Fields: `owner_id`, `friend_id`, `can_view_wardrobe`, `can_style_for_me`, `can_request_borrow`
- **Default:** all false. Nothing is shared just by becoming friends.
- **See:** owner, and the friend it's about. **Change:** owner only.

### `reports` 🔒 (Phase 10)
- Fields: `reporter_id`, `reported_user_id`, `reason`, `details`
- **See:** reporter only. **Create:** reporter only. Reviewed by an admin via the dashboard.

---

## Borrowing

### `borrow_requests` (Phase 2 tables, Phase 12 screens)
- Fields: `id`, `item_id`, `owner_id`, `borrower_id`, `start_date`, `end_date`, `handover_method` (`pickup` · `ship`), `note`, `status`, timestamps per step
- **Lifecycle:** `requested` → `approved` (reserved) **or** `declined` → `lent_out` → `returning` → `returned`. `cancelled` is possible before `lent_out`.
- **Overlap protection:** the database refuses two `approved`/`lent_out`/`returning` borrows of the same item with overlapping dates (exclusion constraint).
- **See:** owner and borrower only.
- **Create:** via `request_borrow`, which checks the friend has `request_borrow` permission, can view the item, and the item is borrowable.
- **Change:** only via the transition functions. Each one checks *who* is acting and *what state* the borrow is in.

### `borrow_events` (Phase 12)
History of every status change.
- Fields: `borrow_id`, `actor_id`, `from_status`, `to_status`
- **See:** owner and borrower. **Change:** nobody (written by the transition functions).

### `borrow_shipping_details` 🔒+ (Phase 12)
The address for one "please send it" borrow.
- Fields: `borrow_id`, `recipient_id`, `name`, `address_lines`, `postal_code`, `city`, `country`, `tracking_number`, `erase_after`
- **See:** only the two people in that borrow, and only while it's active.
- **Change:** recipient edits the address; the sender adds the tracking number.
- **Erased** automatically 14 days after `returned` / `cancelled` / `declined`.

---

## Vault (Phase 2 skeleton, Phase 13 screens)

### `vault_records` 🔒
Marks an item as a Vault item and holds its private asset details.
- Fields: `item_id`, `owner_id`, `exact_model`, `reference_number`, `serial_number`, `purchase_date`, `retail_price`, `price_paid`, `currency`, `purchase_location`, `condition`, `completeness` (box, dust bag, papers), `insurance_notes`, `private_notes`
- **See / change:** owner only. Friends who can see the item can't tell it's in the Vault.

### `vault_documents` 🔒
- Fields: `id`, `item_id`, `owner_id`, `kind` (`receipt` · `authenticity` · `certificate` · `insurance` · `other`), `storage_path` (in the owner-only `vault-documents` bucket)
- **See / change:** owner only.

### `valuations` 🔒 (later)
- Fields: `item_id`, `owner_id`, `low`, `high`, `currency`, `confidence` (`low` · `medium` · `high`), `valued_at`, `source_provider`, `source_metadata` (JSON, which comparables were used)
- **See:** owner only. **Change:** server only. Never created without real market data.

---

## Subscriptions (Phase 14)

### `entitlements` 🔒
- Fields: `user_id`, `plan` (`free` · `premium` · `premium_plus`), `source` (`revenuecat`), `expires_at`
- **See:** yourself. **Change:** server only (RevenueCat webhook).

---

## Storage buckets

| Bucket | Path pattern | Who can read | Who can upload |
|---|---|---|---|
| `wardrobe-images` | `{owner_id}/{item_id}/{image_id}.jpg` | `can_view_item` for the matching `item_images` row | Owner, into own folder |
| `scan-uploads` | `{owner_id}/{scan_id}/…` | Owner | Owner, into own folder |
| `vault-documents` | `{owner_id}/{item_id}/…` | Owner | Owner |
| `avatars` | `{user_id}/avatar.jpg` | Owner + accepted friends | Owner |
| `exports` | `{user_id}/{export_id}.zip` | Owner (deleted after 24 h) | Server only |

All buckets are private and served through signed links that expire after ~30 minutes.

---

## Privacy tests (must pass before a phase is complete)

For every table above, automated tests (`supabase/tests/database/`) check that:
1. A **stranger** (logged in, not connected) sees nothing.
2. A **friend without permission** sees nothing.
3. A **friend with permission** sees only what's allowed, and never 🔒 tables.
4. A **"selected" item** is visible only to the chosen friends.
5. A **removed or blocked friend** immediately loses access.
6. **Not logged in** (`anon`) gets nothing at all.
7. **Storage** follows the same rules as the tables.
