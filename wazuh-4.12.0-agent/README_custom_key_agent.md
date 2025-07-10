# Using a Custom AES Key on the Wazuh Agent

This document explains how to adapt the Wazuh agent to use a custom AES key, ensuring compatibility with a manager that uses custom key handling. It covers how the agent currently loads, generates, derives, and uses keys, and what you must change for a custom setup.

---

## 1. How the Agent Currently Handles Keys

### a. Key Loading from File

- The agent reads its key from a file defined by the macro `KEYS_FILE` (see `src/headers/defs.h`).
- The function responsible is `OS_ReadKeys` in `src/os_crypto/shared/keys.c`:

```c
fp = wfopen(keys_file, "r");
...
strncpy(key, valid_str, KEYSIZE - 1);
OS_AddKey(keys, id, name, ip, key, 0);
```

### b. Key Generation (if not provided)

- If a key is not provided, it is generated using agent info, time, and random numbers, then hashed with MD5.
- Example from `src/addagent/manage_agents.c`:

```c
os_snprintf(str1, STR_SIZE, "%d%s%d", (int)(time3 - time2), name, (int)rand1);
os_snprintf(str2, STR_SIZE, "%d%s%s%d", (int)(time2 - time1), ip, id, (int)rand2);
OS_MD5_Str(str1, -1, md1);
OS_MD5_Str(str2, -1, md2);
snprintf(key, 65, "%s%s", md1, md2);
```

### c. Key Derivation for Encryption

- The loaded or generated key is further processed in `OS_AddKey`:

```c
OS_MD5_Str(name, -1, filesum1);
OS_MD5_Str(id, -1, filesum2);
snprintf(_finalstr, sizeof(_finalstr), "%s%s", filesum1, filesum2);
OS_MD5_Str(_finalstr, -1, filesum1);
filesum1[15] = '\0';
filesum1[16] = '\0';
OS_MD5_Str(key, -1, filesum2);
snprintf(_finalstr, sizeof(_finalstr), "%s%s", filesum2, filesum1);
os_strdup(_finalstr, keys->keyentries[keys->keysize]->encryption_key);
```
- The result is stored as `encryption_key` and used for AES operations.

### d. Key Usage in Encryption/Decryption

- The derived key is used in functions like `OS_AES_Str` (see `src/os_crypto/aes/aes_op.c`):

```c
int OS_AES_Str(const char *input, char *output, const char *charkey, long size, short int action);
```
- The `charkey` argument is set to the derived key.

---

## 2. How to Implement Custom Key Handling on the Agent

### a. Using a Raw/Custom Key (Bypassing Derivation)

- If the manager uses a raw key (no MD5 derivation), the agent must do the same.
- In `OS_AddKey` (in `src/os_crypto/shared/keys.c`), replace the MD5 derivation block with:

```c
os_strdup(key, keys->keyentries[keys->keysize]->encryption_key);
```
- This ensures the agent uses the key exactly as provided by the manager.

### b. Key File Format

- Ensure the agent’s key file matches the manager’s format (e.g., `id name ip key`).
- If you change the format, update the parsing logic in `OS_ReadKeys`.

### c. Key Length and Format

- For AES-256, the key must be 32 bytes.
- If the manager provides a key of a different length, ensure both sides agree on the length and padding/truncation logic.

### d. Key Usage in Code

- When calling `OS_AES_Str`, pass the raw key as `charkey`:

```c
OS_AES_Str(input, output, my_key, size, OS_ENCRYPT);
```
- If you use a custom IV, update the function to accept and use it.

---

## 3. Agent Installation/Enrollment Steps (with Custom Key Logic)

1. **Receive or generate the key** using the new logic (from the manager or during enrollment).
2. **Store the key** in the agent’s key file in the new format.
3. **Update the agent’s code** to use the key as intended (raw or derived).
4. **Test communication** between agent and manager to confirm compatibility.

---

## 4. Example: Minimal Agent-Side Code Change

**Before (default, with MD5 derivation):**
```c
// In OS_AddKey
OS_MD5_Str(name, -1, filesum1);
OS_MD5_Str(id, -1, filesum2);
snprintf(_finalstr, sizeof(_finalstr), "%s%s", filesum1, filesum2);
OS_MD5_Str(_finalstr, -1, filesum1);
filesum1[15] = '\0';
filesum1[16] = '\0';
OS_MD5_Str(key, -1, filesum2);
snprintf(_finalstr, sizeof(_finalstr), "%s%s", filesum2, filesum1);
os_strdup(_finalstr, keys->keyentries[keys->keysize]->encryption_key);
```

**After (bypass derivation, use raw key):**
```c
// In OS_AddKey
os_strdup(key, keys->keyentries[keys->keysize]->encryption_key);
```

---

## 5. Where Keys Are Stored

- The file responsible for storing (writing) keys is defined by the macro `KEYS_FILE`.
- `KEYS_FILE` is defined in `src/headers/defs.h` as:

```c
#define KEYS_FILE       "etc/client.keys"
#define KEYS_FILE       "client.keys"
```

- Depending on your build or platform, `KEYS_FILE` will point to either `etc/client.keys` or `client.keys`.

---

## 6. Source Files for Code Snippets

- **src/os_crypto/shared/keys.c**
  - Key loading, derivation, and writing.
- **src/addagent/manage_agents.c**
  - Key generation if not provided.
- **src/headers/defs.h**
  - Definition of the key storage file macro.
- **src/os_crypto/aes/aes_op.c** (function signature reference)
  - Usage example for direct key usage in code.

---

## 7. Summary Table

| Step                | Default Wazuh                | With Custom Key Handling         |
|---------------------|------------------------------|----------------------------------|
| Key Generation      | MD5-based derivation         | Your logic (raw, custom, etc.)   |
| Key File Format     | id name ip key               | Must match your new format       |
| Key Usage in Code   | Derived key                  | Raw or custom key                |
| Agent Code Change   | None                         | Update to match manager logic    |

---

## 8. Notes

- The IV (Initialization Vector) is currently hardcoded. For best security, consider making it configurable.
- Always ensure your key is 32 bytes for AES-256.
- If you modify the key handling, review all usages to ensure compatibility.

---