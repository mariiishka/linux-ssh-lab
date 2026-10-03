# How to Set Up SSH Securely

A short guide to logging in to a Linux server with keys and disabling password and root login. Tested on Ubuntu/Debian with OpenSSH. A detailed breakdown of common failures and how I diagnosed them is in [notes.md](notes.md).

> In the examples, `user` is the username on the server and `SERVER_IP` is the server's address. Replace them with your own values.

## The Golden Rule

**Do not close your working session until you have verified the new connection in a separate terminal window.** That way, even if you make a mistake in the configuration, you won't lose access. If your server has a console in the hosting or virtualization panel, that is another fallback.

## 1. Generate a Key (on your own machine)

```bash
ssh-keygen -t ed25519 -C "my-laptop"
```

- The private key (`~/.ssh/id_ed25519`) stays with you only and is never copied anywhere.
- The public key (`~/.ssh/id_ed25519.pub`) is placed on the server.
- Set a passphrase: it protects the key if the file ends up in the wrong hands.

## 2. Install the Public Key on the Server

```bash
ssh-copy-id user@SERVER_IP
```

If `ssh-copy-id` is not available, add the contents of the `.pub` file to `~/.ssh/authorized_keys` on the server manually (one key per line).

## 3. Verify Key-Based Login

```bash
ssh user@SERVER_IP
```

The server should not ask for the user's password (only the key's passphrase, if you set one). **Do not move on to the next step until this works.**

## 4. Disable Password and Root Login

On the server, edit `/etc/ssh/sshd_config`:

```
PasswordAuthentication no
KbdInteractiveAuthentication no
PermitRootLogin no
```

On older OpenSSH versions, `ChallengeResponseAuthentication` may be used instead of `KbdInteractiveAuthentication`.

**Check that the settings are not overridden.** The main file may include additional files from `/etc/ssh/sshd_config.d/`, and they can change values:

```bash
sudo grep -r PasswordAuthentication /etc/ssh/
sudo sshd -T | grep -E 'passwordauthentication|permitrootlogin'
```

The second command shows the values that are actually in effect.

## 5. Check the Syntax and Apply

```bash
sudo sshd -t                      # empty output = no errors
sudo systemctl reload ssh         # on some systems the service is called sshd
```

## 6. Verify the Result in a NEW Terminal Window

```bash
ssh user@SERVER_IP                # key-based login should work

# password login should be rejected:
ssh -o PubkeyAuthentication=no -o PreferredAuthentications=password user@SERVER_IP
# expected: Permission denied (publickey)
```

Only after that is it safe to close the old session.

## 7. Convenience: an Alias in `~/.ssh/config`

```
Host myserver
    HostName SERVER_IP
    User user
    IdentityFile ~/.ssh/id_ed25519
```

After that, `ssh myserver` is enough.

## Permissions

SSH refuses to work if permissions are too open:

| What                                     | Permissions |
| ---------------------------------------- | ----------- |
| `~/.ssh` (directory)                     | `700`       |
| `~/.ssh/authorized_keys` (on the server) | `600`       |
| Private key                              | `600`       |

## Troubleshooting

| Symptom                                  | What to check                                                                                 |
| ---------------------------------------- | --------------------------------------------------------------------------------------------- |
| `Permission denied (publickey)`          | Is the public key in `authorized_keys`; permissions on `~/.ssh`; is it the right user         |
| `WARNING: UNPROTECTED PRIVATE KEY FILE!` | Private key permissions: `chmod 600`                                                          |
| A change in `sshd_config` has no effect  | Files in `/etc/ssh/sshd_config.d/`, the output of `sshd -T`, whether the service was reloaded |
| Unclear reason for rejection             | Client: `ssh -v user@SERVER_IP`. Server: `sudo journalctl -u ssh -n 50`                       |

## What Not to Do

- Do not copy or publish your private key. If it leaks, immediately remove its public half from `authorized_keys`.
- Do not disable password login before you have verified key-based login.
- Do not change `sshd_config` without running `sshd -t` before reloading the service.
- Do not commit private keys, real IP addresses, or secrets to the repository.
