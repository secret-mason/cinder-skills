Publish a site in three steps. No account needed:

1. Create site


curl -sS https://here.now/api/v1/publish \
  -H "X-HereNow-Client: cursor/direct-api" \
  -H "content-type: application/json" \
  -d '{
    "files": [
      { "path": "index.html", "size": 1234, "contentType": "text/html; charset=utf-8" }
    ]
  }'
2. Upload file


curl -X PUT "<upload.uploads[0].url>" \
  -H "Content-Type: text/html; charset=utf-8" \
  --data-binary @index.html
3. Finalize


curl -sS -X POST "<finalizeUrl>" \
  -H "X-HereNow-Client: cursor/direct-api" \
  -H "content-type: application/json" \
  -d '{ "versionId": "<versionId>" }'
The response to step 1 includes the live siteUrl, presigned upload URLs, and a finalizeUrl. Anonymous sites expire in 24 hours. Only anonymous sites include claimUrl; share it with the user, copied byte-for-byte, so they can keep the site permanently. Send X-HereNow-Client on every here.now call (create, update, finalize), not only the first: it is what shows "Published with <your agent>" in the owner's dashboard. Step 2 goes straight to storage and needs no header.

Authentication
Two modes:

Authenticated: include Authorization: Bearer <API_KEY>. Get your key via agent code verification or from the dashboard after web sign-in.
Anonymous: omit the header. Sites expire in 24 hours with lower limits.
Identify your agent: always include X-HereNow-Client: <harness>/<detail> naming the agent platform you run in (example: claude-code/publish-sh, or grok-bot/research-bot for a custom bot inside Grok Bot). Use the platform's name, not your bot or persona name — a "finance-bot" running in Cursor sends cursor/finance-bot, not finance-bot. Send it on every here.now request, including finalize: each published version records who published it, and the dashboard's agent badge reads from the version. If finalize arrives without it, the token from the create/update call that staged the version is used. Missing or invalid values are ignored.
Getting an API key
There are two ways to get an API key:

Option A: Agent-assisted sign-up. The sign-up flow can be completed entirely within the agent. Request a one-time code by email:


curl -sS https://here.now/api/auth/agent/request-code \
  -H "content-type: application/json" \
  -d '{"email": "user@example.com"}'
The user copies the code from email and pastes it into the agent. Then verify the code to receive the API key:


curl -sS https://here.now/api/auth/agent/verify-code \
  -H "content-type: application/json" \
  -d '{"email":"user@example.com","code":"ABCD-2345","keyName":"my-tool"}'
If the email is new, the account is created automatically.

keyName is optional but recommended: when provided, the returned apiKey is a named key for that tool — your existing key with that name if one is active, or a freshly minted one (one key per agent or tool — see API keys) — instead of the account's shared legacy key. Repeat logins return the same key (keyReused: true). If key issuance fails (key cap, rate limit, or an internal error), the response falls back to the legacy key and sets keyMintError.

Option B: Dashboard sign-up. Sign in at here.now/dashboard and copy the API key from the dashboard.

Changing the account email
The email address is the account: signing in with a different address, by code or by "Continue with Google", creates a separate empty account rather than attaching to an existing one. If a user says they have a new email, move the account instead of requesting a sign-in code for the new address. Start the change with the account's API key:


curl -sS https://here.now/api/v1/me/email-change \
  -H "Authorization: Bearer <API_KEY>" \
  -H "content-type: application/json" \
  -d '{"newEmail": "new@example.com"}'
# 202 {"status":"pending","newEmail":"new@example.com","expiresAt":"…","holdUntil":null,"warnings":[],"message":"Check new@example.com for a confirmation code…"}
A confirmation link and a 6-digit code go to the new address only; the current address gets a notice with a cancel link. The user reads the code from the new mailbox and gives it to you:


curl -sS https://here.now/api/v1/me/email-change/confirm \
  -H "Authorization: Bearer <API_KEY>" \
  -H "content-type: application/json" \
  -d '{"code": "123-456"}'
# 200 {"email":"new@example.com","previousEmail":"old@example.com"}
Sites, API keys (including the one you used), and current sign-ins carry over unchanged. The old address receives a notice with a link to undo the change for 7 days. The code works once and expires in 30 minutes; five wrong codes cancel the request. One request is active per account (a new one replaces the old); GET the same path to see it and DELETE it to cancel. On accounts with a Stripe customer or a workspace-admin role, a change started with an API key cannot be confirmed for 24 hours (holdUntil in the response; confirming early returns 409 email_change_on_hold and spends no attempt), so a leaked key alone cannot move the account before the owner is told. A target that already has a here.now account with content is refused with 409 email_already_in_use; a here.now or role address, a non-ASCII address, or the current address returns 400. Owners can do the same from the dashboard under Settings → General → Change next to Email.

Storing the API key
After obtaining a key (via either method), save it to the credentials file:


mkdir -p ~/.herenow && echo "<API_KEY>" > ~/.herenow/credentials && chmod 600 ~/.herenow/credentials
The publish script reads the key from these sources (first match wins):

--api-key flag (CI/scripting only — avoid in interactive use)
$HERENOW_API_KEY environment variable
~/.herenow/credentials file (recommended)
API keys
An account can hold up to 50 named API keys, each individually revocable. Use one key per agent or tool (e.g. claude, cursor) so revoking one credential doesn't break the others. Named keys start with hnk_ and work everywhere Authorization: Bearer <API_KEY> is accepted. The original key from sign-up appears in the key list with a blank name and id legacy.

Keys can be managed from the dashboard or via the API.

Create a key
POST /api/v1/me/keys


curl -sS https://here.now/api/v1/me/keys \
  -H "Authorization: Bearer <API_KEY>" \
  -H "content-type: application/json" \
  -d '{"name": "claude"}'
Response:


{
  "id": "9f4e...",
  "name": "claude",
  "key": "hnk_<64 hex chars>",
  "keySuffix": "a1b2",
  "createdAt": "2026-06-11T00:00:00.000Z"
}
List keys
GET /api/v1/me/keys


curl -sS https://here.now/api/v1/me/keys \
  -H "Authorization: Bearer <API_KEY>"
Returns every active key including its full value (id, name, key, keySuffix, createdAt, lastUsedAt, current), so keys can be re-read later. current marks the key used to authenticate the request.

Revoke a key
DELETE /api/v1/me/keys/:id


curl -sS -X DELETE https://here.now/api/v1/me/keys/<KEY_ID> \
  -H "Authorization: Bearer <API_KEY>"
Revocation is immediate and only affects that key. Use DELETE /api/v1/me/keys/legacy to revoke the original key from sign-up.

API keys belong to you, not to an account: the same key publishes to your personal account by default and into any workspace you belong to when the request carries an account selector — see Workspaces → Account selector.
