---
title: "Codexに「メモリ観測できる環境を探って」と頼んだら構成提案から構築まで全部やってくれた"
emoji: "🔬"
type: "tech"
topics: ["codex", "opentelemetry", "grafana", "playwright", "cdp"]
published: true
---

## 何が起きたのか

Webアプリのパフォーマンス改善に取り組んでいたとき、まずは計測環境を整備する必要がありました。Chrome DevTools Protocolでメモリを取りたい、できればログやメトリクスも可視化したい――ただ、具体的にどのツールを組み合わせるかまでは固まっていません。

そこでCodex（OpenAIのコーディングエージェント）に、まずこう伝えました。

> Chrome DevToolsと連携してメモリ使用量を観測しつつ、色々試作を打てる環境を作りたい。それに合っているものを取り入れてみて。codexというワードを入れつつWeb検索を使っていい方法を探って。

Codexはこの指示を受けてWeb検索を行い、Chrome DevTools MCP・Playwright経由のCDP接続・ローカルLGTMスタックの組み合わせを提案してきました。構成として筋が通っていたので、そのまま許可を出しました。

> では、Codex + Chrome DevTools MCP + Playwright(CDP) + ローカルLGTM(Logs/Metrics/Traces) の組み合わせのセットアップを行なって

自分がやったのは「方向性を示す」と「提案を承認する」の2ステップだけです。Codexはここから **11ファイルの追加・変更** を含む完全な観測基盤を1コミットで仕上げてきました。

## Codexが自動構築した全体像

構築された環境は、大きく5つのレイヤに分かれます。

### 1. 観測基盤（LGTM スタック）

`grafana/otel-lgtm` のDockerイメージを使い、1コンテナで以下を起動する `compose.observability.yaml` が生成されました。

- **Grafana** — ダッシュボード
- **Prometheus** — メトリクス収集
- **Loki** — ログ収集
- **Tempo** — 分散トレーシング
- **OpenTelemetry Collector** — OTLP受信の入り口

```yaml
# compose.observability.yaml（Codexが生成）
# grafana/otel-lgtm を使い、Grafana / Prometheus / Loki / Tempo / OTel Collector を1コンテナで起動
```

`make obs-up` 一発で全部立ち上がり、`http://127.0.0.1:3300` でGrafanaにアクセスできます。

### 2. ブラウザ連携（CDP）

CDPポートを開いた状態でChromeを起動するシェルスクリプトが2本。専用の `--user-data-dir` を切ることで、普段のブラウザプロファイルと干渉しない設計になっていました。

```bash
# CDP有効でChrome起動
make cdp-chrome-up

# headlessで起動する場合
make cdp-chrome-up-headless
```

### 3. 自動計測プローブ（Playwright + CDP）

ここが一番手が込んでいた部分です。Codexが書いた `cdp-memory-probe.ts` は、以下を自動で実行します。

- Playwrightの `connectOverCDP` で起動済みChromeに接続
- CDP経由でメモリ使用量・DOM数・リスナー数を時系列取得
- `PerformanceObserver` でlong taskを検出
- `Performance.getMetrics` からLayout回数やScript実行時間を取得
- 収集したデータをOTLP HTTPでLGTMスタックに送信
- 同時にローカルファイル（NDJSON/JSON）にもバックアップ保存

```bash
# 対象URLに対して180秒間、1秒間隔で計測
pnpm perf:cdp:probe \
  --target-url=http://127.0.0.1:3000/projects/1/pages/1 \
  --duration-sec=180 \
  --interval-ms=1000
```

出力されるファイルは3種類です。

| ファイル | 内容 |
|---|---|
| `samples-*.ndjson` | 毎秒のサンプルデータ（時系列） |
| `summary-*.json` | p95/max/latestなどの統計サマリ |
| `sampling-profile-*.json` | CDPのサンプリングプロファイル |

### 4. クエリ導線

収集したデータをCLIから直接クエリできるスクリプトも生成されていました。

```bash
# Lokiのログを検索
make query-logql

# Prometheusのメトリクスを検索
make query-promql

# Tempoのトレースを検索
make query-traceql
```

GrafanaのUIを開かなくても、ターミナルから即座にデータを確認できます。

### 5. 実行導線（Makefile統合）

上記すべてが `Makefile` のターゲットに整理されていました。

```makefile
obs-up              # LGTM起動
obs-down            # LGTM停止
obs-ps              # コンテナ状態確認
cdp-chrome-up       # CDP Chrome起動
cdp-chrome-down     # CDP Chrome停止
perf-probe          # 計測プローブ実行
query-logql         # LogQLクエリ
query-promql        # PromQLクエリ
query-traceql       # TraceQLクエリ
```

## 指示していないのにCodexが判断した作業

特筆すべきは、明示的に頼んでいない作業をCodexが自律的に行っていた点です。

- **Makefile導線の整備** — 繰り返し使うコマンドをターゲット化
- **セットアップ手順書の作成** — `CODEX_CDP_LGTM_SETUP.md` を自動生成
- **`.gitignore` の更新** — 計測成果物（`.perf/`）をGit管理対象外に設定
- **Chrome DevTools MCPの追加と疎通確認** — `codex mcp add` の実行とリスト確認
- **OTLP投入の疎通テスト** — logs/metrics/tracesの3系統すべてで疎通確認
- **後片付け** — 起動したコンテナとChromeを停止してクリーンな状態に復元

「パフォーマンス計測の環境を作る」という抽象的な依頼から、運用まで見据えた判断をエージェントが下しています。

## 観測できる項目の一覧

構築された環境で取得できるデータをまとめます。

**メモリ関連**
- `usedJSHeapSize` / `totalJSHeapSize` / `jsHeapSizeLimit`（`performance.memory`）
- `measureUserAgentSpecificMemory`（対応環境のみ）
- CDP `Memory.getSamplingProfile`

**DOM/描画負荷**
- DOM nodes / documents / listeners（`Memory.getDOMCounters`）
- `LayoutCount` / `RecalcStyleCount` / `TaskDuration` / `ScriptDuration`（`Performance.getMetrics`）

**long task**
- `PerformanceObserver('longtask')` による件数と累積時間

これらがすべてPrometheus（メトリクス）・Loki（ログ）・Tempo（トレース）に流れ込むため、Grafanaで横断的に分析できます。

## 実運用での使い方

Codexが構築した環境を使った問題特定のフローは以下のとおりです。

1. `make obs-up` と `make cdp-chrome-up` で基盤を起動
2. Codexから Chrome DevTools MCPを使って対象画面を操作
3. 並行して `perf:cdp:probe` を実行し、時系列サンプルを採取
4. `summary-*.json` の `p95Bytes` / `maxBytes` を確認して異常箇所を特定
5. LogQL / PromQL / TraceQL で詳細を掘り下げる
6. 改善実装後に同条件で再計測し、ビフォーアフターを数値で比較

「計測 → 特定 → 修正 → 再計測」のサイクルが、すべてCLIで完結します。

<!-- TODO: この観測基盤を使って実際に弊社アプリのパフォーマンス改善がどう行われたか（計測結果・ボトルネック特定・改善前後の数値比較など）を追記する -->
