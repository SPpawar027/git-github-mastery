# Chapter 05: Credential Management (Git Credential Helpers, Keychains & GCM)

---

## 1. Concept: Ye Kya Hai?

Jab aap HTTPS protocol use karke GitHub se communicate karte hain, toh Git ko har `git pull` aur `git push` par aapka **Personal Access Token (PAT)** chahiye hota hai.

Agar Git ke paas koi credential manager na ho:
- Har 5 minute mein terminal aapse 40-character ka token type karne ko kahega.
- Developer thak kar galti karega ya token ko desktop par plain text notepad mein save karke insecure bana dega.

**Git Credential Helpers** wo background software programs hain jo aapke credentials ko securely store karte hain aur jab bhi Git ko zaroorat hoti hai, automatically provide kar dete hain.

Lekin har credential helper secure nahi hota! Is chapter mein hum **Secure Vaults** (Apple Keychain, Windows Vault, Linux Libsecret) vs **Dangerous Plaintext Storage** (`store`) ke internal mechanics ko samjhenge.

---

## 2. Why: Iski Zarurat Kyu Padi? (The `~/.git-credentials` Catastrophe)

Bohot se developers StackOverflow par ek purani command dekh kar run kar dete hain:
`git config --global credential.helper store`

### The Extreme Security Risk:
`store` helper aapke GitHub tokens aur passwords ko aapki hard drive par ek **unencrypted plain text file** mein save karta hai:
`~/.git-credentials`

Agar aap `cat ~/.git-credentials` chalayein:
`https://john:ghp_SuperSecretToken123456789@github.com`

**The Threat:**
Agar aapke system par koi malicious npm package, Python script, ya browser malware chalta hai, toh wo 1 line ke bash script se `~/.git-credentials` file read karke aapka corporate GitHub token steal kar lega! Pura enterprise code leak ho sakta hai.

### The Professional Alternative:
Hamesha **OS-Level Encrypted Keyrings** ya **Git Credential Manager (GCM)** use karein jo hardware-level AES encryption use karte hain.

---

## 3. Real-World Example: Rotating a Compromised Token

Aapke team ke ek developer ne aadhi raat ko notice kiya ki uska Personal Access Token ek public screencast mein galti se screen par flash ho gaya.
Developer ne turant GitHub web settings mein jakar token ko **Revoke** kar diya aur naya token generate kar liya.

Lekin jab agle din usne terminal par `git push` chalaya, toh Git naya token poochne ke bajaye turant fail ho gaya:
`remote: Invalid username or token.`
`fatal: Authentication failed.`

**Why?**
Kyunki uske local machine ke Credential Helper ne purane invalid token ko cache karke rakha hua tha aur Git bar-bar wahi expired token GitHub ko bhej raha tha!
Developer ko pata hona chahiye ki credential helper ke cache ko terminal se kaise **erase / purge** kiya jata hai.

---

## 4. Internal Working: The Git Credential Protocol

Git kisi external credential helper se communicate karne ke liye ek standardized protocol follow karta hai.
Git helper program ko launch karta hai aur 3 commands pass karta hai:

```text
1. git credential fill:
   Git calls the helper and asks: "Do you have credentials for https://github.com?"
   Helper checks OS Keyring and returns username & token via STDIN/STDOUT.

2. git credential approve:
   Agar push successful raha, Git helper ko bolta hai: "This token worked! Store it safely."

3. git credential reject:
   Agar GitHub ne token reject kar diya, Git helper ko bolta hai: "This token is INVALID! Erase it immediately!"
```

```text
Git CLI ◄──(STDIN / STDOUT)──► Credential Helper ◄──────► OS Encrypted Vault
                                                          ├── macOS: Apple Keychain
                                                          ├── Windows: Windows Vault
                                                          └── Linux: GNOME Keyring / KWallet
```

---

## 5. Commands: Credential Helpers Suite

| Command | Safety | Description |
|---|:---:|---|
| `git config --global credential.helper cache` | `🟡 [CAUTION]` | Tokens ko purely RAM memory mein temporarily cache karta hai (Disk par kuch save nahi hota). |
| `git config --global credential.helper "cache --timeout=3600"`| `🟡 [CAUTION]` | Token cache timeout 1 ghante (3600 seconds) ke liye set karta hai. |
| `git config --global credential.helper store` | `🔴 [DANGEROUS]` | **Extreme Caution:** Tokens ko plain text disk file `~/.git-credentials` mein likhta hai! |
| `git credential reject` | `🟡 [CAUTION]` | Stale/invalid credentials ko cache se forcibly delete karta hai. |
| `git config --global --unset credential.helper` | `🟡 [CAUTION]` | Configured credential helper ko remove karta hai. |

---

## 6. Expected Output: Terminal Par Verification

