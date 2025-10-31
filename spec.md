# Technical specification — **Your Very Own Password Manager (yvopassman)**

Version: **0.0.1** — implementation-ready, prescriptive instructions for a C / GTK native Linux app packaged as a `.deb`.

---

## 1. Project summary

A native Linux password manager written in **C** with a **GTK** GUI, storing credentials in a single **locally-encrypted JSON file** (AES-256-GCM, key derived with PBKDF2-HMAC-SHA256 from a user master password entered via keyring prompt), packaged as a `.deb` and uninstallable so the encrypted file is removed on package removal.

---

## 2. Required stack & libraries

* Language: **C (C17)**
* GUI: **GTK 4** — `libgtk-4-dev`, `libglib2.0-dev`
* Keyring prompt: **libsecret (Secret Service API)** — `libsecret-1-dev`
* Crypto: **OpenSSL** — `libssl-dev`
* JSON: **jansson** (`libjansson-dev`)
* Clipboard: use **GdkClipboard** (via GDK/GTK)
* File locking: POSIX `flock` (`<sys/file.h>`)
* Build system: **Meson + Ninja** (meson provides good C integration and tests)
* Unit tests: **libcheck** (or CMocka) — use **libcheck** (`libcheck-dev`)
* Packaging: **dpkg-deb / debhelper** (`debhelper`, `dh-make`) for .deb packaging

> All instructions that follow will assume the above stack is available in the build environment.

---

## 3. High-level architecture

1. **UI layer (GTK)**

    * MainWindow: list of entries, buttons per entry, Add button, menu for Preferences / Quit.
    * Dialogs: AddEntry dialog (title, login fields), Confirm dialogs, Error dialogs.
2. **Core layer (C library `libyvopassman`)**

3. **App binary**: `yvopassman` GUI program linking GTK and core library.
4. **Packaging scripts**: deb control, `postinst`, `postrm` to register desktop file and remove storage on uninstall.

---

## 4. Data storage — exact format & location

* **Path (per-user):**
  `$XDG_DATA_HOME/yvopassman/passwords.json.enc`
  Fallback: `~/.local/share/yvopassman/passwords.json.enc` if `XDG_DATA_HOME` undefined.
* **File content:** single file: **base64-encoded JSON structure encrypted with AES-256-GCM**. On disk store a compact binary format (not plain JSON) containing metadata + ciphertext. The on-disk layout (binary, little-endian) — everything base64-encoded inside a small JSON wrapper to keep it text-friendly:

```json
{
  "version": "0.0.1",
  "kdf": {
    "algo": "PBKDF2-HMAC-SHA256",
    "salt": "<base64-salt>",          // 16 bytes
    "iterations": 200000
  },
  "enc": {
    "nonce": "<base64-nonce>",        // 12 bytes
    "tag": "<base64-auth-tag>",       // 16 bytes
    "ciphertext": "<base64-ciphertext>"
  }
}
```

* **Plaintext JSON (before encryption)** is a single JSON object:

```json
{
  "meta": {
    "created_at": "2025-10-27T12:34:56Z",
    "updated_at": "2025-10-27T12:45:00Z",
    "file_version": 1
  },
  "entries": [
    {
      "id": "<uuid-v4>",
      "title": "My Email",
      "login": "user@example.com",
      "password": "s3cureP@ssw0rd",
      "created_at": "2025-10-27T12:34:56Z",
      "updated_at": "2025-10-27T12:34:56Z"
    },
    ...
  ]
}
```

* The entire plaintext JSON is **UTF-8** and encrypted as one blob with AES-256-GCM. Store accompanying nonce and auth tag in wrapper above.

---

## 5. Cryptography — exact parameters & APIs

* **KDF:** `PBKDF2-HMAC-SHA256`

    * Salt length: **16 bytes** (generated with RAND_bytes).
    * Iterations: **200,000** (explicit constant `PBKDF2_ITERS = 200000`).
    * Derived key length: **32 bytes** (256-bit).
    * Use OpenSSL: `PKCS5_PBKDF2_HMAC` with `EVP_sha256()`.
* **Symmetric encryption:** `AES-256-GCM` (authenticated encryption)

    * Key: derived 32 bytes from PBKDF2.
    * Nonce (IV): **12 bytes** random per-encryption (RAND_bytes).
    * Tag: **16 bytes** (GCM auth tag).
    * Use OpenSSL EVP AES-GCM APIs: `EVP_CIPHER_CTX_new()`, `EVP_EncryptInit_ex(ctx, EVP_aes_256_gcm(), ...)`, `EVP_EncryptUpdate`, `EVP_EncryptFinal_ex`, `EVP_CIPHER_CTX_ctrl(..., EVP_CTRL_GCM_GET_TAG, 16, tag)`.
