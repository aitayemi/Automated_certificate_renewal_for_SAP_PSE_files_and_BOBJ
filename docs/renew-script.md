# `sap_pse_renew_<env>.sh` - certificate download and local PSE update

Runs on hosts where simple-acme works (RHEL 9). Two jobs:

1. ask the CA for a new certificate (or confirm the current one is still valid) via
   simple-acme and write the PEM files to `PEM_DIR` (`/usr/sap/acme-certs`);
2. replace the own certificate and key in every matching PSE on that host.

> **Read [Known gap: PK list handling](#known-gap-pk-list-handling) before relying on
> job 2.** The import script ([import-script.md](import-script.md)) is the more
> careful implementation of the same job.

## Usage

```
sap_pse_renew_<env>.sh setup      # one-time: register the ACME account, create the renewal
sap_pse_renew_<env>.sh            # normal run (cron): renew, then update PSEs if the certificate changed
sap_pse_renew_<env>.sh --force    # force renewal at the CA (new certificate) and re-import
sap_pse_renew_<env>.sh --reimport # re-import the CURRENT certificate (no new certificate requested)
sap_pse_renew_<env>.sh --dry-run  # list the PSEs that would be updated, change nothing
```

Run as `<sid>adm`. The script refuses to run as root, because PSE credentials and
simple-acme's data belong to that user.

Exit codes: `0` ok, `1` fatal error, `2` one or more PSE updates failed (others may
have succeeded).

## Configuration (environment variables)

| Variable | Default | Meaning |
|---|---|---|
| `CERT_CN` | per environment | certificate name; also the CN a PSE's own certificate must have to be updated |
| `SIMPLE_ACME` | `/opt/simple-acme/simple-acme` | simple-acme executable |
| `ACME_BASEURI` | `https://acme.enterprise.sectigo.com` | ACME directory (the "Universal" account) |
| `ACME_EMAIL` | `sap-admins@edu` | account contact |
| `EAB_KID`, `EAB_HMAC` | empty | External Account Binding credentials; **required only for `setup`** |
| `ACME_VALIDATION_ARGS` | `--validation none` | domains are pre-validated at Sectigo |
| `ACME_FRIENDLY_NAME` | `sap-wildcard-<env>` | name of the renewal inside simple-acme |
| `SIMPLE_ACME_EXTRA` | empty | extra simple-acme options for setup and renew |
| `ACME_LD_LIBRARY_PATH` | empty | newer glibc/libstdc++ path for simple-acme on old hosts; applied to simple-acme only, never to `openssl` or `sapgenpse` |
| `ACME_CONFIG_DIR` | `/home/<sid>adm/.simple-acme` | simple-acme `ConfigurationPath` |
| `PEM_DIR` | `/usr/sap/acme-certs` | where simple-acme writes the PEM files |
| `SAP_BASE` | `/usr/sap` | SAP base directory |
| `PSE_PIN` | empty | PIN of the PSEs (empty for PIN-less PSEs) |
| `TRUST_BUNDLE` | empty | extra CA PEM re-added to each rebuilt PSE |
| `EXTRA_CA_FILE` | `/storage/tiger/acme-certs/usertrust.pem` | CA certificate missing from the served chain (USERTrust root); appended to the chain. Note the default is the base directory, not the per-environment one |
| `RESTART_ICM` | `0` | `1` = soft-exit each updated instance's ICM (SIGINT) |
| `PSE_ONLY` | empty | space-separated PSE file names to restrict the run to (testing) |
| `BACKUP_KEEP` | `5` | backup directories kept per instance |
| `STATE_FILE` | `$PEM_DIR/.last_imported.sha256` | SHA256 of the last imported certificate |
| `NODES`, `HOST_SHORT`, `SAPGENPSE` | derived | see [architecture.md](architecture.md#naming-conventions) |

The five renew scripts differ only in `CERT_CN`, `ACME_FRIENDLY_NAME` and one
comment.

## Flow

1. **`ensure_acme_config`** makes sure `settings.json` (next to the simple-acme
   binary) exists and its `ConfigurationPath` points to `ACME_CONFIG_DIR`. simple-acme
   has no command-line flag for this.
2. **`setup` mode** (one-time) calls simple-acme with `--accepttos --baseuri
   --emailaddress --eab-key-identifier --eab-key --source manual --host <CN>
   --friendlyname --validation none --store pemfiles --pemfilespath <PEM_DIR>
   --installation none` and exits.
3. Instance directories are discovered; abort if none exists.
4. **`simple-acme --renew --baseuri ...`** runs (adds `--force` with `--force`). The
   output is scanned for `No scheduled renewals found`, which means `setup` was
   never run for this user.
5. The three PEM files must exist; the **SHA256 of the certificate** is compared to
   `STATE_FILE`. Unchanged means "nothing to do" and exit `0` (unless `--force`,
   `--reimport` or `--dry-run`).
6. **Sanity checks:** certificate and key public keys match; certificate not
   expiring within a day.
7. The key is converted to PKCS#8; a chain file is built: leaf, then
   `-chain-only.pem`, then `EXTRA_CA_FILE`.
8. **For every PSE of every instance** whose own certificate has `CN=<CERT_CN>`
   (and which is in `PSE_ONLY` if set): back up the PSE (and its `cred_v2`), delete
   it, run `sapgenpse import_p8 -c <chain> <key>`, then `post_import` (trust bundle
   and, for PIN PSEs, `seclogin`). On failure the backup is restored.
9. Old backup directories are pruned; the ICM is soft-exited if `RESTART_ICM=1`.
10. The new certificate hash is written to `STATE_FILE`, unless `PSE_ONLY` was used
    (a partial test run must not suppress later full runs).

## Known gap: PK list handling

Step 8 uses the original replace-the-PSE approach:

```
rm -f "$pse"
sapgenpse import_p8 -p "$pse" -x "$PSE_PIN" -c "$FULLCHAIN" "$P8"
```

Consequences, all consistent with the behaviour documented in
[import-script.md](import-script.md#what-sapgenpse-does-observed-behaviour):

* A new PSE starts with an **empty PK list**. After this step it holds only the
  certificates of `$FULLCHAIN` (wildcard, intermediates, USERTrust) plus whatever is
  in `TRUST_BUNDLE`. Every other entry that was in the old list (for example partner
  or SNC certificates, DigiCert/VeriSign roots, application-specific certificates)
  is gone.
* There is **no verification step**; success means only that `import_p8` returned 0.
* Because the renewed PSE now holds the current serial, the import script treats it
  as "already current" and **never repairs** the PK list.

### Recommended fix

Make the renew script **download-only** and let the import script perform all PSE
updates, on RHEL 9 hosts as well:

1. Add a switch to the renew script (for example `DOWNLOAD_ONLY=1`) that skips
   step 8 onwards, but still records the hash.
2. On those hosts run the import script with `CERTS_DIR=/usr/sap/acme-certs`, 30
   minutes after the renewal, exactly as RHEL 7 hosts do with the share.

This is also the end state planned for the RHEL 7 workaround (see
[architecture.md](architecture.md#host-roles)). Until it is done, check the PK
lists of PSEs on renewal hosts after each renewal (`sapgenpse maintain_pk -l`), and
use the PEM archive in the backup directory to restore missing entries.
