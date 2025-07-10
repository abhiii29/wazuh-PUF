# Custom AES Key Implementation for Wazuh Manager

## Overview
This guide explains how to implement and use a custom AES key for encryption in the Wazuh manager. The goal is to hardcode a custom key (for demonstration) and optionally bypass the MD5 derivation step.

---

## 1. Key Handling Logic
- The manager loads keys from a file (`KEYS_FILE`) or generates them if not present.
- For a custom key, you will:
  - Hardcode a 32-byte key in the manager’s code.
  - (Optionally) Bypass the MD5 derivation in `OS_AddKey` so the key is used as-is.

---

## 2. Implementation Steps

### 2.1. Locate Key Handling Code
- File: `src/os_crypto/shared/keys.c`
- Function: `OS_AddKey` (key derivation and storage)
- File: `src/os_crypto/aes/aes_op.c` (AES operations)

### 2.2. Insert Hardcoded Key
- Define your 32-byte key in a suitable location (e.g., as a static const char*).
- Modify the logic to always use this key when calling `OS_AES_Str`.

### 2.3. Bypass MD5 Derivation (Optional)
- In `OS_AddKey`, replace the MD5 derivation block with:
  ```c
  os_strdup(key, keys->keyentries[keys->keysize]->encryption_key);
  ```
- This ensures the key is used as provided.

### 2.4. Key File Format
- Ensure the key file (defined by `KEYS_FILE` in `src/headers/defs.h`) contains the correct format:
  ```
  id name ip key
  ```
- The key should be 32 bytes for AES-256.

---

## 3. Example
```c
// Example: Hardcoded key in manager
const char *custom_aes_key = "your-32-byte-long-key-goes-here......";
OS_AES_Str(input, output, custom_aes_key, size, OS_ENCRYPT);
```

---

## 4. Key File Format and Storage
- The key file is defined by the `KEYS_FILE` macro in `src/headers/defs.h`:
  ```c
  #define KEYS_FILE "etc/client.keys"
  #define KEYS_FILE "client.keys"
  ```
- The format should be:
  ```
  id name ip key
  ```
- The key must be 32 bytes for AES-256.

---

## 5. Security Notes
- The IV (Initialization Vector) is currently hardcoded. For best security, consider making it configurable.
- Always use a secure, random 32-byte key for AES-256 in production.
- Review all usages of the key to ensure compatibility after changes.

---

## 6. Summary Table
| Step                | Default Wazuh                | With Custom Key Handling         |
|---------------------|------------------------------|----------------------------------|
| Key Generation      | MD5-based derivation         | Hardcoded (manager)              |
| Key File Format     | id name ip key               | id name ip key                   |
| Key Usage in Code   | Derived key                  | Raw or custom key                |
| Code Change         | None                         | Update to match new logic        |
