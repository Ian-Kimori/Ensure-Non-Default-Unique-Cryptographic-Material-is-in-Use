# Ensure-Non-Default-Unique-Cryptographic-Material-is-in-Use

Here is the complete procedure with explicit pass/fail conditions for every single step:

---

**PART 1 — MariaDB database checks**

**1a. SSL enabled**
```sql
SHOW VARIABLES LIKE '%ssl%';
```
| Condition | Status |
|---|---|
| `have_ssl = YES` | ✅ PASS |
| `have_openssl = YES` | ✅ PASS |
| `have_ssl = NO` or `DISABLED` | ❌ FAIL |

Look for ssl_cert, ssl_key, ssl_ca variable values — these tell you the exact paths.

**1b. SSL actively in use**
```sql
SHOW STATUS LIKE 'Ssl_cipher';
```
| Condition | Status |
|---|---|
| Returns a cipher name e.g. `TLS_AES_256_GCM_SHA384` | ✅ PASS |
| Returns empty value | ❌ FAIL — SSL not active on current connection |

**1c. SSL enforced per user**
```sql
SELECT user, host, ssl_type 
FROM mysql.user 
WHERE user NOT IN ('mysql.infoschema','mysql.session','mysql.sys')
ORDER BY user;
```
| Condition | Status |
|---|---|
| `ssl_type = ANY` | ✅ PASS — SSL required |
| `ssl_type = X509` | ✅ PASS — client cert required |
| `ssl_type = SPECIFIED` | ✅ PASS — specific cipher required |
| `ssl_type` is empty | ❌ FAIL — SSL not enforced for that user |

---

**PART 2 — Server certificate checks**

**2a. Auto-generated certificate check**
```bash
openssl x509 -in /etc/my.cnf.d/certs/server.crt -subject -noout | grep Auto_Generated_Server_Certificate
```
| Condition | Status |
|---|---|
| Command returns no output | ✅ PASS — not auto-generated |
| Command returns a match | ❌ FAIL — default certificate in use |

**2b. Full certificate details**
```bash
openssl x509 -in /etc/my.cnf.d/certs/server.crt -text -noout | \
  grep -iE "subject|issuer|not before|not after|signature algorithm|serial number"
```
| Field | Pass condition | Fail condition |
|---|---|---|
| `Subject CN` | Your hostname or organisation name | Generic, MariaDB default, or empty |
| `Issuer` | Your own internal CA | MariaDB default CA |
| `Signature Algorithm` | `sha256WithRSAEncryption` or better | `sha1WithRSAEncryption` or weaker |
| `Not After` | Future date | Past date = expired |
| `Serial Number` | Any non-zero unique value | Same across multiple nodes |

**2c. Expiry check**
```bash
openssl x509 -in /etc/my.cnf.d/certs/server.crt -enddate -noout
```
| Condition | Status |
|---|---|
| Date is in the future | ✅ PASS |
| Date is today or in the past | ❌ FAIL — certificate expired |
| Expiry within 30 days | ⚠️ WARNING — renew immediately |

**2d. Key strength**
```bash
openssl rsa -in /etc/my.cnf.d/certs/server.key -text -noout 2>/dev/null | grep "Private-Key"
```
| Condition | Status |
|---|---|
| `Private-Key: (4096 bit)` | ✅ PASS — strong |
| `Private-Key: (3072 bit)` | ✅ PASS — strong |
| `Private-Key: (2048 bit)` | ✅ PASS — acceptable minimum |
| `Private-Key: (1024 bit)` | ❌ FAIL — too weak |
| `Private-Key: (512 bit)` | ❌ FAIL — critically weak |

**2e. Certificate fingerprint**
```bash
openssl x509 -in /etc/my.cnf.d/certs/server.crt -fingerprint -sha256 -noout
```
| Condition | Status |
|---|---|
| Fingerprint is different on every node | ✅ PASS |
| Same fingerprint found on two or more nodes | ❌ FAIL — shared certificate |

**2f. Serial number**
```bash
openssl x509 -in /etc/my.cnf.d/certs/server.crt -serial -noout
```
| Condition | Status |
|---|---|
| Serial number is different on every node | ✅ PASS |
| Same serial number on two or more nodes | ❌ FAIL — same certificate reused |

---

