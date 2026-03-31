# Security Audit Report

**Repository:** `azmove/leaked-claude-code`  
**Audit Date:** 2026-03-31  
**Auditor:** GitHub Copilot Coding Agent  
**Scope:** Full review of all source files, assets, and repository metadata

---

## Executive Summary

This repository contains source code that appears to be genuine Anthropic Claude Code internal TypeScript source (leaked via an exposed `.map` file in the npm package). The **TypeScript source files themselves do not contain malicious code**. However, the repository's README.md contains highly suspicious and deceptive content injected by the repository author that constitutes a significant security risk to anyone who follows its instructions.

**Verdict: The source code is clean; the README contains the author's injected malicious/deceptive content.**

---

## Findings

### 🔴 CRITICAL: README.md — Deceptive Binary Distribution

**File:** `README.md`

The README was authored by the repository owner (not Anthropic) and contains several red flags consistent with a credential-stealing trojan campaign:

| Issue | Detail |
|-------|--------|
| **Deceptive marketing** | Claims to be "Claude Code Unlocked" with "no limits, no censorship" and "Enterprise-level features unlocked" |
| **Bypasses payment restrictions** | Explicitly states: *"utilizes browser fingerprint spoofing and token rotation methods to bypass paid access restrictions"* |
| **Pre-compiled binary** | Directs users to download `ClaudeCode_x64.7z` from the releases page — a Windows executable not present in the source repository |
| **API key harvesting** | The binary prompts the user to enter their Anthropic API Key on first run ("securely stored using the Windows Credential Manager") |
| **Disclaimer deflection** | Adds a "Security Research" disclaimer to deflect legal responsibility |

**Risk:** The pre-compiled binary (`ClaudeCode_x64.7z`) is the primary threat vector. It is not part of the source repository and **cannot be audited here**. Its distribution pattern — a "cracked" tool that requests an API key on first run — is a classic credential-harvesting technique. The binary may:
- Exfiltrate the entered Anthropic API key to a third-party server
- Install additional malware on the victim's machine
- Perform actions on the victim's Anthropic account

**Do not download or run `ClaudeCode_x64.7z` or any binary from this repository's releases.**

---

### 🟡 NOTABLE: `buddy/types.ts` — `String.fromCharCode` Encoding

**File:** `buddy/types.ts` (lines 14–50)

```typescript
const c = String.fromCharCode
export const duck = c(0x64,0x75,0x63,0x6b) as 'duck'
export const goose = c(0x67, 0x6f, 0x6f, 0x73, 0x65) as 'goose'
// ... etc.
```

**Assessment: BENIGN — Explained by comment in the file.**

The file contains a comment explaining the encoding:

> *"One species name collides with a model-codename canary in excluded-strings.txt. The check greps build output (not source), so runtime-constructing the value keeps the literal out of the bundle while the check stays armed for the actual codename. All species encoded uniformly."*

All 18 encoded values decode to innocent animal species names used by the "buddy" companion sprite feature:
`duck, goose, blob, cat, dragon, octopus, owl, penguin, turtle, snail, ghost, axolotl, capybara, cactus, robot, rabbit, mushroom, chonk`

---

### 🟡 NOTABLE: `bridge/bridgeConfig.ts` — `USER_TYPE === 'ant'` Developer Overrides

**File:** `bridge/bridgeConfig.ts`

```typescript
export function getBridgeTokenOverride(): string | undefined {
  return (
    (process.env.USER_TYPE === 'ant' &&
      process.env.CLAUDE_BRIDGE_OAUTH_TOKEN) ||
    undefined
  )
}
```

**Assessment: BENIGN — Standard Anthropic internal developer override pattern.**

The `USER_TYPE === 'ant'` (Anthropic employee) checks gate developer-only environment variable overrides for `CLAUDE_BRIDGE_OAUTH_TOKEN` and `CLAUDE_BRIDGE_BASE_URL`. These are only active when the `USER_TYPE` env var is explicitly set to `'ant'`. This pattern appears in several files (`bridgeMain.ts`, `bridgeConfig.ts`) and is consistent with an internal developer testing mechanism. No external attacker can trigger this without already controlling the environment.

