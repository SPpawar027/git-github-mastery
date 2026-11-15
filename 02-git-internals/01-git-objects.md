# Chapter 01: Git Object Model (The 4 Fundamental Object Types)

---

## 1. Concept: Ye Kya Hai?

Git ka internal architecture ek simple aur elegant principle par chalta hai: **Everything in Git is an Object.**

Git ke pure universe mein sirf **4 fundamental object types** hote hain:
1. **Blob (Binary Large Object):** File ka actual content (code, text, binary data). Note: Blob mein file ka naam ya permissions nahi hote, sirf raw content hota hai.
2. **Tree (Directory):** Folders aur directories ko represent karta hai. Tree ke andar files ke naam, unke permissions (`100644`, `100755`), aur unke corresponding Blob ya Subtree ke SHA-1 hashes store hote hain.
3. **Commit:** Ek specific snapshot ka metadata. Isme root Tree ka hash, parent commit(s) ka hash, author info, committer info, timestamp, aur commit message hota hai.
4. **Annotated Tag:** Ek permanent milestone marker (jaise `v1.0.0`). Isme commit ka hash, tagger info, date, aur GPG cryptographic signature hota hai.

---

## 2. Why: Iski Zarurat Kyu Padi? (The Elegance of Minimalist Design)

Unix philosophy ka golden rule hai: *"Do one thing and do it well."*

Linus Torvalds ne Git ka data model itna modular banaya ki complex version control operations (branches, merges, cherry-picks, rebases) inhi 4 basic building blocks ke aapas mein connect hone se ho jaate hain.

Agar Git har feature ke liye naya file format banata, toh:
- Database corrupt hone ke chances 100x badh jaate.
- Network transfer bohot complex ho jata.
- Tooling aur performance slow ho jaati.

Sirf 4 objects aur Content-Addressable Key-Value model ke dam par Git Linux kernel ke 1.2+ million commits ko bina kisi lag ke fraction of a second mein traverse kar leta hai.

---

## 3. Real-World Example: Full-Stack Project Snapshot

Maan lijiye aapka ek project hai:
```text
my-app/
├── README.md
└── src/
    └── index.js
```

Git ise 4 objects ke web ke roop mein kaise dekhta hai?
1. `README.md` ka content ──► **Blob Object A**
2. `src/index.js` ka content ──► **Blob Object B**
3. `src/` folder ──► **Tree Object 1** (Contains: `index.js -> Blob B`)
4. Root folder `my-app/` ──► **Tree Object 2** (Contains: `README.md -> Blob A`, `src -> Tree 1`)
5. Aapka commit ──► **Commit Object** (Points to: `Tree 2`)

```text
               ┌──────────────────────────────┐
               │    Commit Object (SHA: 1a2b) │
               └──────────────┬───────────────┘
                              │ points to
                              ▼
               ┌──────────────────────────────┐
               │  Root Tree Object (SHA: 3c4d)│
               └──────┬────────────────┬──────┘
                      │                │
            README.md │                │ src/
                      ▼                ▼
     ┌──────────────────────┐   ┌───────────────────────────┐
     │  Blob Object A       │   │  Subtree Object (SHA: 5e6f)│
     │  (Content: # App)    │   └─────────────┬─────────────┘
     └──────────────────────┘                 │ index.js
                                              ▼
                                 ┌───────────────────────────┐
                                 │  Blob Object B            │
                                 │  (Content: console.log...)│
                                 └───────────────────────────┘
```

---

## 4. Internal Working: Object Anatomy & Storage Format

Har object disk par `.git/objects/` ke andar kaise store hota hai?

### 1. The Git Header:
Git kisi bhi content ko store karne se pehle ek header prepend karta hai:
```text
<object_type> + " " + <size_in_bytes> + "\0" (null byte)
```

Example for a blob with text "Hello World\n" (12 bytes):
`"blob 12\0Hello World\n"`

### 2. SHA-1 Calculation:
Is poore string (Header + Content) ka SHA-1 hash nikalta hai:
`557db03de997c86a4a028e1ebd3a1ceb225be238`

