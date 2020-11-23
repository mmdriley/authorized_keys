# Passwordless `sudo`

To allow a user named `sphinx` to run commands as `root` using `sudo` without being prompted for their password, add this snippet to `/etc/sudoers` or to a file in `/etc/sudoers.d/`:

```
sphinx ALL=(ALL) NOPASSWD:ALL
```

Interpreting the line left to right, this means:
- the user `sphinx`
- whatever the identity of the current host
- can run, as any user
- without being prompted for their password
- any command.

The second `ALL` isn't _strictly_ necessary. It's a `Runas_Spec` that lets the user run commands as _any_ user using `sudo`. Without it, they can only run commands as `root` -- though from there, of course, they can run commands as any user with `su`.

This snippet matches the one [used in `cloud-init`](https://github.com/canonical/cloud-init/blob/f99d4f96b00a9cfec1c721d364cbfd728674e5dc/config/cloud.cfg.tmpl#L171).

## References

- [Understanding `sudoers(5)` syntax](https://toroid.org/sudoers-syntax)
- [How to setup passwordless `sudo` on Linux?](https://serverfault.com/a/160587)
- [`sudoers(5)` - Linux manual page](https://man7.org/linux/man-pages/man5/sudoers.5.html)
