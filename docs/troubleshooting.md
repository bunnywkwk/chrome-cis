# Troubleshooting: chrome_cis

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

## T-C2. RHEL 8: "Could not detect a supported package manager from the following list: ['rpm']"

**When:** first test-project run on rhel8 (2026-10-08), task `PRELIM | AUDIT | Gather installed packages`. Screenshot:
`~/chrome_cis_test/docs/evidence/image.png`.

```text
[WARNING]: Requested package manager rpm was not usable by this module: Found executable at /bin/rpm. Failed to
import the required Python library (rpm) on rhel8's Python /usr/bin/python3.12. ...
[ERROR]: Task failed: Module failed: Could not detect a supported package manager from the following list: ['rpm'],
or the required Python library is not installed. Check warnings for details.
fatal: [rhel8]: FAILED! => {"changed": false, "msg": "Could not detect a supported package manager from the following
list: ['rpm'], or the required Python library is not installed. Check warnings for details."}
```

**Cause:** the run used a newer ansible-core (the `[ERROR]: Task failed: ... Origin:` format is 2.19+), not the
2.16 venv. ansible-core 2.17+ no longer supports Python 3.6 on targets, so on RHEL 8 it picks `/usr/bin/python3.12`.
The `rpm` (and `dnf`) Python bindings exist only for RHEL 8's system Python 3.6 (`/usr/libexec/platform-python`), so
`package_facts` (and later `dnf`) can't work. Same root cause as the control-node decision D15 in `mongodb8_cis`.

**Fix:** run from the 2.16 venv: `source ~/.ansible-2.16.1-env/bin/activate`, check `ansible --version` shows
`core 2.16.x`, rerun.

**Prevent:** always activate the 2.16 venv before running the test project (`docs/control-node-setup.md`); prelim
only asserts `>= 2.16.1`, so a newer core passes the check and fails here on RHEL 8.
