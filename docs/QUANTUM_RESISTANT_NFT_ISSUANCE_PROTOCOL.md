> 📄 **Spark Processor全体仕様 →** [README.md](./README.md)

# Quantum-Resistant NFT Issuance Protocol (β)

**Symbol Blockchain + FN-DSA-512 (Falcon) PQC Layer**  
**バージョン**: β v1.1 (2026-02-26)  
**ライセンス**: CC0 1.0 (パブリックドメイン)

## 1. 概要

本プロトコルは、Symbolブロックチェーンのネイティブ機能（Multisig + Aggregate Transaction）を最大限活用しつつ、最終状態の真正性を **FN-DSA-512 (NIST標準 PQC)** で保証する**二層構造型量子耐性NFT発行プロトコル**です。

- **コンセンサス層**：SymbolネットワークのEd25519 + Multisig + Aggregate で原子性・承認記録を保証  
- **量子耐性層**：最終状態ハッシュに対して**1回のみ**FN-DSA-512署名を適用（O(1)コスト）

これにより、署名サイズ・手数料を最小化しつつ、量子コンピュータ耐性を付与します。

## 2. 二層アーキテクチャ

| 層             | 保証内容                     | 使用技術                              | 検証場所     |
|----------------|------------------------------|---------------------------------------|--------------|
| コンセンサス層 | 原子性・請負人承認記録       | Ed25519, Multisig, Aggregate Tx       | Symbolノード |
| 量子耐性層     | 最終状態真正性（量子耐性）   | FN-DSA-512 + Merkle Tree + SHA3-512   | オフチェーン |

## 3. コアフロー（4ステップ）

1. **一時的 2-of-2 Multisig アカウント作成**  
   オーナー + 請負人アドレスで一時Multisigを作成（発行プロセス中のみ使用）

2. **Multisig下でAggregate発行**  
   - NFT Mosaic作成（supply=1, divisibility=0）  
   - Merkle Tree圧縮メタデータ付与  
   - 所有権をMultisigへ設定  
   （この時点ではPQC未使用）

3. **請負人によるCosignature**  
   Ed25519でCosign → Aggregate成立 → NFT正式発行

4. **Multisig解除 + 所有権移転 + PQC署名**  
   オーナーが単独で **AggregateComplete** を発行し、以下の処理を原子的に実行：  
   - Multisig解除  
   - Mosaic所有権をオーナー通常アドレスへ移転  
   - 最終状態ハッシュに対するFN-DSA-512署名をMosaic Metadataに記録

## 4. データモデル（v1.1改善版）

### 最終状態ハッシュ（FinalStateDescriptorV2）

```typescript
interface FinalStateDescriptorV2 {
  networkGenerationHash: string;        // チェーン固有
  mosaicIdHex: string;
  merkleRootHex: string;
  ownerAddressRaw: string;              // 最終オーナー
  finalizedHeight: string;
  finalizedGenerationHash: string;      // 最終確定時のGH（フォーク耐性強化）
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

PQCメタデータ形式（CBOR固定構造推奨）

JSON/hex → CBOR に移行（パース高速・サイズ最小・将来拡張性向上）。

5. 実装フロー（Symbol TypeScript SDK例）
（完全版はリポジトリの examples/ を参照してください。以下は主要抜粋）

5.4 最終ステップ（Multisig解除 + 所有権移転 + PQC署名）

// 最終Aggregate（Complete型で即時確定）

const finalizeAggregateTx = AggregateTransaction.createComplete(
  Deadline.create(epochAdjustment),
  [
    multisigRemoveTx.toAggregate(owner.publicAccount),     // Multisig解除
    transferOwnershipTx.toAggregate(owner.publicAccount), // 所有権移転
    pqcMetadataTx.toAggregate(owner.publicAccount),       // PQC CBOR Metadata
  ],
  networkType,
  [],
  UInt64.fromUint(4000000)
);

const signedFinalizeTx = owner.sign(finalizeAggregateTx, networkGenerationHash);
// announceAggregateComplete で即時確定


6. オフチェーン検証仕様（リファレンス）


Mosaic IDからMerkle Root・PQC Metadataを取得

オフチェーンメタデータからMerkle Treeを再構築しRoot一致確認

FinalStateDescriptorV2を構築 → computeFinalStateHashV2

FN-DSA-512公開鍵で署名検証


→ すべて成功すれば量子耐性真正性が証明されます。

7. セキュリティ特性


請負人承認はSymbolコンセンサス層で強制記録

最終真正性はFN-DSA-512（量子耐性）で固定

Multisigは発行時のみ使用 → 永続依存なし

PQC署名は1回のみ → 漏洩影響を最小化

Merkle証明により部分データ検証も可能


8. 実装ステータス（2026-02-26時点）

設計：完了（v1.1改善済み）
実装：準備中（β）
テスト：未実施（testnet検証を優先）

9. 次に推奨するステップ
Phase 1（　日） testnetでMultisig → Aggregate → 解除フロー全通し
Phase 2（　日） Merkle + PQC検証CLI作成
Phase 3   　  ダミーFN-DSA-512実装 → 本ライブラリ統合

参考資料

Symbol公式ドキュメント（Metadata / Aggregate / Multisig）
NIST FIPS 206 (FN-DSA)
プロジェクト全体：Spark Processor β1.0.0




