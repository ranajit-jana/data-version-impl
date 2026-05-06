# Data Versioning with DVC

Hands-on implementation of DVC (Data Version Control) using a dummy CSV dataset.
Demonstrates how Git + DVC + S3 work together to version large data files.

---

## Mental model

```
Git     →  which version to use  (stores .dvc pointer files)
DVC     →  how to fetch it       (resolves hash → remote)
S3      →  where the data lives  (stores actual files by hash)
```

S3 has no concept of versions or commits — that lives entirely in Git.

---

## Project structure

```
.
├── data/
│   ├── customers.csv          # active dataset (tracked by DVC, not Git)
│   ├── customers.csv.dvc      # pointer file committed to Git
│   ├── customers_v1.csv       # reference: 5 customers
│   ├── customers_v2.csv       # reference: 7 customers
│   └── customers_v3.csv       # reference: 10 customers
├── .dvc/
│   └── config                 # remote storage config (committed to Git)
├── pyproject.toml             # Python dependencies
├── .gitignore
├── .dvcignore
├── setup.txt                  # environment setup instructions
└── plan.md                    # step-by-step implementation plan
```

---

## Prerequisites

- Python 3.8+
- Git
- uv (Python package manager)

```bash
git --version     # git version 2.x.x
python3 --version # Python 3.8+
uv --version      # uv 0.x.x
```

---

## Setup

### 1. Install dependencies

```bash
uv sync
```

Installs `dvc>=3.0.0` and `dvc-s3>=3.0.0` into `.venv/`.

```bash
dvc --version
# Expected: dvc 3.x.x
```

### 2. Configure Git identity

```bash
git config --global user.name "Your Name"
git config --global user.email "your@email.com"
```

### 3. Initialize DVC

```bash
dvc init
git add .dvc .dvcignore
git commit -m "init: dvc setup"
```

### 4. Configure AWS CLI (for S3 remote)

```bash
uv tool install awscli
echo 'export PATH="$HOME/.local/bin:$PATH"' >> ~/.bashrc
source ~/.bashrc

aws --version
aws configure    # enter Access Key, Secret Key, region
```