```bash
# Setting in-memory RAM cache with 2-hour timeout:
$ git config --global credential.helper "cache --timeout=7200"

# Verifying credential helper configuration:
$ git config --list --show-origin | grep credential
file:/home/user/.gitconfig	credential.helper=cache --timeout=7200

# Erasing a rejected credential manually via Git plumbing:
$ printf "protocol=https\nhost=github.com\n" | git credential reject
# Token is instantly purged from local cache!
```

---

## 7. Visual Comparison: Storage Helper Safety Spectrum

```text
LEVEL 1: THE DANGEROUS DISK STORE (credential.helper store)
Hard Drive: ~/.git-credentials ──► [PLAIN TEXT UNENCRYPTED TOKEN!] ◄── High Malware Vulnerability!


LEVEL 2: IN-MEMORY RAM CACHE (credential.helper cache)
System RAM Memory: [Token stored ephemerally] ──► Auto-wiped after timeout or reboot. 100% disk-safe!


LEVEL 3: HARDWARE ENCRYPTED KEYCHAIN (Git Credential Manager / Apple Keychain / Libsecret)
Operating System Vault: [AES-256 Encrypted with User Login Password] ──► Gold Standard!
```

---

## 8. Common Mistakes: Developers Kya Galti Karte Hain?

1. **Using `credential.helper store` on Shared Workstations:**
   - Mistake: Shared office desktop par apna token store kar dena. Agla developer jo terminal open karega wo aapke GitHub account se commits push kar sakta hai!
2. **Infinite Token Caching on Laptops:**
   - Always configure a reasonable timeout (e.g. 1 hour ya 8 hours work shift) agar RAM cache use kar rahe hain.
3. **Not Erasing Credentials When Changing Accounts:**
   - HTTPS clone ke baad ek account logout karne ke baad doosre account se push fail hota hai jab tak purana credential purge na kiya jaye.

---

## 9. Troubleshooting Guide: The 7-Step Diagnostic Framework

Agar token update karne ke baad bhi Git purana token bhejta rahe:

```text
1. OBSERVE:    Command: `git push origin main`
               Error: "fatal: Authentication failed for 'https://github.com/...'"
               Git does NOT ask for a new token; it fails instantly.

2. INSPECT:    Identify which credential helper is intercepting:
               `git config --show-origin --get-all credential.helper`

3. UNDERSTAND: Local credential helper ke paas purana invalid token cached hai
               aur wo Git ko naya prompt generate nahi karne de raha.

4. PLAN:       Git credential plumbing call se stale credential purge karein.

5. EXECUTE:    Run in terminal:
               printf "protocol=https\nhost=github.com\n" | git credential reject

6. VERIFY:     Run `git push origin main` again; Git terminal par fresh
               "Username" aur "Password/Token" prompt open karega!

7. PREVENT:    HTTPS tokens ke bajaye SSH keys use karein (Chapter 03 & 04)
               taaki token caching issues permanent eliminate ho sakein.
```

---

## 10. Practice Lab: Mastering In-Memory Credential Caching 💻

Chaliye apne terminal par secure RAM cache aur credential rejection plumbing execute karte hain:

```bash
# Step 1: Configure secure in-memory caching for 15 minutes (900 seconds)
git config --global credential.helper "cache --timeout=900"

# Step 2: Verify active helper
git config credential.helper

# Step 3: Simulate manually injecting a test credential into the helper
# (This tests the Git plumbing protocol)
cat << 'EOF' | git credential approve
protocol=https
host=mock-github.internal
username=test-developer
password=mock-secret-token-12345
EOF

# Step 4: Ask Git to fill the credential back to us!
cat << 'EOF' | git credential fill
protocol=https
host=mock-github.internal
EOF
# Notice: Git reads it directly from RAM memory and outputs:
# username=test-developer
# password=mock-secret-token-12345

# Step 5: Reject and purge the credential!
cat << 'EOF' | git credential reject
protocol=https
host=mock-github.internal
EOF

# Step 6: Verify it was completely wiped from memory:
cat << 'EOF' | git credential fill
protocol=https
host=mock-github.internal
EOF
# Git will prompt interactively because memory is now 100% clean! Press Ctrl+C to cancel.

# Cleanup
git config --global --unset credential.helper
```

---

## 11. Challenge: Test Your Understanding 🧠 (Scenario-Based Problems)

### 📌 Scenario Challenge 1: The Multi-Account HTTPS Token Collision
> **Problem Statement:**
> Ek developer HTTPS use karta hai aur uske do GitHub accounts hain:
> - `account-work`
> - `account-personal`
> 
> Usne operating system par Git Credential Manager configure kiya hua hai.
> Jab wo work repo push karta hai, GCM automatically work account se push kar deta hai.
> Lekin jab wo personal repo (`https://github.com/personal-user/my-blog.git`) push karta hai, toh Git bina prompt kiye work account ka token bhej deta hai aur error aata hai:
> `remote: Permission to personal-user/my-blog.git denied to account-work.`
> 
> **Question:** HTTPS URLs mein kaunsa parameter add karne se Git Credential Manager ko force kiya ja sakta hai ki wo specific username ka token authenticate kare?

