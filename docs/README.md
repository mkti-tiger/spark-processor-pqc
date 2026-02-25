## Quantum-Resistant NFT Issuance Protocol (β)

本プロジェクト独自の量子耐性NFT発行プロトコル（Symbol Multisig + Aggregateフル活用）です。

### Symbolの強みをフル活用した設計（4ステップ）

**ステップ①**　一時的な2-of-2 Multisigアカウントを作成  
**ステップ②**　MultisigからAggregate Transactionを発行（Mosaic作成＋Merkle圧縮Metadata＋所有権設定）  
**ステップ③**　請負人がCosignatureで承認  
**ステップ④**　Multisig解除＋所有権移転（Ed25519標準署名で実行）

### 二層構造による量子耐性実現
- **コンセンサス層**（Symbolネットワーク保証）  
  Multisig、一時的承認、Aggregate、Cosignature、Ed25519署名

- **量子耐性層**（アプリケーション保証）  
  FN-DSA-512によるPQC署名（1回のみ）、Merkle Tree圧縮、Metadata格納

**最終的な真正性保証**はオーナーのPQC単一鍵により付与されます（PQC署名はアプリケーション層で検証されます）。  
Multisig解除および所有権移転はSymbol標準署名（Ed25519）で実行され、その最終状態に対してPQC署名が1回のみ適用されます。

### 本質的強み
複数データをマークルツリーで圧縮し、  
マークルルート集約後にAggregate Transactionを発行。  
その全体に対しPQC署名を1回のみ適用することで、  
署名容量と手数料を最小化。  

**これがSymbolブロックチェーンの本質的強みである。**

※ PQC署名はコンセンサスレベルでのネットワーク検証ではなく、アプリケーション層での真正性証明となります。将来的なSymbolプロトコル拡張によりネイティブ検証も可能になる予定です。
