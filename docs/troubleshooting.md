# Troubleshooting: chrome-cis

Every error hit while building or testing the role: exact output, cause, fix, prevent. IDs `T-Cn`.

## T-C1. google-chrome RPM: "key 1 import failed" during install

**When:** discovery step 2, `sudo dnf install -y https://dl.google.com/linux/direct/google-chrome-stable_current_x86_64.rpm`
on RHEL 8, 9 and 10 (2026-10-07). The install itself finished (`Complete!`).

```text
  Running scriptlet: google-chrome-stable-155.0.8059.39-1.x86_64                                       6/6
error: can't create transaction lock on /var/lib/rpm/.rpm.lock (Resource temporarily unavailable)
error: /tmp/google.sig.Uzg8ZC: key 1 import failed.
```

RHEL 10 shows the same with `/usr/lib/sysimage/rpm/.rpm.lock` (the rpm database moved there).
Afterwards `rpm -q gpg-pubkey --qf '%{VERSION}-%{RELEASE} %{SUMMARY}\n' | grep -i google` prints nothing.

**Cause:** the RPM's post-install script runs `rpm --import` for Google's signing key while `dnf` still holds the rpm
database lock, so the import fails. The package came from a URL (a local file to dnf), which dnf does not
signature-check by default, so nothing stopped the install.

**Fix (manual):** `sudo rpm --import https://dl.google.com/linux/linux_signing_key.pub`. The daily cron
`/etc/cron.daily/google-chrome` also retries the import.

**Prevent (role):** opt-in install imports the key first (`ansible.builtin.rpm_key`), then adds the repo
(`ansible.builtin.yum_repository`, same content as Google's file), then installs from the repo with `dnf`, so the
package is verified and the post-install import has nothing to do.
