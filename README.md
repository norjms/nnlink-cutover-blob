# nnlink-cutover-blob

`nnlink-cutover.enc` is an **AES-256-CBC (PBKDF2, 600k iter)** encrypted tar of the
cutover keys, the eero desired-state, and the topology reports. It is **ciphertext
only** - useless without the passphrase, which is held offline (1Password).

Decrypt:
```
openssl enc -d -aes-256-cbc -pbkdf2 -iter 600000 -in nnlink-cutover.enc -pass pass:'<passphrase>' | tar -xf -
```
Runbook: claude.ai Project doc `claude/cutover-runbook.md`.
