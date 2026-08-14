# OMV SSH Authentication — Reboot & Troubleshooting Notes

## Overview

SSH access to an OpenMediaVault (OMV) server was configured using **Ed25519 public-key authentication** and the Windows OpenSSH client.

After a Windows reboot, an SSH authentication issue was investigated to verify that:

* The Windows SSH agent was functioning.
* The correct private key was loaded.
* The server contained the corresponding public key.
* The OMV user had SSH access.
* File ownership and permissions were correct.

The eventual cause was a **case-sensitive Linux username mismatch**, rather than a problem with the SSH key or SSH agent.

---

## Standard SSH Connection

Use the correct Linux username when connecting:

```powershell
ssh <username>@<server-ip>
```

Linux usernames may be case-sensitive. The username supplied to SSH must exactly match the account configured on the server.

For example, if the account is named `AdminUser`, attempting to authenticate as `adminuser` may fail because they can be treated as different usernames.

---

## SSH Key Files

The Windows OpenSSH client stores the Ed25519 key pair in the user's `.ssh` directory.

Using environment variables avoids exposing a specific Windows username:

```text
$env:USERPROFILE\.ssh\id_ed25519
$env:USERPROFILE\.ssh\id_ed25519.pub
```

The files serve different purposes:

* `id_ed25519` — **private key**
* `id_ed25519.pub` — **public key**

The private key must remain secret and should never be committed to a public repository, pasted into documentation, or otherwise publicly distributed.

The public key is intended to be installed on systems where authentication is required.

---

## Verify the Windows SSH Agent

After a Windows reboot, verify that the SSH agent has the expected key loaded:

```powershell
ssh-add -l
```

The output should identify the expected Ed25519 key.

If the expected key appears, the SSH agent has access to the private key.

A properly configured public-key login should then work without first performing a password-based login to the server:

```powershell
ssh <username>@<server-ip>
```

---

## OMV User Configuration

The OMV user used for SSH should have:

* Membership in the `_ssh` group.
* An interactive shell such as `/usr/bin/bash`.
* The appropriate SSH public key registered with the account.

Example configuration:

```text
Shell: /usr/bin/bash
Groups: _ssh, sudo, users
```

Administrative privileges such as membership in `sudo` should only be assigned when required for the account's intended purpose.

The public key can be managed through:

**OMV → Users → Users → Edit User → SSH public keys**

---

## Server-Side Authorized Keys

OpenSSH normally stores a user's authorized public keys in:

```text
/home/<username>/.ssh/authorized_keys
```

The file can be inspected from an authenticated server shell:

```bash
cat ~/.ssh/authorized_keys
```

Ownership and permissions can be checked with:

```bash
ls -ld ~/.ssh ~/.ssh/authorized_keys
```

The `authorized_keys` file should generally be writable only by the account owner. A typical permission setting is:

```text
-rw------- authorized_keys
```

which corresponds to mode `600`.

Improper ownership or overly permissive permissions can cause OpenSSH to reject an otherwise valid key.

---

## Diagnosing Authentication Problems

OpenSSH's verbose mode provides detailed authentication information:

```powershell
ssh -v <username>@<server-ip>
```

Useful diagnostic messages include:

```text
Offering public key:
```

and, when successful:

```text
Server accepts key:
```

If the client offers the expected key but the server rejects it, investigate the server-side configuration, including:

* Correct username
* User's SSH permissions
* Registered public key
* `authorized_keys`
* File ownership
* File permissions
* Effective SSH server configuration

---

## Testing Password Authentication

When troubleshooting, password authentication can be explicitly requested with:

```powershell
ssh -o PreferredAuthentications=password -o PubkeyAuthentication=no <username>@<server-ip>
```

This is useful for determining whether the server is actually permitting password authentication independently of public-key authentication.

Password authentication does not need to remain enabled when the intended security configuration uses SSH keys exclusively.

---

## Recovery Access with CTerm

OMV's CTerm service can provide temporary browser-based shell access when normal SSH access is unavailable.

For troubleshooting, CTerm can be temporarily enabled and the affected account added to the `cterm` group.

After SSH access is restored:

1. Remove the user from the `cterm` group.
2. Disable CTerm.
3. Disable CTerm's host-shell functionality.
4. Disable its reverse-proxy functionality if it was enabled.
5. Apply the OMV configuration changes.

This restores the server to its normal configuration without leaving an unnecessary management interface enabled.

---

## Root Cause

The SSH infrastructure itself was functioning correctly.

The Windows system:

* Had the expected Ed25519 private key.
* Had the SSH agent running.
* Had the expected key loaded.
* Successfully offered the key to the server.

The OMV server:

* Had SSH running.
* Had public-key authentication enabled.
* Had the correct public key installed.
* Had the user assigned to the SSH group.
* Had valid `authorized_keys` ownership and permissions.

The actual issue was **username capitalization**.

The failed connection used a username with incorrect capitalization:

```powershell
ssh <incorrect-case-username>@<server-ip>
```

Using the exact Linux account name immediately restored public-key authentication:

```powershell
ssh <correct-case-username>@<server-ip>
```

---

## Lessons Learned

SSH authentication failures do not necessarily indicate a damaged or missing key. Before replacing keys or resetting passwords, verify the entire authentication chain:

1. Confirm network connectivity to the server.
2. Confirm the exact username, including capitalization.
3. Verify the SSH agent is running.
4. Verify the expected key is loaded with `ssh-add -l`.
5. Use `ssh -v` to determine which key is being offered.
6. Confirm the server has the corresponding public key.
7. Verify `authorized_keys` ownership and permissions.
8. Check the server's effective authentication configuration.

Verbose SSH output can distinguish between **network connectivity**, **client-side key management**, and **server-side authentication** problems and can prevent unnecessary configuration changes.

## Security Notes

Public-facing documentation should avoid exposing unnecessary environment-specific information. Examples and screenshots should sanitize:

* Personal/local usernames
* Internal IP addresses
* VPN or overlay-network addresses
* Hostnames
* SSH key fingerprints
* Private directory structures when unnecessary
* Secrets, tokens, passwords, and private keys

The **contents of a private SSH key should never be published**.

Generic placeholders such as `<username>`, `<server-ip>`, and `<hostname>` allow the troubleshooting process to be documented without exposing unnecessary details about the environment.
