# Spark Processor PQC (β)

Symbolブロックチェーンを活用した量子耐性NFT発行のための実験的プロセッサ。

## Quantum-Resistant NFT Issuance Protocol (β)

本プロジェクトは、**Symbol (XYM)** のネイティブ機能である **Multisig + Aggregate Transaction** を最大限に活用した、**独自の量子耐性NFT発行プロトコル**です。

請負人の承認を確実にブロックチェーン上に記録しつつ、最終的なコントロールを常にオーナーのPQC単一鍵 (FN-DSA-512) のみで保持することを目的としています。

### 詳細設計
[Quantum-Resistant NFT Issuance Protocol の詳細仕様](./docs/quantum-resistant-protocol.md)

### 主な特徴
- PQC署名を**1回のみ**に集約（容量・手数料を劇的に削減）
- 一時的2-of-2 Multisigによる請負人承認フロー
- Merkle Tree圧縮 + Aggregate Transactionの最適活用
- Symbolネイティブ機能のフル活用（他チェーンでは困難）
- β段階の柔軟設計（将来的なPQCアルゴリズム移行に容易に対応）

---

このプロトコルは他に類似例のないオリジナル設計です。