* **Security practices:**

    * Immediately `OPENSSL_cleanse()` or `explicit_bzero()` the master password buffer after computing key.
    * Keep derived key in memory only; mark memory non-swappable if possible (`mlock`) for the derived key (attempt `mlock`, but continue if denied).
    * Use atomic save and POSIX `flock` when writing the encrypted file.
* **No network connections**: do not call any network APIs or link network libraries intentionally.

---

## 6. Master password handling & key lifecycle

* **First run / unlock sequence**

    1. On startup, check for existence of encrypted file.

        * If file exists: present a keyring prompt (via **libsecret**) that asks user to enter master password. Do **not** store that master password in any persistent store. Use the prompt exclusively to capture the password string.
        * If file does not exist: present a keyring prompt asking the user to **create** a master password (twice for confirmation), then:

            * Generate salt (16 bytes).
            * Run PBKDF2 with above parameters to derive key.
            * Use key to encrypt an empty data structure (meta + empty entries), store resulting file (include salt in wrapper).
    2. Derive key with PBKDF2 and keep the derived key in process memory only. Do **not** persist the master password or derived key to disk. Keep master password variable overwritten and securely erased after key derivation.
    3. Cache the derived key **in memory for the session only** while the GUI is running. When the program exits, securely zero the derived key and release locked memory.

* **Notes about libsecret usage**

    * Use libsecret only to present the password entry dialog (Secret Service prompt). Do **not** use libsecret to store the password permanently. If desired, store a small sentinel in keyring to indicate that a master password is set (not required; the presence of the encrypted file is the sentinel).

---

## 7. GUI specification & behavior (detailed)

* **Main window**:

    * Title: `Your Very Own Password Manager` (menu and window title).
    * Primary widget: vertical list (`GtkListView` with `GListModel`) of entries.
    * Each row shows:

        * `title` (label)
        * `login` (secondary label, optionally hidden on first glance)
        * Three small buttons aligned on the right:

            * `login` — on click: copy the **login** value to clipboard (GdkClipboard). Then show a short toast/notification that login copied.
            * `password` — on click: copy the **password** value to clipboard. Then show toast/notification.
            * `regenerate` — on click: regenerate a new password (see password generation parameters below), replace the password in-memory and write the file to disk (atomic save). Update `updated_at`.
    * Global controls:

        * `+ Add` button (prominent) — opens AddEntry dialog.

* **AddEntry dialog**:

    * Fields: `Title` (text, required), `Login` (text, required). No password input field — password is automatically generated.
    * On submit: generate secure password; add entry with UUID v4; set created_at/updated_at; persist immediately by encrypting and atomically writing file.
* **Password generation**:

    * Default length: **20 characters**. Use cryptographically secure RNG.
    * Character classes: uppercase, lowercase, digits, symbols (include `!@#$%^&*()-_=+[]{}<>?`), ensure at least one of each class included.
    * Deterministic-seeding not allowed — use OpenSSL `RAND_bytes`.
* **Clipboard behavior (security)**:

    * After copying to clipboard, schedule **automatic clear** after **30 seconds** (use `g_timeout_add_seconds(30, callback_clear_clipboard, ...)`) to replace clipboard with an empty string. If system denies clearing, still zero local buffers.
    * Always securely zero any transient memory holding the copied password.
* **Regeneration behavior**:

    * On regenerate: create new password, replace entry in memory, write file, and show ephemeral notification "Password regenerated and copied" if desired; copy new password to clipboard.


---

## 8. Concurrency and file I/O

* Use `flock(LOCK_EX)` when writing and `flock(LOCK_SH)` when reading.
* Write atomically: write to temporary file in same directory (e.g., `passwords.json.enc.tmp`), fsync file descriptor, rename to final name using `rename()`, then fsync containing directory.
* Validate authentication tag on decrypt; if authentication fails, show a clear error dialog: "Incorrect master password or file corrupted."

---

## 9. Error handling & recovery

* If decrypt fails: do not overwrite file. Offer user to retry entering master password. Log events to a local logfile under `$XDG_STATE_HOME/yvopassman/log` or `~/.local/state/yvopassman/log` (rotated, limited size).
* If write fails: inform user and keep in-memory state consistent; attempt retries with exponential backoff up to a small limit.
* If storage file missing on startup: assume first-run; prompt to create master password.

---

## 10. Packaging (`.deb`) — file layout & scripts

