# Security notes and publishing checklist

## What is sensitive

| Item | Where | Notes |
|---|---|---|
| ACME EAB key identifier and HMAC key | Ansible notes, Sectigo Certificate Manager, screenshot of the account page in the process document | allow registering ACME accounts against the organisation's Sectigo contract. Needed only for `setup` |
| Wildcard **private key** | `/storage/tiger/acme-certs/<dir>/_.<zone>-key.pem` and `/usr/sap/acme-certs/` | one key for a whole landscape. A warning is logged when the share copy is readable by *other* (it was observed with mode 777) |
| PSE copies | `sec/backup/backup-*/` | PSEs contain private keys; the backups keep the original file modes |
| Tomcat keystore password | `KS_PASS` in `bobj_cert_import_<env>.sh` | in clear text in the scripts, also used for the temporary PKCS#12 |
| BOBJ CMC administrator credential | comments at the end of the BOBJ scripts | clear text |
| Copies of PSEs or keys in `/tmp` | operators' working directories | have been seen world-readable; delete after use |

## Recommendations

1. **Restrict the key on the share:** `chgrp <group-of-all-sidadm-users> <key>; chmod 640 <key>`.
   Do the same for the directory. The import script warns but does not change modes.
2. **Remove the credentials from scripts and notes.** Put the keystore password into a
   file readable only by `<sid>adm` (mode 600) and read it in the script; pass EAB
   credentials interactively or from a vault (`ansible-vault`), never in a file that is
   copied to every host. `install_simple_acme.sh` already avoids this on purpose.
3. **Rotate** anything that has been stored in clear text in a shared place, sent in a
   chat, or uploaded for documentation: the EAB credentials (create a new ACME account
   in Certificate Manager), the keystore passwords, and the CMC administrator password.
4. **Remove the CMC credential from the script comments.**
5. **Protect the backups:** `chmod 700 sec/backup` and delete old ones you do not need.
6. **Scripts already do:** write the converted key (`PKCS#8`) to a mode-600 temporary
   file and remove it on exit; keep the keystore password out of `ps` (environment
   variable); refuse to run as root; use `umask 077` in the BOBJ script.
7. **Known exposure:** a PIN given with `-x` appears on the `sapgenpse` command line
   while it runs. PIN-less PSEs (the normal case here) are not affected.

## Checklist before publishing this repository

Treat the repository as **private** unless every item below is done.

- [ ] Delete real values for `EAB_KID` and `EAB_HMAC` from the Ansible notes; use
      placeholders (`<EAB_KID>`, `<EAB_HMAC>`).
- [ ] Delete `KS_PASS` values from the four BOBJ scripts.
- [ ] Delete the CMC administrator credential from the BOBJ script comments.
- [ ] Do **not** add the Word documents or their screenshots; the ACME account page
      screenshot shows EAB data. Use this Markdown instead.
- [ ] Decide whether internal host names (`es1pas`, `jp1bobjcl1`, ...), the share path,
      the contact address and the domain `tiger.edu` may be public. Replace them with
      examples if not.
- [ ] Run a secret scanner over the working tree **and the history**
      (for example `gitleaks detect` or `trufflehog git file://.`). If a secret was ever
      committed, rotate it; removing the file is not enough.
- [ ] Have someone else read the documents once for anything environment specific.
