# Security

Treat secrets, private keys, and client configurations as credentials. Never commit them to this repository.

## Protect credentials

- Store Core, bot, PostgreSQL, VPN host, and website deployment values in an approved secret manager or protected deployment environment. Do not put them in source control, container images, chat, or issue attachments.
- In production, `JWT_SECRET` must contain at least 32 characters. The bot requires `BOT_TOKEN`. Configure the API URLs for the intended environment.
- Limit Core's SSH identity to the peer-management commands it needs. Core currently disables SSH host-key verification with `StrictHostKeyChecking=no` and `UserKnownHostsFile=/dev/null`. Configure host-key verification before relying on this integration in production.
- The bot keeps access tokens in process memory. Protect access to the running process and never log bearer tokens.
- Client configuration files contain a device private key. Deliver them only to the authenticated device owner and protect any downloaded copies.

## Rotate credentials

- **JWT signing key:** Replacing `JWT_SECRET` invalidates tokens signed with the old value. Deploy the new value through the secret manager. Users will need to log in again.
- **Database credentials:** Update the database and Core configuration together. Confirm that Core reconnects and that `GET /health` reports the database as available.
- **Bot token:** Create or rotate the token through Telegram's bot administration workflow. Update the bot's protected configuration, confirm it reconnects, then revoke the old token.
- **VPN SSH key:** Install a restricted replacement key on the VPN host and update Core to use it. Verify peer operations before removing the old authorization.
- **VPN server keys or endpoint:** Coordinate changes to the host and client configurations. Users may need updated configuration files. Verify connectivity after the change.

Record rotation dates and ownership, not credential values. Test recovery and rotation procedures outside production before relying on them during an incident.

## Database and backups

Core stores each device's `privateKey`, public key, IP address, and owning user in PostgreSQL. Database backups therefore contain client authentication material. Encrypt backups, restrict and audit access, and remove copies according to the retention policy.

Treat client configuration downloads, database dumps, and backups containing device records as sensitive credentials. Do not share them in support channels or use them in test environments. Restore the database and VPN host state from compatible recovery points, then reconcile device records with host peers.

## Known security concerns

Core currently skips SSH host-key verification. Pinning the VPN host key is an engineering hardening task. Device deletion also proceeds when peer removal fails, so operators must check host state when Core logs a removal failure.
