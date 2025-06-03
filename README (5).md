# Two-Factor Authentication (2FA) Hardening on Ubuntu Pro 22.04 LTS system

## Overview

This guide documents a secure, NIST-compliant implementation of two-factor authentication (2FA) for an Ubuntu Pro 22.04 LTS system hosted on AWS. It combines biometric protection with time-based one-time passwords (TOTP) stored securely using the Aegis Authenticator app, enforced via PAM and SSH hardening.

---

## Two-Factor Authentication Mechanism

### Authenticator App

**[Aegis Authenticator](https://getaegis.app)** is used as the TOTP app. It supports:
- **Biometric unlock only** (something you are)
- **Local secure device possession** (something you have)
- **Encrypted backups with versioning**

### Backup Process
1. Set a secure folder as the backup location in Aegis.
2. Backups are versioned and updated automatically on any account changes.
3. Upload encrypted backup folder to a secure cloud storage provider.

> Backup files can be verified under **Settings → Backups** in the Aegis app.

---

##  AWS Cloud Environment

### Operating System
- **Ubuntu Pro 22.04 LTS (FIPS 140-3 Certified)**
  - FIPS-validated crypto modules
  - CIS & DISA-STIG hardening
  - Long-term support (10 years)
  - Source: [AWS Marketplace – Ubuntu Pro](https://aws.amazon.com/marketplace/pp/prodview-o5noqh44k3uxm)

### EC2 Configuration
- Instance type: `t2.large`
- **Inbound:** Only allow traffic on port 22 from a specific IP (using `/32` CIDR)
- **Outbound:** All traffic allowed (0.0.0.0/0)

---

## Two-Factor Authentication Hardening Steps

### 1. SSH and System Preparation

```bash
sudo apt update && sudo apt upgrade -y
sudo reboot
```

Set a system password:

```bash
sudo passwd <your-username>
```

Install Google Authenticator PAM module:

\`\`\`bash
sudo apt install libpam-google-authenticator -y
\`\`\`

---

### 2. PAM Configuration

Edit \`/etc/pam.d/common-auth\`:

**Before:**
\`\`\`plaintext
auth optional pam_cap.so
\`\`\`

**After:**
\`\`\`plaintext
auth optional pam_cap.so
auth required pam_google_authenticator.so nullok
\`\`\`

 \`nullok\` allows password-only fallback before 2FA is set. This will be removed in the final step.

---

### 3. SSH Server Configuration

Edit \`/etc/ssh/sshd_config\`:

- Add below line ~62:
  \`\`\`plaintext
  ChallengeResponseAuthentication yes
  \`\`\`
- Add below line ~86:
  \`\`\`plaintext
  AuthenticationMethods keyboard-interactive
  \`\`\`

Then restart SSH:
\`\`\`bash
sudo systemctl restart sshd
\`\`\`

---

### 4. PAM SSHD Configuration

Edit \`/etc/pam.d/sshd\`:

**Add at the top:**
\`\`\`plaintext
auth required pam_google_authenticator.so nullok
\`\`\`

---

### 5. Initialize Google Authenticator

Run:

\`\`\`bash
google-authenticator
\`\`\`

- Choose **time-based tokens** (\`y\`)
- Use **secret key** (not QR code) to add account to Aegis
- Answer the following prompts with \`y\`:
  - Update \`.google_authenticator\` file
  - Disallow token reuse
  - Increase window for clock skew
  - Enable rate limiting

Back up emergency codes to encrypted storage.

---

### 6. Secure \`pam_google_authenticator\` Enforcement

Remove \`nullok\` from:
- \`/etc/pam.d/common-auth\`
- \`/etc/pam.d/sshd\`

**Final form (\`common-auth\`):**
\`\`\`plaintext
auth optional pam_cap.so
auth required pam_google_authenticator.so
\`\`\`

**Final form (\`sshd\`):**
\`\`\`plaintext
auth required pam_google_authenticator.so
\`\`\`

Restart SSH:

\`\`\`bash
sudo systemctl restart sshd
\`\`\`

---

## Final Login Flow

After SSH login:
1. Prompt for first verification code (TOTP)
2. Prompt for system password
3. Prompt for second TOTP (a different valid one)
4. Successful authentication

---

## Compliance with NIST 800-63B

| Requirement                             | Compliant | Reference                       |
|----------------------------------------|-----------|---------------------------------|
| MFA with two distinct factors          | YES        | NIST SP 800-63B §5.1.1          |
| Biometric use not as sole authenticator| YES       | §5.2.3                          |
| Cryptographic protection of TOTP secrets| YES       | §5.1.4.1 (via Aegis + biometric)|
| Mandatory use of 2FA (no \`nullok\`)     | YES        | §5.1.5                          |
| Rate limiting login attempts           | YES        | §5.2.2                          |

---

## Future-Proofing

| Component                    | Viability     | Notes                                   |
|-----------------------------|---------------|-----------------------------------------|
| Aegis w/ biometric unlock    | YES 2025–2030+ | Secure TOTP with local biometric access |
| FIPS 140-2 hardware key      | Deprecated by 2030 | Consider moving to FIPS 140-3 |
| Google Authenticator (TOTP) | YES Still valid | Recommend shift to FIDO2/WebAuthn       |
| Ubuntu Pro 22.04 LTS        | YES Until 2032 | 10-year support lifecycle               |

---

## References

- [NIST SP 800-63B: Digital Identity Guidelines - Authentication & Lifecycle Management](https://pages.nist.gov/800-63-3/sp800-63b.html)
- [FIPS 140-3 Compliance Program](https://csrc.nist.gov/projects/cryptographic-module-validation-program)
- [Google Authenticator on Ubuntu](https://idroot.us/set-up-two-factor-authentication-ubuntu-24-04/)
- [Red Hat \`nullok\` PAM Warning](https://access.redhat.com/solutions/6890171)
- [AWS Ubuntu Pro FIPS Image](https://aws.amazon.com/marketplace/pp/prodview-o5noqh44k3uxm)

---

## Related Security Practices

- Use \`auditd\` or \`fail2ban\` for additional login monitoring.
- Enforce SSH key authentication before TOTP if FIDO keys are unavailable.
- Consider using **WebAuthn + TPM/FIDO2 security keys** for even stronger assurance (AAL3).

---

## 📦 License

MIT License – see \`LICENSE\` file.
