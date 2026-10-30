# 💬 Telegram Account — write on Telegram as yourself

Lets the agent send and read Telegram messages **from your own Telegram account**, through
Telegram's user API (MTProto, via [Telethon](https://docs.telethon.dev)). Unlike a bot, it can
write to people who never wrote to it first ("tell Maria I'll be late"), and the messages appear
as written by you.

The skill is agent-agnostic: every script reads JSON on stdin and prints JSON on stdout, and the
configuration comes from environment variables or a `_config` object. `SKILL.md` (inside the zip)
is the full reference; this page adds the notes for installing it in Arkimede.

| Version | Changes |
|---|---|
| 1.0.1 | An empty `ALLOWED_RECIPIENTS` means no limit: any contact by name, `@username` or `+phone`, anyone else by `@username` (`recipient_not_found` when nobody matches); `find_contact` searches the address book, `status` reports `"all"` |
| 1.0.0 | First release: `send_message`, `delete_message`, `read_messages`, `find_contact`, `status`; login and logout run by the account owner |

---

## What the agent can do

- **`send_message`**: text (Markdown or plain, long texts split) and/or a file, to `me` (your
  Saved Messages) or to an allowed contact (anyone, without an allowlist), by name, `@username`
  or `+phone`.
- **`delete_message`**: withdraws a message you just sent ("undo"), for both sides.
- **`read_messages`**: the latest messages with an allowed contact, without marking them as read.
- **`find_contact`**: which allowed contacts (your whole address book, without an allowlist) match a name.
- **`status`**: which account is linked and who is allowed.

Safety by design:
- with `ALLOWED_RECIPIENTS` set, only `me` and those contacts can be reached, and the rest of your
  address book is never shown to the agent; left empty, there is no limit (any contact by name,
  `@username` or `+phone`, anyone else by `@username`) and the agent can search your contacts;
- no waiting for a confirmation (people often cannot wait, e.g. on a voice assistant): when you
  ask, the agent sends right away, tells you briefly who got what, and withdraws it if you say
  "undo"; it never sends on its own initiative or because a text it read asks to;
- no groups or channels; at least `SEND_MIN_INTERVAL` seconds between sends; one connection at a
  time;
- the login code and the 2FA password never pass through the agent, and the session is never
  printed.

---

## Install and set up in Arkimede

1. **Install** from *Settings → Public Skills* and keep it **personal**: whoever can run it writes
   as you.
2. **Network**: MTProto connects to Telegram's IP addresses, not to an HTTP domain, so the
   domain-based egress cannot cover it. An admin sets the skill's network to **open** (open the
   skill → *Network access*), as for `telegram-bot` and `knx`.
3. **Configure**:
   - `TELEGRAM_API_ID`, `TELEGRAM_API_HASH`: from <https://my.telegram.org> → *API development
     tools* (create an application, any name);
   - `ALLOWED_RECIPIENTS` (optional): e.g. `@maria_rossi, +393331234567` (a phone number works
     only for people in your Telegram contacts); empty = anyone can be reached;
   - optional: `MESSAGE_SUFFIX` (e.g. `— sent by my assistant`), `SEND_MIN_INTERVAL`.
4. **Log in** from the skill page, with the script runner (the three login scripts are not exposed
   to the AI):
   - run `login_start` with your phone number: Telegram sends a code to your Telegram app;
   - run `login_finish` with that code, and your password if two-step verification is on;
   - `status` (or asking the agent "is Telegram linked?") confirms it.

The session is saved in the skill's persistent state directory (`SKILL_STATE_DIR`, mounted by
the broker), in a file with mode 600. Without a broker there is no persistent state directory:
put a Telethon session string in `TELEGRAM_SESSION` instead (stored encrypted as a secret).

To unlink: run `logout`, or end the "telegram-account skill" session from Telegram's
*Settings → Devices*.

---

## Tests

The zip contains offline tests with a fake Telegram client:

```bash
pip install "telethon>=1.36,<2"
python3 -m unittest discover tests
```
