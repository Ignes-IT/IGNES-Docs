# Operations

This page describes the build and deployment configuration in the source repositories. It does not describe the state of a live environment.

## Build and deployment

- **Core** uses TypeScript, Express, Prisma, and PostgreSQL. Available scripts include `npm run build`, `npm start`, and `npm test`. Tests load `.env.test`.
- **Telegram Bot** uses TypeScript and Telegraf. Available scripts include `npm run build`, `npm run typecheck`, and `npm start`.
- **Website** uses Next.js. Available scripts include `npm run build`, `npm run lint`, and `npm start`.
- **Shared** is a TypeScript package built with `npm run build`.

The Core Dockerfile expects `IGNES-Shared/` and `IGNES-Core/` directories in its build context. A standalone Core checkout does not have this layout. Confirm the build context and shared-package version before building the image.

Core's Compose file expects operator-provided `core.env` and `postgres.env` files. It waits for PostgreSQL's health check and mounts named volumes for database data and SSH keys. The API port is mapped to host loopback. Keep these environment files and all deployment values in an approved secret manager or protected deployment environment. Do not add them to this repository.

The Core entrypoint creates a peer-management SSH key in its mounted volume if no key exists, then prints the public key for operator setup. The VPN host should restrict this key to the peer-management commands using the forced-command dispatcher in the Core repository.

## Operational checks

1. Confirm that Core starts with valid environment variables and can connect to PostgreSQL. Check the `db` field in `GET /health`; the top-level `status` can remain `ok` if the database check fails.
2. Confirm that the API URL, website `NEXT_PUBLIC_CORE_API_URL`, bot `CORE_API_URL`, and Core CORS policy match the target environment. Core currently allows only the local website origin.
3. If VPN management is enabled, check the SSH identity, forced-command setup, and peer list, add, and remove operations with a test account and device.
4. Check Core and host logs after device changes. If the log contains `removePeer failed (continuing)`, Core has still deleted the database record. Check and reconcile the peer on the host.
5. After restoring the database or maintaining the VPN host, compare device records with host peer state. Verify client configuration generation before restoring service.

## Recovery precautions

- PostgreSQL backups contain device private keys, account data, and subscription data. Encrypt backups, limit and audit access, and apply a defined retention period.
- Keep database backups and VPN host peer configuration from compatible recovery points. Restoring only one side can leave database records and VPN peers out of sync.
- Protect the Core SSH private key and its persistent volume. If the key is lost or exposed, replace it, update the authorized key on the VPN host, and remove the old authorization.
- Restore credentials through the approved secret manager or another protected channel. Do not put secret values, client configurations, or unredacted environment files in logs, tickets, or documentation.
- Test restores in a restricted environment. Do not copy production data or keys into development or test systems.
