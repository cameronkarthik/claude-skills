---
name: wallet-safety-check
description: Use before any git push, public repo creation, or code review of crypto projects. Audits for private key exposure, credential leaks, unsafe storage patterns, and git history contamination.
version: 0.1.0
tools: Read, Grep, Glob, Bash
---

# Wallet Safety Check

Security audit skill for crypto projects. Catches private key exposure, credential leaks, and unsafe storage before they cost real money.

## When to Use

- Before any `git push` on a crypto project
- Before making a private repo public
- After adding wallet functionality to any codebase
- When reviewing code that handles private keys, seed phrases, or API keys
- When the user asks "is this safe to push?"

## Audit Checklist

Run these checks in order. Stop and flag immediately on any failure.

### 1. Git Staged Files Check

```bash
# Check what's about to be committed
git diff --cached --name-only

# Look for dangerous files in staging
git diff --cached --name-only | grep -E '\.(env|key|pem|p12|pfx|secret)$'
```

Flag: ANY match is a hard stop.

### 2. Secret Pattern Scan

Search the entire repo for these patterns:

```bash
# Private keys (hex)
grep -rn "0x[a-fA-F0-9]{64}" --include="*.ts" --include="*.js" --include="*.tsx" --include="*.jsx" --include="*.json"

# Solana keypair arrays (65-byte arrays in JSON)
grep -rn "\[[0-9]{1,3},[0-9]{1,3},[0-9]{1,3}," --include="*.json" --include="*.ts" --include="*.js"

# Seed phrases (12/24 word patterns)
grep -rn "abandon\|acoustic\|aircraft\|mnemonic\|seed.*phrase\|SEED.*PHRASE" --include="*.ts" --include="*.js" --include="*.env*"

# Base58 private keys (Solana format, 87-88 chars)
grep -rn "[1-9A-HJ-NP-Za-km-z]\{87,88\}" --include="*.ts" --include="*.js" --include="*.json"

# API keys
grep -rn "sk-\|api[_-]key\|apiKey\|API_KEY\|PRIVATE_KEY\|SECRET_KEY" --include="*.ts" --include="*.js" --include="*.env*"
```

### 3. Environment File Audit

```bash
# Check .gitignore exists and covers env files
cat .gitignore | grep -E "\.env"

# Check for committed env files
git ls-files | grep -E "\.env"

# Check git history for env files (even if removed now)
git log --all --diff-filter=A --name-only -- '*.env' '*.env.*'
```

Critical: If env files exist in git history, the secrets are ALREADY leaked. They need rotation, not just removal.

### 4. Storage Pattern Audit

Search for unsafe key storage patterns:

| Pattern | Risk | Fix |
|---|---|---|
| `localStorage.setItem("key"` | Persists in plain text, survives page reload | Use encrypted IndexedDB vault |
| `sessionStorage.setItem("key"` | Plain text in memory, XSS vulnerable | Use in-memory only with encryption |
| `console.log(privateKey` | Keys in browser console, may be logged | Remove all key logging |
| `JSON.stringify(keypair)` | Serialized keys may end up in unexpected places | Never serialize raw keys |
| `Buffer.from(secretKey)` | In browser, Buffer polyfills may leak | Use Uint8Array directly |

### 5. Network Exposure Check

```bash
# Check for keys being sent over network
grep -rn "fetch\|axios\|XMLHttpRequest" --include="*.ts" --include="*.js" -l | while read f; do
  grep -n "key\|secret\|private\|mnemonic" "$f"
done
```

### 6. Git History Deep Scan

```bash
# Search entire git history for secrets
git log -p --all -S "PRIVATE_KEY" -- '*.ts' '*.js' '*.json' '*.env'
git log -p --all -S "0x" -- '*.env*'
```

If secrets are found in history:
1. Do NOT just delete and recommit -- the history still contains them
2. Rotate ALL exposed credentials immediately
3. Use `git filter-branch` or BFG Repo Cleaner to purge history
4. Force push (with user confirmation)
5. Revoke and regenerate all API keys

## Risk Severity Levels

| Level | Examples | Action |
|---|---|---|
| CRITICAL | Private key in source code, env file committed, seed phrase in logs | Stop immediately. Do not push. Rotate credentials. |
| HIGH | API key in client-side code, unencrypted localStorage, key logged to console | Fix before push. May need rotation. |
| MEDIUM | No .gitignore for env files, missing encryption on key storage | Fix before push. Add to .gitignore. |
| LOW | Verbose error messages that could leak key fragments | Fix when convenient. |

## Response Template

After running the audit, report:

```
WALLET SAFETY CHECK
===================
Severity: [CRITICAL/HIGH/MEDIUM/LOW/CLEAN]

Findings:
- [Finding 1 with file:line reference]
- [Finding 2 with file:line reference]

Required Actions:
- [ ] [Action 1]
- [ ] [Action 2]

Credential Rotation Needed: [YES/NO]
Safe to Push: [YES/NO]
```
