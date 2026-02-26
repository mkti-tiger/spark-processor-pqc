# Quantum-Resistant NFT Issuance Protocol (β1.1.0)

**Symbol Blockchain + FN-DSA-512 (Falcon) PQC Layer**  
**バージョン**: β1.1.0 (2026-02-26)  
**ライセンス**: CC0 1.0 (パブリックドメイン)

> 📄 **Spark Processor全体仕様 →** [README.md](./README.md)

---

## 目次

- [概要](#1-概要)
- [二層アーキテクチャ](#2-二層アーキテクチャ)
- [コアフロー（4ステップ）](#3-コアフロー4ステップ)
- [データモデル](#4-データモデルv11改善版)
- [実装フロー](#5-実装フローsymbol-typescript-sdk例)
- [5.4 実装上の注意点](#54-実装上の注意点gemini指摘β110追記)
- [オフチェーン検証仕様](#6-オフチェーン検証仕様リファレンス)
- [セキュリティ特性と制約](#7-セキュリティ特性と制約)
- [容量設計](#8-容量設計)
- [実装ステータス](#9-実装ステータス)
- [ロードマップ](#10-ロードマップ)
- [参考資料](#参考資料)

---

## 1. 概要

本プロトコルは、Symbolブロックチェーンのネイティブ機能（Multisig + Aggregate Transaction）を最大限活用しつつ、最終状態の真正性を **FN-DSA-512 (NIST標準 PQC)** で保証する**二層構造型量子耐性NFT発行プロトコル**です。

- **コンセンサス層**：SymbolネットワークのEd25519 + Multisig + Aggregate で原子性・承認記録を保証
- **量子耐性層**：最終状態ハッシュに対して**1回のみ**FN-DSA-512署名を適用（O(1)コスト）

これにより、署名サイズ・手数料を最小化しつつ、量子コンピュータ耐性を付与します。

> ⚠️ **β1.1.0明確化**：PQC署名はアプリケーション層で検証されます。  
> Symbolノードは現行仕様ではEd25519のみをコンセンサスレベルで検証します。  
> NFT発行・真正性証明・監査用途では十分に有効です。

---

## 2. 二層アーキテクチャ

| 層 | 保証内容 | 使用技術 | 検証場所 |
|---|---|---|---|
| **コンセンサス層** | 原子性・請負人承認記録 | Ed25519, Multisig, Aggregate Tx | Symbolノード（自動） |
| **量子耐性層** | 最終状態真正性（量子耐性） | FN-DSA-512 + Merkle Tree + SHA3-512 | オフチェーン（検証ツール） |

---

## 3. コアフロー（4ステップ）

```
Step 1: 一時的 2-of-2 Multisig アカウント作成
        オーナー + 請負人アドレスで一時Multisigを作成
        （発行プロセス中のみ使用・完了後に解除）
        ↓
Step 2: Multisig下でAggregate発行
        - NFT Mosaic作成（supply=1, divisibility=0）
        - Merkle Tree圧縮メタデータ付与
        - 所有権をMultisigへ設定
        （この時点ではPQC未使用）
        ↓
Step 3: 請負人によるCosignature
        Ed25519でCosign → Aggregate成立 → NFT正式発行
        ↓
Step 4: Multisig解除 + 所有権移転 + PQC署名（原子的実行）
        オーナーが単独でAggregateCompleteを発行：
        - Multisig解除
        - Mosaic所有権をオーナー通常アドレスへ移転
        - 最終状態ハッシュに対するFN-DSA-512署名をMetadataに記録
```

---

## 4. データモデル（v1.1改善版）

### 最終状態ハッシュ（FinalStateDescriptorV2）

```typescript
interface FinalStateDescriptorV2 {
  networkGenerationHash: string;       // チェーン固有（チェーン間の使い回し防止）
  mosaicIdHex: string;                 // NFT識別子
  merkleRootHex: string;               // メタデータのMerkle Root
  ownerAddressRaw: string;             // 最終オーナーアドレス
  finalizedHeight: string;             // 確定ブロック高
  finalizedGenerationHash: string;     // 最終確定時のGH（フォーク耐性強化）
}

function computeFinalStateHashV2(desc: FinalStateDescriptorV2): Uint8Array {
  const payload = Buffer.concat([
    Buffer.from(desc.networkGenerationHash, 'hex'),
    Buffer.from(desc.mosaicIdHex, 'hex'),
    Buffer.from(desc.merkleRootHex, 'hex'),
    Buffer.from(desc.ownerAddressRaw, 'hex'),
    Buffer.from(desc.finalizedHeight, 'utf8'),
    Buffer.from(desc.finalizedGenerationHash, 'hex'),
  ]);
  return sha3_512(payload); // Symbol互換SHA3-512
}
```

### PQCメタデータ形式（CBOR固定構造推奨）

JSON/hex → CBOR に移行することで、パース高速・サイズ最小・将来拡張性向上を実現します。

```typescript
// CBORメタデータ構造（概念）
{
  v: 2,                          // プロトコルバージョン
  alg: "FN-DSA-512",             // アルゴリズム識別子
  keyId: "<sha3-256 of pubkey>", // 公開鍵レジストリ参照ID
  sig: <Uint8Array>,             // FN-DSA-512署名（約690バイト）
  stateHash: <Uint8Array>,       // FinalStateHashV2
}
```

---

## 5. 実装フロー（Symbol TypeScript SDK例）

完全版はリポジトリの `examples/` を参照してください。以下は主要抜粋です。

### 5.1 一時Multisig作成

```typescript
const multisigTx = MultisigAccountModificationTransaction.create(
  Deadline.create(epochAdjustment),
  1,                              // minApproval: 2-of-2
  1,                              // minRemoval
  [contractor.address],           // 請負人を追加
  [],
  networkType
);
```

### 5.2 NFT Mosaic作成（Aggregate内）

```typescript
const mosaicDefinitionTx = MosaicDefinitionTransaction.create(
  Deadline.create(epochAdjustment),
  nonce,
  mosaicId,
  MosaicFlags.create(false, true, false), // supply=固定, transferable=true
  0,                              // divisibility=0（非分割）
  UInt64.fromUint(0),             // 無期限
  networkType
);
```

### 5.3 最終ステップ（Multisig解除 + 所有権移転 + PQC署名）

```typescript
// FinalStateDescriptorV2を構築
const descriptor: FinalStateDescriptorV2 = {
  networkGenerationHash: networkGenerationHash,
  mosaicIdHex: mosaicId.toHex(),
  merkleRootHex: computedMerkleRoot,
  ownerAddressRaw: owner.address.encoded(),
  finalizedHeight: finalizedHeight.toString(),
  finalizedGenerationHash: finalizedGenerationHash,
};

// FN-DSA-512署名を生成（アプリケーション層）
const stateHash = computeFinalStateHashV2(descriptor);
const pqcSignature = fnDsa512.sign(stateHash, pqcPrivateKey);

// PQC署名をCBORエンコードしてMetadataに記録
const pqcMetadataTx = AccountMetadataTransaction.create(
  Deadline.create(epochAdjustment),
  owner.address,
  KeyGenerator.generateUInt64Key('PQC_ANCHOR_V2'),
  pqcCborPayload.length,
  pqcCborPayload,                 // CBORエンコード済みPQC署名
  networkType
);

// 最終Aggregate（Complete型で即時確定）
const finalizeAggregateTx = AggregateTransaction.createComplete(
  Deadline.create(epochAdjustment),
  [
    multisigRemoveTx.toAggregate(owner.publicAccount),      // Multisig解除
    transferOwnershipTx.toAggregate(owner.publicAccount),   // 所有権移転
    pqcMetadataTx.toAggregate(owner.publicAccount),         // PQC CBOR Metadata
  ],
  networkType,
  [],
  UInt64.fromUint(4000000)
);

const signedFinalizeTx = owner.sign(finalizeAggregateTx, networkGenerationHash);
// announceAggregateComplete で即時確定
```

### 5.4 実装上の注意点（Gemini指摘・β1.1.0追記）

**① CBORライブラリは決定論的エンコードを使用すること（必須）**

CBORのキー順序が変わると `stateHash` が不一致になり、検証が必ず失敗します。
決定論的（Canonical）エンコードが保証されたライブラリを選定してください。

```typescript
// ✅ 推奨：cbor-x（決定論的エンコード対応）
import { encodeCanonical } from 'cbor-x';
const pqcCborPayload = encodeCanonical({
  v: 2,
  alg: "FN-DSA-512",
  keyId: keyId,
  sig: pqcSignature,
  stateHash: stateHash,
});

// ❌ 非推奨：キー順序が不定のライブラリ（stateHash不一致の原因）
// JSON.stringify() も順序が保証されないため使用禁止
```

**② Metadata更新権限の管理（必須）**

Step 4完了後、`PQC_ANCHOR_V2` Metadataを上書きされると真正性が破壊されます。
以下の運用ルールを必ず適用してください。

```
【推奨運用ルール】
- PQC_ANCHOR_V2 の更新権限はオーナーのみ
- 発行完了後は Metadata更新トランザクションを意図的にブロック
  （運用ポリシーとして明文化・監査ログに記録）
- より強固にしたい場合：Metadataをイミュータブルなアドレスに紐付け
  （Symbolのアカウント制限機能を活用）
```

**③ 公開鍵レジストリの再利用（コスト最適化）**

同一オーナーが2個目以降のNFTを発行する場合、`PQC_PUBKEY_V2` の再記録は不要です。
`keyId`（SHA3-256 of publicKey）で既存の公開鍵レジストリを参照するだけで済みます。

```
1枚目のNFT発行: PQC_ANCHOR_V2 + PQC_PUBKEY_V2 を記録（2エントリ）
2枚目以降:      PQC_ANCHOR_V2 のみ記録（keyIdで既存公開鍵を参照）
                → 手数料・容量コストを大幅削減
```

---

## 6. オフチェーン検証仕様（リファレンス）

```
1. Mosaic IDからMerkle Root・PQC Metadataを取得
        ↓
2. オフチェーンメタデータからMerkle Treeを再構築しRoot一致確認
        ↓
3. FinalStateDescriptorV2を構築 → computeFinalStateHashV2でハッシュ生成
        ↓
4. keyIdからPQC公開鍵レジストリを参照（Symbolメタデータ）
        ↓
5. FN-DSA-512公開鍵で署名検証
        ↓
→ すべて成功 = 量子耐性真正性が証明された状態
```

---

## 7. セキュリティ特性と制約

### 保証される特性

- 請負人承認はSymbolコンセンサス層で強制記録（Ed25519・改ざん不可）
- 最終真正性はFN-DSA-512（量子耐性）で固定
- Multisigは発行時のみ使用 → 永続依存なし
- PQC署名は1回のみ → 漏洩影響を最小化
- Merkle証明により部分データ検証も可能
- `finalizedGenerationHash` によるフォーク耐性（別チェーンへの署名使い回し防止）

### 制約・注意点

- PQC署名の検証は**アプリケーション層**（Symbolノードは自動検証しない）
- **Metadataの上書きリスク** → 発行完了後のMetadata更新権限を運用ポリシーで明示的に制限すること（5.4参照）
- **CBORエンコードの決定論的保証** → ライブラリ選定を誤ると `stateHash` 不一致が発生（5.4参照）
- 請負人がオフラインの場合のフォールバック設計は別途必要

---

## 8. 容量設計

### メタデータ容量試算

```
FN-DSA-512 署名:         690 バイト
CBORオーバーヘッド:       ~50 バイト
stateHash (SHA3-512):     64 バイト
keyId参照:                32 バイト
─────────────────────────────────────
署名メタデータ合計:      約836 バイト
```

> Symbolのメタデータ制限は1エントリ **1,024バイト**。  
> 署名（836バイト）は1エントリに収容可能です。  
> 公開鍵（897バイト）は**別エントリの公開鍵レジストリ**として管理します。

### 推奨分割構成

| メタデータキー | 内容 | サイズ |
|---|---|---|
| `PQC_ANCHOR_V2` | CBOR署名データ（署名+stateHash） | ~836バイト |
| `PQC_PUBKEY_V2` | 公開鍵レジストリ（別エントリ） | ~930バイト |

---

## 9. 実装ステータス（2026-02-26時点）

| 項目 | ステータス |
|---|---|
| 設計 | ✅ 完了（v1.1改善済み） |
| アーキテクチャ明確化（β1.1.0） | ✅ 完了 |
| 実装 | 🔲 準備中（β） |
| testnet検証 | 🔲 未実施 |
| 検証CLI | 🔲 未実施 |

---

## 10. ロードマップ

```
Phase 1（優先）
  🔲 testnetでMultisig → Aggregate → 解除フロー全通し
  🔲 容量検証（Metadataエントリ分割の実動作確認）

Phase 2
  🔲 Merkle + PQC検証CLI作成
  🔲 公開鍵レジストリの標準実装

Phase 3
  🔲 ダミーFN-DSA-512実装 → 本ライブラリ統合
  🔲 IPFS / ArweaveへのMerkle Proof自動パブリッシュ
```

---

## 参考資料

- [Symbol公式ドキュメント（Metadata / Aggregate / Multisig）](https://docs.symbol.dev)
- [NIST FIPS 206 (FN-DSA / Falcon)](https://csrc.nist.gov/pubs/fips/206/final)
- [NIST FIPS 204 (ML-DSA / Dilithium)](https://csrc.nist.gov/pubs/fips/204/final)
- [Spark Processor β1.1.0 → README.md](./README.md)

---

*License: CC0 1.0 (パブリックドメイン) | Designed by @mkti_tiger | Spark Processor β1.1.0*