### 3. Zlib Compression:
Is poore payload ko Git standard **zlib deflate** algorithm se compress karta hai.

### 4. Storage Directory Path:
Hash ke first 2 characters bante hain folder name, aur bache 38 characters bante hain file name:
`.git/objects/55/7db03de997c86a4a028e1ebd3a1ceb225be238`

---

## 5. Commands: Inspecting Git Objects (Plumbing)

| Command | Safety | Description |
|---|:---:|---|
| `git hash-object -w --stdin` | `🟡 [CAUTION]` | Raw text ko hash karke `.git/objects` mein Blob create karta hai. |
| `git cat-file -t <hash>` | `🟢 [SAFE]` | Batata hai object ka type kya hai (`blob`, `tree`, `commit`, `tag`). |
| `git cat-file -s <hash>` | `🟢 [SAFE]` | Object ka uncompressed size bytes mein batata hai. |
| `git cat-file -p <hash>` | `🟢 [SAFE]` | Object ke content ko human-readable format mein pretty-print karta hai. |
| `git verify-pack -v <pack>`| `🟢 [SAFE]` | Compressed packfiles ke andar stored objects aur unke sizes inspect karta hai. |

---

## 6. Expected Output: Terminal Par Object Verification

```bash
# Hash and store a test string:
$ echo "Senior Git Engineer" | git hash-object -w --stdin
42b3bfec28e08d66df21b06c0c279c165ef49a37

# Check object type:
$ git cat-file -t 42b3bfec28e08d66df21b06c0c279c165ef49a37
blob

# Check size:
$ git cat-file -s 42b3bfec28e08d66df21b06c0c279c165ef49a37
20

# Read content back:
$ git cat-file -p 42b3bfec28e08d66df21b06c0c279c165ef49a37
Senior Git Engineer
```

---

## 7. Visual Explanation: The 4 Object Types

```text
┌────────────────────────────────────────────────────────────────────────┐
│ 1. BLOB                                                                │
│    Stores ONLY raw bytes of a file.                                    │
│    NO filename. NO timestamps. NO permissions.                         │
├────────────────────────────────────────────────────────────────────────┤
│ 2. TREE                                                                │
│    Stores directory entries:                                           │
│    100644 blob 42b3bfec...  app.js                                     │
│    040000 tree 8a12cc99...  components/                                │
├────────────────────────────────────────────────────────────────────────┤
│ 3. COMMIT                                                              │
│    Stores:                                                             │
│    tree 9a8b7c... (Root tree pointer)                                  │
│    parent 1f2e3d... (Previous commit)                                  │
│    author John Doe <john@work.com> 1731562800 +0530                    │
│    committer John Doe <john@work.com> 1731562800 +0530                 │
│    Commit message                                                      │
├────────────────────────────────────────────────────────────────────────┤
│ 4. ANNOTATED TAG                                                       │
│    Stores:                                                             │
│    object <commit_sha>                                                 │
│    type commit                                                         │
│    tag v1.0.0                                                          │
│    tagger Lead <lead@corp.com> 1731562800 +0530                        │
│    Release notes message & GPG signature                               │
└────────────────────────────────────────────────────────────────────────┘
```

---

## 8. Common Mistakes: Developers Kya Galti Karte Hain?

1. **Assuming Blobs Store Filenames:**
   - Mistake: Sochna ki blob ke andar file ka naam hota hai.
   - Consequence: Agar aap ek hi file ko rename karte hain (`app.js` to `main.js`), toh Blob ka hash change **nahi** hota! Naya Blob create nahi hota. Sirf Tree object naya banta hai jo purane blob hash ko naye filename ke sath map karta hai.
2. **Attempting to Read `.git/objects` with `cat` directly:**
   - Mistake: `cat .git/objects/42/b3bfec...`
   - Result: Terminal par garbage binary characters aayenge kyunki file zlib compressed hai. Hamesha `git cat-file -p` use karein ya Python zlib se decompress karein.

---

## 9. Troubleshooting: Diagnosing Loose Object Issues

