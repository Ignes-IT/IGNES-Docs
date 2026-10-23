# Architecture

This page describes how the current IGNES services fit together. See [Operations](OPERATIONS.md) for build and runtime information, and [Security](SECURITY.md) for credential handling.

## Components

| Component | Responsibility | Integration |
| --- | --- | --- |
| [IGNES-Core](https://github.com/Ignes-IT/IGNES-Core) | Express API, authentication, plans, subscriptions, device records, and VPN configuration generation. Prisma stores application data in PostgreSQL. | Connects to the VPN host over SSH to manage peers. |
| [IGNES-Telegram-Bot](https://github.com/Ignes-IT/IGNES-Telegram-Bot) | Telegraf bot for account, subscription, and device actions. | Calls Core's HTTP API. Logged-in sessions remain in process memory. |
| [IGNES-Website](https://github.com/Ignes-IT/IGNES-Website) | Next.js landing page and plan display. | Fetches public plan data from Core and links to the Telegram bot. |
| [IGNES-Shared](https://github.com/Ignes-IT/IGNES-Shared) | Shared TypeScript types, validation schemas, and brand configuration. | Imported by Core and other TypeScript clients. |
| VPN host | Runs the peer add, remove, and list operations. | Receives restricted peer-management commands from Core over SSH. It maintains state separately from the Core database. |

Core handles API authorization and application data. The bot and website act as clients. The website fetches plans in the browser using `NEXT_PUBLIC_CORE_API_URL`. Core currently allows the browser origin `http://localhost:3001`; other website origins require a matching CORS configuration.

## Request flows

1. The website requests `GET /api/plans` and links visitors to the Telegram bot.
2. The bot registers or logs in through Core. Core returns a JWT, which the bot keeps in an in-memory session.
3. The bot sends authenticated requests to Core. Core checks the account and plan, stores device records and keys in PostgreSQL, and uses SSH peer commands when VPN management is enabled.
4. Core creates a client configuration from the device key and server settings and returns it to the authenticated user.
5. When a device is deleted, Core attempts to remove its VPN peer and then deletes its database record. If `removePeer` fails, Core logs the error and still deletes the record. The peer can remain on the VPN host.

## HTTP API

The endpoints below are served by Core. Registration, login, health, plans, and brand configuration are public. All other routes require a bearer JWT.

| Method and path | Purpose |
| --- | --- |
| `GET /health` | Returns API status and a separate database status field. |
| `POST /api/auth/register`, `POST /api/auth/login` (public) | Register an account or log in. Both return a JWT. |
| `GET /api/auth/me`, `DELETE /api/auth/me` | Read or delete the authenticated account. |
| `GET /api/plans` (public) | List available plans. |
| `GET /api/brand` (public) | Return the brand configuration. |
| `GET /api/subscriptions/me`, `POST /api/subscriptions` | Read the current subscription or create a trial. |
| `POST /api/subscriptions/mock-upgrade` | Mock PRO upgrade. This endpoint does not process payments. |
| `POST /api/devices`, `GET /api/devices`, `DELETE /api/devices/:id` | Create, list, or delete the authenticated user's devices. |
| `GET /api/vpn/config/:uuid` | Download the authenticated user's WireGuard-compatible configuration. |

### Account deletion

`DELETE /api/auth/me` returns HTTP 409 with the message `Cannot delete user with active devices` if the account has any device records. The `deviceCount` query filters by `userId` only, so it includes inactive devices as well. Delete the device records before retrying account deletion.

## Environment

Core validates its environment variables at startup. The following six variables have defaults:

| Variable | Default |
| --- | --- |
| `NODE_ENV` | `development` |
| `PORT` | `3000` |
| `JWT_EXPIRES_IN` | `7d` |
| `WIREGUARD_DNS` | `1.1.1.1` |
| `VPN_SSH_USER` | `root` |
| `VPN_MANAGE_SCRIPT` | `/root/awg/manage_amneziawg.sh` |

Core requires `DATABASE_URL`, `JWT_SECRET`, `SERVER_PUBLIC_KEY`, `WIREGUARD_ENDPOINT`, `WIREGUARD_SUBNET_PREFIX`, `WIREGUARD_SERVER_IP`, `VPN_SSH_HOST`, and `VPN_SSH_KEY_PATH`. `DATABASE_URL` must use PostgreSQL. In production, `JWT_SECRET` must contain at least 32 characters. Core validates the formats of the WireGuard public key, endpoint, subnet prefix, and server IP.

Setting `VPN_SSH_KEY_PATH` to `/dev/null` disables VPN management. The bot requires `BOT_TOKEN`; `CORE_API_URL` defaults to `http://localhost:3000`. The website's `NEXT_PUBLIC_CORE_API_URL` has the same local default. Configure these values for the target environment.

Core reads `VPN_MANAGE_SCRIPT` into its configuration, but the current VPN service does not use it. The service invokes `peer-add`, `peer-remove`, and `peer-list` directly.

## Current limitations

- Bot sessions are stored in a process-local map. Restarting the bot clears them.
- Core's `/health` response can return `status: "ok"` while `db` is `"fail"`. Check both fields.
- Core's SSH client disables host-key verification and does not use a persistent known-hosts file. See [Security](SECURITY.md).
- Device deletion continues when peer removal fails. A VPN peer can remain after its database record is deleted.
- The account deletion message refers to “active devices,” but the count includes every device record, regardless of `isActive`.
- The mock subscription upgrade is not a payment flow. Core's current CORS configuration allows only the local website origin.

## Source references

This summary was checked against the `main` branches of [IGNES-Core](https://github.com/Ignes-IT/IGNES-Core), [IGNES-Telegram-Bot](https://github.com/Ignes-IT/IGNES-Telegram-Bot), [IGNES-Website](https://github.com/Ignes-IT/IGNES-Website), and [IGNES-Shared](https://github.com/Ignes-IT/IGNES-Shared) on 2026-10-24. Core files relevant to these details include `src/config/environment.ts`, `src/app.ts`, `src/controllers/auth.controller.ts`, `src/controllers/device.controller.ts`, and `src/services/vpn.service.ts`.
