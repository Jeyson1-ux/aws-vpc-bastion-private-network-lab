# ⚠️ Gotchas & Fixes

This file documents all issues encountered during the lab and how they were resolved.

---

## 1. Private EC2 had no internet access

**Issue:**

```
sudo yum update -y
```

was hanging.

**Cause:**
No NAT Gateway has yet been configured.

**Fix:**

* I created NAT Gateway
* Added route: `0.0.0.0/0 → NAT`

---

## 2. S3 access failed after NAT removal

**Issue:**

```
aws s3 ls
```

was hanging.

**Cause:**
No internet path to S3.

**Fix:**

* Created S3 Gateway Endpoint
* Attached to private route table

---

## 3. S3 endpoint creation failed

**Issue:**
DNS error when creating endpoint.

**Cause:**
VPC DNS settings disabled.

**Fix:**
Enabled:

* DNS resolution
* DNS hostnames

---

## 4. AccessDenied errors with S3

**Issue:**

```
AccessDenied: s3:ListBucket
```

**Cause:**
Incorrect IAM policy.

**Fix:**

* Replaced with custom least privilege policy
* Scoped to specific bucket

---

## 5. AWS CLI not working on Windows

**Issue:**

```
aws not recognized
```

**Fix:**
Installed AWS CLI using this command in the terminal instead of downloading it locally:

```
winget install Amazon.AWSCLI
```

---

## 6. SSM session failed initially

**Issue:**
403 Unauthorized / plugin missing

**Fix:**

* Installed Session Manager plugin
* Configured AWS CLI credentials

---

## 7. Bucket region mismatch

**Issue:**
S3 commands hanging

**Cause:**
Wrong region used

**Fix:**
Used correct region:

```
--region ap-south-1
```

---

## 8. Route table confusion

**Issue:**
Private subnet still had no internet

**Cause:**
Route table not associated properly

**Fix:**
Re-associated correct route table to private subnet

---

## 9. NAT Gateway placement

**Issue:**
The NAT not working

**Cause:**
Placed in wrong subnet

**Fix:**
Moved NAT to public subnet

---

## 10. Security group rules misunderstanding

**Issue:**
SSH worked even after tightening rules

**Cause:**
Existing TCP session still active

**Fix:**
Restarted connection to validate rules

---

# 🧠 Final Note

These mistakes were actually the most valuable part of the lab. Understanding why things failed made the architecture much clearer.