```text
1. OBSERVE:    Command: `git fsck`
               Output: "dangling blob a1b2c3d4..."

2. INSPECT:    Dangling blob ka content inspect karein:
               `git cat-file -p a1b2c3d4`

3. UNDERSTAND: Dangling object ka matlab hai ki ek object .git/objects mein
               exist karta hai lekin koi commit, tree, branch ya tag use point
               nahi kar raha (e.g. Staged kiya tha fir unstage kar diya, ya
               amend commit kiya). This is normal garbage, not repository corruption!

4. PLAN:       Agar ye unwanted garbage hai, toh ise Git ke automatic garbage
               collector par chhod dein, ya explicitly prune karein.

5. EXECUTE:    git gc --prune=now

6. VERIFY:     `git fsck` run karein; dangling object ab gayab ho chuka hoga.

7. PREVENT:    Kuch karne ki zarurat nahi hai; Git periodically khud garbage
               clean karta rehta hai.
```

---

## 10. Practice Lab: Manual Git Object Construction 💻

Is practical lab mein hum dekhenge ki Git ke objects physically zlib compressed kaise hote hain aur Python ke zlib module se unhe manually decompress karenge:

```bash
# Step 1: Initialize temporary lab
mkdir -p /tmp/git-objects-deep-lab && cd /tmp/git-objects-deep-lab
git init

# Step 2: Create a blob manually
HASH=$(echo "Deep Internals Lab" | git hash-object -w --stdin)
echo "Object Hash: $HASH"

# Step 3: Find the physical file on disk
DIR_NAME=${HASH:0:2}
FILE_NAME=${HASH:2}
OBJECT_PATH=".git/objects/$DIR_NAME/$FILE_NAME"
ls -la $OBJECT_PATH

# Step 4: Prove that it is compressed by decompressing with Python!
python3 -c "
import zlib
with open('$OBJECT_PATH', 'rb') as f:
    raw_data = zlib.decompress(f.read())
    print('Raw Decompressed Bytes:', raw_data)
"
# Notice the output: b'blob 19\x00Deep Internals Lab\n'
# It has the Git header [blob 19\0] followed by the exact content!

# Step 5: Verify with git cat-file
git cat-file -p $HASH

# Cleanup
cd ~ && rm -rf /tmp/git-objects-deep-lab
```

---

## 11. Challenge: Test Your Understanding 🧠 (Scenario-Based Problems)

### 📌 Scenario Challenge 1: The Raw SHA-1 Hashing Calculation
> **Problem Statement:**
> Ek text string hai: `"Hello Git"` (length: 9 characters, no newline).
> Git ke rule ke mutabiq header hoga: `"blob 9\0Hello Git"`.
> 
> Kya aap bash terminal mein `printf` aur standard Linux `sha1sum` use karke exact wahi 40-character hash calculate kar sakte hain jo `echo -n "Hello Git" | git hash-object --stdin` calculate karta hai?
> 
> **Question:** Command kya hogi aur dono ka output exact match kaise karein?

<details>
<summary><b>💡 Click to Reveal Complete Solution & Walkthrough</b></summary>

#### Solution & Terminal Verification:

```bash
# 1. Calculate hash using Git's internal engine:
GIT_HASH=$(echo -n "Hello Git" | git hash-object --stdin)
echo "Git Hash:    $GIT_HASH"

# 2. Calculate hash using pure Linux bash printf and sha1sum:
# Notice \0 represents the null byte:
BASH_HASH=$(printf "blob 9\0Hello Git" | sha1sum | awk '{print $1}')
echo "Bash Hash:   $BASH_HASH"

# Output Comparison:
# Git Hash:    b7e6e4f971a3f4f08ed5074d3d01e36135c7e138
# Bash Hash:   b7e6e4f971a3f4f08ed5074d3d01e36135c7e138
```
Both outputs are 100% identical! This demystifies Git's hashing algorithm completely.
</details>

---

