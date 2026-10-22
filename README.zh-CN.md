# milon-python-sdk（中文）

Milon 链的 Python SDK：密钥与地址、IDL 指令编解码、交易构建与签名、RPC 查询。
本仓库是官方 Go SDK（`milon-go-sdk`）的逐文件对译实现，功能与测试与 Go 侧对齐。

[English](README.md)

## 能力

- **四条曲线**：secp256k1、Ed25519、BLS12-381、FN-DSA-512（后者移植中，暂为占位）。
  地址派生使用域分离 Blake3，并自动跳过合约保留区（首字节 `0xFF`）。
- **运行时 IDL 层**：随包分发 `provider/IDL/*.json`，11 个链上应用（system、account、token、
  staking、identity、sftoken、dex、validator、keyless、social、demo）的指令判别器与 typeTag
  始终与链当前 IDL 一致，无需重新生成代码。
- **类型化构建器**：`gen.TOKEN.MINT.args(...).encode()`；view 响应直接解码为 Python 值，
  `Result` 的 `Err` 分支解码为结构化 `ApiFailure`。
- **交易构建器与多种签名模式**：统一 payer、逐指令签名、多签 k-of-n、SkipPubKey（链上定位地址）、
  vote gate 授权位。
- **纯 Python 密码学**：依赖 `blake3`、`coincurve`、`cryptography`、`py_ecc`、`base58`，
  不需要 Go/CGO，也不需要本地守护进程。

## 安装

```bash
pip install .
```

Python >= 3.10。

## 快速开始

```python
import milon_sdk as m
from milon_sdk import gen
from milon_sdk.crypto import new_address_from_public_key, new_pure_classical_secret_key
from milon_sdk.helper import check_tx_success
from milon_sdk.lib import PubKeySignatureMode, new_transaction_builder

client = m.new_client(m.DEV_NET)
rpc = client.RpcClient

# 1. 生成密钥对并派生链上地址
sk = new_pure_classical_secret_key()
pk = sk.ed25519_public()
addr = new_address_from_public_key(pk)

# 2. devNet 领水（账户自付 gas）
rpc.claim_faucet(sk, addr, PubKeySignatureMode(PublicKey=pk))
print("MIL 余额：", rpc.balance_of(addr))

# 3. 构建 / 签名 / 提交一条 IDL 指令
wire = gen.TOKEN.MINT.args(token=addr, to=addr, amount=1_000_000).encode()
tx = (
    new_transaction_builder([wire])
    .with_payer(addr)
    .add_ix_and_payer_sig(addr, sk, 0, PubKeySignatureMode(PublicKey=pk))
    .build()
)
rpc.submit_tx(tx)
check_tx_success(rpc.wait_for_transaction(tx.tx_hash()).BodyTxHistory)
```

可直接运行的链上脚本在 [`examples/`](examples)：

- `e2e_token_demo.py`：领水 → 建代币 → 铸币 → 转账 → 交易详情 → 事件解码 → 模拟预估（10 步）
- `sig_modes_demo.py`：签名模式矩阵（纯赞助、SkipPubKey、2-of-2 多签、逐指令分签、BLS 账户，12 项）

## 网络

| 网络        | RPC                                   | Chain ID  |
|-------------|---------------------------------------|-----------|
| `DEV_NET`   | `https://devnet.milonlabs.com/v1/rpc` | 900000001 |
| `LOCAL_NET` | `http://127.0.0.1:6280/v1/rpc`        | 900000001 |

## 测试

```bash
pip install -e ".[test]"
pytest
```

`tools/check_gen_parity.py` 用于与 Go SDK 对账 IDL 数据（指令集合、判别器、typeTag 全等）：

```bash
python tools/check_gen_parity.py ../milon-go-sdk
```

## 现状与未支持项

如实列出暂未覆盖的部分：

- **gRPC 区块流**（`blockStream`、`framedHistory`、`printFramedHistory`）未移植，请用 RPC 轮询
  （`wait_for_transaction`、`get_block_by_height`）替代。
- **FN-DSA-512** 后量子曲线仍为占位实现，相关测试以 skip 标注；其余曲线完整可用。
- 批提交 / 批模拟的方法号（`6`、`11`）仅作为协议常量存在，尚未提供 client 封装；
  批 view（`16`）已启用。
- Python 命名遵循 `docs/PORTING.md` 的机械映射规则，部分字段保留 Go 的 PascalCase 以便对读。

## 文档

- [`docs/PORTING.md`](docs/PORTING.md)：Go → Python 移植约定（命名、错误处理、整数位宽、
  JSON/postcard 映射、密码学库选型、测试规则）。改动编解码与密码学代码前请先读。

## 许可

MIT，见 [LICENSE](LICENSE)。
