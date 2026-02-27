# MINIMAL VERIFIER v1.2.0
**真正性プリミティブ — 不変コア**  
言語非依存の疑似コード（可読性のためPythonスタイル）  
総行数: 182行（コメント含む）  
作成日: 2026-02-27  
ライセンス: CC0 1.0 Universal（パブリックドメイン）

---

## 目的

このファイルは **Spark Processor** の魂を定義する、数学的に固定された最小ルールセットです。  
どの言語による実装であっても、何十年・何百年後であっても、  
**検証可能な真正性を維持するために、これらのルールを厳密に遵守しなければなりません。**

> ⚠️ このファイルは不変のコアです。1ビットでも逸脱すると真正性プリミティブが破壊されます。

---

## 1. 正規シリアライズ規則（CBOR RFC 8949 厳格正規エンコーディング）

```pseudocode
def canonical_serialize(data):
    # ⚠️ 注意: bool は int のサブクラスであるため、必ず int より先にチェックする
    if data is None:
        return encode_cbor_null()
    elif isinstance(data, bool):          # bool を int より先に判定（重要）
        return encode_cbor_boolean(data)
    elif isinstance(data, int):
        return encode_cbor_integer(data)
    elif isinstance(data, float):
        return encode_cbor_float_ieee754_binary64(data)
    elif isinstance(data, str):
        return encode_cbor_text(data.encode('utf-8'))
    elif isinstance(data, bytes):
        return encode_cbor_bytes(data)
    elif isinstance(data, list):
        return encode_cbor_array([canonical_serialize(item) for item in data])
    elif isinstance(data, dict):
        # キーをUTF-8バイト列でソートして決定論的な順序を保証する
        sorted_keys = sorted(data.keys(), key=lambda k: k.encode('utf-8'))
        sorted_items = [(k, canonical_serialize(data[k])) for k in sorted_keys]
        return encode_cbor_map(sorted_items)
    else:
        raise ValueError(f"サポートされていない型: {type(data)}")
```

---

## 2. マークルツリー構築規則

```pseudocode
def sha3_512(data):
    return hash_sha3_512(data)  # 常に64バイトのダイジェストを返す

def build_merkle_tree(leaves):
    # 空リストの場合は空バイト列のハッシュを返す
    if not leaves:
        return sha3_512(b"")

    # リーフをUTF-8バイト順でソート（決定論的順序の保証）
    # ⚠️ 注意: leaves の型が混在する場合（dict, bytes, str など）は
    #          型ごとに正規化してからソートすること
    sorted_leaves = sorted(
        leaves,
        key=lambda x: canonical_serialize(x) if isinstance(x, (dict, list, str, int, float, bool)) else x
    )

    # 各リーフをハッシュ化
    # dict / list は canonical_serialize を経由してから SHA3-512 にかける
    # それ以外（bytes）はそのまま SHA3-512 にかける
    nodes = [
        sha3_512(canonical_serialize(leaf)) if isinstance(leaf, (dict, list, str, int, float, bool))
        else sha3_512(leaf)
        for leaf in sorted_leaves
    ]

    # ボトムアップでマークルルートを計算する
    while len(nodes) > 1:
        new_nodes = []
        for i in range(0, len(nodes), 2):
            left  = nodes[i]
            right = nodes[i + 1] if i + 1 < len(nodes) else nodes[i]  # 奇数の場合は自身を複製
            new_nodes.append(sha3_512(left + right))
        nodes = new_nodes

    return nodes[0]  # マークルルート（64バイト）
```

---

## 3. 署名入力形成規則

```pseudocode
def form_signature_input(merkle_root, spec_genesis_hash, timestamp_rfc3339):
    """
    署名対象のペイロードを構築する。

    Args:
        merkle_root        : bytes  — build_merkle_tree() の戻り値（64バイト）
        spec_genesis_hash  : bytes  — この仕様ファイル自体のジェネシスハッシュ
        timestamp_rfc3339  : str    — RFC 3339 形式のタイムスタンプ（例: "2026-02-27T00:00:00Z"）

    Returns:
        bytes — canonical_serialize されたペイロード（署名・検証に使用）
    """
    payload = {
        "merkleRoot"       : merkle_root,
        "specGenesisHash"  : spec_genesis_hash,
        "timestamp"        : timestamp_rfc3339
    }
    return canonical_serialize(payload)
```

---

## 4. 検証手順

```pseudocode
def verify_authenticity(anchor_data, public_key, signature):
    """
    真正性アンカーデータの完全な検証を行う。

    Args:
        anchor_data : dict     — 検証対象のアンカーデータ
        public_key  : bytes    — PQC（耐量子暗号）公開鍵
        signature   : bytes    — PQC署名

    Returns:
        bool — True: 検証成功 / False: 検証失敗
    """

    # ステップ1: SpecGenesisHash の一致確認
    # 仕様が改ざんされていないことを保証する
    if anchor_data["specGenesisHash"] != CURRENT_SPEC_GENESIS_HASH:
        return False

    # ステップ2: マークルルートの再計算
    computed_root = build_merkle_tree(anchor_data["leaves"])

    # ステップ3: 署名入力の再形成
    sig_input = form_signature_input(
        computed_root,
        anchor_data["specGenesisHash"],
        anchor_data["timestamp"]
    )

    # ステップ4: PQC 署名の検証
    if not pqc_verify(sig_input, signature, public_key):
        return False

    # ステップ5: マークルパスの検証（オプション）
    # merklePath が存在する場合のみ検証を実施する
    if "merklePath" in anchor_data:
        if not verify_merkle_path(anchor_data["merklePath"], computed_root):
            return False

    return True
```

---

## 変更点サマリー（v1.2.0 日本語版での修正）

| # | 箇所 | 問題 | 修正内容 |
|---|------|------|----------|
| 1 | `canonical_serialize` | `bool` は Python で `int` のサブクラスのため、`int` より後に判定すると `True/False` が整数として扱われる | `bool` チェックを `int` より**前**に移動 |
| 2 | `build_merkle_tree` のソート | `leaves` に `dict`, `bytes`, `str` が混在すると `sorted()` が型エラーを起こす可能性がある | ソートキーに `canonical_serialize` を使用し、型を統一 |
| 3 | `build_merkle_tree` のハッシュ化 | `dict`/`list` 以外の型（`str`, `int` など）が来た場合の処理が未定義 | `bytes` 以外の全型に `canonical_serialize` を適用するよう修正 |

---

## ライセンス

CC0 1.0 Universal — このファイルはパブリックドメインです。  
著作権の放棄により、いかなる制限もなく自由に使用・改変・配布できます。