### 📌 Scenario Challenge 2: Accidental Giant File in Git History
> **Problem Statement:**
> Ek developer ne accidentally ek 200MB ki `.zip` archive file commit kar di:
> ```bash
> git add dataset.zip
> git commit -m "add dataset"
> ```
> Uske agle hi commit mein developer ko galti samajh aayi aur usne file delete kar di:
> ```bash
> git rm dataset.zip
> git commit -m "remove dataset"
> ```
> Lekin jab teammate ne repo clone ki, toh cloning abhi bhi 200MB download hone ki wajah se super-slow ho rahi thi!
> 
> **Questions:**
> 1. Working tree se file delete ho jaane ke baad bhi clone size 200MB kyu hai?
> 2. Wo 200MB ka Blob Git ke kis folder mein zinda betha hai?

<details>
<summary><b>💡 Click to Reveal Complete Solution & Walkthrough</b></summary>

#### Solution & Explanation:

1. **Why clone is still 200MB:**
   Kyunki Git ek **immutable version control system** hai!
   Commit 1 mein 200MB ka Blob object create hua aur `.git/objects/` mein store ho gaya.
   Commit 2 ne sirf working tree se file hatayi, lekin Commit 1 ki history abhi bhi database mein exist karti hai (taaki aap kabhi bhi Commit 1 par checkout karke wo file dekh sakein).
   Jab koi clone karta hai, Git pure DAG ko download karta hai, isliye 200MB ka Blob remote se local machine par download hota hi hai!

2. **Where does it live?**
   Ye `.git/objects/` ke andar loose object ya `.git/objects/pack/` packfile ke andar zinda rehta hai.
   Ise completely remove karne ke liye history rewrite tool (jaise `git-filter-repo` ya BFG Repo-Cleaner) aur `git gc --prune=now` chalana padta hai.
</details>

---

## 12. Interview Questions (FAANG & Product Companies)

### Q1: What are the 4 fundamental object types in Git?
**Answer:** The four fundamental objects are:
1. **Blob:** Stores pure file data (no metadata or filename).
2. **Tree:** Stores directory listings, mapping filenames and permissions to Blob or Subtree SHAs.
3. **Commit:** Stores snapshot pointer (root tree), parent commit SHA(s), author/committer metadata, and log message.
4. **Annotated Tag:** Stores a pointer to a specific commit along with a human-readable tag name, message, and optional GPG signature.

### Q2: Why does Git store file permissions inside Tree objects rather than Blob objects?
**Answer:** Blobs represent pure content to maximize **deduplication**. If file permissions were stored inside Blobs, changing a file's executable permission (`chmod +x script.sh`) would require creating a completely new duplicate Blob of the entire file. By keeping permissions (`100644` vs `100755`) inside the Tree object, the Blob remains identical and reusable.

### Q3: What is the difference between a Lightweight Tag and an Annotated Tag?
**Answer:**
- A **Lightweight Tag** is simply a pointer (a 41-byte text file in `.git/refs/tags/<name>`) containing a commit SHA. It is NOT an object in `.git/objects`.
- An **Annotated Tag** is a full object stored in `.git/objects/`. It has its own SHA-1 hash, author, timestamp, message, and can be cryptographically signed with GPG for official release validation.

---

## 13. Team Lead Perspective: Real Teams Mein Iska Use

1. **Repository Health & Packfiles:**
   Over time, loose objects thousands ki sankhya mein ho jaate hain. Team Lead CI pipeline mein check rakhta hai ki repository ka size exponentially grow na kare. Git automatically loose objects ko compress karke single `.pack` file banata hai with reverse indexing (`.idx`), jo disk size ko 70-90% tak compress kar deta hai.
2. **Git SHA-256 Migration (Object Format):**
   SHA-1 theoretically collision vulnerable ho chuka hai (SHAttered attack, 2017). Modern Git ab SHA-256 object format support karta hai (`git init --object-format=sha256`). Security-critical defense aur financial enterprises mein Team Leads modern repositories ko SHA-256 standard par initialize karwate hain.

---

### 🔗 Official Documentation References:
- [Git SCM - Git Internals: Git Objects](https://git-scm.com/book/en/v2/Git-Internals-Git-Objects)
- [Git SCM - Object Storage Architecture](https://git-scm.com/docs/gitrepository-layout)

