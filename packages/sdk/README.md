# @splitstream/sdk

Thin, typed client for the [SplitStream](https://github.com/SplitStream-Labs/splitstream-sdk-cli) contributor payout vault on Stellar — a [Soroban](https://soroban.stellar.org) contract that pays open-source contributors pro-rata for the issues they closed in a cycle.

The SDK reads vault state, parses the per-cycle payout manifests the [splitstream-actions](https://github.com/SplitStream-Labs/splitstream-actions) bridge publishes, and reconstructs the Merkle proofs a claim needs. It **never holds funds and never holds a private key**.

Browser-safe: no `node:crypto`, no CLI-only dependencies.

```bash
npm install @splitstream/sdk
```

## Design rules

- **It never signs.** The `build*` methods return *unsigned* transactions. You sign them with a wallet, a hardware device or a keypair, then hand the signed envelope back to `submitSigned`.
- **Token amounts are `bigint` end to end.** Decimal strings appear only at the display and parse boundaries. A rounded payout is a wrong payout, so there is no `Number` conversion anywhere in the amount path.
- **Nothing is defaulted about the network.** Every network identity is a required option, and the constructor validates them up front.

## Quick start

```ts
import { SplitStreamClient } from '@splitstream/sdk';

const client = new SplitStreamClient({
  rpcUrl: process.env.SPLITSTREAM_RPC_URL!,
  networkPassphrase: process.env.SPLITSTREAM_NETWORK_PASSPHRASE!,
  vaultContractId: process.env.SPLITSTREAM_VAULT_CONTRACT_ID!,
  tokenContractId: process.env.SPLITSTREAM_TOKEN_CONTRACT_ID!,
});

const balance = await client.getBalance(address);   // bigint, base units
const root = await client.getCycleRoot(3);          // lowercase hex, or null
const claimed = await client.hasClaimed(3, address);
const decimals = await client.getTokenDecimals();   // read from the token contract

const tx = await client.buildClaimTx(address, 3, amount, proof); // unsigned
const signedXdr = await myWallet.sign(tx);                       // you sign it
const { hash, status } = await client.submitSigned(signedXdr);
```

### Client options

| Option | Required | Meaning |
| --- | --- | --- |
| `rpcUrl` | yes | Soroban RPC endpoint; an absolute http(s) URL |
| `networkPassphrase` | yes | Must match `rpcUrl`'s network |
| `vaultContractId` | yes | Vault contract id (`C...`) |
| `tokenContractId` | yes | Payout token contract id (`C...`) |
| `server` | no | Inject an `rpc.Server` (a test seam) |
| `pollIntervalMs` | no | Poll interval while awaiting a submission; default `2000` |
| `pollTimeoutMs` | no | Give up after this long; default `60000` |

A missing or malformed value throws `SplitStreamError` at construction, so a misconfigured client fails immediately rather than at the first call.

### Reads

Reads are read-only simulations. They need no funded account.

| Method | Returns |
| --- | --- |
| `getBalance(contributor)` | Claimable balance in the vault, in base units |
| `getCycleRoot(cycleId)` | Published Merkle root, lowercase hex, or `null` if unposted |
| `hasClaimed(cycleId, contributor)` | Whether that contributor has pulled that cycle |
| `getVesting(contributor)` | `VestingInfo`, or `null` when there is no schedule |
| `getTokenDecimals()` | Decimals read from the token contract |
| `getReserveBalance()` | The vault's own token balance, reserved for future cycles |

### Unsigned transactions

| Method | Builds |
| --- | --- |
| `buildClaimTx(contributor, cycleId, amount, proof)` | `credit_claim` |
| `buildWithdrawTx(contributor)` | `withdraw` |
| `buildClaimVestedTx(contributor)` | `claim_vested` |

Each returns an assembled, **unsigned** transaction with the footprint and resource fees already simulated. `buildClaimTx` rejects a negative amount or an empty proof before touching the network.

`submitSigned(signedXdr)` decodes the envelope against the client's passphrase, submits it and polls until the transaction is final, returning `{ hash, status }` where `status` is `'SUCCESS'` or `'FAILED'`. On failure it raises `SplitStreamError` with the contract's error code decoded into a name when the diagnostics carry one.

## Helpers

Exported alongside the client:

- **Merkle proofs** — `buildClaimProof(manifest, identifier)` is the one call a contributor needs; it returns the proof, the leaf, the computed root and whether they agree. `verifyMerkleProof`, `computeMerkleRoot`, `computeManifestRoot`, `merkleLeaf`, `hashPair`, `buildMerkleTree`, `merkleProofForLeaf` and `manifestToLeafInputs` are there so the scheme can be checked locally.
- **Manifests** — `parseManifest` validates and normalizes a raw manifest and throws `ManifestParseError` naming the offending field; `findManifestEntry`, `parseVesting`, `isValidStellarAddress` and `isValidContractId` round it out.
- **Amounts** — `formatTokenAmount`, `formatTokenAmountWithSeparators`, `parseTokenAmount`, `percentOf`, `truncateAddress`. `parseTokenAmount` rejects excess precision rather than rounding.
- **Errors** — `SplitStreamError` carries the numeric contract `code`, its decoded `codeName` and the raw `details`. `SPLITSTREAM_CONTRACT_ERRORS`, `contractError`, `parseContractErrorCode`, `splitstreamErrorName` and `toSplitStreamError` are exported for callers that classify failures themselves.
- **XDR and bytes** — the ScVal and byte helpers, so the rest of your code never has to import the XDR classes directly.

## The Merkle format is frozen

The leaf pre-image and the tree construction are shared **byte-for-byte** with [splitstream-actions](https://github.com/SplitStream-Labs/splitstream-actions) (which builds the root) and the vault (which verifies it). They are not this package's invention and cannot be changed here alone:

```
leaf(stellar, amount) = SHA-256( ScVal(Address(stellar)).toXDR() || ScVal(i128(amount)).toXDR() )

Address XDR (44 bytes): u32(SCV_ADDRESS=18) | u32(SC_ADDRESS_TYPE_ACCOUNT=0) | u32(publickey ED25519=0) | ed25519(32)
i128 XDR    (20 bytes): u32(SCV_I128=10) | int64 hi | uint64 lo

node(a, b)             = SHA-256( min(a, b) || max(a, b) )
```

Leaves are ordered by **ascending Stellar public-key bytes** — not by leaf hash and not by manifest row order — so the root is reproducible from the manifest alone. An unpaired node is promoted unchanged rather than hashed with itself, which is why a proof can be shorter than the tree is tall.

A different pre-image, leaf order or odd-node rule fails every claim with `InvalidProof` even though the manifest is correct.

## Requirements

Node.js **20 or newer** for `engines`, though the package itself is browser-safe.

## Documentation

- [SDK reference](https://splitstream.gitbook.io/splitstream-sdk-cli/sdk-reference) — the full API, the contract error table and the manifest format
- [Documentation site](https://splitstream.gitbook.io/splitstream-sdk-cli/)
- [Repository](https://github.com/SplitStream-Labs/splitstream-sdk-cli) — the CLI built on this SDK, and the current deployed contract ids
- [Issues](https://github.com/SplitStream-Labs/splitstream-sdk-cli/issues)

## License

MIT
