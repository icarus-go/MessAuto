<p align="center">
  <img src="resources/icon_256.png" height="256">
  <h1 align="center">MessAuto</h1>
  <h4 align="center"> Automatic extraction of SMS and email verification codes on Mac</h4>
<p align="center">
<a href="https://github.com/icarus-go/MessAuto/blob/master/LICENSE.txt">
<img src="https://img.shields.io/github/license/icarus-go/MessAuto"
            alt="License"></a>
<a href="https://github.com/icarus-go/MessAuto/releases">
<img src="https://img.shields.io/github/downloads/icarus-go/MessAuto/total.svg"
            alt="Downloads"></a>
<a href="https://img.shields.io/badge/-macOS-black?&logo=apple&logoColor=white">
<img src="https://img.shields.io/badge/-macOS-black?&logo=apple&logoColor=white"
            alt="macOS"></a>
</p>

<p align="center">
  [<a href="./README.md">English</a>] [<a href="docs/README-CN.md">中文</a>]<br>
</p>

# MessAuto

MessAuto is a macOS application that automatically extracts SMS and email verification codes. Built with Rust, it works with any app.

This fork adds authenticated HTTP Webhooks and an embedded ngrok tunnel, so an Android SMS forwarding app can send verification messages directly to your Mac. The current package version is **1.4.0**.

The following demonstrates the SMS login process with the help of MessAuto:

https://github.com/LeeeSe/MessAuto/assets/44465325/6e0aca37-377f-463b-b27e-a12ff8c1e70b

🎉🎉🎉 MessAuto now supports the Mail app

https://github.com/LeeeSe/MessAuto/assets/44465325/33dcec87-61c4-4510-a87c-ef43e69c4e9d

