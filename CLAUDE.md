# CLAUDE.md — simple_digital_calendly

## What this folder is
A setup/ops workspace (not an app) for wiring MAG's Calendly to Simple Digital's
third-party **MarketingIQ** attribution tool via a Calendly webhook. Contents:
`credentials.txt`, the reference email PDF, a short note on MAG's own edge
functions, and `chat_history/`. The one real action taken here was registering
the Calendly webhook against the API.

## Key facts
- **Calendly account:** James Castrission — user `DDCBR44S6ZIKC53U`, org
  `HAECQY77EPHMW3ZX`, `calendly.com/myadventuregroup`, tz Australia/Sydney.
- **MarketingIQ callback:** `https://app.marketingiq.com.au/api/webhooks/calendly/cmu6g36zq0000ssi8f4ybnw52`
  (trailing id = Simple Digital's Project ID). Payloads signed with the shared
  `CALENDLY_WEBHOOK_SIGNING_KEY`.
- **Registered webhook:** subscription `549095f0-d123-4b3a-8206-8116f613efee`,
  ACTIVE, event `invitee.created`, scope `user` (MarketingIQ filters to Discovery
  Call on their side).

## Gotchas learned this session
- **Calendly webhooks are API-only.** There is NO Calendly UI to create a webhook
  subscription — only `POST https://api.calendly.com/webhook_subscriptions`. The
  "API & webhooks" settings page just mints PATs. (The on-form "Reference code /
  Tracking ID" question IS a UI thing — don't confuse the two.)
- **PAT scopes matter and the errors are specific:**
  - Creating the subscription needs `webhooks:write` (a read-only PAT → silent
    dead end / no write).
  - Subscribing to `invitee.created` ALSO needs `scheduled_events:read`
    (HTTP 400 "missing required scope: scheduled_events:read" otherwise). That
    scope is under the **Scheduling** group on the token-creation screen.
  - So a working registration PAT = Webhooks(write) + Scheduling
    (scheduled_events:read) + User management (users:read, organizations:read).
- **Claude Code's auto-mode classifier blocks the POST** (treats it as an
  "Account & Standing-Rule Changes" action). Can't run it from a tool call —
  hand the user the command to run via the in-session `! ` prefix, or they add a
  Bash permission rule.
- Decode a Calendly PAT's scopes without calling the API:
  `echo "$PAT" | cut -d. -f2 | tr '_-' '/+' | base64 -d`.

## Secrets / safety
- `credentials.txt` holds TWO live Calendly PATs + the signing key in PLAINTEXT.
  It is gitignored. The write PAT is one-time setup — rotate/delete after use.
- Canonical signing key lives in Google Secret Manager
  (`CALENDLY_WEBHOOK_SIGNING_KEY`, project `claudegwscli502400`).
- Never write any token/key value into committed files (as_built.txt, CLAUDE.md,
  chat history) — redact.

## Register / inspect the webhook
```bash
# values pulled from credentials.txt; run with the "! " prefix (classifier blocks tool-run POSTs)
PAT=$(grep '^CALENDLY_PAT_WEBHOOK_WRITE=' credentials.txt | sed 's/^[^=]*=//')
SIGNING_KEY=$(grep '^CALENDLY_WEBHOOK_SIGNING_KEY=' credentials.txt | sed 's/^[^=]*=//')
curl -s -X POST https://api.calendly.com/webhook_subscriptions \
  -H "Authorization: Bearer $PAT" -H "Content-Type: application/json" \
  -d "{\"url\":\"https://app.marketingiq.com.au/api/webhooks/calendly/cmu6g36zq0000ssi8f4ybnw52\",\"events\":[\"invitee.created\"],\"organization\":\"https://api.calendly.com/organizations/HAECQY77EPHMW3ZX\",\"user\":\"https://api.calendly.com/users/DDCBR44S6ZIKC53U\",\"scope\":\"user\",\"signing_key\":\"$SIGNING_KEY\"}"

# list existing subscriptions (GET, read-only):
curl -s "https://api.calendly.com/webhook_subscriptions?organization=https://api.calendly.com/organizations/HAECQY77EPHMW3ZX&scope=user&user=https://api.calendly.com/users/DDCBR44S6ZIKC53U" \
  -H "Authorization: Bearer $PAT"
```

## MAG's own parallel path (context, lives in Supabase, not here)
Project `aivitcomiywiysrfwqxt` (MAGTestProject): edge functions
`calendly-source-sync` (cron, Calendly→ActiveCampaign source write) and
`new-paid-leads-sync`. Independent of MarketingIQ; both kept by agreement.
See `mag_calendly_ac_sync_edge_functions.txt` and `as_built.txt`.
