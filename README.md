# Ensure-Non-Default-Unique-Cryptographic-Material-is-in-Use

**Step 1 — Find where the SSL files are located**
```sql
-- Run in MariaDB to find the actual paths in use
SHOW VARIABLES LIKE '%ssl%';
```
Look for `ssl_cert`, `ssl_key`, `ssl_ca` variable values — these tell you the exact paths.

Or check the config files:
```bash
grep -iE "ssl_cert|ssl_key|ssl_ca|ssl_capath" /etc/mysql/*.cnf /etc/mysql/conf.d/* /etc/mysql/mariadb.conf.d/* 2>/dev/null
```

---

**Step 2 — Run the benchmark check on the certificate**
```bash
# Navigate to the data/ssl directory and check the certificate subject
openssl x509 -in /path/to/server-cert.pem -subject -noout
```

| Output | Status |
|---|---|
| Contains `Auto_Generated_Server_Certificate` | ❌ FAIL — default cert in use |
| Contains `Auto_Generated_CA_Certificate` | ❌ FAIL — default CA in use |
| Custom CN (e.g. company name, hostname) | ✅ PASS |
| No output / file not found | ⚠️ Check path |

---

**Step 3 — Check full certificate details**
```bash
# Full certificate inspection
openssl x509 -in /path/to/server-cert.pem -text -noout | grep -iE \
  "subject|issuer|not before|not after|serial|signature algorithm"
```

Look for:

| Field | What to check | Status |
|---|---|---|
| `Subject CN` | Should be your server hostname/company | ✅/❌ |
| `Issuer` | Should be your own CA, not MariaDB default | ✅/❌ |
| `Signature Algorithm` | Should be `sha256WithRSAEncryption` or better | ✅/❌ |
| `Not After` | Should not be expired | ✅/❌ |
| Serial number | Should be unique per certificate | ✅/❌ |

---

**Step 4 — Check certificate is unique to this instance**
```bash
# Get the certificate fingerprint
openssl x509 -in /path/to/server-cert.pem -fingerprint -noout

# Get the serial number
openssl x509 -in /path/to/server-cert.pem -serial -noout
```

Compare this fingerprint/serial against your other MariaDB nodes (Kenya and Ireland DCs). If they match it means the **same certificate is shared across instances** which is a fail.

| Finding | Status |
|---|---|
| Unique fingerprint per node | ✅ PASS |
| Same fingerprint across multiple nodes | ❌ FAIL |

---

**Step 5 — Check the CA certificate**
```bash
openssl x509 -in /path/to/ca-cert.pem -subject -noout
openssl x509 -in /path/to/ca-cert.pem -text -noout | grep -iE "subject|issuer|not after"
```

| Finding | Status |
|---|---|
| CA is your own internal/organisational CA | ✅ PASS |
| CA subject contains `Auto_Generated_CA_Certificate` | ❌ FAIL |
| CA is shared with non-MariaDB systems inappropriately | ❌ FAIL |

---

**Step 6 — Check key strength**
```bash
# Check the private key size
openssl rsa -in /path/to/server-key.pem -text -noout 2>/dev/null | grep "Private-Key"
# or for EC keys
openssl ec -in /path/to/server-key.pem -text -noout 2>/dev/null | grep "ASN1 OID"
```

| Key type | Minimum acceptable | Status |
|---|---|---|
| RSA | 2048 bit (3072+ preferred) | ✅/❌ |
| EC | 256 bit (P-256 or better) | ✅/❌ |
| RSA 1024 bit | Too weak | ❌ FAIL |

---

**Overall pass/fail criteria:**

| Check | Required for PASS |
|---|---|
| Certificate is not auto-generated | ✅ Mandatory |
| Certificate CN matches server identity | ✅ Mandatory |
| Certificate is unique per instance | ✅ Mandatory |
| CA is not MariaDB default | ✅ Mandatory |
| Certificate is not expired | ✅ Mandatory |
| Key strength is adequate | ✅ Mandatory |
| Cryptographic material not shared across instances | ✅ Mandatory |

Start with Step 1 to get the actual SSL file paths from the running MariaDB instance, then run Step 2 — the benchmark check will give you an immediate pass or fail on the auto-generated certificate question.

Run these commands in sequence on the DB server:

---

**Block 1 — Auto-generated certificate check (the benchmark test)**
```bash
openssl x509 -in /etc/my.cnf.d/certs/server.crt -subject -noout
```
- Contains `Auto_Generated` → ❌ FAIL immediately
- Does not contain `Auto_Generated` → continue to next checks

---

**Block 2 — Certificate validity and identity**
```bash
openssl x509 -in /etc/my.cnf.d/certs/server.crt -text -noout | grep -iE "subject|issuer|not before|not after|signature algorithm|serial"
```
Looking for:
- CN is your server hostname or organisation name
- Not expired (`Not After` is in the future)
- Signature algorithm is `sha256WithRSAEncryption` or better
- Issuer is your own internal CA

---

**Block 3 — Key strength**
```bash
openssl rsa -in /etc/my.cnf.d/certs/server.key -text -noout 2>/dev/null | grep "Private-Key"
```
- 2048 bit minimum → ✅
- Less than 2048 bit → ❌ FAIL

---

**Block 4 — CA certificate check**
```bash
openssl x509 -in /etc/my.cnf.d/certs/ca.crt -subject -noout
openssl x509 -in /etc/my.cnf.d/certs/ca.crt -enddate -noout
```
- Contains `Auto_Generated` → ❌ FAIL
- Expired → ❌ FAIL

---

**Block 5 — Uniqueness check across your Galera nodes**
```bash
# Run on each node and compare outputs — they must all be different
openssl x509 -in /etc/my.cnf.d/certs/server.crt -fingerprint -sha256 -noout
```
```bash
openssl x509 -in /etc/my.cnf.d/certs/server.crt -serial -noout
```
- Same fingerprint on multiple nodes → ❌ FAIL
- Unique per node → ✅ PASS

---

**Decision tree — work top to bottom, stop at first fail:**

```
Is the cert auto-generated?          YES → FAIL, stop here
                                      NO → continue
```
```
Is the cert expired?                 YES → FAIL, stop here
                                      NO → continue
```
```
Is the CN your own identity?          NO → FAIL, stop here
                                     YES → continue
```
```
Is the key 2048 bit or stronger?      NO → FAIL, stop here
                                     YES → continue
```
```
Is the CA auto-generated?            YES → FAIL, stop here
                                      NO → continue
```
```
Is the cert unique per Galera node?   NO → FAIL, stop here
                                     YES → PASS
```
