# Quantum-Resistant NFT Issuance Protocol (β)

本プロジェクト独自の量子耐性NFT発行プロトコルです。  
Symbolブロックチェーンの**Multisig + Aggregate Transaction**をフル活用し、請負人承認を確実に記録しつつ、最終コントロールを常にオーナーのPQC単一鍵のみに保持します。

## コア設計（4ステップ）

### 1. 一時的2-of-2 Multisigアカウント作成
- オーナーアドレス＋請負人アドレスの **2-of-2 Multisig** を生成  
- これにより請負人の承認をブロックチェーン上で強制的に記録

### 2. MultisigからAggregate Transactionを発行
- 1回のAggregateで以下を原子的に実行  
  - Mosaic（NFT）作成（supply=1, divisibility=0）  
  - Merkle Treeで圧縮したMetadata追加  
  - Mosaicの所有権をMultisigアカウントに設定  
- PQC署名はまだ使用しない（容量削減のため）

### 3. 請負人がCosignature（承認）
- 請負人がウォレットで **Cosign** を実行  
- これでAggregateが成立し、Mosaicが正式発行される

### 4. オーナーのPQC単一鍵でMultisig解除＋所有権移転
- オーナーの **FN-DSA-512鍵** のみで新しいAggregateを実行  
  - Multisig解除  
  - Mosaic所有権をオーナーのPQC単一アドレスへ移転  
  - PQC署名付きMetadataを最終追加（**ここでPQC署名は1回のみ**）

## Symbolの強みを活かした効果
- PQC署名容量・手数料を劇的に削減（1回のみ）  
- 請負人承認をネイティブMultisigで強制記録  
- 最終コントロールは常にオーナーのPQC単一鍵のみ  
- Merkle Tree証明による各データの真正性担保  
- 他チェーンでは実現困難な**ネイティブAggregate + Multisig**の組み合わせ

## 現状（β段階）
- 2026年2月25日現在、設計完了・実装準備中  
- 本設計は他に類似例のないオリジナルプロトコルです

詳細は随時更新いたします。