* Package name: `yvopassman` (machine: `yvopassman`)
* Binary installed to `/usr/bin/yvopassman`
* Desktop file installed to `/usr/share/applications/yvopassman.desktop` with proper categories (`Utility;Security;`) so it appears in application list.
* Icons: `/usr/share/icons/hicolor/.../apps/yvopassman.png` (if provided)
* **Deb scripts**:

    * `debian/postinst`:

        * Run `update-desktop-database` and `gtk-update-icon-cache` if necessary (or `dh_install` covers most).
    * `debian/prerm` / `debian/postrm`:

        * On **purge** (i.e., package removed with purge flag), delete user's data file **only** for the local user? Packaging scripts run as root at system-level; we must remove the encrypted file for the system user who installed? The vision requires: *When the package is uninstalled the file storing the passwords should be deleted.* Implement this behavior as:

            * During package removal, remove system-wide installed user data under `/var/lib/yvopassman/*` **and** attempt to remove per-user files under `~/.local/share/yvopassman/passwords.json.enc` for every user in `/home/*`.
            * Concrete script (prescriptive): in `postrm` on `purge` action iterate `/home/*` and `root` and remove `$HOME/.local/share/yvopassman/passwords.json.enc` and `$XDG_DATA_HOME/yvopassman/passwords.json.enc` if present, and then remove `/var/lib/yvopassman` if used. Ensure script exercises caution and only deletes files under those app-specific directories. (This meets the Vision: remove file on uninstall.)
* **deb control dependencies**:

    * `Depends: ${shlibs:Depends}, ${misc:Depends}, libgtk-4-1, libsecret-1-0, libssl3, libjansson4` (name versions per distribution).
* **Build & package commands**:

    * `meson setup build`
    * `ninja -C build`
    * `DESTDIR=debian/tmp ninja -C build install`
    * Use `dpkg-deb --build debian/tmp yvopassman_0.0.1_amd64.deb` or use `debhelper` scripts to create final package.

---

## 11. Testing & CI (must include autotests)

* **Unit tests (libcheck)**:

    * `test_crypto`: PBKDF2 produces expected-length key; AES-GCM encrypt/decrypt roundtrip; tampered ciphertext / tag fails authentication.
    * `test_storage`: JSON <-> structure roundtrip; atomic write + read tests (simulate concurrent writes).
    * `test_pwgen`: generated passwords meet class coverage and length stats.
* **Integration tests**:

    * Headless GUI smoke: launch the app in an Xvfb environment (`xvfb-run`) and programmatically exercise: create entry, copy login/password, regenerate, exit. Use CLI test helper that uses DBus/AT-SPI if needed. Implement a simple built-in `--test-mode` that exposes test hooks (use only in CI).

* Tests must be runnable with `meson test`.

---

## 12. Logging and diagnostics

* Keep minimal logging to `~/.local/state/yvopassman/log` with timestamps. Do not log plaintext passwords or master password. Log only high-level events: "file saved", "decrypt failed", "invalid format".

---

## 14. Security considerations (must be implemented)

* Derived key and plaintext must be zeroed after use (`OPENSSL_cleanse` / `explicit_bzero`).
* Try to `mlock` the derived key memory; if unsuccessful, continue but log a warning.
* File permissions: ensure encrypted file mode is `0600`. Directory `~/.local/share/yvopassman` set `0700`.
* Avoid printing sensitive info to stdout/stderr or logs.
* Ensure clipboard clearing after 30s;

---

## 15. Uninstall & removal behavior

* On package removal (`postrm` stage `purge`), delete:

    * `/var/lib/yvopassman/*` (if used)
    * For each user in `/home/*` and `root`, remove `$HOME/.local/share/yvopassman/passwords.json.enc` and `$HOME/.local/share/yvopassman` directory if empty.
* Additionally, `postrm` should remove desktop entry and icons (handled by package manager).
* Document this behavior in README: uninstall will remove encrypted password file.

---

## 16. Documentation & deliverables

* **README.md** with:

    * Build instructions (dependencies, meson/ninja commands), run instructions.
    * User guide: install `.deb` via `dpkg -i`, launch app, how to add entries, change master password, uninstall behavior.
    * Security notes: master password in memory only, clipboard clearing.
* **man page** `yvopassman.1` documenting CLI flags (e.g., `--test-mode`, `--version`).
* **LICENSE** file (choose appropriate license; the Vision did not specify — include permissive license file or leave blank if undesired). *(If license must be chosen, include it in repo.)*

---

## 19. Final notes (strict to Vision)

* The program **must not** make any network connections. Avoid linking or calling any networking libraries.
* The encrypted file **must** be deleted when the package is uninstalled/purged.
* Master password entry uses a **keyring-style prompt** (libsecret) and the master password is **cached only in memory** for the session.
* Encryption uses **AES-256-GCM** and key derivation **PBKDF2-HMAC-SHA256** with **200,000 iterations** and **16-byte salt**.
