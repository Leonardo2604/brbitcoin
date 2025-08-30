# 🇧🇷 BrBitcoin Javascript SDK

![hello-world](https://github.com/user-attachments/assets/4953259b-d889-4c74-ae3f-432f564dee83)

## Design Principles

- **Automatic Zeroization**: Sensitive data wiped from memory using context managers
- **Hierarchical Security**: BIP32/BIP39/BIP44 compliant HD wallets with encrypted backups
- **Network Agnostic**: Unified API for Regtest/Testnet/Mainnet operations
- **Full RPC Access**: Direct Bitcoin Core JSON-RPC integration
- **Type Safety**: Comprehensive type hints for better developer experience

## Features

- 🔐 Secure key management with memory zeroization
- 💳 HD wallet support (BIP32, BIP39, BIP44, BIP84)
- 📡 Multiple network backends (Bitcoin Core, Electrum, Custom)
- 📦 PSBT (Partially Signed Bitcoin Transaction) support
- ⚡️ Async-first architecture for network operations
- 🔄 UTXO management with automatic coin selection
- 📊 Blockchain data inspection utilities
- 🛠️ Low-level Bitcoin script builder

## 📦 Installation

### Via NPM

```bash
npm install brbitcoin
```

### Via Yarn

```bash
yarn add brbitcoin
```

## 🚀 Quick Start

> [!WARNING]
> Always test with REGTEST before MAINNET usage.

### 1. Wallet Management

```typescript
import { Wallet, Network } from "brbitcoin";

// Create random HD wallet (testnet by default)
async function createRandomWallet() {
  using wallet = Wallet.create();

  console.log(`Testnet new address: ${wallet.address}`);

  await wallet.exportEncrypted("wallet.json", process.env.WALLET_PASS);
}

// Import from existing key
function importFromPrivateKey() {
  using wallet = Wallet.fromPrivateKey("beefcafe...", Network.REGTEST);

  console.log(`Regtest address: ${wallet.address}`);
}

// Create from BIP39 mnemonic
async function createFromMnemonic() {
  const mnemonic =
    "absorb lecture valley scissors giant evolve planet rotate siren chaos";

  using wallet = await Wallet.fromMnemonic(mnemonic, Network.MAINNET);

  console.log(`Regtest address: ${wallet.address}`);
}
```

### 2. Blockchain Interaction

#### 2.1 Address Information

```typescript
import { getAddressInfo } from "brbitcoin";

async function getAddressInfoExample() {
  const info = await getAddressInfo("bc1q...", Network.MAINNET);

  console.log(`Balance: ${info.balance} satoshis`);
  console.log(`UTXOs: ${info.utxos.length}`);
}

async function getWalletBalance() {
  using wallet = new Wallet({ network: Network.TESTNET });

  const balance = await wallet.balance();
  console.log(`Wallet balance: ${balance} sats`);
}
```

#### 2.2 Transaction Inspection

```typescript
import { getTransaction, Wallet, Network } from "brbitcoin";

async function getTransactionExample() {
  const tx = await getTransaction("aabb...", Network.REGTEST);

  console.log(`Confirmations: ${tx.confirmations}`);

  for (const output of tx.outputs) {
    console.log(`Output value: ${output.value}`);
  }
}

async function getWalletUTXOs() {
  using wallet = new Wallet({ network: Network.TESTNET });

  const utxos = await wallet.utxos();

  console.log(`UTXOs: ${utxos.length}`);
}
```

#### 2.3 Block Exploration

```typescript
import { getBlock, Network } from "brbitcoin";

async function getBlockExample() {
  // By hash
  const block = await getBlock("000000000019d6...", Network.MAINNET);
  console.log(`Block height: ${block.height}`);

  // By number
  const genesis = await getBlock(0, Network.MAINNET);
  console.log(`Genesis timestamp: ${genesis.timestamp}`);
}
```

### 3. Transaction Building

#### 3.1 High-Level (Recommended)

```typescript
import { Wallet } from "brbitcoin";

const RECEIVER = "tb123...";
const AMOUNT = 0.001; // BTC

async function sendExample() {
  using wallet = new Wallet({ network: Network.REGTEST });

  const txid = await wallet.send({ to: RECEIVER, amount: AMOUNT });
  console.log(`Broadcasted TX ID: ${txid}`);
}
```

#### 3.2 Mid-Level Control

```typescript
import { Wallet, Transaction, toBtc } from "brbitcoin";

const RECEIVER = "tb123...";
const AMOUNT = 100_000; // Satoshis == 0.001 BTC
const FEE = 500; // Satoshi == 0.0000005 BTC

async function sendExample() {
  using wallet = new Wallet({ network: Network.REGTEST });

  const utxos = await wallet.utxos();

  const txid = await new Transaction({ network: wallet.network })
    .addInput(utxos[0])
    .addOutput(RECEIVER, toBtc(AMOUNT))
    .fee(toBtc(FEE))
    // .estimateFee()
    .sign(wallet)
    .broadcast();

  console.log(`Broadcasted TX ID: ${txid}`);
}
```

#### 3.3 Low-Level Scripting

```typescript
import { Wallet, Script, Transaction } from "brbitcoin";

async function lowLevelScriptingExample() {
  // Create a P2SH lock script
  const lockScript = Script()
    .pushOpDup()
    .pushOpHash160()
    .pushBytes(pubkeyHash)
    .pushOpEqualVerify()
    .pushOpCheckSig();

  using wallet = new Wallet({ network: Network.REGTEST });

  const inputs = await wallet.utxos();
  const AMOUNT = 0.0001; // BTC
  const txid = new Transaction({ network: wallet.network })
    .addInput(inputs[0])
    .addOutputScript(lockScript, AMOUNT)
    .sign(wallet)
    .broadcast();

  console.log(`Broadcasted TX ID: ${txid}`);
}
```

### 4. Taproot Transactions (BIP340/341/342)

#### 4.1 Generating Taproot Address

```typescript
import { Wallet, TaprootBuilder, Script, Op, Network } from "brbitcoin";

// Generate internal key
async function generateTaprootAddress() {
  using wallet = new Wallet({ network: Network.MAINNET });

  const internalKey = wallet.taprootInternalKey();

  // Build Taproot script tree
  const script = Script()
    .pushOpHash160()
    .pushBytes("my_hash160")
    .pushOpEqual();
  const taproot = TaprootBuilder(internalKey)
    .addLeafScript(script)
    .finalize();

  console.log(`Taproot Address: ${taproot.address}`);
  console.log(`Control Block: ${taproot.controlBlock}`);
}
```

#### 4.3 Spending from Taproot (Key Path)

```typescript
import { Wallet, Transaction, Network } from "brbitcoin";

// Spending using Schnorr signature
async function spendFromTaproot() {
  using wallet = Wallet.fromTaprootInternalKey("internal_key_hex");

  const utxo = wallet.utxos()[0];

  const tx = await new Transaction({ network: Network.MAINNET })
    .addTaprootInput(utxo)
    .addOutput("bc1q...", 0.009)
    .setChange(wallet.taprootAddress)
    .estimateFee()
    .signTaproot(wallet)
    .broadcast();

  console.log(`Key path spend TX: ${tx.txid}`);
}
```

#### 4.4 Spending from Taproot (Script Path)

```typescript
import { Wallet, TaprootScriptSolution, Script, Network } from "brbitcoin";

// Reveal script and provide solution
async function spendFromTaprootScriptPath() {
  const preimage = "secret123";
  const script = new Script()
    .pushOpHash160()
    .pushBytes(hash160(preimage))
    .pushOpEqual();

  const solution = new TaprootScriptSolution({
    script,
    solutionOps: [Script.opPushBytes, preimage],
  });

  using wallet = new Wallet({ network: Network.REGTEST });

  const utxo = wallet.utxos()[0];

  const tx = await new Transaction({ network: Network.REGTEST })
    .addTaprootInput(utxo, solution)
    .addOutput("bc1q...", 0.0095)
    .signTaproot(wallet)
    .broadcast();

  console.log(`Script path spend TX: ${tx.txid}`);
}
```

#### 4.5 Taproot Benefits

- Privacy: All spends look identical on-chain
- Efficiency: Smaller witness size vs traditional multisig
- Flexibility: Combine multiple spending conditions
- Standard: BIP340 (Schnorr), BIP341 (Taproot), BIP342 (Tapscript)

### 5. Security Practices

#### 5.1 Encrypted Private Key Backup

```typescript
import { Wallet } from "brbitcoin";

const PATH = "wallet.json";
const PASS = process.env.WALLET_PASS;

async function createWallet() {
  using wallet = Wallet.create();

  await wallet.exportEncrypted({ path: PATH, password: PASS });
}
```

#### 5.2 Restore from Encrypted backup

```typescript
import { Wallet } from "brbitcoin";

const PATH = "wallet.json";
const PASS = process.env.WALLET_PASS;

async function restoreWallet() {
  using wallet = await Wallet.fromEncrypted({
    path: PATH,
    password: PASS,
  });

  console.log(`Recovered address: ${wallet.address}`);
}
```

#### 5.3 Zeroization Guarantees

```typescript
// Keys are wiped:
// - When context manager exits
// - After signing/broadcast
// - On object destruction
async function zeroizationExample() {
  using wallet = Wallet.fromPrivateKey("c0ffee...");

  const txid = await wallet.send("bc1q...", 0.001);
  console.log(`TX ID: ${txid}`);
}
```

### 6. Node Management

#### 6.1 Network Configuration

```typescript
import { NodeClient, Network } from "brbitcoin";

// Connect to Bitcoin Core
const client = new NodeClient({
  network: Network.REGTEST,
  rpcUser: "user",
  rpcPassword: "pass",
  host: "localhost",
  port: 18443,
});
```

#### 6.2 Node Operations

```typescript
async function nodeOperationsExample() {
  // Get blockchain info
  const info = await client.getBlockchainInfo();
  console.log(`Blocks: ${info.blocks}, Difficulty: ${info.difficulty}`);

  // Generate regtest blocks
  if (client.network === Network.REGTEST) {
    const blocks = await client.generateToAddress(10, "bcrt1q...");
    console.log(`Mined block: ${blocks[-1]}`);
  }

  // Get fee estimates
  const fees = await client.estimateFee([1, 3, 6]);
  console.log(`1-block fee: ${fees[1]} BTC/kvB`);
}
```

#### 6.3 Direct RPC Access

```typescript
async function rpcExample() {
  // Raw RPC commands
  const mempool = await client.rpc("getmempoolinfo");
  console.log(`Mempool size: ${mempool["size"]}`);

  // Batch requests
  const results = await client.batchRpc([
    ("getblockcount", []),
    ("getblockhash", [0]),
    ("getblockheader", ["000000000019d6..."]),
  ]);
  console.log(`Block count: ${results[0]}`);
}
```

#### 6.4 Bitcoin Core RPC Command Reference (Partial)

| Category       | Command                | Description                   | Example Usage                                                      |
| -------------- | ---------------------- | ----------------------------- | ------------------------------------------------------------------ |
| **Blockchain** | `getblockchaininfo`    | Returns blockchain state      | `getblockchaininfo`                                                |
|                | `getblock`             | Get block data by hash/height | `getblock "blockhash" 2`                                           |
|                | `gettxoutsetinfo`      | UTXO set statistics           | `gettxoutsetinfo`                                                  |
| **Wallet**     | `listtransactions`     | Wallet transaction history    | `listtransactions "*" 10 0`                                        |
|                | `sendtoaddress`        | Send to Bitcoin address       | `sendtoaddress "addr" 0.01`                                        |
|                | `backupwallet`         | Backup wallet.dat             | `backupwallet "/path/backup.dat"`                                  |
| **Network**    | `getnetworkinfo`       | Network connections/version   | `getnetworkinfo`                                                   |
|                | `addnode`              | Manage peer connections       | `addnode "ip:port" "add"`                                          |
| **Mining**     | `getblocktemplate`     | Get mining template           | `getblocktemplate {"rules":["segwit"]}`                            |
|                | `submitblock`          | Submit mined block            | `submitblock "hexdata"`                                            |
| **Utility**    | `validateaddress`      | Validate address              | `validateaddress "bc1q..."`                                        |
|                | `estimatesmartfee`     | Estimate transaction fee      | `estimatesmartfee 6`                                               |
| **Raw Tx**     | `createrawtransaction` | Create raw transaction        | `createrawtransaction '[{"txid":"...","vout":0}]' '{"addr":0.01}'` |
|                | `signrawtransaction`   | Sign raw transaction          | `signrawtransaction "hex"`                                         |
| **Control**    | `stop`                 | Shut down node                | `stop`                                                             |
|                | `uptime`               | Node uptime                   | `uptime`                                                           |

### 7. Hierarchical Deterministic (HD) Wallets

#### 7.1 Creating HD Wallets (BIP32/BIP44 compliant)

```typescript
import { Wallet, Network } from "brbitcoin";

async function createHDWallet() {
  using hdWallet = await Wallet.createHD();
  const firstAddress = hdWallet.deriveAddress(0);
  const secondAddress = hdWallet.deriveAddress(1);
  const hundredthAddress = hdWallet.deriveAddress(99);

  console.log(`Master xpub: ${hdWallet.xpub}`);
  console.log(`Derivation path: ${hdWallet.derivationPath}`);
  console.log(`First Address: ${firstAddress}`);
  console.log(`Second Address: ${secondAddress}`);
  console.log(`Hundredth Address : ${hundredthAddress}`);
}
```

#### 7.2 Advanced Derivation Paths

```typescript
import { Wallet, Network } from "brbitcoin";

async function advancedDerivationPaths() {
  // Custom derivation schemes
  using segwitWallet = await Wallet.createHD({
    purpose: 84, // BIP84 (SegWit)
    network: Network.MAINNET,
    accountIndex: 3,
  });

  console.log(`Native SegWit address: ${segwitWallet.address}`);

  // Custom derivation path
  using hdWallet = await Wallet.createHD({
    network: Network.MAINNET,
    path: "m/44'/0'/1'",
  });

  console.log(`Custom path derivation address: ${hdWallet.deriveAddress(2)}`);
}
```

#### 7.3 Hardware Wallet Integration

```typescript
import { Wallet, Network } from "brbitcoin";

async function hardwareWalletIntegration() {
  using hwWallet = await Wallet.fromHardwareDevice({
    deviceType: "ledger",
    network: Network.MAINNET,
  });

  const txid = await hwWallet.send("bc1q...", 0.01);
  console.log(`Broadcasted TX ID: ${txid}`);
}
```

#### 7.4 HD Wallet Supported Standards

| Standard | Purpose                     | Example Path        |
| -------- | --------------------------- | ------------------- |
| BIP32    | Hierarchical Key Derivation | m/0'/1              |
| BIP39    | Mnemonic Phrase Generation  | 24-word seed        |
| BIP44    | Multi-Account Structure     | m/44'/0'/0'/0/0     |
| BIP84    | Native SegWit (Bech32)      | m/84'/0'/0'/0/0     |
| BIP174   | PSBT (Partially Signed Tx)  | PSBT format support |

---

# [License: MIT](../LICENSE)
