# Debugging Notes — Manual Provisioning

This is a log of the actual issues hit while manually provisioning the 5-VM stack, and how each was diagnosed and fixed. Kept in the order they occurred.

---

## 1. Vagrant "machine is locked" error

**Symptom:**
```
Vagrant can't use the requested machine because it is locked!
```

**Diagnosis:** No stale lock file was present under `.vagrant/machines/app01/`, and VirtualBox itself showed the VM as "Powered Off" (not running) — so it wasn't a VM-state or file-lock issue. Checked for lingering `vagrant`/`ruby` processes via Task Manager and `ps aux`, found none.

**Fix:** A full machine restart cleared it — most likely an orphaned Windows-level file handle from an interrupted `vagrant up` that wasn't visible to Task Manager or `ps`.

---

## 2. web01 timing out on boot

**Symptom:**
```
Timed out while waiting for the machine to boot.
```

**Diagnosis:** Checked the Vagrantfile and found `vb.gui = true` set only on web01 (every other VM ran headless). This forces VirtualBox to open a visible GUI window, which interfered with Vagrant's boot detection.

**Fix:** Changed `vb.gui = true` → `false`, added `boot_timeout = 600` as a safety margin.

---

## 3. 502 Bad Gateway (Nginx → Tomcat)

**Symptom:** Browser showed `502 Bad Gateway` when hitting web01's IP.

**Diagnosis:**
```bash
# From web01
curl -I http://app01:8080
# → curl: (7) Failed to connect to app01 port 8080: No route to host
```
Instant rejection (not a timeout) pointed to a firewall block, not a routing issue. Confirmed on app01:
```bash
sudo firewall-cmd --list-all
# ports: <empty>
```
Port 8080 had never been opened.

**Fix:**
```bash
sudo firewall-cmd --zone=public --add-port=8080/tcp --permanent
sudo firewall-cmd --reload
```

---

## 4. "User Not Found" on login/registration

**Symptom:** App loaded fine (Nginx → Tomcat path worked), but login and registration both failed.

**Diagnosis:**
```bash
journalctl -u tomcat -f
# ERROR ... Cannot create PoolableConnectionFactory (Communications link failure)
```
Same pattern as issue #3 — checked db01's firewall:
```bash
sudo firewall-cmd --list-all
# ports: <empty>
```
Port 3306 was never opened either.

**Fix:**
```bash
sudo firewall-cmd --zone=public --add-port=3306/tcp --permanent
sudo firewall-cmd --reload
```
Also confirmed MariaDB was correctly bound to all interfaces (`ss -tlnp | grep 3306` → `*:3306`), ruling out a MySQL-config issue.

---

## 5. RabbitMQ connection error on registration

**Symptom:** App showed: `RabbitMQ server is off. Please start the RabbitMQ server and try again.`

**Diagnosis:** Same pattern again — rmq01's firewall had port 5672 unopened.

**Fix:**
```bash
sudo firewall-cmd --zone=public --add-port=5672/tcp --permanent
sudo firewall-cmd --reload
```

---

## 6. Memcache service crash (different root cause)

**Symptom:**
```bash
systemctl status memcached
# Active: failed (Result: exit-code)
# bind(): Cannot assign requested address
# failed to listen on TCP port 11211: Cannot assign requested address
```

**Diagnosis:** This one broke the pattern from #3–#5 — it wasn't a firewall issue, since the service wasn't even reaching a listening state. Checked the config directly:
```bash
cat /etc/sysconfig/memcached
# OPTIONS="-l 0.0.0.0,::1"
```
Memcache was configured to bind to both IPv4 (`0.0.0.0`) and IPv6 localhost (`::1`). The VM's IPv6 wasn't properly available, so the `::1` bind failed and took the whole service down with it.

**Fix:**
```bash
# Edited /etc/sysconfig/memcached
OPTIONS="-l 0.0.0.0"
```
```bash
systemctl restart memcached
```
Then opened the firewall port as well (same as the other services):
```bash
firewall-cmd --zone=public --add-port=11211/tcp --permanent
firewall-cmd --zone=public --add-port=11111/udp --permanent
firewall-cmd --reload
```

---

## Overall Pattern

4 of the 5 backend services (app01, db01, rmq01, and partially mc01) had firewall ports that were never opened, despite the setup guide including the `firewall-cmd` commands to do so — almost certainly because provisioning steps that ran once didn't get reliably re-applied after the reloads triggered by issues #1 and #2. The Memcache crash was a genuinely separate bug (an invalid IPv6 bind), correctly distinguished from the firewall pattern rather than assumed to be the same thing.
