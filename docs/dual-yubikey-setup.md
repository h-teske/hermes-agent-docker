# Dual YubiKey Setup Guide

Anleitung für die Einrichtung von zwei YubiKeys (Primary + Backup) für:
- **Mac Login** (Smart Card / PIV)
- **Git Commit Signing** (SSH-basiert)
- **SSH Login** auf Server

## Grundprinzip

Jeder YubiKey generiert sein **eigenes Schlüsselpaar**. Man kann private Keys nicht zwischen YubiKeys kopieren. Deshalb muss jeder Dienst **beide Public Keys** kennen.

---

## 1. Voraussetzungen

```bash
# Homebrew-Pakete installieren
brew install ykman openssh pinentry-mac gnupg

# Optional: YubiKey Manager GUI
brew install --cask yubico-yubikey-manager
```

Firmware prüfen:
```bash
ykman info
```

---

## 2. PIV (Smart Card) einrichten – für Mac Login

Jeder YubiKey hat ein PIV-Modul (Personal Identity Verification) mit eigenen Zertifikaten.

### 2.1 PIV auf beiden Keys initialisieren

**Für jeden YubiKey einzeln durchführen** (immer nur einen einstecken):

```bash
# PIN ändern (Standard: 123456)
ykman piv access change-pin

# PUK ändern (Standard: 12345678)
ykman piv access change-puk

# Management Key ändern (Standard: 010203040506070801020304050607080102030405060708)
ykman piv access change-management-key --generate --protect
# --protect speichert den Management Key PIN-geschützt auf dem YubiKey selbst
```

### 2.2 Zertifikat für Smart Card Login generieren

Auf **jedem** YubiKey:

```bash
# Selbstsigniertes Zertifikat im Slot 9a (PIV Authentication) generieren
ykman piv keys generate --algorithm ECCP384 9a pubkey-yubikey1.pem
ykman piv certificates generate \
  --subject "CN=Dein Name" \
  --valid-days 3650 \
  9a pubkey-yubikey1.pem

# Für den zweiten Key:
ykman piv keys generate --algorithm ECCP384 9a pubkey-yubikey2.pem
ykman piv certificates generate \
  --subject "CN=Dein Name" \
  --valid-days 3650 \
  9a pubkey-yubikey2.pem
```

### 2.3 Mac Smart Card Login aktivieren

macOS erkennt PIV-fähige Smart Cards automatisch über CryptoTokenKit:

```bash
# Smart Card Pairing erzwingen
sc_auth pairing_ui -s enable

# Token-basierte Authentifizierung prüfen
sc_auth identities
```

**Beide Keys pairen:**
1. YubiKey 1 einstecken → macOS fragt nach Pairing → PIN eingeben → mit Account verknüpfen
2. YubiKey 2 einstecken → gleichen Vorgang wiederholen

Danach kannst du dich mit **beiden** Keys am Mac anmelden.

**Wichtig:** Unter *Systemeinstellungen → Benutzer & Gruppen → Erweitert* (oder via `sc_auth`):
```bash
# Beide Identitäten dem Account zuordnen
sc_auth pair -u <username> -h <hash-von-yubikey1>
sc_auth pair -u <username> -h <hash-von-yubikey2>
```

Prüfe die gepairten Identitäten:
```bash
sc_auth list
```

---

## 3. SSH Keys auf dem YubiKey – für SSH Login

Es gibt zwei Ansätze. **Empfohlen: FIDO2/Resident Keys** (einfacher, moderner).

### 3.1 Option A: FIDO2 Resident Keys (empfohlen, ab YubiKey 5)

```bash
# YubiKey 1 einstecken
ssh-keygen -t ed25519-sk -O resident -O verify-required \
  -C "yubikey1@hostname" -f ~/.ssh/id_ed25519_sk_yk1

# YubiKey 2 einstecken
ssh-keygen -t ed25519-sk -O resident -O verify-required \
  -C "yubikey2@hostname" -f ~/.ssh/id_ed25519_sk_yk2
```

- `-O resident` speichert den Key direkt auf dem YubiKey (discoverable)
- `-O verify-required` erfordert PIN + Touch bei jeder Nutzung
- `ed25519-sk` nutzt den FIDO2-Standard

### 3.2 Option B: PIV-basierte SSH Keys

Falls du den PIV-Slot (gleicher wie Mac Login) auch für SSH nutzen willst:

```bash
# Public Key aus PIV-Slot 9a exportieren (pro YubiKey)
ssh-keygen -D /usr/lib/opensc-pkcs11.so -e > ~/.ssh/id_piv_yk1.pub

# In ~/.ssh/config den PKCS11-Provider konfigurieren
```

`~/.ssh/config`:
```
Host *
  PKCS11Provider /usr/lib/opensc-pkcs11.so
  # oder auf macOS:
  # PKCS11Provider /usr/local/lib/opensc-pkcs11.so
```

### 3.3 Beide Public Keys auf Servern hinterlegen

Auf **jedem** Zielserver in `~/.ssh/authorized_keys` **beide** Public Keys eintragen:

```bash
# Auf dem Server:
cat >> ~/.ssh/authorized_keys << 'EOF'
ssh-ed25519-sk ... yubikey1@hostname
ssh-ed25519-sk ... yubikey2@hostname
EOF
```