**PART 3 — CA certificate checks**

**3a. CA auto-generated check**
```bash
openssl x509 -in /etc/my.cnf.d/certs/ca.crt -subject -noout | grep Auto_Generated_CA_Certificate
```
| Condition | Status |
|---|---|
| Returns no output | ✅ PASS — not auto-generated |
| Returns a match | ❌ FAIL — default CA in use |

**3b. CA full details**
```bash
openssl x509 -in /etc/my.cnf.d/certs/ca.crt -text -noout | \
  grep -iE "subject|issuer|not before|not after|signature algorithm"
```
| Field | Pass condition | Fail condition |
|---|---|---|
| `Subject CN` | Your organisation or internal CA name | MariaDB default or generic |
| `Signature Algorithm` | `sha256WithRSAEncryption` or better | `sha1WithRSAEncryption` or weaker |
| `Not After` | Future date | Past date = expired |

**3c. CA expiry**
```bash
openssl x509 -in /etc/my.cnf.d/certs/ca.crt -enddate -noout
```
| Condition | Status |
|---|---|
| Date is in the future | ✅ PASS |
| Date is today or in the past | ❌ FAIL — CA expired |
| Expiry within 90 days | ⚠️ WARNING — plan renewal urgently |

**3d. Verify server cert is signed by this CA**
```bash
openssl verify -CAfile /etc/my.cnf.d/certs/ca.crt \
  /etc/my.cnf.d/certs/server.crt
```
| Condition | Status |
|---|---|
| Output is `server.crt: OK` | ✅ PASS |
| Output contains `error` or `unable to verify` | ❌ FAIL — cert not signed by this CA |

---

**PART 4 — Cross-node uniqueness**

```bash
# Run on each node
echo "=== NODE NAME ==="
openssl x509 -in /etc/my.cnf.d/certs/server.crt -fingerprint -sha256 -noout
openssl x509 -in /etc/my.cnf.d/certs/server.crt -serial -noout
openssl x509 -in /etc/my.cnf.d/certs/server.crt -subject -noout
```

| Condition | Status |
|---|---|
| Every node has a different fingerprint | ✅ PASS |
| Every node has a different serial number | ✅ PASS |
| Every node has a unique subject CN | ✅ PASS |
| Any two nodes share the same fingerprint | ❌ FAIL |
| Any two nodes share the same serial number | ❌ FAIL |
| All nodes have identical subject CN | ❌ FAIL — same cert deployed everywhere |

---

**PART 5 — File permissions**

```bash
ls -la /etc/my.cnf.d/certs/
stat /etc/my.cnf.d/certs/server.key
stat /etc/my.cnf.d/certs/server.crt
stat /etc/my.cnf.d/certs/ca.crt
```

| File | Permission | Owner | Status |
|---|---|---|---|
| `server.key` | `600` | `mysql` | ✅ PASS |
| `server.key` | `640` or looser | any | ❌ FAIL |
| `server.crt` | `644` or `640` | `mysql` | ✅ PASS |
| `server.crt` | `777` or world-writable | any | ❌ FAIL |
| `ca.crt` | `644` or `640` | `mysql` | ✅ PASS |
| `ca.crt` | world-writable | any | ❌ FAIL |
| Any file owned by non-mysql non-root user | any | other | ❌ FAIL |

---

**Master scorecard — all must be ✅ for overall PASS:**

| Part | Check | Pass condition |
|---|---|---|
| 1a | SSL enabled | `have_ssl = YES` |
| 1b | SSL in use | Cipher name returned |
| 1c | SSL enforced per user | No empty `ssl_type` |
| 2a | Not auto-generated | No output from grep |
| 2b | Strong signature algorithm | `sha256` or better |
| 2c | Certificate not expired | Future `Not After` date |
| 2d | Key strength adequate | 2048 bit minimum |
| 2e | Unique fingerprint per node | All nodes differ |
| 2f | Unique serial per node | All nodes differ |
| 3a | CA not auto-generated | No output from grep |
| 3b | CA uses strong algorithm | `sha256` or better |
| 3c | CA not expired | Future `Not After` date |
| 3d | Cert signed by correct CA | `OK` returned |
| 4 | Unique cert per node | Fingerprint and serial differ |
| 5 | File permissions correct | `server.key` is `600` |
