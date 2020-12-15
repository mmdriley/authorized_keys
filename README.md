# Manage SSH with Google Docs

Steps to use a Google Doc as your `authorized_keys` file across some servers.

Probably most appropriate in a deployment with a small numbers of users and servers.

## One-time setup

### Create the `authorized_keys` Google Doc

1.  Create a [new Google Doc](https://docs.new).

2.  Add the `authorized_keys` contents, e.g.:

    ```
    # computer1
    ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABAQC1LQMZJkoWNJrai7qJ6ZI7yqXTZijQd9E/onI01dR2bA1Mvrmbz/BL0tJIrwNxVpNCUn9Os4svPy9ITIrkKg6rlxHMwW1D9oEc7grrFaM2jvhaF/GrMKuD1gC+kRYW5eaZqdcP7njRO8+ciwVImb3sw+mSAvSKUcIvHby8yGEVU2I+p3I35YRSSN1KH+BFPQRE/jd0U4Qm1a5ZI5LWL6cUbFLv5OzHp8nun+BNQStxMe6bjHcXJRtH+8LxXs5meTTo/MOUSUgPIFSYlUF1ujHJio02NXJatlWn6t1IbHMm86JAc6uSOvQNUmEB0PbUdAbV8QCS9k84xz7AzpAJC/U3

    # computer2
    ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIAGDUSry9MFpslgCZhingWShnvszp9Aw7KuDlutVi+bl
    ```

3. Change the sharing settings for the document so "anyone with the link can **view**".

## Per-machine bringup

Assuming we'll be logging on as the user `sphinx`.

### Create the login user

1.  Create the user with a home directory

    ```bash
    useradd sphinx --shell /bin/bash --create-home
    ```

2.  Give the user passwordless `sudo` privilege

    Create `/etc/sudoers.d/sphinx-passwordless-sudo` containing:

    ```
    sphinx ALL=(ALL) NOPASSWD:ALL
    ```

    This line is explained in more detail [here](sudoers.md).

### Install `authorized_keys` script

1.  Download the script

    ```bash
    curl -L "${URL_FOR_SCRIPT}" -o /usr/local/download_authorized_keys

    # The script must be owned by, and only writable by, root.
    chmod 755 /usr/local/download_authorized_keys
    ```

2.  Configure the script

    In `download_authorized_keys`, set `GOOGLE_DOC_ID` to the ID of the document you created. For example, if the document URL is:
    ```
    https://docs.google.com/document/d/4oVJ6K5g2LOhqlgrblto5WYTasVebsPJGbsHSmVXNyQe/edit
    ```

    then set `GOOGLE_DOC_ID` as:
    ```
    GOOGLE_DOC_ID=4oVJ6K5g2LOhqlgrblto5WYTasVebsPJGbsHSmVXNyQe
    ```

3.  Create a user to run the script

    ```bash
    # Add a system user with no login privilege.
    useradd authorized_keys_command_user --system --shell /bin/false
    ```

### Configure `sshd`

We're going to require `publickey` authentication for all users and let `sphinx` log in with keys from the Google doc.

Add to the end of `/etc/ssh/sshd_config`:

```
Match all
  AuthenticationMethods publickey

Match User sphinx
  AuthorizedKeysCommand /usr/local/download_authorized_keys
  AuthorizedKeysCommandUser authorized_keys_command_user
```

We use `Match` blocks as an easy way to override values that might be set earlier in the file. That breaks if there are already `Match` blocks, but there usually aren't.

If we set `AuthorizedKeysCommand` but not `AuthorizedKeysCommandUser`, `sshd` will reject the config and fail to start.

### Read new `sshd` configuration

Make sure to keep an `ssh` session open in case `sshd` can't read the new configuration.

```bash
# Restart sshd and read sshd_config
service ssh restart

# Check that sshd_config was read successfully
service ssh status

# ... also test logging in as sphinx.
```

## Debugging

If `AuthorizedKeysCommand` is not respected, check permissions on `/usr/local` and perhaps fix them with `chmod 755 /usr/local`. If the permissions are wrong, you'll see lines like this in `/var/log/auth.log`:

```
Jul 29 08:06:31 hostname sshd[11457]: error: Unsafe AuthorizedKeysCommand "/usr/local/download_authorized_keys": bad ownership or modes for directory /usr/local
```

## Belt and suspenders

If you rely entirely on `AuthorizedKeysCommand` to download `authorized_keys` from Google Docs, you might lose SSH access to your hosts if:
- You lose access to your Google account
- The document is deleted
- Google is down
- Google changes the URL scheme for exporting documents as text

To account for this, you may want to create a "fallback" key and put it in `authorized_keys` on each host. This key should probably stay offline or be rooted at a Yubikey since it will be so hard to revoke.

See the page on [Doomsday Keys](doomsdaykeys.md) for one strategy for keeping backup credentials.
