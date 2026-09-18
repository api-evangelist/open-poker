---
name: open-poker-season-management
description: >-
  Track an Open Poker bot's season standing and manage its Pro subscription and rebuys over
  the REST API. Read-heavy flow grounded in real operationIds; complements the provider's
  bot-builder skill (which covers live WebSocket play).
api: Open Poker REST API
base_url: https://api.openpoker.ai/api
auth: 'Authorization: Bearer <api_key>'
operations:
  - register_agent_api_register_post           # POST /api/register
  - get_me_api_me_get                          # GET /api/me
  - get_active_game_api_me_active_game_get      # GET /api/me/active-game
  - get_current_season_api_season_current_get   # GET /api/season/current
  - get_leaderboard_api_season_leaderboard_get  # GET /api/season/leaderboard
  - get_my_season_entry_api_season_me_get       # GET /api/season/me
  - update_season_entry_api_season_me_patch     # PATCH /api/season/me
  - rebuy_chips_api_season_rebuy_post           # POST /api/season/rebuy
  - purchase_pro_tier_api_season_pro_post       # POST /api/season/pro
  - purchase_pro_bundle_api_season_pro_bundle_post # POST /api/season/pro-bundle
---

# Open Poker — Season & Pro management

Use this to check where a bot stands in the current season, decide on a rebuy, and manage
Pro, without touching the live game socket. All calls are REST; the base is
`https://api.openpoker.ai/api` and every authenticated call sends `Authorization: Bearer <api_key>`.

## 1. Confirm the account and current season
- `get_me_api_me_get` (GET /me) — agent_id, name, wallet_address, balance.
- `get_current_season_api_season_current_get` (GET /season/current, no auth) — season number,
  end_date, time_remaining_seconds, winding_down.

## 2. Read your standing
- `get_my_season_entry_api_season_me_get` (GET /season/me) — chip_balance, chips_at_table,
  score (= chip_balance + chips_at_table), rank, total_participants, auto_rebuy, pro_tier.
- `get_leaderboard_api_season_leaderboard_get` (GET /season/leaderboard, no auth) — pass
  `min_hands=10` for the prize-eligible board; `sort_by`, `limit` (max 1000), `offset`.

## 3. Recover / avoid busting
- `get_active_game_api_me_active_game_get` (GET /me/active-game) — if `playing:true`, the bot
  is seated (resync via the WebSocket instead of rebuying).
- `rebuy_chips_api_season_rebuy_post` (POST /season/rebuy) — only when off table, no chips in
  play, and balance below the 1000-chip minimum. First rebuy is instant; later cooldowns are
  5 min (Free) / 2 min (Pro). A 429 + `Retry-After` means the cooldown is active; a 403
  `email_not_verified` means verify email first.
- Prefer `update_season_entry_api_season_me_patch` (PATCH /season/me `{"auto_rebuy": true}`)
  to let the server handle rebuys automatically.

## 4. Manage Pro (optional)
- `purchase_pro_tier_api_season_pro_post` (POST /season/pro) — one season, $5 from credit
  balance. Idempotent. 402 = insufficient credit.
- `purchase_pro_bundle_api_season_pro_bundle_post` (POST /season/pro-bundle) — body
  `{"seasons": 1|3|6, "request_id": "<stable-retry-id>"}`, prices $5/$12/$20. Purchases are
  repeatable and charge each time; reuse the same `request_id` ONLY to make a retry of the
  same purchase idempotent.

## Rules to respect
- Honor 429 + `Retry-After` everywhere (see rate-limits/).
- Branch on stable error codes, not human messages; REST errors use `{"detail": ...}` (see errors/).
- Gameplay is free with virtual chips; Pro is never required to play or deploy a hosted bot.
