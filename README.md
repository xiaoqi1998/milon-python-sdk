# milon-python-sdk

Python SDK for the [Milon](https://milonlabs.com) blockchain: key management, address derivation,
IDL instruction encoding/decoding, transaction building and signing, and RPC queries.
It is a faithful, test-locked port of the official Go SDK (`milon-go-sdk`).

[中文文档](README.zh-CN.md)

## Features

- **Four curves** — secp256k1, Ed25519, BLS12-381 and FN-DSA-512 (pending), with domain-separated
  Blake3 address derivation that never lands in the contract-reserved `0xFF` address space.
- **Runtime IDL layer** — 11 on-chain apps (system, account, token, staking, identity, sftoken,
  dex, validator, keyless, social, demo) are loaded from the bundled `provider/IDL/*.json`, so
  instruction builders, discriminators and type tags always match the chain's current IDL.
- **Typed builders** — `gen.TOKEN.MINT.args(...).encode()`; view responses decode straight into
  Python values, including `Result` `Err` payloads as structured `ApiFailure`.
- **Transaction builder with signature modes** — unified payer, per-instruction signers,
  multisig (k-of-n), skip-pubkey (address lookup on chain) and vote-gated auth bits.
- **Pure Python crypto stack** — `blake3`, `coincurve`, `cryptography`, `py_ecc`, `base58`;
  no Go/CGO dependency and no daemon required.

## Install

```bash
pip install .
```

Requirements: Python >= 3.10.

## Quickstart

```python
import milon_sdk as m
from milon_sdk import gen
from milon_sdk.crypto import new_address_from_public_key, new_pure_classical_secret_key
from milon_sdk.helper import check_tx_success
from milon_sdk.lib import PubKeySignatureMode, new_transaction_builder

client = m.new_client(m.DEV_NET)
rpc = client.RpcClient

# 1. Create an account key pair and its chain address
sk = new_pure_classical_secret_key()
pk = sk.ed25519_public()
addr = new_address_from_public_key(pk)

# 2. Claim test MIL from the devnet faucet (self-paid gas)
rpc.claim_faucet(sk, addr, PubKeySignatureMode(PublicKey=pk))
print("MIL balance:", rpc.balance_of(addr))

# 3. Build, sign and submit an IDL instruction
wire = gen.TOKEN.MINT.args(token=addr, to=addr, amount=1_000_000).encode()
tx = (
    new_transaction_builder([wire])
    .with_payer(addr)
    .add_ix_and_payer_sig(addr, sk, 0, PubKeySignatureMode(PublicKey=pk))
    .build()
)
rpc.submit_tx(tx)
check_tx_success(rpc.wait_for_transaction(tx.tx_hash()).BodyTxHistory)

# 4. Read a view method and decode the result
print(rpc.balance_of(addr))  # decoded view value
```

Runnable end-to-end scripts live in [`examples/`](examples):

- `e2e_token_demo.py` — faucet → create token → mint → transfer → tx history → event decode → simulate
- `sig_modes_demo.py` — signature-mode matrix (unified payer, skip-pubkey, 2-of-2 multisig,
  separate instruction signers, BLS accounts)

## Networks

| Network  | RPC                                   | Chain ID    |
|----------|---------------------------------------|-------------|
| `DEV_NET`| `https://devnet.milonlabs.com/v1/rpc` | 900000001   |
| `LOCAL_NET` | `http://127.0.0.1:6280/v1/rpc`     | 900000001   |

## Testing

```bash
pip install -e ".[test]"
pytest
```

`tools/check_gen_parity.py` cross-checks the bundled IDL data against a `milon-go-sdk` checkout:

```bash
python tools/check_gen_parity.py ../milon-go-sdk
```

## Status and known gaps

Honest list of what this SDK does **not** do yet:

- **Block streaming over gRPC** (`blockStream`, `framedHistory`, `printFramedHistory`) is not
  ported. Use RPC polling (`wait_for_transaction`, `get_block_by_height`) instead.
- **FN-DSA-512** (post-quantum) keys, signatures and verification are stubbed; the corresponding
  tests are skipped (`_fndsa` pure-Python port in progress). All other curves are complete.
- Batch submit / batch simulate method types (`6`, `11`) exist as wire constants only; no client
  helper is exposed yet. Batch view (`16`) is in use.
- Python API names follow a documented mechanical mapping from Go (`docs/PORTING.md`); some
  identifiers keep Go PascalCase field names for traceability.

## Documentation

- [`docs/PORTING.md`](docs/PORTING.md) — the Go→Python translation conventions (naming, error
  handling, integer widths, JSON/postcard mapping, cryptography library choices, test rules).
  Read this before touching codec or crypto code.

## License

MIT — see [LICENSE](LICENSE).
