# Custom AES Key Implementation for Wazuh Agent

## Overview
This guide explains how to implement and use a custom AES key for encryption in the Wazuh agent. The goal is to dynamically read the key from a file or environment variable (demonstrating liveness) and bypass the MD5 derivation step.

---

## 1. Key Handling Logic
- The agent loads its key from a file (`KEYS_FILE`) or generates it if not present.
- For liveness, the agent will read the key from a file or environment variable each time it is needed.

---

## 2. Implementation Steps

### 2.1. Locate Key Handling Code
- File: `src/os_crypto/shared/keys.c`
- Function: `OS_AddKey` (key derivation and storage)
- File: `src/os_crypto/aes/aes_op.c` (AES operations)

### 2.2. Implement Key Retrieval Function
- Create a function to read the key from a file (e.g., `/etc/wazuh-agent/aes.key`) or environment variable (`WAZUH_AGENT_AES_KEY`).
- Use this function in place of the static key.

### 2.3. Bypass MD5 Derivation
- In `OS_AddKey`, replace the MD5 derivation block with:
  ```c
  os_strdup(key, keys->keyentries[keys->keysize]->encryption_key);
  ```
- This ensures the key is used as provided.

### 2.4. Demonstrate Liveness
- Log or print the key value each time it is read.

---

## 3. Example
```c
// Example: Read key from file or env
char key[33] = {0};
if (getenv("WAZUH_AGENT_AES_KEY")) {
    strncpy(key, getenv("WAZUH_AGENT_AES_KEY"), 32);
} else {
    FILE *f = fopen("/etc/wazuh-agent/aes.key", "r");
    if (f) {
        fread(key, 1, 32, f);
        fclose(f);
    }
}
// Use key as needed
OS_AES_Str(input, output, key, size, OS_ENCRYPT);
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
| Key Generation      | MD5-based derivation         | Dynamic (agent)                  |
| Key File Format     | id name ip key               | id name ip key                   |
| Key Usage in Code   | Derived key                  | Raw or custom key                |
| Code Change         | None                         | Update to match new logic        | 