<details>
<summary><b>💡 Click to Reveal Complete Solution & Walkthrough</b></summary>

#### Solution: Embedding Username in the HTTPS URL

Git Credential Manager URL ke **User Component** ko inspect karta hai!
Agar URL generic hai: `https://github.com/personal-user/my-blog.git`, toh GCM default active account use karta hai.

#### The Fix:
Remote URL ke andar target account ka **username** explicitly inject kar dein (Password/token mat dalein, sirf username!):

```bash
git remote set-url origin https://personal-user@github.com/personal-user/my-blog.git
```

#### What happens internally:
Git Credential Manager dekhega ki host `github.com` hai aur requested username `personal-user` hai! GCM OS Keychain se sirf `personal-user` ka token select karega aur work account se collision 100% solve ho jayega!
</details>

---

### 📌 Scenario Challenge 2: Auditing Plaintext Secrets on Developer Laptops
> **Problem Statement:**
> Ek Enterprise Security Audit chal rahi hai.
> Security Lead ne notice kiya ki kuch developers ne galti se `credential.helper store` chala rakha tha.
> 
> **Questions:**
> 1. Terminal par kaunsi command chalane se pata chalega ki kya local machine par plaintext `~/.git-credentials` file exist karti hai?
> 2. Is security leak ko purge karke safely OS keychain ya in-memory cache par switch karne ka command script kya hoga?

<details>
<summary><b>💡 Click to Reveal Complete Solution & Walkthrough</b></summary>

#### Solution & Remediation Script:

```bash
# Step 1: Check if plaintext file exists:
if [ -f ~/.git-credentials ]; then
    echo "CRITICAL WARNING: Plaintext credentials file detected!"
    cat ~/.git-credentials
    
    # Step 2: Securely purge the plaintext file:
    rm -f ~/.git-credentials
    echo "Plaintext credentials permanently deleted!"
fi

# Step 3: Remove the insecure store helper from global config:
git config --global --unset-all credential.helper

# Step 4: Configure secure RAM cache:
git config --global credential.helper "cache --timeout=3600"
```
Audit passed! Zero plaintext secrets left on disk.
</details>

---

## 12. Interview Questions (FAANG & Product Companies)

### Q1: What is the internal architecture of the Git Credential Helper protocol?
**Answer:** The Git Credential Helper protocol operates via standard input and standard output over a newline-delimited key-value format (e.g. `protocol=https`, `host=github.com`, `username=alice`). When authentication is required, Git executes the helper with the `fill` argument to retrieve credentials. After the network transaction, Git calls the helper with `approve` (if authentication succeeded) to persist credentials, or `reject` (if authentication failed with 401 Unauthorized) to erase invalid cached tokens.

### Q2: Why is `credential.helper store` strictly prohibited in corporate enterprise environments?
**Answer:** `credential.helper store` writes authentication credentials (usernames and personal access tokens) completely unencrypted in plain text to the local filesystem (`~/.git-credentials`). Any non-privileged process, local script, or malicious dependency running under the user's account can inspect this file, compromising sensitive corporate access tokens and bypassing multi-factor authentication controls.

### Q3: What is Git Credential Manager (GCM) and how does it handle Modern Authentication?
**Answer:** Git Credential Manager (GCM) is an open-source, cross-platform credential helper developed by Microsoft and the Git community. Unlike basic helpers, GCM integrates natively with cloud hosting APIs (GitHub, GitLab, Azure DevOps) to support modern authentication workflows including browser-based OAuth 2.0 logins, Multi-Factor Authentication (2FA), Conditional Access Policies, and hardware-backed storage via OS-level native keychains (Windows Credential Manager, macOS Keychain, and Linux Secret Service).

---

## 13. Team Lead Perspective: Real Teams Mein Iska Use

1. **Enterprise Endpoint Hardening:**
   DevOps aur IT security policy ke tehat Tech Lead automated MDM scripts (Jamf / Microsoft Intune) distribute karta hai jo developer laptops par `credential.helper store` ko permanently ban karta hai aur native OS Keychain integration enforce karta hai.
2. **Automating Token Expiration Alerts:**
   GitHub Organization level par Team Lead **Fine-Grained PAT Expiration Webhooks** setup karta hai. Jab kisi developer ka token expire hone mein 5 din bache hon, toh internal Slack bot automatically developer ko alert bhejta hai taaki active sprint work block na ho.

---

### 🔗 Official Documentation References:
- [Git SCM - Git Credential Storage](https://git-scm.com/book/en/v2/Git-Tools-Credential-Storage)
- [Git SCM - git-credential Documentation](https://git-scm.com/docs/git-credential)
- [GitHub - Git Credential Manager Documentation](https://github.com/git-ecosystem/git-credential-manager)

