# How Subfolder Fetch Works — A Full Walkthrough

This explains, step by step, what your tool is actually doing under the hood
when you paste a GitHub folder URL and click "Download ZIP."

## The big picture — what problem are we solving?

GitHub's website lets you *browse* a folder, but it never gives you a "download
just this folder" button. GitHub only offers:

- Clone the whole repo (`git clone`)
- Download the whole repo as a ZIP (`Code → Download ZIP`)

Neither of those isolates a single subfolder. So instead of using GitHub's
website, the tool talks directly to **GitHub's REST API** — a separate,
machine-friendly interface into the same data. Think of the website as the
"showroom" and the API as the "warehouse door around back" — same inventory,
but built for programs instead of humans clicking around.

## Step 1 — Parsing your URL

You paste something like:
```
github.com/RiyadunNabi/BUET_CSE/tree/main/3-1/CSE318_AI/2_MaxCut
```

The tool splits this on `/` and picks out five pieces:

| Piece | Value | Meaning |
|---|---|---|
| owner | `RiyadunNabi` | whose account the repo lives in |
| repo | `BUET_CSE` | which repository |
| (skip) | `tree` | GitHub's own marker meaning "this is a folder view" |
| branch | `main` | which version/timeline of the repo |
| path | `3-1/CSE318_AI/2_MaxCut` | everything after — the folder you actually want |

This is just string-splitting — no network calls yet. It's the tool figuring
out *what to ask for* before it asks.

## Step 2 — Turning the branch into a "snapshot ID"

Here's a subtlety: a branch name like `main` isn't a fixed thing — it moves
every time you commit. So the tool's first API call is:

```
GET https://api.github.com/repos/{owner}/{repo}/branches/{branch}
```

This is like asking "what does `main` currently point to?" The answer includes
a **commit SHA** — a long hexadecimal ID (like `a1b2c3d...`) that uniquely and
permanently identifies the exact state of the repo at that moment. Think of a
branch name as a sticky note that gets moved to a new page every time you
write something new, while the SHA is the page number itself — fixed forever.

We need that fixed page number for the next step, because trees (below) are
addressed by SHA, not by branch name.

## Step 3 — Getting the full file listing (the "tree")

Next call:
```
GET https://api.github.com/repos/{owner}/{repo}/git/trees/{sha}?recursive=1
```

This asks Git's internal object database for a complete, flat list of *every
file in the entire repository* at that snapshot — full paths, not just one
folder. `recursive=1` tells it to include files in every nested subfolder
too, not just the top level.

The response looks conceptually like:
```json
[
  { "path": "3-1/CSE318_AI/2_MaxCut/main.cpp", "type": "blob", "sha": "..." },
  { "path": "3-1/CSE318_AI/1_something/other.py", "type": "blob", "sha": "..." },
  { "path": "README.md", "type": "blob", "sha": "..." }
]
```
`"blob"` is Git's term for "this is a file" (as opposed to `"tree"`, which
would mean "this is a folder"). Every blob has its own SHA too — a fingerprint
of that file's exact contents.

## Step 4 — Filtering down to just your folder

This is the actual "subfolder" trick, and it's disappointingly simple: the
tool has the list of *all* files, so it just throws away anything whose path
doesn't start with `3-1/CSE318_AI/2_MaxCut/`. No special API for this exists —
we fetch everything and filter client-side, in your browser, using ordinary
JavaScript `.filter()`.

## Step 5 — Downloading each file's actual content

The tree only gave us metadata (path + SHA) — not the file contents. For each
file that survived the filter, the tool makes one more call:
```
GET https://api.github.com/repos/{owner}/{repo}/git/blobs/{file_sha}
```
This returns the file's content, but **encoded in Base64** — a text-safe way
of representing binary data using only letters, numbers, and a few symbols.
GitHub does this because a file could be an image, a compiled binary,
anything — Base64 guarantees it survives being packed into JSON safely.

The tool then decodes that Base64 back into raw bytes (`atob()` + a byte-array
conversion) — reversing the encoding to get the original file back exactly.

## Step 6 — Zipping it in the browser

Normally, ZIP files are built by an operating system or a server. Here, it's
built entirely *inside your browser tab* using a JavaScript library called
**JSZip**. As each file is decoded, it's added to an in-memory ZIP archive
under its original relative path (so `2_MaxCut/main.cpp` stays
`2_MaxCut/main.cpp` in the final ZIP — this is what preserves your folder
structure).

Once every file is added, `zip.generateAsync()` compresses everything into a
single downloadable `Blob` (browser-speak for "a chunk of binary data").

## Step 7 — Triggering the actual download

The tool creates an invisible `<a>` (link) element, points its `href` at the
ZIP blob using `URL.createObjectURL()`, sets `download="foldername.zip"`, and
clicks it programmatically. This is the standard browser trick for saving
generated content to disk — same mechanism many "export to CSV" buttons use
on other websites.

## Where the token fits in

Every API call above goes through `buildHeaders()`, which attaches:
```
Authorization: token YOUR_TOKEN
```
...if you provided one. GitHub checks this header on every request. For a
**public** repo, it's optional — GitHub lets anyone read public data, just
with a stricter rate limit (60 requests/hour without a token vs. 5,000/hour
with one). For a **private** repo, every single request above would return
`404 Not Found` without a valid token — not because the file doesn't exist,
but because GitHub deliberately hides the *existence* of private resources
from unauthorized requests, rather than saying "403 Forbidden" and confirming
something's there.

## Why nothing gets saved anywhere

The token only ever lives in a JavaScript variable in your browser tab's
memory. It's never written to `localStorage`, a cookie, or sent to any server
except `api.github.com` directly. Close the tab, and that variable — and the
token with it — is gone. This is different from tools that save your token to
LocalStorage for convenience (like the ones we tried earlier) — that's more
convenient for repeat use, but means the token lingers on that machine until
manually cleared, which matters on a shared lab PC.

## The full request flow, summarized

```
Your click
   │
   ├─▶ GET /repos/{owner}/{repo}/branches/{branch}      → get commit SHA
   │
   ├─▶ GET /repos/{owner}/{repo}/git/trees/{sha}?recursive=1  → get full file list
   │
   ├─▶ filter list down to just your folder's paths
   │
   ├─▶ GET /repos/{owner}/{repo}/git/blobs/{file_sha}   → repeated per file
   │        (decode Base64 → raw bytes)
   │
   ├─▶ JSZip: pack all files into one archive, preserving paths
   │
   └─▶ create Blob → fake <a> click → browser saves the .zip
```

Everything here runs client-side. There's no backend server of yours involved
at all — it's just your browser talking straight to GitHub's API.
