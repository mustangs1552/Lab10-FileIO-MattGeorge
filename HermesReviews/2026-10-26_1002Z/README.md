# Security Review — mustangs1552/Lab10-FileIO-MattGeorge @ 7e3208f748

- **Repo:** mustangs1552/Lab10-FileIO-MattGeorge
- **Pinned commit:** `7e3208f74800ed71e00df1dd6fe371a596ee3dbd` (7e3208f748, "Initial upload", Matthew George, 2015-12-14T17:33:07Z)
- **Reviewed:** 2026-10-26 (UTC), automated security review (Hermes)
- **Branch reviewed:** `master` (the repo's only commit — HEAD of default branch — IS the pinned commit)
- **Scope:** all 15 tracked files; all 6 hand-written Java sources read end-to-end (InputReader.java 45 LOC, Responder.java 196 LOC, SupportSystem.java 74 LOC, TechSupport.java 8 LOC), plus README.TXT, both response data files, all three BlueJ `.ctxt` files, and package.bluej.

## Owner exceptions applied

None. `HermesReviews/reviewNotes.md` was fetched before any code was read and returned **404 Not Found** — no owner notes exist, so no findings were suppressed and there are no exceptions to quote. No notes-vs-code conflicts to reconcile.

## Summary verdict

**No Critical or High findings.** This is a self-contained BlueJ teaching project (a keyword-matching "tech support" chatbot from *Objects First with Java*) with no network, no persistence beyond stdout, no auth surface, and no build/dependency system. The only write-adjacent behavior is printing canned strings. Two Low findings and a few informational notes follow.

## Findings

### [LOW] F1 — Crash on malformed line in `responses.txt` (uncaught `ArrayIndexOutOfBoundsException`)

- **File:** `tech-support-io-v7/Responder.java:140-141` (parser loop for `EXTRA_RESPONSES`)
- **What is wrong:** Startup parses each line of `responses.txt` with `response.split(" ")` and immediately dereferences the first token: `String[] words = response.split(" "); ... responseMap.put(words[0].toLowerCase(), resp);`. An empty line in the file produces `words.length == 0`, so `words[0]` throws an uncaught `ArrayIndexOutOfBoundsException` out of the constructor; the program dies before its input loop. Whitespace-only lines are an easy mistake (an editor writing trailing blank lines). Impact is confined to local startup of a lab program, hence Low — but it is a real, deterministic DoS/crash triggered purely by file content.
- **Concrete fix:** Guard each iteration, e.g. `if (response == null || response.trim().isEmpty()) continue; String[] words = response.split("\\s+");` and additionally check `words.length >= 1` before `words[0]` (treat a lone keyword with no response as `resp = ""`).

### [LOW] F2 — Stale compiled IDE artifacts committed beside sources

- **Files:** `tech-support-io-v7/*.class` (InputReader.class, Responder.class, SupportSystem.class, TechSupport.class), plus `*.ctxt` and `package.bluej` (BlueJ metadata)
- **What is wrong:** Committed `.class` binaries can be loaded ahead of their `.java` sources (classpath-ordered runs, BlueJ's cached recompile), and repo artifacts can go stale when only sources are edited. For a reviewer this also means the repo distributes executable bytecode that is not covered by the project's source review. Verified clean today: a strings-level pass over all four committed `.class` files found only string constants, standard `java.util`/`java.io` references, and no dangerous APIs (no `Runtime.exec`/`ProcessBuilder`, sockets, HTTP, reflection, native or file-write calls) — i.e. the binaries match the benign sources as of this commit. Still, binary artifacts have no review trail: a later push that quietly swaps a `.class` while leaving `Responde.java`-style mismatched sources would be easy to miss.
- **Concrete fix:** Remove the `*.class` files from version control and add them to `.gitignore` (BlueJ metadata files are conventionally kept only if needed; typically skip `*.ctxt` and `package.bluej` too). Recompile from source when running.

## Informational notes (no action required)

- **No secrets anywhere in the tree.** All 15 files (sources, data files, README, BlueJ metadata) read in full; no tokens, keys, passwords, credential-bearing URLs, or `.env`-like content. Committed artifact types are only: Java sources, UTF/ASCII text data files, `.class` bytecode, BlueJ `.ctxt`/`package.bluej` metadata.
- **No injection, no deserialization, no command/SQL surface.** The only executable constructs are string concatenation and `HashMap` lookups; responses read from `default.txt`/`responses.txt` are printed verbatim by the local terminal chatbot. `InputReader.java:34-36`'s `split(" ")`/`trim().toLowerCase()` never leaves process memory. A strings pass over all four committed `.class` files corroborated this: no `Runtime`/`ProcessBuilder`/`Socket`/reflect/HTTP references.
- **No auth/authorization code exists** (nothing to misconfigure); the program has no network I/O at all.
- **No dependency or build files exist** (no `pom.xml`/`build.gradle`); only the standard library is used, so there are no unpinned external dependencies to flag.
- **No sensitive data in logs or config.** The only stdout/stderr output is canned response strings and two file-open failure messages (`Responder.java:147-151, 172-176`).
- **Minor robustness nits, not security issues:** pinned `US-ASCII` charset on both readers (`Responder.java:130, 163`) will mojibake non-ASCII responses; `split(" ")` in `InputReader.java:36` mishandles runs of multiple spaces and tabs (empty tokens); `java.util.Random` (`Responder.java:40`) is fine here because picks are cosmetic, not security tokens.

## Method

Read-only review at the pinned SHA. Clone was a shallow, blobless `--filter=blob:none` clone checked out at 7e3208f748; owner notes endpoint consulted first via API (404). Every tracked file was enumerated from `git ls-tree -r` and read end-to-end; the four `.class` files were additionally scanned at the strings level for dangerous API references, and the commit history (exactly one commit) checked for prior review docs — none exist; this document is the first entry in `HermesReviews/`.