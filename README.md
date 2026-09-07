# uiautomator2 Automation Script

A Flask API that drives real Android apps (Grab, Gojek, Zig, Tada, Ryde, Deliveroo, Foodpanda, WhatsApp) through UI automation instead of any official API. It runs **on** an Android device (via Termux), controls the on-device apps with [uiautomator2](https://github.com/openatx/uiautomator2), and exposes the results over HTTP so an external orchestrator (e.g. an [n8n](https://n8n.io) workflow or chatbot) can book rides, order food, check prices, and send WhatsApp messages on the user's behalf.

## Why this exists

Grab, Gojek, Zig, Tada, and Ryde don't expose public consumer APIs for booking rides or ordering food. This project automates their real Android apps by reading the UI hierarchy (`uiautomator2`) and simulating taps/text input, so a booking agent can act on a user's actual logged-in accounts without any of those platforms offering an integration.

## How it works

1. `server.py` runs a Flask server directly on the Android phone (under Termux).
2. Each request connects to the local device with `uiautomator2` (`u2.connect()`), reads the screen's accessibility tree, and drives the target app (tap buttons, type text, wait for screens) to perform an action.
3. Multi-step actions (booking a ride, ordering food, sending a WhatsApp message) are modeled as **finite-state-machine flows** — each HTTP call advances one step and returns the next state, so a caller can drive the whole flow one request at a time.
4. A Cloudflare Tunnel (`start-tunnel.sh`) exposes the local Flask server to the internet so a remote backend can reach the phone.

```
Remote orchestrator (n8n / bot)
        │  HTTPS (Cloudflare Tunnel)
        ▼
   Flask server (server.py) ── runs on the Android device (Termux)
        │  uiautomator2
        ▼
   Grab / Gojek / Zig / Tada / Ryde / Deliveroo / Foodpanda / WhatsApp
   (real apps, driven by simulated UI taps)
```

## Supported apps & actions

| App | Check price | Book / order | Cancel | Login (OTP) |
|---|---|---|---|---|
| Grab | ✅ transport + food | ✅ | ✅ | ✅ |
| Gojek | ✅ | ✅ | ✅ | ✅ |
| Zig | ✅ | ✅ | ✅ | ✅ |
| Tada | ✅ | ✅ | ✅ | — |
| Ryde | ✅ | ✅ | ✅ | ✅ |
| Deliveroo | ✅ (price check only) | — | — | — |
| Foodpanda | ✅ (price check only) | — | — | — |
| WhatsApp | — | send message, add/choose contact | — | ✅ (QR) |

Each app's automation lives in its own package (`Grab/`, `gojek/`, `zig/`, `tada/`, `ryde/`, `deliveroo/`, `foodpanda/`, `whatsapp/`), with a `login.py`, `check_price.py` / `confirmation_check.py`, `book_ride.py` / `food.py`, and `cancel_ride.py` per app, plus a shared `utils.py` for popup dismissal, permission handling, and element lookup helpers.

## API

All routes except `/health_check` and `/register-tunnel` require an `X-Signature` header — an HMAC-SHA256 of the raw request body, signed with a shared secret stored on-device at `~/.secret/secret.key` (see `verify_hmac` in [server.py](server.py)).

### Health & lifecycle
- `GET /health_check` — liveness check.
- `POST /register-tunnel` — provisions and starts the Cloudflare Tunnel (`start-tunnel.sh`) and a health-check cron job.
- `POST /stop-tunnel` — stops the tunnel, removes the cron job and the local secret.
- `POST /update` — triggers the on-device update script.

### Multi-step flows (recommended)
- `POST /transport/flow` — drives the full ride-booking FSM: `start → awaiting_missing_info → login → login_otp_pending → confirmation_check_pending → awaiting_user_confirmation → booking_in_progress → handle_waiting_time → cancel_and_restart → done`. Works across Grab, Gojek, Zig, Tada, and Ryde; when no `app` is given it queries all of them and returns combined price options.
- `POST /food/flow` — same FSM pattern for food ordering (currently Grab).
- `POST /messaging/flow` — WhatsApp flow: login/QR, choosing among ambiguous contacts, sending the message.

### Per-app single actions
- `POST /grab`, `POST /gojek`, `POST /zig`, `POST /ryde`, `POST /foodpanda` — take `{"action": ..., "args": {...}}` for one-off actions (`book_ride`, `order_food`, `confirmation_check`, `cancel_ride`) without going through the flow state machine.
- `POST /all_transport_app` — fan-out price check (`confirmation_check`) across Grab, Gojek, Zig, and Ryde in one call.

## Requirements

- A rooted or debug-enabled Android device with **Termux** and **ADB** access, with Grab/Gojek/Zig/Tada/Ryde/Deliveroo/Foodpanda/WhatsApp installed and logged in (or reachable for login).
- Python 3 with:
  ```bash
  pip install -r requirements.txt
  ```
- [`uiautomator2` init'd on the device](https://github.com/openatx/uiautomator2#installation) (`python -m uiautomator2 init`) so the ATX agent is running.
- `cloudflared` on-device if you want to expose the server via `start-tunnel.sh`.

## Running locally (on-device)

```bash
python server.py
```

Flask listens on the default port; pair it with `start-tunnel.sh <tunnel-name> <user-id> <secret>` to expose it publicly through Cloudflare, or call it over `adb forward` / local network for testing without a tunnel.

## Project layout

```
server.py         # Flask app: routes, HMAC auth, flow state machines
server_dev.py      # dev/experimental server variant
server_dev2.py      # dev/experimental server variant
general.py         # shared uiautomator2 helpers (popups, permissions, element lookup)
data.py            # dataclasses for flow state (TransportBookingData, FoodOrderData, MessageData, ...)
start-tunnel.sh     # Cloudflare Tunnel bootstrap for the Termux device
Grab/, gojek/, zig/, tada/, ryde/, deliveroo/, foodpanda/, whatsapp/
                    # per-app automation: login, price check, booking/ordering, cancellation
```

## Notes

- `server_dev.py` and `server_dev2.py` are in-progress/experimental variants of the main server and may not be feature-complete.
- This automates third-party apps' UI directly; it is not affiliated with or endorsed by Grab, Gojek, Zig, Tada, Ryde, Deliveroo, Foodpanda, or WhatsApp, and is intended for personal/authorized use only — automating another user's account or bypassing app terms of service is the user's own responsibility.
