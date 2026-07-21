# 📧 Gmail — Complete Email Management

Integration with Gmail through the official Google API. Lets you send, read,
reply to, forward emails and check for new messages.
Includes a **real-time monitoring daemon** (watch) that automatically
notifies every new email in the INBOX.

---

## ⚙️ Configuration

You have **two ways** to configure this skill:

### 🤖 Guided mode (recommended) — via chat

Simply tell the assistant:

> **"Connect Gmail"** or **"Configure the Gmail skill"**

The assistant will guide you step by step: it will generate a link to open in the browser,
you will authorize access to your Gmail account and paste the resulting URL into the chat.

The configuration is saved **automatically and securely** directly
in the backend — OAuth tokens never appear in the chat history.
No copy-pasting of credentials required.

### ✏️ Manual mode — direct entry

If you prefer to fill in the fields yourself, go to
**Settings → Skills → gmail → Configure** and fill in:

| Variable | Where to find it |
|-----------|--------------|
| `GMAIL_CLIENT_ID` | Google Cloud Console → Credentials → OAuth client ID |
| `GMAIL_CLIENT_SECRET` | Google Cloud Console → Credentials → OAuth client ID |
| `GMAIL_REFRESH_TOKEN` | [OAuth2 Playground](https://developers.google.com/oauthplayground) (see below) |
| `GMAIL_USER_EMAIL` | Your Gmail address (e.g. `mario@gmail.com`) |

<details>
<summary>📖 How to obtain the credentials manually</summary>

**1. Create the Google Cloud project**
1. Go to [console.cloud.google.com](https://console.cloud.google.com)
2. New project → give it a name (e.g. `Arkimede Gmail`)
3. **APIs & Services → Library** → search for **Gmail API** → Enable
4. **APIs & Services → Credentials** → **Create OAuth client ID** → type **Desktop app**
5. Copy **Client ID** and **Client Secret**

**2. Add yourself as a test user**
1. **APIs & Services → OAuth consent screen**
2. **"Test users"** section → add your Gmail address

**3. Get the Refresh Token**
1. Go to [OAuth2 Playground](https://developers.google.com/oauthplayground)
2. ⚙️ → check *"Use your own OAuth credentials"* → enter Client ID and Secret
3. Select scope: `https://mail.google.com/`
4. **Authorize APIs** → sign in with Gmail → **Exchange authorization code for tokens**
5. Copy the **Refresh token**

</details>

---

## ⏱️ Token lifetime

| Google app status | Refresh token lifetime |
|-----------------|---------------------|
| **Testing** (default) | 7 days — then it must be reconnected |
| **Production** | Unlimited (expires only after 6 months of inactivity) |

To switch to production: **OAuth consent screen → Publish app**.

To reconnect an expired token, simply say **"Reconnect Gmail"** in chat.

---

## ✅ Available features

| Action | How to ask for it |
|--------|----------------|
| Connect / reconfigure | `"Connect Gmail"`, `"Configure Gmail"` |
| Send email | `"Send an email to X"`, `"Send a message to..."` |
| Send email with attachment | `"Send to X attaching file Y"` |
| Show mail | `"Show my emails"`, `"Do I have email?"`, `"Check my mail"` |
| Read a message | `"Read the email from X"`, `"Open the message..."` |
| Reply | `"Reply to X"`, `"Send a reply to..."` |
| Reply with attachment | `"Reply attaching the report"` |
| Forward | `"Forward to X"`, `"Pass this email along to..."` |
| Forward with attachment | `"Forward to X and also attach the PDF"` |
| New emails (manual) | `"Did I receive anything?"`, `"Are there any new emails?"` |
| **Automatic monitoring** | Start the daemon — see section below |

---

## 🔔 Real-time monitoring (Daemon / Watch)

The `daemon_emails.py` daemon runs in the background and sends a **push event**
every time a new email arrives in the INBOX — without needing to query
the mailbox manually.

### How it works

The daemon uses the **Gmail History API**: on each cycle it downloads only the changes
since the last check, without re-downloading the entire mailbox.

The polling interval is configurable through the `GMAIL_POLL_INTERVAL` variable
(settable in **Settings → Skills → Gmail → Configure**, default: **30 seconds**).
Recommended values:
- `15` — more responsive, more API calls
- `30` — balanced (default)
- `120` — fewer API calls, less responsive

### Emitted event

When new emails arrive the daemon sends a `new_emails` event with:
- the number of emails received
- for each one: id, subject, sender, date, preview

How these events are presented depends on the application hosting the skill.

### Errors and reconnection

| Emitted event | Cause | Solution |
|---------------|-------|-----------|
| `auth_error` | OAuth token expired or revoked | Say `"Reconnect Gmail"` in chat, then restart the daemon |
| `daemon_exit` | Unrecoverable critical error | Check the daemon logs and restart |

> **Note:** A Google app in *Testing* mode generates tokens that expire after **7 days**.
> If you use the daemon continuously, switch the app to *Production* mode in
> Google Cloud Console → OAuth consent screen → Publish app.

---

## Dependencies

- `google-api-python-client>=2.100`
- `google-auth>=2.20`
- `google-auth-httplib2>=0.2`
- `httplib2>=0.22`

## Version

`1.3.0` — May 2026 · MIT License · Author: info@rstonline.it  
_New in v1.3.0: config var `GMAIL_POLL_INTERVAL` to customize the daemon polling interval (default: 30s); the value is read first from `_config`, then from the environment variable._  
_New in v1.2.0: outgoing attachment support on `send_email`, `reply_email`, `forward_email` (absolute paths, auto-detected MIME type)._  
_New in v1.1.0: real-time monitoring daemon (watch via Gmail History API), persistent notifications in the DB._

## Network

This skill makes calls to external domains. They are declared in `skill.yaml` (`network:` field) and visible to the admin during review; with the **egress** overlay, domains not listed are blocked at the network level.

| Domain | Use |
|---|---|
| `accounts.google.com` | OAuth2 — authorization page |
| `oauth2.googleapis.com` | OAuth2 — token exchange and refresh |
| `www.googleapis.com` | Gmail API (send/list/read/reply/forward/history) |

> Google OAuth2 (authorization, token exchange/refresh) and Gmail API to send, list, read, reply to and forward emails.
