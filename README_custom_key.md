# Using a Custom AES Key in Wazuh

This document explains how to introduce your own AES key for encryption in Wazuh, how the key is currently loaded or generated in the codebase, and where keys are stored.

---

## 1. How the Key is Currently Loaded or Generated

### a. Key Loading from File
- Wazuh reads keys from a file (usually `KEYS_FILE`).
- Each line contains agent information and a key.
- The function responsible is `OS_ReadKeys` in `src/os_crypto/shared/keys.c`:

```c
fp = wfopen(keys_file, "r");
...
strncpy(key, valid_str, KEYSIZE - 1);
OS_AddKey(keys, id, name, ip, key, 0);
```

### b. Key Generation (if not provided)
- If a key is not provided, it is generated using agent info, time, and random numbers, then hashed with MD5.
- Example from agent management code:

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
filesum1[15] = '\\0';
filesum1[16] = '\\0';
OS_MD5_Str(key, -1, filesum2);
snprintf(_finalstr, sizeof(_finalstr), "%s%s", filesum2, filesum1);
os_strdup(_finalstr, keys->keyentries[keys->keysize]->encryption_key);
```
- The result is stored as `encryption_key` and used for AES operations.

---

## 2. How to Introduce Your Own AES Key

### a. Direct Usage in Code
- The AES functions accept a key as the `charkey` argument.
- You can provide your own key (must be 32 bytes for AES-256) when calling `OS_AES_Str`:

```c
const char *my_key = "your-32-byte-long-key-goes-here......";
OS_AES_Str(input, output, my_key, size, OS_ENCRYPT);
```

### b. Modifying Key Loading
- To always use your own key, you can:
  1. Modify the code where `OS_AES_Str` is called to pass your key.
  2. Or, replace the key in the key file with your desired value.

### c. (Optional) Bypass Key Derivation
- If you want to use your key directly (without MD5 derivation), you can modify `OS_AddKey` to skip the MD5 processing and store your key as `encryption_key`:

```c
// In OS_AddKey, replace the MD5 derivation with:
os_strdup(key, keys->keyentries[keys->keysize]->encryption_key);
```

---

## 3. Where Keys Are Stored

- The file responsible for storing (writing) keys is defined by the macro `KEYS_FILE`.
- The main function that writes keys is `OS_WriteKeys` in `src/os_crypto/shared/keys.c`:

```c
int OS_WriteKeys(const keystore *keys) {
    ...
    if (TempFile(&file, KEYS_FILE, 0) < 0)
        return -1;

    for (i = 0; i < keys->keysize; i++) {
        keyentry *entry = keys->keyentries[i];
        fprintf(file.fp, "%s %s %s %s\\n", entry->id, entry->name, OS_CIDRtoStr(entry->ip, cidr, IPSIZE) ? entry->ip->ip : cidr, entry->raw_key);
    }
    ...
    if (OS_MoveFile(file.name, KEYS_FILE) < 0) {
        goto error;
    }
    ...
}
```

- `KEYS_FILE` is defined in `src/headers/defs.h` as:

```c
#define KEYS_FILE       "etc/client.keys"
#define KEYS_FILE       "client.keys"
```

- Depending on your build or platform, `KEYS_FILE` will point to either `etc/client.keys` or `client.keys`.

---

## 4. Summary Diagram

```mermaid
flowchart TD
    A[Does key file exist?] -- Yes --> B[Read key from file]
    A -- No --> C[Generate key from agent info, time, random, MD5]
    B & C --> D[Derive encryption key using MD5 hashes]
    D --> E[Store in memory (keys->keyentries[id]->encryption_key)]
    E --> F[Pass as charkey to OS_AES_Str for AES operations]
```

---

## 5. Notes
- The IV (Initialization Vector) is currently hardcoded. For best security, consider making it configurable.
- Always ensure your key is 32 bytes for AES-256.
- If you modify the key handling, review all usages to ensure compatibility.

---