# Stalwart Administration

Stalwart runs on `stalwart.example.com`. SSH access via `provision@stalwart.example.com`, prefix commands with `mdo` for elevated privileges.

The CLI connects to the local HTTP admin API:

```sh
stalwart-cli -u http://localhost -c admin:change-me <command>
```

## Config

Config file: `/usr/local/etc/stalwart/config.toml`

List all config keys:

```sh
stalwart-cli -u http://localhost -c admin:change-me server list-config
```

List keys under a prefix:

```sh
stalwart-cli -u http://localhost -c admin:change-me server list-config <prefix>
```

Add or update a config key:

```sh
stalwart-cli -u http://localhost -c admin:change-me server add-config <key> <value>
```

Delete a config key:

```sh
stalwart-cli -u http://localhost -c admin:change-me server delete-config <key>
```

Reload config without restarting:

```sh
stalwart-cli -u http://localhost -c admin:change-me server reload-config
```

## Accounts

Stalwart uses LDAP as its user directory. Accounts cannot be created or modified through Stalwart — manage them in LDAP instead.

The LDAP `mail` attribute is used as the account name (one account per email address). Users must be members of `cn=mail,dc=group,dc=ldap` and have `userClass=active` to be recognised by Stalwart.

## DKIM

Stalwart auto-discovers DKIM signers by key name. The name **must** follow the pattern `rsa-{domain}` (e.g. `rsa-example.com`). A mismatch produces a `dkim.signer-not-found` warning and mail is sent unsigned.

> **Important:** `stalwart-cli server add-config` writes to the database, which is overridden by `config.toml` on restart. Always write `sign` config directly to `config.toml`.

### Add DKIM signing for a new domain

**1. Generate the key pair**

```sh
stalwart-cli -u http://localhost -c admin:change-me dkim create rsa example.com rsa-example.com mail
```

This creates a key named `rsa-example.com` internally.

**2. Get the public key for DNS**

```sh
stalwart-cli -u http://localhost -c admin:change-me dkim get-public-key rsa-example.com
```

Add the output as a DNS TXT record:

```
Name:  mail._domainkey.example.com
Type:  TXT
Value: v=DKIM1; k=rsa; p=<public-key>
```

Verify with:

```sh
drill TXT mail._domainkey.example.com
```

**3. Enable outbound signing**

Edit `config.toml` and add or update `[queue.outbound]`:

```toml
[queue.outbound]
sign = ["rsa-example.com"]
```

For multiple domains:

```toml
[queue.outbound]
sign = ["rsa-example.com", "rsa-other.com"]
```

Then restart Stalwart (a config reload is not sufficient for new keys):

```sh
service stalwart restart
```

**4. Verify**

Send a test email to an external address. In Gmail use **Show original** and confirm `dkim=pass` in the authentication results.

### List configured signatures

```sh
stalwart-cli -u http://localhost -c admin:change-me server list-config signature
```

### Notes

- `ed25519` keys follow the same pattern (`ed25519-{domain}`) but are optional. A warning for the ed25519 signer not found is harmless if only RSA is configured.
- Key files are stored under `/usr/local/etc/stalwart/`.

## Hostname

The SMTP banner hostname is set in `config.toml`:

```toml
[server]
hostname = "example.com"
```

Verify the SMTP banner:

```sh
echo "QUIT" | nc -w 3 stalwart.example.com 25
```

## Queue

Inspect the outbound mail queue:

```sh
stalwart-cli -u http://localhost -c admin:change-me queue list
```

## Restart

```sh
service stalwart restart
```
