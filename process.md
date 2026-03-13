# Stoneline Data — Client Onboarding Process

## Pricing & Onboarding Fees
*(update with your final numbers)*

| Tier | Storage | Monthly | Onboarding Fee |
|------|---------|---------|----------------|
| Personal | 1TB | $__ /mo | $__ |
| Small Business | 5TB | $__ /mo | $__ |
| Professional | 10TB+ | $__ /mo | $__ |
| HIPAA/Compliance | Custom | $__ /mo | $__ |

---

## Redundancy Architecture

Every client gets the 3-2-1 stack:
- **3** copies of their data
- **2** different storage providers (AWS + GCP)
- **1** local copy on your NAS

---

## Step-by-Step Account Setup

### Step 1 — AWS Setup (Primary)

1. Log into your AWS root account
2. Create a new IAM user per client: `stoneline-[clientname]`
   - Attach policy: `AmazonS3FullAccess` scoped to their bucket only
   - Enable MFA on the IAM user
3. Create a dedicated S3 bucket: `stoneline-[clientname]-primary`
   - Region: us-east-1 (or your preferred primary region)
   - Block all public access: ON
   - Versioning: ENABLED (protects against accidental deletion)
   - Default encryption: SSE-S3 or SSE-KMS (KMS if HIPAA tier)
4. Enable S3 Replication to a second AWS region (us-west-2)
   - Creates bucket: `stoneline-[clientname]-replica`
   - Replication rule: replicate all objects
5. Set lifecycle policy on replica bucket:
   - Transition to S3 Glacier after 30 days (cost optimization)
6. Enable S3 Access Logging on primary bucket → log bucket: `stoneline-logs`
7. Set up CloudWatch alarm for any unexpected access or large deletions

---

### Step 2 — GCP Setup (Secondary / Redundant)

1. Log into Google Cloud Console
2. Create a new project: `stoneline-[clientname]`
3. Create a Cloud Storage bucket: `stoneline-[clientname]-gcp`
   - Location: multi-region (US) for built-in geographic redundancy
   - Storage class: Standard (or Nearline if cold archive)
   - Public access: Prevented
   - Encryption: Google-managed (default) or CMEK for compliance tier
4. Create a dedicated service account: `stoneline-[clientname]@project.iam.gserviceaccount.com`
   - Role: Storage Object Admin (scoped to their bucket only)
   - Download JSON key — store in 1Password
5. Enable audit logging: Admin Read, Data Read, Data Write
6. Set retention policy on bucket if required (legal/compliance clients)

---

### Step 3 — Sync Configuration (Rclone)

Rclone handles automated sync between all three locations.

```bash
# Install rclone
curl https://rclone.org/install.sh | sudo bash

# Configure AWS remote
rclone config
# name: aws-[clientname]
# type: s3
# provider: AWS
# credentials: IAM user access key + secret

# Configure GCP remote
rclone config
# name: gcp-[clientname]
# type: google cloud storage
# service_account_file: /path/to/keyfile.json

# Test sync (dry run first)
rclone sync /local/stoneline/[clientname] aws-[clientname]:stoneline-[clientname]-primary --dry-run

# Live sync
rclone sync /local/stoneline/[clientname] aws-[clientname]:stoneline-[clientname]-primary
rclone sync /local/stoneline/[clientname] gcp-[clientname]:stoneline-[clientname]-gcp
```

---

### Step 4 — Automate with Cron

```bash
# Edit crontab
crontab -e

# Run sync nightly at 2am
0 2 * * * rclone sync /local/stoneline/[clientname] aws-[clientname]:stoneline-[clientname]-primary --log-file=/var/log/stoneline/[clientname]-aws.log
0 2 * * * rclone sync /local/stoneline/[clientname] gcp-[clientname]:stoneline-[clientname]-gcp --log-file=/var/log/stoneline/[clientname]-gcp.log

# Weekly integrity check
0 3 * * 0 rclone check /local/stoneline/[clientname] aws-[clientname]:stoneline-[clientname]-primary --log-file=/var/log/stoneline/[clientname]-check.log
```

---

### Step 5 — Encryption (Client-Side)

For maximum security (and minimum liability), encrypt before upload so neither AWS nor GCP can read the data.

```bash
# Rclone crypt remote — wraps any existing remote
rclone config
# name: aws-[clientname]-crypt
# type: crypt
# remote: aws-[clientname]:stoneline-[clientname]-primary
# filename_encryption: standard
# password: [generate strong password — store in 1Password]
```

Now sync to the crypt remote instead — files are encrypted before they leave your machine.

---

### Step 6 — Billing & Autopay

1. Set up Stripe customer for client
2. Create recurring subscription at agreed monthly rate
3. Add AWS and GCP costs to your internal tracker (not billed to client — built into your margin)
4. Send client receipt + storage report monthly

---

### Step 7 — Monthly Verification Report

Send each client a simple monthly email confirming:
- Last successful sync date (AWS + GCP)
- Storage used vs allocated
- Any errors or alerts (and resolution)
- Integrity check result (pass/fail)

Template lives in `/templates/monthly-report.md`

---

### Step 8 — Offboarding

If a client leaves:
1. Export/deliver their data (S3 presigned URL or physical drive)
2. Confirm delivery in writing
3. Delete all buckets and IAM users after 30-day hold period
4. Cancel Stripe subscription
5. Archive their folder in your client records

---

## Cost Reference (Your Infrastructure)

| Service | Cost |
|---------|------|
| AWS S3 Standard | ~$0.023/GB/mo |
| AWS S3 Glacier | ~$0.004/GB/mo |
| AWS S3 Replication | ~$0.015/GB transferred |
| GCP Cloud Storage Standard | ~$0.020/GB/mo |
| GCP Nearline | ~$0.010/GB/mo |
| Rclone | Free |

1TB across AWS primary + GCP secondary ≈ **$43/mo** your cost before Glacier tiering.
With Glacier on the replica after 30 days, drops to roughly **$28/mo** per TB.

---

## Client Folder Structure (Local NAS)

```
/stoneline/
  [clientname]/
    active/          ← live data, synced nightly
    archive/         ← older versions, retained per agreement
    logs/            ← sync and integrity logs
    docs/            ← signed agreement, BAA if applicable
```