## Usage
### Download and Install
Packages published for this fork are listed under [Releases](https://github.com/icarus-go/MessAuto/releases). Upstream packages do not include this fork's Webhook features; if no fork release is published yet, build from this repository using the commands below.

- **Apple Silicon (M-series)**: choose `aarch64`.
- **Intel**: choose `x86_64`.

Quit the existing MessAuto before replacing it. Open the matching `.dmg` and drag `MessAuto.app` into **Applications**, or unzip the `.app.zip` and copy the app there. Configuration in `~/Library/Application Support/messauto/` is preserved when replacing the application.

Local packages use ad-hoc signing and are not notarized by Apple. Replacing a build can require granting Accessibility permission again, as described below.
### Permissions
- Full Disk Access: System Settings -> Privacy & Security -> Full Disk Access
- Accessibility: System Settings -> Privacy & Security -> Accessibility
### Keep Messages and Mail Apps Running in the Background
Keeping these apps running in the background shortens the time it takes to sync messages from iPhone to Mac.

## Configuration
- Auto Paste: Simulates keyboard input to automatically paste the verification code from the clipboard into the input field.
- Auto Return: Automatically presses the Return key after pasting the verification code.
- Don't Occupy Clipboard: MessAuto simulates keyboard input to directly type the verification code without occupying the clipboard.
- Listen for SMS: When enabled, monitors the built-in Messages app on Mac (works best when the app is running in the background; otherwise there may be a delayed response — this is not a MessAuto issue).
- Listen for Email: When enabled, monitors the built-in Mail app on Mac (same as above).
- Hide Icon: Temporarily hides the menu bar icon. The icon will reappear when the app restarts (you can stop it via Activity Monitor).
- Config: Quickly opens the TOML configuration file where you can customize regex patterns and keywords.
- Log: Quickly opens the log file.
- Floating Window: Automatically shows a popup near the cursor when a verification code is detected. This forces the "Don't Occupy Clipboard" mode.

> Keywords: Also called trigger words. The program will only perform subsequent operations when a message contains keywords such as "verification code". Otherwise, the message will be ignored.

## Webhook and embedded ngrok

Webhook is enabled by default at `0.0.0.0:8978`. POST to `/webhook` (or `/`) with the full message in `value`. It uses the same parser, floating window, direct input, clipboard, paste and Return settings as Messages and Mail.

On first launch, MessAuto generates a random Webhook token. Copy it from the **Webhook** tray submenu:

```bash
TOKEN='<your Webhook token>'
curl -X POST 'http://127.0.0.1:8978/webhook' \
  -H "Authorization: Bearer $TOKEN" \
  -H 'Content-Type: application/json' \
  -d '{"value":"Your verification code is 131964"}'

# Alternatively, pass the token in the URL:
curl -X POST "http://127.0.0.1:8978/webhook?token=$TOKEN" \
  -H 'Content-Type: application/json' \
  -d '{"value":"Your verification code is 131964"}'
```

A match returns `{"ok":true,"code":"131964"}` and queues the configured action. A message without a code returns HTTP 200 with `ok:false`; invalid JSON or a missing `value` returns 400; missing/incorrect credentials return 401. The Authorization header takes precedence over the query token.

The tray submenu lets you enable Webhook/ngrok, view their status, copy public or LAN URLs (including the token), copy/regenerate the Webhook token, and set the ngrok authtoken in a hidden input dialog. Regenerating the token invalidates old forwarding URLs. LAN address discovery tries `en0`, then `en1`.

ngrok runs inside MessAuto via the Rust SDK; no CLI installation is required. It is enabled by default when Webhook is running. If no authtoken is configured, MessAuto tries to import it from `~/Library/Application Support/ngrok/ngrok.yml` or `~/.config/ngrok/ngrok.yml`; otherwise choose **Set ngrok Token…**. Set `ngrok_domain` in `config.toml` to the dev domain assigned to your ngrok account before connecting. If it is empty, the tray and a notification explain how to configure it; the local Webhook keeps running. SDK-generated random URLs cannot be fixed and reused on the free plan.

**Stop any external ngrok CLI using the same domain before connecting.** MessAuto will notify you of conflicts; it does not kill other processes. Disabling ngrok keeps the local Webhook running. A forwarding-only CLI entry such as `smsCodeForward` is no longer needed; unrelated tunnels such as SSH can remain in your CLI configuration.

Configuration is stored in `~/Library/Application Support/messauto/config.toml`:

```toml
listen_webhook = true
webhook_host = "0.0.0.0"
webhook_port = 8978
webhook_token = "<generated token>"
ngrok_enabled = true
ngrok_authtoken = "<ngrok account authtoken>"
ngrok_domain = "<your-assigned-domain>.ngrok-free.dev"
version = 2
```

Existing settings are preserved when upgrading to configuration version 2. After editing the domain, toggle ngrok off/on; restart MessAuto after editing the listener address or port.

Treat both tokens and copied URLs as secrets. The ngrok authtoken is stored as plain text in the configuration file. Public ngrok URLs use HTTPS; local HTTP traffic is unencrypted, so use a trusted LAN. Prefer the Bearer header where possible to avoid putting the Webhook token in URL history/logs. Authorized requests can trigger keyboard input according to your current settings.

## FAQ
### Cannot Open Because the Developer Cannot Be Verified
For a package you trust, locate `MessAuto.app` in Finder and use the macOS per-app Open/Privacy & Security flow. The local package does not carry an Apple Developer ID notarization ticket.

### Accessibility Is Enabled, but Input Still Fails
An old permission entry can refer to a previous build's code signature. This was observed during local verification: the macOS TCC log reported a code requirement mismatch even though the switch was on.

Quit MessAuto, remove its stale Accessibility entry (or run the app-specific command below), add `/Applications/MessAuto.app` again in **System Settings → Privacy & Security → Accessibility**, enable it, and reopen the app:

```bash
tccutil reset Accessibility com.leeese.messauto
```

Full Disk Access is a separate permission used to read Messages/Mail; Accessibility permits keyboard input. When macOS asks to quit and reopen after granting access, make sure it reopens the current app in `/Applications`.

### Webhook Cannot Listen on Port 8978
Quit another app using the same port (such as AutoCode), or change `webhook_port` in `config.toml` and restart MessAuto.

## TODO

- [x] Add in-app updates
- [x] Optimize verification code extraction logic
- [ ] Publish to Homebrew
- [ ] Support third-party email clients
- [ ] Support Android and Windows

Update checks use the official `cargo-packager-updater` 0.2.3. To use a proxy, set `HTTP_PROXY`, `HTTPS_PROXY` or `ALL_PROXY` in the environment before launching MessAuto; macOS System Settings proxy detection from the former custom fork is no longer used.

## Development

```bash
git clone https://github.com/icarus-go/MessAuto.git
cd MessAuto
cargo test --locked

# Build both macOS architectures
rustup target add aarch64-apple-darwin x86_64-apple-darwin
cargo build --locked --release --target aarch64-apple-darwin --target x86_64-apple-darwin

cargo install cargo-packager --version 0.11.8 --locked
cargo packager --release --target aarch64-apple-darwin --formats app,dmg
cargo packager --release --target x86_64-apple-darwin --formats app,dmg
```

The existing packaging metadata expects a `MessAuto_SC` signing certificate. For local ad-hoc packages, set `[package.metadata.packager.macos].signing-identity` to `"-"` in your local configuration before packaging. For distribution with Developer ID signing and notarization, use your own signing identity and credentials.

The generated bundles are `target/aarch64-apple-darwin/release/MessAuto.app` and `target/x86_64-apple-darwin/release/MessAuto.app`, with `.dmg` files in the same directories. Local verification covered native Apple Silicon execution; Intel packages are cross-compiled and their architecture/signature checked, and still need execution on an Intel Mac.

This fork's packages are updated manually. The built-in update checker still points to the upstream repository.

## Acknowledgments

- Thanks to [@尚善若拙](https://sspai.com/post/73072) for providing the idea of extracting SMS verification codes.
