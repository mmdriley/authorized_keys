# Download `authorized_keys`

* Arguments depend on the tokens configured in `AuthorizedKeysCommand`.

* The `download` script must be owned and only writable by `root`.

* `AuthorizedKeysCommand` cannot be used without `AuthorizedKeysCommandUser`.
  The latter should be set to a user created just for this purpose, e.g.:

  ```bash
  useradd --system --shell /bin/false --user-group authorized_keys_command_user
  ```
