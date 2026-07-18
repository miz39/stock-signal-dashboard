# agents/ コードレビュー（2026-07-15）

> **対応状況（同日対応）**: High 4件・Medium 4件・Low 6件のうち、感応度エージェントの重み集中（Medium #8 の重み変更部分）を除きすべて修正済み。
> - バグ3件（負PER/PBR、NaNデッドクロス、comps confidence）→ 修正
> - 冗長フェッチ → data.py にプロセス内キャッシュ追加（財務TTL 1h / 指数OHLCV TTL 10分。通常銘柄の価格はキャッシュしない）
> - confidence 定義 → 全エージェント「データ充足度」ベースに統一
> - テクニカルの逆張り採点 → 順張り整合に変更（RSI 50-65 スイートスポット加点、BB下限割れは減点）
> - 単位正規化 → data.py `_normalize_financial_data()` に一元化（canonical: percent）
> - coordinator → 重みキーを英語キー直結に、Trading/Valuation 統合比率を `valuation.combined_trading_weight` で config 化
> - Low 4件（ベータddof、DCFターミナル下限、モメンタムoff-by-one）→ 修正（DCF成長率推定の年欠落は据え置き）
> - テスト → `tests/test_trading_agents.py` 追加（25テスト）、`test_portfolio.py` の更新漏れ3件を recent_pf 基準に修正
> - **見送り**: バリュエーション重みの再配分（DCF+感応度=0.40 の集中）は戦略判断のため未変更。変更する場合はバックテスト前後比較を推奨

対象: `agents/` 全10ファイル（Trading Layer 4 + Valuation Layer 5 + coordinator）
方法: 全ファイル精読 + `data.py` / `config.yaml` / `tests/` との突き合わせ

## サマリー

全体として構造は良い（各エージェントが同一の戻り値スキーマ `score/confidence/reasons/metrics` を守り、coordinator が重み付け統合するだけの疎結合）。一方で **スコアリングの正確性に関わるバグが3件**、**パフォーマンスに大きく効く冗長フェッチが1件**、設計上の不整合が数件ある。

| 重要度 | 件数 | 内容 |
|--------|------|------|
| High（バグ） | 3 | 負のPER/PBRを「割安」判定、NaN比較の誤判定、comps の confidence 誤計算 |
| High（性能） | 1 | `fetch_financial_statements` が1銘柄あたり4回呼ばれる（キャッシュなし） |
| Medium（設計） | 4 | confidence の意味が不統一、テクニカルの逆張り採点、単位推定の脆さ、DCF/感応度の重複 |
| Low | 4 | ベータ計算のddof不整合ほか |

---

## High: バグ

### 1. 負のPER/PBRが「割安」と誤判定される — `fundamental.py:97-128`

```python
per = fin.get("per")
if per and per < 500:      # per=-10 でも通過
    ...
    if per < 10:
        score += 0.5       # 赤字企業が「PER -10.0 → 割安」で +0.5
```

赤字企業（負のPER）が最高評価の「割安」を得る。PBRも同様（`if pbr:` のみで、債務超過の負PBRが `pbr < 1.0` → +0.4「割安」）。
`comps.py:115` は正しく `0 < target_per < 500` とガードしており、同じガードを入れるべき。

**修正案**: `if per and 0 < per < 500`、`if pbr and pbr > 0`。負のPERは別途減点（赤字）としてもよい。

### 2. データ不足時にNaN比較で「デッドクロス」と誤判定 — `technical.py:77-82`

`sma_long`（100日）や `sma_short` は上場間もない銘柄・短いDataFrameでNaNになるが、NaNガードがあるのは `sma_trend` のみ。`nan > nan` は False なので else 側に落ち、**score -= 0.5「SMAデッドクロス」** が付く。RSI・MACD も同様にNaNのまま metrics に入る（`float(nan)`）。

また `histogram.iloc[-2]`（technical.py:108）は行数2未満で IndexError。

**修正案**: 先頭で `len(df) < sma_long期間` なら confidence 0 で早期リターン、または各指標にNaNチェック。

### 3. 対象銘柄のPER/PBRが取れなくても confidence が高いまま — `comps.py:174-177`

```python
data_points = len(peer_pers) + len(peer_pbrs)   # ピア側のデータ数
confidence = min(100, int(data_points / 20 * 100))
if not (target_per or target_pbr):
    confidence = max(confidence, 10)             # ← min の間違い？
```

対象銘柄のPER/PBRが両方Noneだと比較は一切行われず score=0 なのに、ピアのデータ数だけで confidence が最大100になる。`max(confidence, 10)` は下限を引き上げるだけで意図（データ欠損時に信頼度を下げる）と逆。

**修正案**: `confidence = min(confidence, 10)` に変更。

---

## High: パフォーマンス

### 4. `fetch_financial_statements` が1銘柄あたり4回呼ばれる（キャッシュなし）

`analyze_valuation()` の1回の実行で、dcf / three_statement / operating_model / sensitivity がそれぞれ独立に `fetch_financial_statements(ticker)` を呼ぶ。1回の呼び出しは `yf.Ticker().financials + balance_sheet + cashflow + info` の4リクエスト相当なので、**1銘柄で約16リクエスト、うち12は完全に重複**。

