# 💬 Telegram — the agent on Telegram, as an account and as a bot

One skill for Telegram (it replaces `telegram-bot` and `telegram-account`). The agent uses a
**Telegram account** — **its own** (a number of its own) or **yours** — through Telegram's user
API (MTProto, via [Telethon](https://docs.telethon.dev)), and optionally a **bot**:

- it writes to **any contact of the account** (or an allowlist), also people who never wrote to
  it first, reads conversations and withdraws a message;
- the **listener** daemon hands the contacts' messages to the agent, which **answers them**;
- Arkimede delivers what the memory (Recordare) decided to say — a timed reminder, a request, a
  promise — straight through it, with no model in between (`user-channel`);
- the optional **bot** lets every Arkimede user chat with the agent from Telegram, and makes a
  notification to the user themself ring.

`SKILL.md` (inside the zip) is the full reference; this page adds the notes for Arkimede.

| Version | Changes |
|---|---|
| 1.0.0 | First release, from `telegram-account` 1.0.1 and `telegram-bot` 1.2.1: two modes (`personal`, `dedicated`), the listener daemon (`listen.py`: contacts' messages to the agent and back; the scripts run on its connection), the bot daemon (`bot.js`), `send_message` without `to` for the user themself (through the bot it rings), the host's `user-channel` (`_config.DELIVERY_ADDRESS`) |

---

## Two modes (`ACCOUNT_MODE`)

| Mode | The account is | Messages the agent gets |
|---|---|---|
| `personal` (default) | yours: the agent writes as you | only replies to what the agent wrote (a reply to its message, or within `REPLY_WINDOW_HOURS`, 24, of its last one to that person) — everything else stays yours |
| `dedicated` | the agent's own (its own number) | every contact's private message |

Never handed over: groups, channels, bots, people outside the account's contacts; with
`ALLOWED_RECIPIENTS` set, only those people.

**What the agent may do for them.** A contact gets **answers only**: no action — nothing at home,
no message, no reminder, no tool — and your memories stay private. In `dedicated` mode the
`AUTHORIZED_CONTACTS` (e.g. `@maria_rossi, +393331234567`) may have the agent do anything, as its owner.
In `personal` mode no contact is ever authorized: you talk to the agent with full powers through
the **bot**.

The turn runs as the Arkimede account that owns the daemon: in `dedicated` mode install and start
the skill on the agent's own account (e.g. the account voice runs as), in `personal` mode on yours.
Each person gets a chat of their own in that account, and the memory knows who is speaking from
their Telegram id — bind it once as their `telegram` address on their card in Recordare's console
(the same address the reminders are delivered to).

---

## Install and set up in Arkimede

1. **Install** from *Settings → Public Skills*, on the account that will own the conversations
   (above), and keep it **personal**.
2. **Network**: nothing to do. MTProto connects to Telegram's data centres by IP address, not by
   name: the skill declares their published ranges (<https://core.telegram.org/resources/cidr.txt>)
   next to `api.telegram.org` (the bot), and goes through the host's egress proxy (Arkimede with
   IP ranges in `network:`). On an older Arkimede, an admin sets the skill's network to **open**
   (open the skill → *Network access*).
3. **Configure**:
   - `ACCOUNT_MODE`: `personal` or `dedicated`;
   - `TELEGRAM_API_ID`, `TELEGRAM_API_HASH`: from <https://my.telegram.org> → *API development
     tools* (create an application, any name);
   - `ALLOWED_RECIPIENTS` (optional): e.g. `@maria_rossi, +393331234567`; empty = any contact;
   - optional: `AUTHORIZED_CONTACTS` (dedicated mode), `REPLY_WINDOW_HOURS`, `MESSAGE_SUFFIX`, `SEND_MIN_INTERVAL`, `TELEGRAM_BOT_TOKEN`.
4. **Log in** from the skill page with the script runner (not exposed to the AI): `login_start`
   with the phone number, then `login_finish` with the code Telegram sent (and the 2FA password,
   if any). `status` confirms it.
5. **Start the listener** (`listen.py`) from *Settings → Skills → Background*.
6. **Bot** (optional): set `TELEGRAM_BOT_TOKEN`, start `bot.js`; each user sends
   `/login email password` to the bot.
7. **Reminders by message**: in Recordare's console give the client the channel `messaging`
   and bind each person's `telegram` address (their numeric Telegram id) on their card.

The session is saved in the skill's persistent state directory (`SKILL_STATE_DIR`), mode 600,
shared by the scripts and the listener; while the listener runs, the scripts go through its
connection (a session connected twice can be revoked by Telegram). Without a persistent state
directory, put a Telethon session string in `TELEGRAM_SESSION` (stored encrypted).

To unlink: run `logout`, or end the "telegram skill" session from Telegram's *Settings → Devices*.

**Migrating** from `telegram-account`: same configuration keys; log in again (a new state
directory). From `telegram-bot`: put the same `TELEGRAM_BOT_TOKEN`; users `/login` again.

---

## Tests

The zip contains offline tests with a fake Telegram client:

```bash
pip install "telethon>=1.36,<2"
python3 -m unittest discover tests
```
