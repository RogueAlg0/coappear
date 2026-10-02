# coappear

Presence-as-a-script-tag: live cursors and shared notes for any website. Zero signup.

Any website becomes a shared place with one script tag. Visitors appear as live cursors, each carrying a warm, randomly assigned name (no real identity, just a kind stranger), and anyone can pin shared note boxes onto the page that everyone else sees instantly.

## Principles

- **Invisible by design.** No chrome, no branding, no UI to learn. Presence should feel like a natural property of the host page.
- **Privacy-first.** No cookies, no accounts, no stored IPs. Rate limits live in memory only.
- **Free and featherlight.** One small server, one tiny script, SQLite for state. Runs comfortably on a free tier.

## Status

Early scaffold. v1 scope is locked and the architecture plan is being finalized. See [SCOPE.md](docs/SCOPE.md).

## v1 (locked)

- One `<script>` embed per site, no signup or dashboard, per-site namespaces keyed by domain
- Live cursors with empowering random names plus visitor count
- Shared note boxes, 5 per visitor, big notes supported
- Strict profanity filter, including lookalike-character tricks
- Per-site persistent board state in SQLite
- Secret-key admin wipe for emergencies

Out of v1: accounts, DMs, edit history, images, moderation UI.

## License

MIT. See [LICENSE](LICENSE).