---

### 🟢 CLEAN: All Network Calls

**Files reviewed:** All `*.ts` / `*.tsx` files

All `axios.post`, `axios.get`, `axios.put`, `axios.delete`, `axios.patch`, and `fetch` calls use URLs derived from:
- `deps.baseUrl` (injected via constructor, resolves to `api.anthropic.com`)
- `getOauthConfig().BASE_API_URL` (Anthropic's production API)
- `sessionUrl` (Anthropic session ingress)
- `sseUrl` (derived from the above)

**No calls to third-party, attacker-controlled, or hardcoded external domains were found.**

---

### 🟢 CLEAN: No Hardcoded Secrets

A thorough search for hardcoded API keys, tokens, passwords, or credentials found none. The `debugUtils.ts` file even contains active secret redaction logic (`redactSecrets()`) that masks tokens in debug logs.

---

### 🟢 CLEAN: `assets/hmv4dn7elu.png` — No Steganography

**File:** `assets/hmv4dn7elu.png`

PNG chunk analysis confirmed:
- Standard chunks only: `IHDR`, `sRGB`, `IDAT` (×22), `IEND`
- No text/metadata chunks (`tEXt`, `zTXt`, `iTXt`)
- No data appended after the `IEND` marker

The image is a clean 2000×313 RGBA PNG screenshot used in the README.

---

### 🟢 CLEAN: No Eval / Dynamic Code Execution

A search for `eval()`, `Function()`, `execSync()`, and similar patterns found only legitimate references:
- `child_process.spawn` is referenced in `bridge/sessionRunner.ts` for spawning legitimate Claude Code child processes (the bridge spawns new Claude sessions)
- No dynamic code construction or eval of user/network-provided strings

---

### 🟡 MINOR: `.gitignore` — Irrelevant Template

**File:** `.gitignore`

The `.gitignore` contains a template for Microsoft Dynamics 365 Business Central (AL language) projects, which is completely unrelated to this TypeScript codebase. This indicates the repository was set up carelessly. It poses no security risk but is evidence of the repository owner's low-effort setup.

---

## Summary Table

| File / Area | Finding | Severity |
|-------------|---------|----------|
| `README.md` | Deceptive content; promotes untrusted binary that harvests API keys | 🔴 CRITICAL |
| `buddy/types.ts` | `String.fromCharCode` encoding — BENIGN (animal names, explained) | 🟢 CLEAN |
| `bridge/bridgeConfig.ts` | `USER_TYPE=ant` dev overrides — BENIGN (Anthropic internal) | 🟢 CLEAN |
| All `.ts`/`.tsx` network calls | All point to Anthropic endpoints only | 🟢 CLEAN |
| `assets/hmv4dn7elu.png` | No steganography or hidden data | 🟢 CLEAN |
| Hardcoded secrets | None found | 🟢 CLEAN |
| Dynamic code execution | None found | 🟢 CLEAN |
| `.gitignore` | Wrong template (AL/Business Central) — low-effort setup | 🟡 MINOR |
| Pre-compiled binary in Releases | Not auditable; highly suspicious distribution pattern | 🔴 CRITICAL |

---

## Recommendations

1. **Do not download or execute `ClaudeCode_x64.7z`** from the releases page. It is not from Anthropic, cannot be audited, and the distribution pattern strongly suggests credential theft.

2. **Do not enter your Anthropic API key** into any binary obtained from this repository.

3. If you have already run the binary and entered your API key, **immediately revoke the key** at [console.anthropic.com](https://console.anthropic.com) and rotate any associated credentials.

4. The TypeScript source code in this repository appears to be genuine leaked Anthropic source code. Using or distributing it may violate Anthropic's intellectual property rights.

5. The claim that this software "bypasses paid access restrictions" is both illegal (violating Anthropic's Terms of Service) and likely a social engineering lure to get users to run the malicious binary.
