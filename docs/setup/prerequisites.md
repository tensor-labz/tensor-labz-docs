# Prerequisites

Everything you need before running the project locally.

---

## Required accounts

| Service  | Purpose         | URL                                                |
| -------- | --------------- | -------------------------------------------------- |
| Supabase | Database + Auth | [supabase.com](https://supabase.com)               |
| AWS      | S3 + Lambda     | [aws.amazon.com](https://aws.amazon.com)           |
| Firebase | Staging hosting | [firebase.google.com](https://firebase.google.com) |
| GitHub   | Source control  | [github.com](https://github.com)                   |

---

## Required tools

### Node.js

```bash
node --version  # must be ≥ 18
npm --version
```

Install via [nvm](https://github.com/nvm-sh/nvm):

```bash
nvm install 20
nvm use 20
```

### AWS CLI

```bash
aws --version  # must be v2
```

Install: [docs.aws.amazon.com/cli](https://docs.aws.amazon.com/cli/latest/userguide/install-cliv2.html)

Configure with the `tensor` profile:

```bash
aws configure --profile tensor
# AWS Access Key ID: <your key>
# AWS Secret Access Key: <your secret>
# Default region: eu-north-1
# Default output format: json
```

All Lambda / S3 commands in this project use `--profile tensor`.

### Firebase CLI

```bash
npm install -g firebase-tools
firebase login
firebase projects:list  # should show tensor-labz-website
```

!!! note "Headless server"
On a server without a browser, use `firebase login --no-localhost`.

### Git + GitHub CLI (optional but useful)

```bash
git --version
gh --version  # optional: github.com/cli/cli
```

---

## Repositories

Clone both repos side-by-side:

```bash
git clone git@github.com:ThanuMahee12/tensor-labz-website.git
git clone git@github.com:ThanuMahee12/tensor-labz-image-lambda.git
```

Expected layout:

```
~/dev/personal/
  tensor-labz-website/          ← React frontend + admin
  tensor-labz-image-lambda/     ← AWS Lambda image service
```

The CLAUDE.md in the frontend repo references `../tensor-labz-image-lambda/` when mentioning Lambda deploy commands.
