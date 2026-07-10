# Canton / Daml Local Development Setup

SDK version: **3.4.11**

## 1. Install the Daml SDK

```bash
curl -sSL https://get.daml.com/ | sh -s 3.4.11
```

If `daml: command not found` after install, the binary isn't on PATH yet:

```bash
export PATH=$PATH:~/.daml/bin
echo 'export PATH=$PATH:~/.daml/bin' >> ~/.zshrc   # or ~/.bashrc
source ~/.zshrc
```

Verify:

```bash
daml version
```

## 2. Build the DAR

From the project root (contains `daml.yaml`):

```bash
daml build
```

Output:

```
.daml/dist/canton-demo-etf-0.1.0.dar
```

## 3. Run the Local Sandbox

Use `daml start` — **not** bare `daml sandbox`.

```bash
daml start
```

`daml start` builds the DAR, uploads it, and starts the sandbox + JSON API **with a domain/synchronizer already connected**, all in one step.

> **Why not `daml sandbox`?** Running `daml sandbox` on its own starts a bare participant node with no synchronizer domain connection. Contracts will create successfully, but any ACS/read query fails with:
> ```
> UNKNOWN_INFORMEES: The participant is not connected to any synchronizer where the given informees are known.
> ```
> Manually running `daml ledger allocate-parties` afterward does not fix this — the missing piece is the domain connection itself, which `daml start` provisions automatically.

## 4. Upload the DAR to Localnet (manual path)

`daml start` handles this automatically, but if you need to upload a DAR to an already-running sandbox (e.g. after a rebuild):

```bash
daml ledger upload-dar \
  --host localhost \
  --port 6865 \
  .daml/dist/canton-demo-etf-0.1.0.dar
```

Port `6865` is the local sandbox's Ledger API port.

## Known Gotchas

- **`daml sandbox` alone ≠ working ledger reads.** Always use `daml start` for local dev unless you specifically need the bare participant.
- **Party fingerprints regenerate on sandbox restart.** Any hardcoded party ID (e.g. in JWT validators, test constants, seed scripts) needs to be updated after a fresh `daml start`.
- **`dpm sandbox`** is the upcoming replacement for `daml sandbox` per Canton DevRel — expect this tooling to shift.