同様の重複:
- `fetch_financial_data`: fundamental（Trading層）と comps（対象銘柄分）で重複。comps はさらにピア最大30社分を毎回フェッチ
- `^N225`: sentiment（3mo）と risk_agent（1y）が銘柄ごとに毎回フェッチ → 候補10銘柄で20回

CLAUDE.md に「run_full_analysis は1-2分かかる」とあるが、この重複解消だけでバリュエーション層は理論上半分以下になる。yfinance のレート制限リスクも下がる。

**修正案**: `data.py` に `_earnings_date_cache` と同じパターンのプロセス内キャッシュを追加（`fetch_financial_statements` / `fetch_financial_data` / 指数データ）。エージェント側の変更は不要。

---

## Medium: 設計

### 5. confidence の意味がエージェント間で不統一

| エージェント | confidence の定義 |
|---------------|------------------|
| technical / sentiment / risk | `abs(score)/2*100` — スコアの強さ（中立=0%） |
| fundamental | データ充足度とスコア強度の min |
| DCF / 三表 / オペレーティング | データ充足度 |
| 感応度 | `above_ratio*50 + 年数*15` — **割安度が高いほど confidence が上がる** |

coordinator はこれらを同質のものとして加重平均している。特に感応度は「自信を持って割高と判断した」場合に confidence が下がる、という逆転が起きる。中立判断（score 0）で confidence 0% になるのも「判断に自信がない」と「強い材料がない」の混同。

**推奨**: confidence は「データの量・質」のみで定義し、方向・強さは score に一本化する。

### 6. テクニカルエージェントが本体戦略と逆方向の採点

本体戦略は GC + RSI 50-62 + ADX≥28 の順張りだが、`technical.py` は RSI<30 に +0.5（売られすぎ買い）、ボリンジャー下限タッチに +0.3（反発期待）と逆張り採点。エントリーゲートを通過した銘柄（RSI 50-62）はこのエージェントからRSI加点をほぼ得られず、逆にゲート外の押し目銘柄が高スコアになる。マルチエージェント判断が本体フィルタと綱引きする構造。意図的な多様性なら CLAUDE.md 等に明記、そうでなければ順張り整合に寄せるべき。

### 7. 単位推定ヒューリスティックが脆い — `fundamental.py`

```python
div_pct = div_yield if div_yield > 0.2 else div_yield * 100
growth_pct = revenue_growth * 100 if abs(revenue_growth) <= 5 else revenue_growth
equity_pct = equity_ratio * 100 if equity_ratio <= 1 else equity_ratio
```

yfinance のバージョンで比率/パーセントの返り値仕様が変わった経緯への対処だが、境界値で誤変換する（例: 配当利回り0.15%が「15%の高配当」判定で +0.3）。**単位の正規化は data.py のプロバイダ層で1回だけ行い、エージェントは正規化済みの値を信頼する**構造にすべき。

### 8. 感応度エージェントは実質DCFの再パッケージ

`sensitivity.py` は dcf.py のプライベート関数を import して同じ計算を±2%/±1%振っているだけで、独立した情報源ではない。重み上は DCF 0.30 + 感応度 0.10 = 0.40 が同一モデルに載っており、バリュエーション層の分散が見かけより低い。coordinator で DCF の計算結果（base_fcf, net_debt, shares等）を渡して再フェッチ・再計算を省くか、DCF の metrics に感応度テーブルを含めて統合する選択肢もある。

---

## Low

- **risk_agent.py:65-67**: ベータ計算で `np.cov`（ddof=1）と `np.var`（ddof=0）が混在し、ベータが約 n/(n-1) 倍過大。`np.var(nr, ddof=1)` に。
- **dcf.py:213-216**: `wacc <= terminal_growth` 時のフォールバック `wacc - 0.02` は wacc<2% で永続成長率が負になり、reasons にはその旨の注記がない。
- **dcf.py:90-96**: 成長率推定で負のFCF年を除外してからYoYを取るため、年の連続性が崩れた系列で成長率を計算している。
- **sentiment.py:48**: `close.iloc[-1]/close.iloc[-5]` は実質4営業日モメンタム（off-by-one）。20日側も同様。
- **coordinator.py:372**: Trading/Valuation の 50/50 統合比率がハードコード。config 化の余地。
- **coordinator.py:97**: 重みのキーが日本語ラベル（`r["agent"]`）依存。ラベル変更で黙って既定値 0.25 にフォールバックする。

## テストカバレッジ

- `tests/test_valuation_agents.py`（22テスト）がバリュエーション層をカバー — 良好
- **Trading層4エージェント（technical / fundamental / sentiment / risk_agent）の単体テストは無い**（test_full_analysis でモックされているのみ）。上記バグ1・2はいずれも単体テストがあれば検出できた類。負のPER、短いDataFrame、NaN入りデータのケースを最低限追加推奨

## 推奨アクション（優先順）

1. バグ3件の修正（fundamental の負PER/PBR、technical のNaNガード、comps の confidence）— 小さい変更
2. `data.py` にプロセス内キャッシュ追加（financial_statements / financial_data / 指数）— 効果大・変更局所的
3. Trading層エージェントの単体テスト追加（バグ再発防止）
4. confidence 定義の統一（設計判断が必要、要相談）
5. テクニカルの順張り/逆張り方針の整理（戦略判断、要相談）
