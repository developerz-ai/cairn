# 🔐 Security and access control

> One monorepo holds the most sensitive code next to everything else. The server enforces access, never the client.

| Area | Rule |
|---|---|
| **Identities** | Humans via OIDC (optional) or local accounts. Agents and CI get service principals with short-lived tokens scoped by path prefix, ref and RPC allowlist |
| **Path ACLs** | Permissions attach to directories. Tree reads hide entries the caller can't see. Out-of-scope paths answer exactly like missing ones |
| **Ref policies** | Who may land on which ref; required approvals; required verifier status for release refs |
| **Signed changesets** | Server-created changesets carry an Ed25519 signature and the authenticated actor. Audit trail doesn't trust the free-text author field |
| **Redaction** | Quarantine and purge a blob without changing any id. For leaked secrets. History stays intact; reads return a tombstone |
| **Audit** | Every ref move, ACL change, token issue and redaction is logged with actor and reason |
| **Untrusted input** | Server parses bytes from clients and agents. Every decoder fuzzed; ASan/UBSan in CI; decompression-bomb limits; path normalization in one audited function |
| **Transport** | TLS everywhere outside localhost. mTLS between tiers optional |

## Threat model (short)
- **Malicious or confused agent**: limited by token scope; can't see or land outside its prefixes; every action attributed.
- **Compromised cache tier**: can't forge objects (ids are verified by clients); can only deny service.
- **Hostile input**: oversized objects, zip-bomb zstd frames, path traversal (`..`, absolute, NUL, case-folding collisions) rejected at the boundary.
- **Leaked secret in history**: redaction, then rotate. Ids don't change, so nothing downstream breaks.
