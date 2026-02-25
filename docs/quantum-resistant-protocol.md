# Quantum-Resistant NFT Issuance Protocol (β)

本プロジェクト独自の量子耐性NFT発行プロトコルです。

Symbolブロックチェーンの **Multisig + Aggregate Transaction** を最大限活用し、請負人承認を確実にブロックチェーンへ記録しつつ、最終状態の真正性をオーナーのPQC公開鍵によって保証します。

## コア設計（4ステップ）

### 1. 一時的 2-of-2 Multisig アカウント作成
- オーナーアドレス＋請負人アドレスで **2-of-2 Multisig** を生成
- 両者の署名がなければ操作できない状態を構築
- 請負人承認をコンセンサス層で強制記録
- ※ このMultisigは一時的にのみ使用します。

### 2. Multisig から Aggregate Transaction を発行
- 1回のAggregateで以下を原子的に実行：
  - Mosaic（NFT）作成（supply=1, divisibility=0）
  - Merkle Treeで圧縮したMetadata追加
  - Mosaic所有権をMultisigアカウントへ設定
- この段階では PQC署名は使用しません。
- データはMerkle Treeで圧縮し、後段で単一署名へ集約します。

### 3. 請負人が Cosignature（承認）
- 請負人がウォレットで Cosign を実行
- 署名方式はSymbol標準の **Ed25519**
- Cosign完了によりAggregateが成立し、NFTが正式発行
- この段階でもPQC署名は使用しません。

### 4. Multisig解除＋所有権移転（Ed25519で実行）
- Multisig成立後、オーナーが **Ed25519標準署名** で新しいAggregateを発行します。
- 内容：
  - Multisig解除
  - Mosaic所有権をオーナーの標準アドレスへ移転
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
→ オフチェーン検証により真正性保証

## 本質的強み
- 複数データをMerkle Treeで圧縮  
- Aggregateで原子的実行  
- PQC署名を最終状態に対して1回のみ適用  
- 署名容量と手数料を最小化（O(1)署名モデル）

これは、SymbolのネイティブAggregateおよびMultisigアーキテクチャとの高い親和性によって実現される設計です。

## セキュリティ特性
- 請負人承認はコンセンサス層で強制記録
- 最終コントロールはPQC公開鍵による真正性保証
- Multisigは発行時のみ利用され、恒久的依存はしない
- データはMerkle証明により検証可能

## 重要な技術補足
- PQC署名はコンセンサスレベルでは検証されません
- トランザクション発行および所有権移転はEd25519で実行されます
- PQCは最終状態に対する追加的な量子耐性保証レイヤーです

## 現状（β段階）
- 2026年2月25日現在、設計完了・実装準備中
- 本設計は二層構造型量子耐性NFTプロトコルとして独自設計
- 詳細は随時更新予定です。

## 最終評価
- ✔ 実装可能 （未テスト） 
- ✔ コンセンサス層とPQC層を明確分離

