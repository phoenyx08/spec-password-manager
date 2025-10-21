Here is a **technical description** for a programmer to implement **Your Very Own Password Manager (yvopassman)** according to the intent and requirements.

---

## **1. Overview**

**Project name:** Your Very Own Password Manager
**Executable name:** `yvopassman`
**Language:** C
**GUI toolkit:** GTK (GTK 4 recommended)
**Target OS:** Linux (Debian-based distributions)
**Distribution format:** `.deb` package (installable via `dpkg`)
**Data storage format:** Encrypted JSON file (local-only, no network connections)

The program is a simple native GTK desktop application for securely storing and managing passwords offline.

---

## **2. Functional Requirements**

### **2.1 Main Features**

1. **Password listing:**

   * On startup, display a list of saved password entries.
   * Each entry shows its **title** and three buttons:

      * `login`: copies the login to clipboard
      * `password`: copies the password to clipboard
      * `regenerate`: regenerates a new secure password and replaces the stored one

2. **Add password:**

   * Button labeled `Add Password`.
   * Opens a form dialog to enter:

      * `Title` (string, required)
      * `Login` (string, required)
   * System automatically generates a secure password and stores the new entry.

3. **Clipboard management:**

   * Copied data (login or password) should clear automatically after 15 seconds.

4. **Data persistence:**

   * All data stored in a single encrypted JSON file:
     `~/.local/share/yvopassman/data.json.enc`
   * File permissions restricted to the current user only (`chmod 600`).

5. **Encryption:**

   * AES-256 in GCM mode for authenticated encryption.
   * Encryption key derived using PBKDF2-HMAC-SHA256 from a user-defined master password stored via a keyring prompt at first run.
   * Master password cached in memory for the session only.

6. **Password generation:**

   * Randomly generate 16–20 character passwords using:

      * Uppercase/lowercase letters
      * Digits
      * Special characters
   * Use `/dev/urandom` as a source of entropy.

7. **No network access:**

   * The application must not initiate any network connection or use online services.

---

## **3. Data Structure**

### **3.1 Encrypted File Format**

* Stored at `~/.local/share/yvopassman/data.json.enc`
* AES-256-GCM encrypted binary blob

### **3.2 Internal JSON Schema (before encryption)**

```json
{
  "entries": [
    {
      "title": "example.com",
      "login": "user@example.com",
      "password": "MySecurePassword123!"
    }
  ]
}
```

---

## **4. Directory Structure**
 
* All deliverables, including Makefile, README.md etc., are stored in the `yvopassman` directory.
* Code sources are stored in `yvopassman/src`
* Installable package is stored in `yvopassman/build`

---

## **5. GTK UI Specification**

### **5.1 Main Window**

* Title: **Your Very Own Password Manager**
* Layout:

   * **Header bar:** App title + “Add Password” button.
   * **Main list view (GtkListBox):**

      * Rows dynamically populated from decrypted JSON entries.
      * Each row:

        ```
        [Title Label] [Login Button] [Password Button] [Regenerate Button]
        ```
   * **Footer:** (Optional) version label (e.g., "yvopassman v1.0")

### **5.2 Add Password Dialog**

* `GtkDialog` with fields:

   * `GtkEntry` for “Title”
   * `GtkEntry` for “Login”
   * “Cancel” and “Save” buttons
* On save:

   * Validate inputs
   * Generate secure password
   * Append to entries list
   * Save encrypted JSON to file

---

## **6. Security Considerations**

* Use **`mlock()`** (if available) to prevent the master password from being swapped to disk.
* Clipboard should be cleared after 15 seconds using a `GTimeout` callback.
* File permissions enforced via `chmod(600)` after every save.
* Do not log sensitive data.
* No network sockets or HTTP requests.

---

## **7. Build and Packaging**

### **7.1 Dependencies**

* GTK 4 (`libgtk-4-dev`)
* JSON-C (`libjson-c-dev`)
* OpenSSL (`libssl-dev`)
* pkg-config
* dpkg-dev, debhelper (for packaging)

## **8. User Documentation**

### **8.1 Installation**

```bash
sudo dpkg -i yvopassman_1.0_amd64.deb
```

### **8.2 Usage**

* Launch from application menu (“Your Very Own Password Manager”).
* Add new password entries or copy existing ones.
* Clipboard clears automatically after 15 seconds.

### **8.3 Uninstallation**

```bash
sudo apt remove yvopassman
```

---

## **9. Testing**

### **9.1 Functional Tests**

* Add, edit, and regenerate passwords.
* Clipboard copy behavior and auto-clear timing.
* File creation and encryption/decryption correctness.

### **9.2 Security Tests**

* Ensure encrypted file is unreadable without master key.
* Confirm no external network connections (`lsof -i` check).
* Validate permissions on data file.
