# emudhra-trust-check

A single, self-contained Bash script that tells you whether a machine would accept
certificates issued by the **eight compromised eMudhra intermediate CAs** (October 2026).

It runs **offline**: the 16 intermediate certificates are embedded in the script,
and their SHA-256 fingerprints are re-checked on every run. All it needs is `bash` and `openssl`.
It doesn't change anything on the machine.

## Background

The compromised keys belong to these intermediates:

- `EM DV TLS CA - G3B`, `EM DV TLS CA - G3C`
- `EM OV TLS CA - G3B`, `EM OV TLS CA - G3C`
- `EM EV TLS CA - G3B`, `EM EV TLS CA - G3C`
- `EM OV CS CA - G3B`, `EM OV CS CA - G3C` (code signing)

Each one exists twice: once signed by an emSign G3 root, and once **cross-signed by
Sectigo "AAA Certificate Services"**, which many trust stores still contain. Having
emSign roots in your bundle isn't the issue in itself. What matters is whether any of
these 16 certificates chains to a root you trust, and removing the emSign roots alone
doesn't close the Sectigo cross-signed paths.

Certificate list source: Kudelski Security / Mozilla bug 2083449.

## Usage

```bash
curl -fsSLO https://raw.githubusercontent.com/mplisov/emudhra-trust-check/main/check-emudhra-trust.sh
bash check-emudhra-trust.sh
```

Other ways to run it:

```bash
./check-emudhra-trust.sh          # if executable
bash -s < check-emudhra-trust.sh  # from stdin, e.g. over ssh
bash check-emudhra-trust.sh -r    # also check revocation online
```

| Option | Meaning |
|---|---|
| `-r` | Also check revocation: downloads CRLs with OpenSSL, and queries OCSP on macOS. This is the only option that uses the network. |
| `-h` | Show help |

| Exit code | Meaning |
|---|---|
| `0` | Not exposed: no compromised TLS intermediate chains to a trusted root (and, on macOS, the system verifier rejected all 16) |
| `1` | Exposed: at least one compromised chain is accepted |
| `2` | Error: `openssl` or `bash` missing, or the embedded certificates are damaged |

## What it checks

For each of the 16 certificates:

1. **OPENSSL**: does it chain to a root in OpenSSL's default trust store, with no
   revocation check? This is how most `curl`, `wget`, Python and other OpenSSL-based
   clients behave on Linux. It honours `SSL_CERT_FILE` / `SSL_CERT_DIR`.
2. **+CRL** (only with `-r`): the same check, but OpenSSL downloads the issuer's CRL.
   `revoked` means a client that checks CRLs would reject this chain.
3. **MACOS** (macOS only): what the system verifier says (`security verify-cert`).
   This is what Safari, Chrome, native apps and Go programs rely on. Without `-r` it
   stays local and doesn't fetch missing CA certificates from the network.

## Example output

A store that still trusts Sectigo AAA, run with `-r`:

```
INTERMEDIATE         USE  ISSUER                             OPENSSL    +CRL
DV TLS CA - G3B      TLS  Sectigo AAA Certificate Services   ACCEPTED   revoked
DV TLS CA - G3B      TLS  emSign Root TLS CA - G3            rejected   n/a
...

RESULT: 6 compromised TLS intermediate(s) chain to a trusted root in the OpenSSL store.
        Tools that use this store and don't check revocation (most curl/wget/python/
        openssl-based clients) would accept forged certs for any domain via these chains.
        Removing the emSign roots alone won't fix the Sectigo AAA cross-signed paths.
        6 of them are revoked in the issuer's CRL: clients that check CRLs reject
        them, but most OpenSSL-based tools don't check, so they are still exposed.
```

## Tested on

Tested on 27 Oct 2026, offline (no network, read-only filesystem, non-root user):

| Platform | OpenSSL | Default store result |
|---|---|---|
| Debian 13 (trixie), ca-certificates 20250419+deb13u1 | 3.5.7 | all 16 rejected |
| Ubuntu 26.04.1 LTS, ca-certificates 20260816 | 3.5.5 | all 16 rejected |
| Alpine 3.23, ca-certificates 20261020 | 3.5.9 | all 16 rejected |
| macOS 27, Homebrew OpenSSL 4.1.0 | 4.1.0 | all 16 rejected |
| macOS 27, system LibreSSL 3.3.6 (`/etc/ssl/cert.pem`) | LibreSSL 3.3.6 | **8 Sectigo cross-signs accepted** |
| macOS 27, system verifier | — | all 16 rejected |

Alpine doesn't include bash by default (`apk add bash`). If you run the script with
plain `sh`, it switches to `bash` when bash is installed, and otherwise exits with a
clear message.

## If you're exposed

- Remove **Sectigo "AAA Certificate Services"** (and the emSign G3 roots) from the
  affected bundle, or update your distro's `ca-certificates` package, then run the script again.
- Prefer clients that check revocation. Sectigo has revoked all eight cross-signs.
- On macOS, `/etc/ssl/cert.pem` is used only by LibreSSL tools such as `/usr/bin/curl`.
  Apps that use the system verifier are covered separately by the MACOS column.

## Verifying the embedded certificates

The fingerprints are listed in the `CERTS` table near the top of the script. Every
embedded PEM is preceded by a comment with its subject, issuer and SHA-256. To check
them yourself against crt.sh:

```bash
curl -s "https://crt.sh/?d=45A883F349AEDFD5F3A649B5498FFB121712BA8B448F3F2A91DBC5A766FCDDBE" \
  | openssl x509 -noout -fingerprint -sha256 -subject -issuer
```