**IAM policy required** — attach this inline policy to your IAM user:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": ["s3:ListBucket"],
      "Resource": "arn:aws:s3:::data-version-impl"
    },
    {
      "Effect": "Allow",
      "Action": ["s3:GetObject", "s3:PutObject", "s3:DeleteObject"],
      "Resource": "arn:aws:s3:::data-version-impl/*"
    }
  ]
}
```

> Note: `ListBucket` targets the bucket ARN; object actions target `ARN/*`. This split is an AWS requirement.

### 5. Add S3 remote

```bash
dvc remote add -d s3remote s3://data-version-impl/dvc-store
git add .dvc/config
git commit -m "config: add S3 DVC remote"
```

Verify:
```bash
aws s3 ls s3://data-version-impl/
# Expected: no error (empty bucket is fine)
```

---

## Creating versioned commits

Each version copies the reference CSV into `data/customers.csv`, tracks it with DVC, and pushes to S3.

**Version 1 — 5 customers**
```bash
cp data/customers_v1.csv data/customers.csv
dvc add data/customers.csv
git add data/customers.csv.dvc data/.gitignore
git commit -m "data: customers v1 - 5 rows"
dvc push
```

**Version 2 — 7 customers, Carol upgraded to Premium**
```bash
cp data/customers_v2.csv data/customers.csv
dvc add data/customers.csv
git add data/customers.csv.dvc
git commit -m "data: customers v2 - 2 new customers, Carol upgraded to Premium"
dvc push
```

**Version 3 — 10 customers, spend refreshed**
```bash
cp data/customers_v3.csv data/customers.csv
dvc add data/customers.csv
git add data/customers.csv.dvc
git commit -m "data: customers v3 - 10 customers, spend refreshed"
dvc push
```

Verify all 3 versions are stored:
```bash
git log --oneline
aws s3 ls s3://data-version-impl/dvc-store/ --recursive
# Expected: 3 objects (one per version hash)
```

---

## Time travel — switching between versions

Use `git checkout` to point to a version's `.dvc` file, then `dvc pull` to fetch that data from S3.

**Go to v1:**
```bash
git checkout 4c407c1 -- data/customers.csv.dvc
dvc pull
cat data/customers.csv
# Expected: 5 rows, Alice spend = 15000
```

**Go to v2:**
```bash
git checkout aba2ce7 -- data/customers.csv.dvc
dvc pull
cat data/customers.csv
# Expected: 7 rows, Carol segment = Premium
```

**Return to v3 (latest):**
```bash
git checkout master -- data/customers.csv.dvc
dvc pull
cat data/customers.csv
# Expected: 10 rows, Alice spend = 21000
```

---

## How DVC fetches a specific data version

This is the core mechanism — understanding this makes everything else obvious.

### Step 1 — `dvc add` computes an MD5 hash

When you run `dvc add data/customers.csv`, DVC computes an MD5 hash of the file contents:

```
MD5(customers.csv contents) → e83c4b9f2a1d...
```

It writes this hash into `data/customers.csv.dvc`:

```yaml
outs:
- md5: e83c4b9f2a1d7c3b9e2f1a4d6c8b0e5f
  size: 312
  path: customers.csv
```

This tiny file is what Git commits. The actual CSV never enters Git.

---

### Step 2 — `dvc push` stores the file by its hash in S3

DVC uploads the file to S3 using the hash as the path.
The full S3 path structure DVC creates is:

```
s3://data-version-impl/dvc-store/files/md5/<first-2-chars>/<rest-of-hash>
```

The first 2 characters of the MD5 hash become a folder, the rest is the filename.
This is content-addressable storage — the path IS the identity of the file.

After pushing all 3 versions, the actual S3 layout looks like this
(confirmed from AWS Console: `data-version-impl > dvc-store > files > md5`):

![S3 bucket structure after dvc push](assets/s3-bucket-structure.png)

```
dvc-store/
└── files/
    └── md5/
        ├── 44/   ← first 2 chars of v? hash  (one of the 3 customer versions)
        ├── 65/   ← first 2 chars of v? hash
        └── 9b/   ← first 2 chars of v? hash
```

Each folder contains exactly one file — the full data file stored under its hash.
S3 stores all versions simultaneously. It has no idea which is "latest" — that is Git's job.

---

### Step 3 — Git commits track which hash is active

Each Git commit holds a snapshot of `customers.csv.dvc` with a specific hash:

```
Commit 4c407c1  →  customers.csv.dvc  →  md5: e83c4b9f...  (v1, 5 rows)
Commit aba2ce7  →  customers.csv.dvc  →  md5: a19f2c3d...  (v2, 7 rows)
Commit d9b4d29  →  customers.csv.dvc  →  md5: 7d2e1f3a...  (v3, 10 rows)
```

Git is the version index. S3 is just a content store.

---

### Step 4 — `git checkout` + `dvc pull` resolves the chain

When you run:

```bash
git checkout 4c407c1 -- data/customers.csv.dvc
dvc pull
```

Here is exactly what happens internally:

```
1. git checkout  →  writes e83c4b9f... into customers.csv.dvc on disk
2. dvc pull      →  reads customers.csv.dvc, extracts md5: e83c4b9f...
3. DVC checks local cache (~/.dvc/cache) — is e83c4b9f... already there?
   YES → copies from cache to data/customers.csv  (no network call)
   NO  → downloads s3://data-version-impl/dvc-store/e8/3c4b9f... from S3
         stores it in local cache
         copies to data/customers.csv
4. You now have v1 data locally
```

The local cache (`~/.dvc/cache`) means repeated pulls of the same version cost nothing — DVC never re-downloads a file it already has.

---

### Full picture in one diagram

```
git checkout <commit>
      │
      ▼
customers.csv.dvc  ←── contains md5 hash of that version
      │
      ▼
dvc pull
      │
      ├── check ~/.dvc/cache/<hash>
      │         hit  → copy to data/customers.csv
      │         miss ↓
      └── S3: dvc-store/files/md5/<first-2-chars>/<rest-of-hash>
                  │
                  ▼
            data/customers.csv  ← exact file for that Git commit
```

This is why DVC is called a "control layer" — it doesn't store data itself,
it controls where data lives and which version maps to which Git commit.

---

## What each step proves

| Action | Concept |
|--------|---------|
| `dvc add` | Replaces the data file with a tiny hash pointer in Git |
| `git commit` (`.dvc` file) | Git stores the pointer, never the actual data |
| `dvc push` | Actual data uploaded to S3 under its hash |
| New `dvc add` on changed file | New hash = new version; old version preserved in S3 |
| `git checkout` + `dvc pull` | Full time travel — code and data move together |

---

## Notes

- Never manually edit `.dvc` files — they are managed by DVC.
- Always run `dvc push` after `git commit` to keep code and data in sync.
- `data/customers.csv` is excluded from Git by `data/.gitignore` (auto-generated by DVC). Do not add `/data/` to the root `.gitignore` — that would block the `.dvc` pointer files too.
