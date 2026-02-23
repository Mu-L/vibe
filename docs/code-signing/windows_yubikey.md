# Windows Code Signing (SSL.com + YubiKey FIPS)

Local hardware-based code signing using **SSL.com Code Signing Certificate**
installed on a **YubiKey FIPS (firmware 5.4.x)** e.g. [yubikey-5c-nano-fips](https://www.yubico.com/il/product/yubikey-5-fips-series/yubikey-5c-nano-fips/)

No cloud signing  
No monthly fees  
Unlimited signings

---

## Overview

In this flow:

- The private key is generated inside the YubiKey
- SSL.com re-issues the certificate bound to that hardware
- Signing works locally using the YubiKey (PIV / smart card)
- No eSigner, no remote service, no .pfx file

---

## Requirements

- Windows machine with physical USB access (no RDP)
- YubiKey FIPS 5.4.x (5.4.3 confirmed)
- SSL.com Code Signing Certificate
- Administrator access on Windows

---

## 1. Install YubiKey tools

Download and install **YubiKey Manager** from Yubico.

This installs:

- YubiKey Manager GUI
- ykman CLI (used for PIV and attestation)

The CLI is installed at:

```
C:\Program Files\Yubico\YubiKey Manager
```

---

## 2. Open PowerShell and add ykman to PATH (session only)

Open **PowerShell as Administrator**, then run:

```
cd $env:USERPROFILE\Desktop
$env:PATH += ";C:\Program Files\Yubico\YubiKey Manager"
```

---

## 3. Verify YubiKey connection

Make sure:

- YubiKey Manager GUI is closed
- The YubiKey is physically connected (not via RDP)

Run:

```
ykman list
```

Expected output example:

```
YubiKey 5C Nano FIPS (5.4.3) [OTP+FIDO+CCID]
```

---

## 4. Reset PIV application (recommended)

This deletes **PIV data only**.

```
ykman piv reset
```

Confirm with `y`.

Defaults after reset:

- PIN: 123456
- PUK: 12345678
- Management Key: default

---

## 5. (Optional) Change PIV PIN

PIN must be 6–8 digits.

```
ykman piv access change-pin
```

---

## 6. Generate private key on the YubiKey (no touch)

Using slot **9c (Digital Signature)**  
Algorithm: RSA 2048  
Touch policy: never

```
ykman piv keys generate --algorithm RSA2048 --pin-policy once --touch-policy never 9c pubkey.pem
```

When prompted for Management Key:

- Press Enter (use default)

---

## 7. Generate CSR from the YubiKey

```
ykman piv certificates request 9c pubkey.pem codesign.csr -s "CN=Your Name"
```

Example:

```
ykman piv certificates request 9c pubkey.pem codesign.csr -s "CN=Yakov Kolani"
```

Enter PIN when prompted.

---

## 8. Generate attestation certificates

### 8.1 Generate attestation

```
ykman piv keys attest 9c attestation.crt
```

### 8.2 Export attestation intermediate

```
ykman piv certificates export f9 attestation-intermediate.crt
```

---

## 9. Submit attestation to SSL.com

1. Login to SSL.com
2. Go to Orders
3. Open the Code Signing order
4. Manage → Attestation
5. Paste:
    - attestation.crt
    - attestation-intermediate.crt
6. Submit

You should see **Successful attestation**.

---

## 10. Certificate re-issuance

After approval:

- SSL.com re-issues the certificate for the YubiKey
- No additional payment required
- eSigner can be canceled

Download format:

- YubiKey (DER)

---

## 11. Install certificate into YubiKey

```
ykman piv certificates import 9c codesign.der
```

---

## 12. Verify certificate

```
ykman piv certificates list
```

Certificate should appear in slot 9c.

---

## 13. Signing binaries on Windows

You can now sign using:

- signtool.exe
- jsign (Windows CAPI or PKCS#11)

Private key never leaves the YubiKey.

---

## Notes

- RDP does not work with YubiKey PIV
- Physical access required for setup
- touch-policy never allows automation
- Unlimited signings
- Token reusable on renewal

---

## Summary

- Hardware-backed private key
- SSL.com certificate bound via attestation
- No cloud signing
- No monthly limits
- Works on Windows, macOS, and Linux
