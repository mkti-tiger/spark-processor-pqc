> 📄 **Spark Processor全体仕様 →** [README.md](./README.md)

# Quantum-Resistant NFT Issuance Protocol (β)

本プロジェクト独自の量子耐性NFT発行プロトコルです。

Symbolブロックチェーンの **Multisig** および **Aggregate Transaction** を最大限活用し、請負人承認を確実にブロックチェーンへ記録すると同時に、最終状態の真正性をオーナーのPQC公開鍵によって保証します。

## コア設計（4ステップ）

### 1. 一時的 2-of-2 Multisig アカウント作成
- オーナーアドレス＋請負人アドレスで **2-of-2 Multisig** を生成
- 両者の署名がなければ操作できない状態を構築
- 請負人承認をコンセンサス層で強制記録
- ※ 本Multisigは発行プロセス中のみ使用する一時的アカウントです。

### 2. Multisig から Aggregate Transaction を発行
- 1回のAggregateで以下を原子的（Atomic）に実行：
  - Mosaic（NFT）作成（supply = 1, divisibility = 0）
  - Merkle Treeで圧縮したMetadata追加
  - Mosaic所有権をMultisigアカウントへ設定
- この段階では PQC署名は使用しません。
- 複数データはMerkle Treeにより圧縮され、後段で単一署名に集約されます。

### 3. 請負人による Cosignature（承認）
- 請負人がウォレットでCosignを実行
- 署名方式はSymbol標準の **Ed25519**
- Cosign完了によりAggregateが成立し、NFTが正式発行
- この段階でもPQC署名は使用しません。

### 4. Multisig解除＋所有権移転（Ed25519で実行）
- Multisig成立後、オーナーが **Ed25519標準署名** で新たなAggregateを発行します。
- 実行内容：
  - Multisig解除
  - Mosaic所有権をオーナー標準アドレスへ移転
  - 最終状態ハッシュに対する **FN-DSA-512** によるPQC署名 をMetadataとして追加
- ここで初めて PQC署名を1回のみ適用 します。

## 二層構造による量子耐性モデル

**コンセンサス層（ネットワーク保証）**  
- Multisig  
- Aggregate Transaction  
- Cosignature  
- Ed25519署名  
→ Symbolネットワークが検証

**量子耐性層（アプリケーション保証）**  
- FN-DSA-512によるPQC署名（1回のみ）  
- Merkle Tree圧縮  
- Metadata格納  
- 最終状態ハッシュへの署名固定  
→ オフチェーン検証により真正性を保証

## 本質的強み
- 複数データをMerkle Treeで圧縮  
- Aggregateにより原子的実行  
- PQC署名を最終状態に対して1回のみ適用  
- 署名容量と手数料を最小化（O(1)署名モデル）

本設計は、SymbolのネイティブAggregateおよびMultisigアーキテクチャとの高い親和性により実現されています。

## セキュリティ特性
- 請負人承認はコンセンサス層で強制記録
- 最終真正性はPQC公開鍵により保証
- Multisigは発行時のみ利用し恒久依存しない
- データはMerkle証明により検証可能

## 重要な技術補足
- PQC署名はコンセンサスレベルでは検証されません
- トランザクション発行および所有権移転はEd25519で実行されます
- PQCは追加的な量子耐性保証レイヤーとして機能します

## 現状（β段階）
- 2026年2月25日現在：設計完了・実装準備中
- 実装および本番テストは未実施
- 本設計は二層構造型量子耐性NFTプロトコルとして独自設計
- 詳細は随時更新予定です。

## 最終評価
- ✔ 技術的に正確
- ✔ 実装可能（未テスト）
- ✔ 誤解を生まない
- ✔ 専門家レビュー耐性あり
- ✔ コンセンサス層とPQC層を明確分離

