# DVC Hands-on Implementation Plan

## Overview

This plan walks through DVC (Data Version Control) using a dummy CSV file.
Goal: understand how Git + DVC + remote storage work together for data versioning.

---

## Step 1 — Set up the environment

Initialize the uv project and sync dependencies from `pyproject.toml`.

```bash
uv sync
```

**Validation:**
```bash
dvc --version
# Expected: dvc 3.x.x

uv pip list | grep dvc
# Expected: dvc and dvc-s3 listed
```

---

## Step 2 — Initialize Git + DVC in the project

```bash
git init
dvc init
```

**Validation:**
```bash
git status
# Expected: .dvc/ folder and .dvcignore appear as new tracked files

ls .dvc/
# Expected: config, .gitignore, tmp/
```

---

## Step 3 — Create a dummy CSV (version 1)

```bash
mkdir data

cat > data/customers.csv << EOF
id,name,city,spend
1,Alice,Mumbai,5000
2,Bob,Delhi,3200
3,Carol,Pune,4100
EOF
```

**Validation:**
```bash
cat data/customers.csv
# Expected: header + 3 data rows

wc -l data/customers.csv
# Expected: 4 lines
```

---

## Step 4 — Track CSV with DVC

```bash
dvc add data/customers.csv
```

**Validation:**
```bash
cat data/customers.csv.dvc
# Expected: md5 hash + size + path entries

cat data/.gitignore
# Expected: /customers.csv  (DVC excludes actual file from git)

git status
# Expected: data/customers.csv.dvc is staged, NOT data/customers.csv
```

---

## Step 5 — Commit version 1 to Git

```bash
git add data/customers.csv.dvc data/.gitignore
git commit -m "v1: 3 customers dataset"
```

**Validation:**
```bash
git log --oneline
# Expected: 1 commit visible
```

---

## Step 6 — Configure a local remote (simulates S3 without needing AWS)

```bash
mkdir -p /tmp/dvc-remote-storage
dvc remote add -d localremote /tmp/dvc-remote-storage
```

**Validation:**
```bash
cat .dvc/config
# Expected: [core] remote = localremote
#           [remote "localremote"] url = /tmp/dvc-remote-storage

git add .dvc/config
git commit -m "add local DVC remote"
```

---

## Step 7 — Push data to the remote

```bash
dvc push
```

**Validation:**
```bash
ls /tmp/dvc-remote-storage/
# Expected: subdirectory named by first 2 chars of hash, with the data file inside
```

---

## Step 8 — Simulate version 2 (update the CSV)

```bash
cat > data/customers.csv << EOF
id,name,city,spend
1,Alice,Mumbai,5500
2,Bob,Delhi,3200
3,Carol,Pune,4100
4,Dave,Bangalore,6000
EOF

dvc add data/customers.csv
git add data/customers.csv.dvc
git commit -m "v2: 4 customers, Alice spend updated"
dvc push
```

**Validation:**
```bash
git log --oneline
# Expected: 3 commits total

cat data/customers.csv.dvc
# Expected: different md5 hash than v1

ls /tmp/dvc-remote-storage/
# Expected: 2 hash folders (both versions stored)
```

---

## Step 9 — Travel back to version 1

```bash
git checkout HEAD~1 -- data/customers.csv.dvc
dvc pull
```

**Validation:**
```bash
cat data/customers.csv
# Expected: 3 rows, Alice spend = 5000 (v1 data)

wc -l data/customers.csv
# Expected: 4 lines (header + 3 rows)
```

---

## Step 10 — Return to version 2

```bash
git checkout main -- data/customers.csv.dvc
dvc pull
```

**Validation:**
```bash
cat data/customers.csv
# Expected: 4 rows, Alice spend = 5500, Dave present

wc -l data/customers.csv
# Expected: 5 lines (header + 4 rows)
```

---

## What each step proves

| Step | Concept proven |
|------|----------------|
| 4    | DVC replaces the big file with a tiny `.dvc` pointer in Git |
| 5    | Git only stores the pointer, never the actual data |
| 7    | Actual data lives in the remote (S3 equivalent here) |
| 8    | New hash = new version; old version still preserved in remote |
| 9    | `git checkout` + `dvc pull` = full time travel including data |
| 10   | Switching back is equally clean and deterministic |

---

## Key mental model

```
Git   →  which version to use  (stores .dvc pointer files)
DVC   →  how to fetch it       (resolves hash → remote)
Remote (S3 / local) →  where the data lives
```
