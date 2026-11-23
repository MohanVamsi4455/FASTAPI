# Setting Up a New VM User (Password Login) — Explained

Step-by-step walkthrough of `login_user_vm_set_up_username_and_password`: how to create a new
non-root user on the EC2 instance and enable password-based SSH login for that user (instead of
only PEM-key auth).

## 1. Connect using the PEM key

Connect as the default `ubuntu` user first, since the new user doesn't exist yet.

```bash
ssh -i "D:\Aws_PEM_KEYS\FASTAPI.pem" ubuntu@YOUR_EC2_PUBLIC_IP
```

## 2. Create the new user

Create the `mohan` account that will own and run the application.

```bash
sudo adduser mohan
```

## 3. Enable password authentication

By default, AWS Ubuntu AMIs disable SSH password login (cloud-init ships a config that forces
key-only auth). This flips that setting to `yes` so `mohan` can log in with a password, not just a
key.

```bash
sudo sed -i 's/^PasswordAuthentication no/PasswordAuthentication yes/' /etc/ssh/sshd_config.d/60-cloudimg-settings.conf
```

## 4. Validate the SSH config

Check the sshd configuration syntax before restarting the service — catches typos/mistakes before
they lock you out.

```bash
sudo sshd -t
```

## 5. Restart SSH

Apply the new config by restarting the SSH daemon.

```bash
sudo systemctl restart ssh
```

## 6. Verify password authentication is active

Dump the effective sshd config and confirm `PasswordAuthentication` (and
`KbdInteractiveAuthentication`, which some clients use instead) are now set to `yes`.

```bash
sudo sshd -T | grep -Ei 'passwordauthentication|kbdinteractiveauthentication'
```

## 7. Set the user's password

`adduser` creates the account but may leave the password unset/locked; this sets (or resets) it
explicitly so password login actually works.

```bash
sudo passwd mohan
```

## 8. Verify the user exists

Confirm the `mohan` user and its UID/GID were created correctly.

```bash
id mohan
```

## 9. Grant root access

Add `mohan` to the `sudo` group so it can run privileged commands (equivalent to root access via
`sudo`).

```bash
sudo usermod -aG sudo mohan
```

> **⚠️ Security note:** enabling SSH password authentication weakens the server's attack surface
> compared to key-only login (password logins are brute-forceable over the internet). If this is a
> production box, consider reverting `PasswordAuthentication` to `no` once key-based access for
> `mohan` is set up, or at minimum use a strong password and fail2ban/rate limiting.
