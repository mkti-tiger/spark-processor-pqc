# Spark Processor (PQC量子耐性アンカー構造) β1.1.0

**「既存のチェーンを一切壊さず、量子コンピューティングの脅威から資産を守る」**

Symbolブロックチェーンの柔軟なプラグイン機能を活用し、  
**既存のL1（BTC、ETH、JPYC等）を一切改変することなく**、量子耐性（PQC）の保護層を提供する非侵入型アーキテクチャです。

> 📄 **アーキテクチャ詳細仕様は末尾の「技術仕様（β1.1.0）」セクションを参照してください。**

---

## 📋 目次

- [プロジェクトの本質](#-プロジェクトの本質)
- [3つの核心的優位性](#-3つの核心的優位性)
- [アーキテクチャ概要（図1）](#-アーキテクチャ概要図1)
- [実装仕様](#実装仕様)
- [Dockerコンテナ内秘密鍵のブラックボックス化](#4-dockerコンテナ内秘密鍵のブラックボックス化)
- [ホットウォレットとの比較とセキュリティ方針](#5-ホットウォレットとの比較とセキュリティ方針)
- [集約閾値の柔軟調整](#6-集約閾値の柔軟調整用途別設定)
- [検証・長期耐久性強化](#7-検証長期耐久性強化)
- [将来の拡張性とエンタープライズ対応](#8-将来の拡張性とエンタープライズ対応)
- [Symbolのオンチェーンデータ記録](#10-symbolのオンチェーンデータ記録のための標準メカニズム)
- [Quantum-Resistant NFT Issuance Protocol (β1.1.0)](#quantum-resistant-nft-issuance-protocol-β110)
- [クイックスタート](#クイックスタート)
- [動作要件](#動作要件)
- [ロードマップ](#ロードマップ)
- [License & Status](#license--status)
- [FAQ](#faq)
- [参考資料・詳細技術分析](#参考資料詳細技術分析)

---

## 🎯 プロジェクトの本質

Spark Processor（SYMBOL請負人）の役割は、**「データ本体を直接守る」**ものではありません。

**データは銀行や企業、クライアントノードに従来の暗号方式で保管**されており、  
量子コンピュータの脅威から守る対象は**データ本体ではなく、その真正性**です。

SYMBOL請負人は、
- クライアントから送られてきた生データ（またはハッシュ）を対象に
- 量子耐性署名（PQC）で検証・証明を行い、
- Symbolブロックチェーン上に**「このデータは量子コンピュータで破られていない」**というアンカーを記録します。

つまり、**「盗まれないようにする」のではなく、「盗まれても分かるようにする」**仕組みを提供します。

データ本体は元の場所（銀行・クライアントノード）に残したまま、  
**真正性の証明だけを量子耐性で強化**する——それがSpark Processorの本質です。


## 📄 仕様ドキュメント

| ファイル | 内容 |
|----------|------|
| [AUTHENTICITY\_PRIMITIVE\_v1.2.0.md](./spec/AUTHENTICITY_PRIMITIVE_v1.2.0.md) | 不変核の正式定義（4規則・Level 0） |
| [MINIMAL\_VERIFIER\_v1.2.0.md](./spec/MINIMAL_VERIFIER_v1.2.0.md) | 言語非依存の疑似コード実装（182行） |

> これらのファイルは Spark Processor の真正性保証の核となる不変ドキュメントです。  
> 内容の変更は SpecGenesisHash の再計算を必要とし、全ての検証に影響します。


金融庁の「耐量子計算機暗号への対応に関する検討会報告書」で懸念されている  
**「真正性の確保（PQC署名の付与）」と「移行期間のハイブリッド運用」**に対する現実的な解が、  
この「Spark Processor」によって具現化されています。

---

## 🚀 3つの核心的優位性

1. **L1 Non-Intrusive（基盤完全非改変）**  
   Symbol Node (Catapult) のコアを1行も変更しません。公式アップデートを即時適用可能。

2. **Docker-Centric PQC（完全隔離ブラックボックス）**  
   すべてのPQC処理（署名・検証・Merkle計算）をDocker Plugin内に閉じ込め、アルゴリズム更新が容易。

3. **Multi-Chain Anchoring（多チェーン対応）**  
   BTC・ETH・JPYCなど複数チェーンのトランザクションを量子耐性でSymbol上に安全にアンカリング。

---

## 🏗 アーキテクチャ概要（図1）

**図1　全体フロー（概念）β1.1.0**

```
【クライアント（ユーザー / アプリ）】← ここから全て始まる！
（生データ生成・TX作成）
        ↓ 生データを転送（E2E暗号）
        |
【クライアントノード】← 専用倉庫 + 受付窓口
（生データ受信・保持・復元依頼受付のみ）
        ↓ ハッシュのみ送信（E2E暗号）
        |
【Symbolプラグイン】← 主役「請負人」
┌─────────────────────────────────────────┐
│  1. ハッシュ集約（E2E経由で蓄積）          │
│     （閾値：1000件 or 1時間）              │
│  2. Merkle Tree構築 → Root導出            │
│     （データ量・手数料を大幅削減）          │
│  3. Aggregate TX作成                      │
│     （Plain MessageにJSONデータ埋め込み）  │
│  4. PQC署名（アプリケーション層・1回）     │
│     ← これがSymbolの強み                  │
│  ※復元時：Merkle Path抽出 + 検証          │
└─────────────────────────────────────────┘
        ↓ 証跡のみ記録
        |
【Symbolチェーン】
（Merkle Root + PQC署名のみ記録）
        ↑ 復元時：Merkle Path + 検証結果返送
        |
【クライアントノード】← 一時受け取り
        ↓ 生データ + 証明を返送（E2E暗号）
        |
【クライアント】← データ完全帰還！
```

> ⚠️ **PQC署名はアプリケーション層で検証されます。**  
> Symbolノードは現行仕様ではEd25519のみをコンセンサスレベルで検証します。  
> 公証・監査・真正性証明用途では十分に有効です。  
> 詳細は末尾「技術仕様（β1.1.0）」セクションを参照してください。

*Designed by @mkti_tiger | Spark Processor (β1.1.0) on X*

---

## 実装仕様

- **PQCアルゴリズム**：NIST標準準拠（FN-DSA-512 / ML-DSA-65）
- **通信方式**：HTTPS + E2E暗号
- **データ圧縮**：Merkle Rootアンカリングによりデータ量を大幅削減

> 詳細な処理フロー・容量設計・検証モデルは末尾「技術仕様（β1.1.0）」セクションを参照してください。

---

## 4. Dockerコンテナ内秘密鍵のブラックボックス化

Spark Processor（β）の最大の特徴は、**クライアントに一切秘密鍵を持たせず、Dockerコンテナ内で完全に閉じた状態でPQC秘密鍵を生成・管理・使用**することです。

### ブラックボックス化の具体的な仕組み

| 対策項目 | 実施方法 | 効果（ノード管理者視点） |
|---|---|---|
| 秘密鍵の生成 | コンテナ起動時にin-memoryで自動生成 | ホスト側にファイルとして一切保存されない |
| 秘密鍵の保管 | Docker Secrets（tmpfsマウント） | `docker exec` でも通常閲覧不可 |
| コンテナ権限 | `--user 1000:1000 --read-only --cap-drop=ALL` | root権限でも書き込み・閲覧が制限される |
| ログ・出力制御 | 鍵関連のログを一切抑制 | `docker logs` や `docker inspect` で情報が漏れない |
| 永続化禁止 | ボリュームマウントを一切使用しない | ホストディスクに鍵が残らない |

### 秘密鍵の定期更新（Key Rotationポリシー）

- **更新頻度**：毎月1回（または重大なセキュリティイベント発生時）
- **更新方法**：Dockerコンテナ内で完全自動ローテーション（ダウンタイムなし）
- **Key Historyの記録**：更新した鍵の有効期間と公開鍵情報を**Symbolのメタデータ**として記録
- **クライアントへの影響**：一切なし（完全に透明）

### HSM（物理分離）への移行パス

- ドライバインターフェースを標準化
- 設定1行変更でAWS CloudHSM / Azure Dedicated HSM / YubiHSMに切り替え可能
- 大口銀行・金融機関の厳格なセキュリティ要件にも対応

### 注意点

ノード管理者がroot権限を持つ場合、理論上は高度な手法で閲覧可能ですが、運用ポリシーと監査ログで対応可能です。  
より強固な分離を希望される場合は、**Symbolノードとは別サーバー**にプラグイン専用Dockerを配置することを推奨します。

---

## 5. ホットウォレットとの比較とセキュリティ方針

本設計はホットウォレットと以下の点で本質的に異なります：

- 鍵は**署名専用**であり、送金機能を持ちません
- 漏洩時の影響は「過去の証明の信頼性低下」のみ。**ユーザーの資産が直接流出することはありません**
- 任意で**HSM連携**や**別サーバー完全分離**（実質コールド運用相当）が可能です

本プロジェクトはCC0で公開しており、透明性を最優先としております。

---

## 6. 集約閾値の柔軟調整（用途別設定）

### 用途別推奨設定例

| 用途 | 推奨設定例 | 最大遅延 | 想定シーン |
|---|---|---|---|
| 銀行・大口利用 | 1000件 or 1時間 | 最大1時間 | 銀行間送金、大量決済バッチ |
| 一般決済 | 100件 or 10分 | 最大10分 | 店舗決済、日常送金 |
| 即時決済（小額） | 10件 or 60秒 | 最大1分 | リアルタイム決済、マイクロペイメント |
| 証明重視（遅延許容） | 1000件 or 1日 | 最大1日 | 証拠保全、監査用途 |

### クライアント主導即時処理オプション（β1.1.0予定）

- クライアントはAPI経由で**「IMMEDIATE=true」フラグ**を付けて生データを送信可能
- **手数料プレミアム**（例：通常の2〜5倍）を課金することで運用者のインセンティブを確保

**調整方法（Docker起動時）**

```bash
docker run -d \
  -e PQC_AGGREGATION_THRESHOLD=100 \
  -e PQC_AGGREGATION_TIMEOUT=600 \
  -e PQC_IMMEDIATE_MODE=true \
  -e PQC_IMMEDIATE_FEE_MULTIPLIER=3 \
  mkti-tiger/spark-processor-pqc
```

---

## 7. 検証・長期耐久性強化

### Merkle Treeの証明データ保存先

**推奨保管先（β1.1.0以降実装予定）**
- **IPFS**（分散型ファイルシステム）
- **Arweave**（永続ストレージ）
- **Filecoin**（経済的インセンティブ付き分散ストレージ）

### 耐量子性の「期限」の明確化

- **チェックポイント・ファイナリティ**：定期的に「PQCアンカーチェックポイント」をSymbol上に記録
- **期限明示**：各アンカーに「有効期限（量子耐性保証期間）」をメタデータとして付与
- **移行メカニズム**：Ed25519破壊が現実味を帯びた時点で、次世代PQCアルゴリズムへ自動移行するスイッチを用意

---

## 8. 将来の拡張性とエンタープライズ対応

- **Key History記録**：鍵更新時に有効期間と公開鍵をSymbolメタデータとして永続記録
- **HSM抽象化ドライバ**：Docker Secrets → AWS CloudHSM / Azure Dedicated HSM / YubiHSMへ1行設定で移行可能
- **Merkle Proof自動パブリッシュ**：即時処理時にIPFSへ自動アップロードし、CIDをトランザクションに記録

---

## 10. Symbolのオンチェーンデータ記録のための標準メカニズム

Symbol Blockchainは、コアプロトコルを一切変更せずにデータ永続化を実現するさまざまな標準トランザクションタイプをサポートしています。

**Transfer Transaction内のPlain Messages**  
最大1,023バイトの非暗号化文字列ペイロード。JSONオブジェクトなどの構造化データを直接オンチェーンに記録可能です。

```json
{
  "v": 1,
  "purpose": "pqc-signature-anchor",
  "createdAt": "2026-02-18T14:47:46",
  "data": "Symbol PQC Apostille"
}
```

**Aggregate Transactionsによるバッチ処理**  
最大100個の内包トランザクションを単一の原子単位に集約。「すべて成功またはすべて失敗」の原子性を保証します。

**PQC署名の統合フロー**

```
1. E2E暗号化リクエスト経由でハッシュを蓄積
2. Merkle Treeを構築し、Root導出
3. Plain MessageにJSONデータを埋め込み、Aggregate全体を構築
4. Aggregate TXハッシュに対してPQC署名を1回生成（アプリケーション層）
5. Ed25519でSymbol Nodeへアナウンス、結果をE2E暗号で返信
```

**実動作例：Symbol初の量子耐性PQC署名アンカー記録**  
KICN-FT（@kicnft.symbol）氏により、2026年2月18日にSymbolブロックチェーン史上初の量子耐性PQC（ML-DSA-65）署名アンカーが記録されました。

```
Hash:         05757EAE5C2CFA6AE933E81DCFA176
Status:       Confirmed
Block Height: 5178120
Timestamp:    2026-02-18 23:48:31
```

詳細：https://x.com/kicnft/status/2024143209098514681

---

## Quantum-Resistant NFT Issuance Protocol (β1.1.0)

本プロジェクト独自の量子耐性NFT発行プロトコルの詳細設計はこちらです。

[📄 詳細仕様 → NFT_PROTOCOL_beta1.1.0.md](./NFT_PROTOCOL_beta1.1.0.md)

---

## クイックスタート

**1. Dockerイメージの取得**
```bash
docker pull ghcr.io/mkti-tiger/spark-processor-pqc:latest
```

**2. コンテナ起動例（銀行用途）**
```bash
docker run -d \
  --name spark-pqc \
  -e PQC_AGGREGATION_THRESHOLD=1000 \
  -e PQC_AGGREGATION_TIMEOUT=3600 \
  -e PQC_IMMEDIATE_MODE=false \
  ghcr.io/mkti-tiger/spark-processor-pqc:latest
```

**3. ログ確認**
```bash
docker logs -f spark-pqc
```

---

## 動作要件

| 項目 | 必要条件 |
|---|---|
| Docker | 最新安定版 |
| OS | Linux / Windows / macOS |
| Symbol Node | 稼働中のノード |
| インターネット接続 | 常時接続（Symbolノードと通信） |

---

## ロードマップ

```
β1.0.0（完了）
  ✅ 概念検証・仕様公開・Dockerブラックボックス設計
  ✅ Symbol初のPQCアンカー記録（2026-02-18）

β1.1.0（進行中）
  ✅ アーキテクチャ正確化・技術仕様をREADMEに統合
  ✅ PQC署名層の正確な定義（アプリケーション層）
  ✅ 全図の修正（VPN→E2E・処理順序正規化）
  🔲 検証CLI作成（最優先）
  🔲 testnetでの全フロー検証
  🔲 HSM連携・複数アルゴリズム自動切替
  🔲 IPFS / Arweave Merkle Proof自動パブリッシュ

β2.0.0（予定）
  🔲 本番運用向け監査ログ・メトリクス機能追加

v1.0.0（予定）
  🔲 正式リリース（Symbolメインネット推奨運用ガイド提供）
  🔲 セキュリティ外部監査
```

---

## License & Status

- **License**：CC0（パブリックドメイン） - 自由に検証・実装・改変可能です
- **Status**：β1.1.0（アーキテクチャ明確化・仕様公開中）
- **Feedback**：X (@mkti_tiger) にて随時受付中

---

## FAQ

**Q. ノード管理者でも秘密鍵が見えてしまいますか？**  
A. 適切なブラックボックス化を実施すれば日常操作では見えません。root権限でも閲覧を困難にしています。

**Q. 決済で遅延が心配です**  
A. Docker環境変数で1分〜1時間まで自由に調整可能です。用途に合わせて最適化してください。

**Q. 検証に秘密鍵は必要ですか？**  
A. いいえ。公開鍵のみで誰でも即時検証可能です。

**Q. ホットウォレットと同じリスクですか？**  
A. いいえ。署名専用鍵のため資産流出リスクはありません。

**Q. PQC署名はSymbolネットワークが自動検証しますか？**  
A. 現行仕様ではアプリケーション層での検証です。公証・監査・真正性証明用途では十分に有効です。詳細は末尾「技術仕様（β1.1.0）」セクションを参照してください。

---

## 参考資料・詳細技術分析

### β1.0.0 実証済みトランザクション記録

```
Transaction Hash: 88E8FB3CC460C49A6681064D6D4519B88B21E32E6BF00E9CC5C5DE8341AD0932
Timestamp:        2026-02-15 19:55:15
```

### Xスレッド（開発経緯）

- 量子耐性化　構造設計が出来ました！  
  https://x.com/mkti_tiger/status/2022458152550306306
- PQCアンカー構造：Symbolプラグインモジュール 仕様書（β1.0.0）  
  https://x.com/mkti_tiger/status/2022874867717279807
- SYMBOL 量子耐性化後のシナリオA  
  https://x.com/mkti_tiger/status/2023697039708877033
- VPN→E2E 変更後のアーキテクチャ図（BTC-ETH-JPYC-OQC請負人仕様を追加）  
  https://x.com/mkti_tiger/status/2025366526845747687

**DEMO Site**：http://sun.s66.xrea.com/Symbol-OS/pqc/

---

### 図2　ETH Node 内部処理図（β1.1.0修正版）

```
【ETHクライアント（ユーザー / DApp）】
（ETH TX生成・署名）
        ↓ ETH TXハッシュを転送（E2E暗号）
        |
【クライアントノード】← ETH専用受付窓口
（TXハッシュ受信・保持のみ）
        ↓ TXハッシュのみ送信（E2E暗号）
        |
【Symbolプラグイン（Docker Plugin）】← 請負人
┌─────────────────────────────────────────┐
│  1. ETH TXハッシュ集約（閾値まで蓄積）    │
│  2. Merkle Tree構築 → Root導出           │
│  3. Aggregate TX作成                     │
│     （Plain MessageにJSON埋め込み）      │
│     ┌──────────────────────────────┐    │
│     │ {                            │    │
│     │   "chain": "ETH",            │    │
│     │   "merkleRoot": "...",       │    │
│     │   "purpose": "pqc-anchor"   │    │
│     │ }                            │    │
│     └──────────────────────────────┘    │
│  4. PQC署名（アプリケーション層・1回）   │
│     アルゴリズム: FN-DSA-512 / ML-DSA-65│
└─────────────────────────────────────────┘
        ↓ 証跡のみ記録（E2E暗号）
        |
【Symbolチェーン】
（ETH Merkle Root + PQC署名を不変記録）
        ↑ 復元時：Merkle Path + 検証結果返送
        |
【クライアントノード】← 一時受け取り
        ↓ TXハッシュ + 量子耐性証明を返送（E2E暗号）
        |
【ETHクライアント】← 証明完了！
```

> ETHのコンセンサス（Proof of Stake）はそのまま維持。  
> Symbolプラグインが**ETH TX真正性の量子耐性証明**のみを担当します。

---

### 図3　BTC Node 内部処理図（β1.1.0修正版）

```
【BTCクライアント（ユーザー / ウォレット）】
（BTC TX生成・UTXO管理）
        ↓ BTC TXハッシュを転送（E2E暗号）
        |
【クライアントノード】← BTC専用受付窓口
（TXハッシュ受信・保持のみ）
        ↓ TXハッシュのみ送信（E2E暗号）
        |
【Symbolプラグイン（Docker Plugin）】← 請負人
┌─────────────────────────────────────────┐
│  1. BTC TXハッシュ集約（閾値まで蓄積）    │
│  2. Merkle Tree構築 → Root導出           │
│  3. Aggregate TX作成                     │
│     （Plain MessageにJSON埋め込み）      │
│     ┌──────────────────────────────┐    │
│     │ {                            │    │
│     │   "chain": "BTC",            │    │
│     │   "merkleRoot": "...",       │    │
│     │   "purpose": "pqc-anchor"   │    │
│     │ }                            │    │
│     └──────────────────────────────┘    │
│  4. PQC署名（アプリケーション層・1回）   │
│     アルゴリズム: FN-DSA-512 / ML-DSA-65│
└─────────────────────────────────────────┘
        ↓ 証跡のみ記録（E2E暗号）
        |
【Symbolチェーン】
（BTC Merkle Root + PQC署名を不変記録）
        ↑ 復元時：Merkle Path + 検証結果返送
        |
【クライアントノード】← 一時受け取り
        ↓ TXハッシュ + 量子耐性証明を返送（E2E暗号）
        |
【BTCクライアント】← 証明完了！
```

> BTCのPoWコンセンサスはそのまま維持。  
> **BTCはSecp256k1（ECDSA）を使用しており、量子コンピュータによる脅威が最も懸念される署名方式**のひとつです。  
> Symbolプラグインが**BTC TX真正性の量子耐性証明**を外部から担当します。

---

### 図4　JPYC 内部処理図（β1.1.0修正版）

```
【JPYCクライアント（ユーザー / 決済アプリ）】
（JPYC送金TX生成・署名）
        ↓ JPYC TXハッシュを転送（E2E暗号）
        |
【クライアントノード】← JPYC専用受付窓口
（TXハッシュ受信・保持のみ）
        ↓ TXハッシュのみ送信（E2E暗号）
        |
【Symbolプラグイン（Docker Plugin）】← 請負人
┌─────────────────────────────────────────┐
│  1. JPYC TXハッシュ集約（閾値まで蓄積）  │
│  2. Merkle Tree構築 → Root導出           │
│  3. Aggregate TX作成                     │
│     （Plain MessageにJSON埋め込み）      │
│     ┌──────────────────────────────┐    │
│     │ {                            │    │
│     │   "chain": "JPYC",           │    │
│     │   "merkleRoot": "...",       │    │
│     │   "purpose": "pqc-anchor"   │    │
│     │ }                            │    │
│     └──────────────────────────────┘    │
│  4. PQC署名（アプリケーション層・1回）   │
│     アルゴリズム: FN-DSA-512 / ML-DSA-65│
└─────────────────────────────────────────┘
        ↓ 証跡のみ記録（E2E暗号）
        |
【Symbolチェーン】
（JPYC Merkle Root + PQC署名を不変記録）
        ↑ 復元時：Merkle Path + 検証結果返送
        |
【クライアントノード】← 一時受け取り
        ↓ TXハッシュ + 量子耐性証明を返送（E2E暗号）
        |
【JPYCクライアント】← 証明完了！
```

> JPYCはEthereum上のERC-20トークン（ECDSA署名）。  
> 金融庁の報告書が指摘する**「決済インフラの真正性確保」**に直接対応します。  
> Symbolプラグインが**JPYC TX真正性の量子耐性証明**を外部から担当します。

---

### 詳細技術分析

本プロジェクトは、Symbolブロックチェーンをポスト量子暗号（PQC）対応に強化するための包括的な提案です。Dockerベースのプラグインアーキテクチャを通じて量子耐性署名を導入し、量子時代における真正性の保存を実現することを目的としています。

**ノード運用者のインセンティブ機構**  
クライアントがプラグインに接続し、Symbolトランザクションと手数料を生成することで運用者に報酬を与えます。経済エコシステムのモデル：`Client → Plugin → Symbol TX → Fee → Plugin Operator`

**実装要件**  
REST APIサーバ（Node.js / Go / Rust）、ハッシュ関数（SHA-256/SHA-3）、PQCライブラリ、Symbol SDK、キューシステム（Redis等）。Symbolコアの修正は不要です。

**開発スケジュール目安**  
経験豊富な開発者1名：プロトタイプ2〜6週間、ベータ版2〜3ヶ月。Layer 1フォークより大幅に簡単です。

**PQCライブラリの利用可能状況**

- Node.js / WebAssembly：pqc.js（Dilithium・Falcon・SPHINCS+対応）
- Rust：pqc_dilithium・falcon_rust（NIST仕様準拠）
- Go：liboqs（Open Quantum Safe プロジェクト）
- 全ライブラリはNIST最終化規格（ML-DSA・FN-DSA・SLH-DSA）に準拠

**設計の本質**  
Symbolの直接修正を避け、新たなサービスレイヤーを構築します。BitcoinのLightning Network、EthereumのInfura、The Graphと同じ「外部レイヤー戦略」です。

**考慮事項**

- パフォーマンス：PQC署名の大サイズはMerkle圧縮 + Aggregateで緩和
- セキュリティ：外部監査・testnetデプロイを経て本番運用へ
- 規格準拠：NIST PQC規格への準拠を継続維持

---

### 元図（β1.0.0）記録用画像

> 以下はβ1.0.0時点の記録画像です。修正済みテキスト図（β1.1.0）は上記を参照してください。

<img width="1654" height="2339" alt="図1 β1.0.0" src="https://github.com/user-attachments/assets/950ab7c5-bd6f-476d-a7d3-c1aac31e5813" />
<img width="1654" height="2339" alt="image" src="https://github.com/user-attachments/assets/6357da43-8c4e-4400-b069-57237154da79" />
<img width="1654" height="2339" alt="image" src="https://github.com/user-attachments/assets/f8809986-bacf-40d8-9806-6c68e5332182" />
<img width="1654" height="2339" alt="image" src="https://github.com/user-attachments/assets/4c9d2eab-e0c4-4efc-8c3d-ad732367486c" />
<img width="782" height="768" alt="image" src="https://github.com/user-attachments/assets/08d79561-2a2c-4a00-b8df-a11b67b84580" />
<img width="777" height="886" alt="image" src="https://github.com/user-attachments/assets/5a17658b-8093-4a33-9be8-9c37d613a09d" />
<img width="787" height="927" alt="image" src="https://github.com/user-attachments/assets/3c08cc31-62aa-4766-b37a-3fb0e58d906f" />
<img width="808" height="1158" alt="ETH Node内部処理図 β1.0.0" src="https://github.com/user-attachments/assets/bad6e495-d298-47d2-9a52-b1fc4bb6b6da" />
<img width="780" height="967" alt="BTC β1.0.0" src="https://github.com/user-attachments/assets/5d5e6b75-81af-482a-a759-e4e850ffee23" />
<img width="777" height="1174" alt="image" src="https://github.com/user-attachments/assets/c8ccb023-ba2d-4d40-8a9d-02dd0a7270ed" />
<img width="792" height="1038" alt="JPYC β1.0.0" src="https://github.com/user-attachments/assets/06b2d83b-1568-4bd2-ac26-5b6a18804b39" />
<img width="726" height="540" alt="image" src="https://github.com/user-attachments/assets/d2f24214-2760-45b1-b161-d7f72edfac36" />

---

## 技術仕様（β1.1.0）

> 本セクションはβ1.0.0で不正確だった技術表現を明確化した正式仕様です。

### β1.0.0からの主要変更点

| 項目 | β1.0.0の表現（不正確） | β1.1.0の正確な表現 |
|---|---|---|
| PQC署名の適用層 | 「Aggregate全体に1回PQC署名」（曖昧） | **アプリケーション層**でのオフチェーンPQC署名 |
| 検証主体 | ネットワーク全体が検証するかのような表現 | **検証ツール・監査アプリ側**で独立検証 |
| コンセンサス層 | PQC対応済みかのような表現 | **Ed25519固定**（現行Symbolプロトコル仕様） |
| 通信方式 | VPN | **E2E暗号**（正式反映） |
| 処理順序 | PQC署名→Merkle Tree（誤り） | **集約→Merkle Tree→Aggregate TX→PQC署名**（正） |

### PQC署名の正確な処理フロー

```
Step 1: 複数データのハッシュをキューに集約
Step 2: Merkle Treeで圧縮 → Root取得
Step 3: Aggregate Transaction を作成
Step 4: Aggregate TX全体のシリアライズハッシュを取得
Step 5: そのハッシュに対してPQC署名を1回生成
        アルゴリズム: FN-DSA-512（Falcon）/ ML-DSA-65（Dilithium）
        署名サイズ: FN-DSA-512 = 約690バイト
Step 6: PQC署名をMetadata TransactionまたはPlain Messageに記録
Step 7: Aggregate全体をEd25519で発行
        → Symbolネットワークはこの署名のみをコンセンサス検証
```

### 二層構造の明確な役割分担

| 層 | 保証内容 | 技術 | 検証場所 |
|---|---|---|---|
| **コンセンサス層** | 原子性・不変記録・タイムスタンプ | Ed25519 + Aggregate TX | Symbolノード（自動） |
| **量子耐性層** | 真正性の量子耐性証明 | FN-DSA-512 / ML-DSA-65 + Merkle | 検証ツール・監査アプリ（独立検証） |

### 用途適合性

| 用途 | 適合性 | 理由 |
|---|---|---|
| NFT発行・真正性証明 | ✅ 実用的 | アプリ層検証で十分 |
| 公証・監査対応 | ✅ 実用的 | 不変記録 + 検証ツールで対応可能 |
| 金融機関向け真正性保証 | ✅ 実用的 | 金融庁報告書の要件に合致 |
| Symbolネットワーク全体の自動PQC検証 | ⚠️ 将来対応 | プロトコル拡張が必要 |

### 容量・手数料設計

```
FN-DSA-512 公開鍵:    897 バイト
FN-DSA-512 署名:      690 バイト
Merkle Root (SHA3):    32 バイト
JSONオーバーヘッド:    ~80 バイト
─────────────────────────────────
署名のみ記録:         約802 バイト（Plain Message 1,023バイト制限内）
公開鍵は別Metadataエントリに分離して管理
```

**コスト比較（概算）**

| 方式 | オンチェーンTX数 | 相対コスト |
|---|---|---|
| 個別TX方式（PQC署名なし） | 1 TX / 1件 | 100% |
| Spark Processor（1000件集約） | 1 TX / 1000件 | **約0.1%** |

### PQC公開鍵の管理仕様

公開鍵はSymbol Metadataとして以下の形式で記録・管理します：

```json
{
  "v": 2,
  "purpose": "pqc-public-key-registry",
  "algorithm": "FN-DSA-512",
  "publicKey": "<hex>",
  "validFrom": "2026-02-26T00:00:00Z",
  "validUntil": "2026-08-26T00:00:00Z",
  "keyId": "<sha3-256 of publicKey>"
}
```

### 完全検証フロー

```
1. TX HashからMerkle RootとPQC署名を取得
2. keyIdからPQC公開鍵レジストリを参照
3. PQC公開鍵でPQC署名を検証
4. IPFS / ArweaveからMerkle Proofを取得
5. Merkle Root + Merkle Proofで個別データの包含を検証
→ すべて成功 = 量子耐性真正性が証明された状態
```

### セキュリティ特性

```
✅ Harvest-now-decrypt-later攻撃への対処
✅ データ改ざんの事後検出
✅ 真正性の長期保証（量子コンピュータがEd25519を破っても独立して検証可能）

⚠️ Symbolチェーン自体のEd25519破綻 → チェックポイント・ファイナリティで対応予定
⚠️ ネットワーク全体の自動PQC検証 → 将来のL1拡張で解決予定
⚠️ クライアント認証 → β1.2.0で策定予定
```

### 参考規格

- [NIST FIPS 204 (ML-DSA / Dilithium)](https://csrc.nist.gov/pubs/fips/204/final)
- [NIST FIPS 206 (FN-DSA / Falcon)](https://csrc.nist.gov/pubs/fips/206/final)
- [Symbol公式ドキュメント](https://docs.symbol.dev)
- [金融庁「耐量子計算機暗号への対応に関する検討会報告書」](https://www.fsa.go.jp)

---

*License: CC0 1.0 (パブリックドメイン) | Designed by @mkti_tiger | Spark Processor β1.1.0*
