# Code Signing

This guide covers code signing for both Windows and macOS.

---

## Windows (SSL.com eSigner)

Cloud-based signing via SSL.com eSigner + Jsign. No USB token or YubiKey needed.

### 1. What to buy

On SSL.com, buy a **Code Signing Certificate**.

During checkout, you MUST enable eSigner cloud signing.

Look for this checkbox and make sure it is checked:

```
Enable certificate issuance for remote signing, team sharing,
and presign malware scanning
```

If this box is NOT checked:

- eSigner credentials will NOT be created
- you will NOT get a Credential ID
- the certificate cannot be fixed after issuance

### 2. Validation

After purchase, SSL.com will perform standard validation
(identity, email, phone – depends on certificate type).

Once validation is complete and the certificate is issued,
you can continue.

### 3. Where to find the eSigner credentials

After issuance:

1. Go to SSL.com Dashboard
2. Open Orders
3. Find your Code Signing order
4. Click Download
5. Scroll to SIGNING CREDENTIALS

You should see:

- Credential ID (UUID)
- signing credential status = enabled

Example:

```
SSL_COM_CREDENTIAL_ID=xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx
```

### 4. Enable TOTP (one-time setup)

In the same Order page, find the eSigner PIN / QR code section.

Steps:

1. Set a 4-digit PIN
2. Choose OTP App
3. Generate QR code
4. Save BOTH:
    - the QR code
    - the Base32 secret shown as text

The Base32 secret looks like:

```
UFZ3SGLG1KVDIDE3KWJEKVAGG24S5PWDQMQTPBAAJDSC566KKFGB
```

This value is your TOTP secret.

### 5. Verify TOTP works (recommended)

Generate a code and compare it against Google Authenticator at the same moment.

On macOS:

```
brew install oath-toolkit
oathtool --totp -b <BASE32_SECRET>
```

On Windows (with uv):

```powershell
uv run --with pyotp python -c "import pyotp; print(pyotp.TOTP('<BASE32_SECRET>').now())"
```

If the 6-digit code matches Google Authenticator, your TOTP setup is correct.

### 6. How signing works with Tauri

Tauri calls a custom sign command for every binary it wants to sign
(main exe, sidecars, NSIS plugins, installer, etc.).

We use `scripts/sign_windows.py` as that command. It **whitelists**
only the files worth signing and skips everything else, keeping us
under the eSigner monthly signing limit.

What gets signed:

- `vibe.exe` (main app)
- `vibe-*setup*.exe` (NSIS installer)

What gets skipped:

- Sidecars (ffmpeg, sona, sona-diarize)
- NSIS plugins and resource DLLs

By default the script runs in **dry run** mode. Set `SIGN_ENABLED=true`
to actually sign.

#### Tauri config

In `desktop/src-tauri/tauri.windows.conf.json`:

```json
"signCommand": {
  "cmd": "python",
  "args": ["scripts/sign_windows.py", "%1"]
}
```

Tauri passes `%1` as the file path. The script checks the filename
against the whitelist and either signs or skips.

#### Prerequisites

```
choco install jsign
choco install temurin
```

#### Required env vars

```
SIGN_ENABLED=true
SSL_COM_CREDENTIAL_ID=xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx
SSL_COM_USERNAME=your@email.com
SSL_COM_PASSWORD=your_ssl_com_password
SSL_COM_TOTP_SECRET=BASE32_SECRET
```

#### Manual signing

```
uv run scripts/sign_windows.py path\to\file.exe
```

#### Verification

```powershell
signtool verify /pa /v path\to\file.exe
```

---

## macOS (Apple Developer)

Local signing via `codesign` + notarization via `notarytool`.
No per-sign limits. Tauri handles everything automatically.

### 1. What to buy

A **$99/year Apple Developer Program** membership at
https://developer.apple.com/programs/

### 2. Create a Developer ID certificate

1. Open Xcode → Settings → Accounts → Manage Certificates
2. Create a **Developer ID Application** certificate
3. Export it as a `.p12` file with a password

### 3. Base64-encode the certificate

CI needs the `.p12` as a base64 string:

```bash
uv run python -c "import base64, pathlib; print(base64.b64encode(pathlib.Path('cert.p12').read_bytes()).decode())"
```

Copy the output into your `APPLE_CERTIFICATE` secret.

### 4. Find your signing identity

```bash
security find-identity -v -p codesigning
```

Look for `Developer ID Application: Your Name (TEAMID)`.

### 5. Generate an app-specific password

Go to https://appleid.apple.com/account/manage → Sign-In and Security →
App-Specific Passwords → Generate.

### 6. Required env vars

Set these in CI:

```
APPLE_CERTIFICATE=<base64 from step 3>
APPLE_CERTIFICATE_PASSWORD=<.p12 password>
APPLE_SIGNING_IDENTITY="Developer ID Application: Your Name (TEAMID)"
APPLE_ID=<your Apple ID email>
APPLE_PASSWORD=<app-specific password from step 5>
APPLE_TEAM_ID=<your 10-char team ID>
```

Find your team ID:

```bash
uv run python -c "import subprocess, re; o=subprocess.check_output(['security','find-identity','-v','-p','codesigning']).decode(); print(set(re.findall(r'\(([A-Z0-9]{10})\)', o)))"
```

### 7. How it works

Tauri automatically:

- Signs all binaries with `codesign` using your Developer ID cert
- Notarizes the `.dmg` with `notarytool`
- Staples the notarization ticket

No custom scripts or whitelists needed. All binaries must be signed
for notarization to pass.

### 8. Verification

```bash
codesign --verify --deep --strict /path/to/Vibe.app
spctl --assess --type exec /path/to/Vibe.app
```

---

## CI (GitHub Actions)

In `.github/workflows/release.yml`:

- **Windows**: check `sign-windows` input to enable signing.
  The workflow sets `SIGN_ENABLED=true` and passes `SSL_COM_*` secrets.
- **macOS**: set `APPLE_*` secrets in repo settings. Tauri signs automatically.

Add all secrets under repo Settings → Secrets and variables → Actions.

---

## Summary

| | Windows | macOS |
|---|---|---|
| Provider | SSL.com eSigner | Apple Developer |
| Cost | ~$60-90/yr + eSigner tier | $99/yr |
| Signing limit | Yes (depends on tier) | No |
| Signing tool | Jsign (via `sign_windows.py`) | `codesign` (built-in) |
| Notarization | N/A | `notarytool` (automatic) |
| Sign everything? | No, whitelist only | Yes, required |
| Custom script | `scripts/sign_windows.py` | None needed |
