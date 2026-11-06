ok# Guardian

Guardian is a Linux security product that protects you from phishing and malicious websites. It has three parts:

- **Browser extension** (Chrome, Chromium, Brave, Edge): checks each site before it loads and shows a warning page with steps to take if you may have been affected.
- **Local backend** (Python): runs as your user, keeps the threat blocklist, and records blocked events.
- **Firewall helper** (Rust): optional. Blocks the addresses of known-bad sites at the system level using nftables.

Everything runs on your computer. Guardian doesn't send your browsing history anywhere.

## What Guardian does and doesn't do

Guardian **does**:

- Warn you before opening known phishing and malicious domains
- Keep a local log of blocked domains (domains only, never full URLs)
- Show a dashboard with recent events and the 24-hour block count
- Optionally block known-bad addresses for all programs on the computer
- Show response steps if you think an account has been compromised

Guardian **does not**:

- Remove malware or undo changes an attacker has already made
- Detect brand-new attacks that aren't in a threat feed
- Protect other devices on your network
- Attack, hack back, or take action against anyone else's system

Blocking by IP address is coarse. Many websites share servers through CDNs, so blocking one site can also affect others on the same address.

## Requirements

- Linux with systemd (Ubuntu, Debian, Fedora, Arch, and similar)
- Python 3.10 or newer
- Rust toolchain (for the firewall helper only)
- nftables (for the firewall helper only)
- Chrome, Chromium, Brave, or Edge
- A licensed threat-intelligence feed, or your own list of domains

## Installation

### 1. Install the backend

```bash
cd guardian
./packaging/install.sh
```

This creates a Python environment under `~/.local/share/guardian/`, generates an API token, and starts the `guardian` service at login.

Check that it's running:

```bash
systemctl --user status guardian
```

### 2. Load the browser extension

1. Open `chrome://extensions` (or `brave://extensions` or `edge://extensions`).
2. Turn on **Developer mode**.
3. Click **Load unpacked** and select the `extension/` folder.

### 3. Connect the extension to the backend

1. Print the API token:

   ```bash
   ~/.local/share/guardian/venv/bin/guardian \
     --token-file ~/.local/share/guardian/api_token --print-token
   ```

2. In the extension, open **Details, then Extension options**, paste the token, and click **Save**.

### 4. Optional: install the firewall helper

Only needed if you want system-level blocking.

```bash
sudo apt install nftables          # or the equivalent for your distribution
cd daemon && cargo build --release && cd ..
sudo ./packaging/install-firewall.sh
```

Then log out and back in, so your session picks up the `guardian` group.

To turn on firewall blocking, open the extension's options page and tick **Also block at the firewall level**, or use the dashboard's **Turn on** button.

## Adding threat data

Guardian is only as good as the blocklist it uses. Until you connect a licensed feed, the only entry is a demo domain used for testing.

Import a feed from a URL:

```bash
guardian-sync --db ~/.local/share/guardian/guardian.db \
  --name my-feed \
  --location https://feeds.example.com/domains.txt \
  --format lines \
  --category phishing
```

For CSV feeds:

```bash
guardian-sync --db ~/.local/share/guardian/guardian.db \
  --name my-feed \
  --location https://feeds.example.com/domains.csv \
  --format csv \
  --domain-column domain \
  --category-column type
```

Feed URLs must use `https://`. Before a real feed goes into production, check that its license allows commercial use in a product like this.

To remove the demo entry:

```bash
sqlite3 ~/.local/share/guardian/guardian.db \
  "DELETE FROM domains WHERE source='static-dev';"
```

## Using the dashboard

Open the extension's **Settings** page, then **Open the Guardian dashboard**. It shows:

- **Connection status:** Connected, Offline, or Token rejected
- **Blocked in last 24 hours:** the count of blocked visits
- **Known bad domains:** how many entries the blocklist holds
- **Firewall blocking:** on or off, with a toggle
- **Recent events:** time, domain, category, and action
- **Response steps:** what to do if you think you've been compromised

The dashboard refreshes every ten seconds.

## Where your data lives

| Data | Location | Leaves your computer? |
|---|---|---|
| Blocklist | `~/.local/share/guardian/guardian.db` | No |
| Event log (domains and times) | same database | No |
| API token | `~/.local/share/guardian/api_token` (readable only by you) | No |
| Settings | same database | No |
| Browsing URLs | Not stored. Checked against the local blocklist only. | No |

Event records contain the domain, the category, the time, and the action. Full URLs, paths, and query strings aren't stored.

## Security model

- The backend listens only on `127.0.0.1`, and rejects requests whose Host header isn't `localhost` or `127.0.0.1`. This blocks DNS-rebinding attacks from web pages.
- Every API request needs the token. The token file is readable only by your user account.
- The firewall helper accepts only two commands, `BLOCK` and `UNBLOCK`, with fixed limits: at least 60 seconds and at most 24 hours per block, at most 100 addresses per request. It refuses private, loopback, and link-local addresses, so it can't cut off your own network.
- The helper manages only its own nftables table (`inet guardian`). It doesn't change rules from ufw, firewalld, or other tools.
- Blocks expire automatically. The helper removes its table when it stops.

Guardian protects against other users, web pages, and casual local programs. It does **not** protect against malware already running as your user account, because that malware can read the token file.

## Troubleshooting

**The extension says "Offline" or "Backend not reachable"**
Check that the service is running: `systemctl --user status guardian`. Restart it with `systemctl --user restart guardian`.

**"Token rejected"**
Print the token again and paste it into Settings. Don't include spaces or line breaks.

**"Firewall helper not found"**
Check the socket: `systemctl status guardian-fwd.socket`. If it's missing, run `sudo ./packaging/install-firewall.sh` again. Then log out and back in.

**Firewall block doesn't seem to work**
Check that enforcement is on, then list the blocked addresses:

```bash
sudo nft list set inet guardian blocked4
```

If the set is empty, look at the helper's log: `journalctl -u guardian-fwd`.

**Pages still load after a block**
The browser may have cached the verdict for up to five minutes. Reload the page after a short wait.

## Development

Run the Python tests:

```bash
cd guardian
pip install -e ".[dev]"
pytest
```

Run the Rust tests:

```bash
cd daemon
cargo test
```

## Project status

Guardian is an early-stage build. Before a public or commercial release:

- Connect and license a threat-intelligence feed
- Compile and test the Rust helper on several distributions
- Get an independent security audit of the API and the root helper
- Publish a privacy policy and terms of service
- Add scheduled feed updates

## License:
GNU GENERAL PUBLIC LICESE
Version 3, June 1991