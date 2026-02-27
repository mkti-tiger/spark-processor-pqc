## Quantum-Resistant NFT Issuance Protocol (β)

本プロジェクト独自の量子耐性NFT発行プロトコルです。

Symbolブロックチェーンの **Multisig** および **Aggregate Transaction** を最大限活用し、請負人承認を確実に記録すると同時に、最終状態の真正性をオーナーのPQC公開鍵によって保証します。

### コア設計（4ステップ）
1. 一時的 2-of-2 Multisig アカウント作成（請負人承認を強制記録）  
2. MultisigからAggregate Transactionを発行（Mosaic作成＋Merkle Tree圧縮＋所有権設定）  
3. 請負人がCosignature（Ed25519）で承認  
4. Ed25519でMultisig解除＋所有権移転後、最終状態ハッシュに対して **FN-DSA-512** によるPQC署名を1回のみ適用

### 二層構造モデル
- **コンセンサス層**（ネットワーク保証）：Multisig / Aggregate / Cosignature / Ed25519  
- **量子耐性層**（アプリケーション保証）：PQC署名（1回のみ） / Merkle Tree圧縮 / Metadata格納

### 本質的強み
複数データをMerkle Treeで圧縮し、Aggregateで原子的に集約した上で、PQC署名を最終状態に対して**1回のみ**適用することで、署名容量と手数料を最小化（O(1)署名モデル）します。  
これはSymbolのネイティブAggregateおよびMultisigアーキテクチャとの高い親和性によって実現される設計です。

**詳細仕様**  
[📄 NFT_PROTOCOL_beta1.1.0.md
 詳細](./NFT_PROTOCOL_beta1.1.0.md
)

※ PQC署名はアプリケーション層での真正性証明となります（コンセンサスレベルでは検証されません）。