### 3.4 Lokale SSH Config für beide Keys

`~/.ssh/config`:
```
Host server-beispiel
  HostName example.com
  User deinuser
  IdentityFile ~/.ssh/id_ed25519_sk_yk1
  IdentityFile ~/.ssh/id_ed25519_sk_yk2
  IdentitiesOnly yes
```

SSH probiert automatisch beide Keys durch und nutzt den, der zum eingesteckten YubiKey gehört.

### 3.5 Resident Keys wiederherstellen (Backup-Szenario)

Falls du den Rechner neu aufsetzt, kannst du die Keys vom YubiKey laden:

```bash
ssh-keygen -K
# Lädt alle Resident Keys vom eingesteckten YubiKey herunter
```

---

## 4. Git Commit Signing mit SSH Keys

Seit Git 2.34 kann man SSH Keys zum Signieren verwenden – kein GPG nötig.

### 4.1 Git konfigurieren

```bash
# SSH als Signaturformat setzen
git config --global gpg.format ssh

# Primären Signing Key setzen (YubiKey 1)
git config --global user.signingkey ~/.ssh/id_ed25519_sk_yk1.pub

# Alle Commits automatisch signieren
git config --global commit.gpgsign true
git config --global tag.gpgsign true
```

### 4.2 Allowed Signers File (für Verifikation)

```bash
# Datei mit allen erlaubten Signierern anlegen
cat > ~/.config/git/allowed_signers << 'EOF'
deine@email.com ssh-ed25519-sk AAAA...key1...
deine@email.com ssh-ed25519-sk AAAA...key2...
EOF

git config --global gpg.ssh.allowedSignersFile ~/.config/git/allowed_signers
```

### 4.3 Zwischen YubiKeys wechseln

Wenn du YubiKey 2 nutzen willst, musst du den Signing Key umstellen:

```bash
# Manuell wechseln:
git config --global user.signingkey ~/.ssh/id_ed25519_sk_yk2.pub
```

**Automatisierung mit einem Wrapper-Skript** (`~/.local/bin/git-sign-detect`):

```bash
#!/bin/bash
# Erkennt welcher YubiKey eingesteckt ist und setzt den richtigen Key

YK_SERIAL=$(ykman info 2>/dev/null | grep "Serial" | awk '{print $NF}')

case "$YK_SERIAL" in
  "12345678")  # Serial deines YubiKey 1
    export GIT_SSH_SIGNING_KEY="$HOME/.ssh/id_ed25519_sk_yk1.pub"
    ;;
  "87654321")  # Serial deines YubiKey 2
    export GIT_SSH_SIGNING_KEY="$HOME/.ssh/id_ed25519_sk_yk2.pub"
    ;;
  *)
    echo "Kein bekannter YubiKey gefunden!" >&2
    exit 1
    ;;
esac

git -c user.signingkey="$GIT_SSH_SIGNING_KEY" "$@"
```

```bash
chmod +x ~/.local/bin/git-sign-detect
# Nutzung: git-sign-detect commit -m "Signierter Commit"
```

### 4.4 Beide Public Keys auf GitHub/GitLab registrieren

**GitHub:**
1. Settings → SSH and GPG keys → "New SSH key"
2. Key type: **Signing Key**
3. Beide Public Keys einzeln hinzufügen

**GitLab:**
1. Preferences → SSH Keys
2. Usage type: **Signing** (oder "Authentication and Signing")
3. Beide Public Keys einzeln hinzufügen

---

## 5. Übersichtstabelle

| Use Case | Wo wird registriert | Was wird registriert |
|----------|-------------------|---------------------|
| Mac Login (PIV) | macOS sc_auth pairing | Beide PIV-Zertifikate |
| SSH Login | `~/.ssh/authorized_keys` auf jedem Server | Beide SSH Public Keys |
| Git Signing | GitHub/GitLab + lokales allowed_signers | Beide SSH Public Keys |

---

## 6. Checkliste nach dem Setup

- [ ] YubiKey 1: PIN geändert (PIV + FIDO2)
- [ ] YubiKey 2: PIN geändert (PIV + FIDO2)
- [ ] Beide Keys für Mac Login gepairt
- [ ] SSH Resident Keys auf beiden Keys generiert
- [ ] Beide SSH Public Keys auf allen Servern hinterlegt
- [ ] Git Signing mit beiden Keys getestet (`git log --show-signature`)
- [ ] Beide Signing Keys auf GitHub/GitLab hochgeladen
- [ ] Backup-Key sicher aufbewahrt (z.B. Safe)

---

## 7. Troubleshooting

### YubiKey wird nicht erkannt
```bash
ykman info                  # Prüfe ob erkannt
sc_auth identities          # Prüfe Smart Card Identitäten
ssh-add -L                  # Prüfe geladene SSH Keys
```

### SSH fragt nicht nach Touch
```bash
# Prüfe ob der Key mit verify-required erstellt wurde
ykman fido credentials list
```

### Git Signing schlägt fehl
```bash
# Prüfe welcher Key konfiguriert ist
git config --global user.signingkey

# Test-Signatur
echo "test" | ssh-keygen -Y sign -f ~/.ssh/id_ed25519_sk_yk1.pub -n test
```

### PIV nach 3 falschen PINs gesperrt
```bash
# Mit PUK entsperren
ykman piv access unblock-pin
```
