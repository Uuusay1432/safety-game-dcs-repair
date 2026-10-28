# Safety-game-based additional-control suggestions for unrealizable DCS models

離散制御器合成（DCS）の実現不能性に対し、追加制御の導入箇所を提示する研究の補足資料です。
元の敗北領域内にあるMixed Stateに着目し、制御不可能事象の直前に新たな制御可能事象を挿入して再合成します。
合成に成功した場合に、それまでの適用内容を追加制御記録として提示します。
任意の実現不能モデルに対する記録生成や、記録・設計変更の最小性を保証するものではありません。

## ディレクトリ構成

```text
.
├── README.md
├── docs/                         論文の補足説明・事例分析
│   ├── paper-supplement/
│   │   ├── README.md
│   │   ├── formal-details.md     形式定義・命題・証明
│   │   ├── formal-details.tex    数式のTeXソース
│   │   ├── benchmark-details.md  評価モデルの変数・要求・改変
│   │   └── evaluation-details.md 評価結果・統計・計算コストの説明
│   ├── benchmark_modifications.md
│   ├── model-source-index.md
│   ├── llm_experiment.md
│   ├── trial-index.md
│   ├── ag-synchronization-case.md
│   ├── bw-kiva-cases.md
│   ├── reproduction.md
│   ├── artifact-status.md
│   └── references.bib
├── models/
│   ├── reference/                改変前・参照用モデル（5モデル）
│   └── unrealizable/             実験入力の実現不能モデル（5モデル）
├── llm-responses/
│   ├── AT/{zero-shot,record-guided}/run-01.md ... run-15.md
│   ├── AM/{zero-shot,record-guided}/run-01.md ... run-15.md
│   ├── AG/{zero-shot,record-guided}/run-01.md ... run-15.md
│   ├── BW/zero-shot/run-01.md ... run-15.md
│   └── KIVA/zero-shot/run-01.md ... run-15.md
└── prompts/
    ├── templates/                修正指示のテンプレート
    │   ├── zero-shot.txt
    │   └── record-guided.txt
    └── filled/                   モデル等を含めた準備済み入力例
        ├── zero-shot.txt
        └── record-guided.txt
```

追加制御記録や修正コードは、保存されている範囲で各試行の原ログに含まれています。
記録・修正コードの抜粋、試行対応表、分析データ、再集計スクリプトを格納する独立したディレクトリは、現在の構成にはありません。

## 資料の案内

| 内容 | 資料 |
|---|---|
| 定義・命題・証明と保証の範囲 | [補足A](docs/paper-supplement/formal-details.md) |
| モデルの出典・改変内容 | [モデル説明](docs/benchmark_modifications.md) |
| 主要変数・規模パラメータ・要求 | [補足B](docs/paper-supplement/benchmark-details.md)、[ソースの宣言索引](docs/model-source-index.md) |
| 実験条件・手順・ログの収録範囲 | [LLM実験](docs/llm_experiment.md) |
| 試行の原ログ | [AT](llm-responses/AT/)、[AM](llm-responses/AM/)、[AG](llm-responses/AG/)、[BW](llm-responses/BW/)、[KIVA](llm-responses/KIVA/) |
| BW・KIVAの追加評価と適用範囲 | [BW・KIVA事例](docs/bw-kiva-cases.md) |
| AGの同期不整合と個別確認 | [AG事例](docs/ag-synchronization-case.md) |
| 統計・計算コストの説明 | [補足C](docs/paper-supplement/evaluation-details.md) |
| MTSAの実行時の注意 | [実行手順](docs/reproduction.md) |
| 保存資料の収録範囲・未確認事項 | [公開範囲の説明](docs/artifact-status.md) |

## 評価モデル

| モデル | 実現不能な入力モデル | 参照用モデル |
|---|---|---|
| AT(2,2) | [AT-2-2.lts](models/unrealizable/AT-2-2.lts) | [AT-2-2-origin.lts](models/reference/AT-2-2-origin.lts) |
| AM(2,5) | [AM-2-5.lts](models/unrealizable/AM-2-5.lts) | [AM-2-5-origin.lts](models/reference/AM-2-5-origin.lts) |
| AG(2,2) | [AG-2-2.lts](models/unrealizable/AG-2-2.lts) | [AG-2-2-realizable.lts](models/reference/AG-2-2-realizable.lts) |
| BW(2,2) | [BW-2-2.lts](models/unrealizable/BW-2-2.lts) | [BW-2-2-origin.lts](models/reference/BW-2-2-origin.lts) |
| KIVA | [KIVA-N-2-robot.lts](models/unrealizable/KIVA-N-2-robot.lts) | [KIVA-N-2-robot-origin.lts](models/reference/KIVA-N-2-robot-origin.lts) |

参照モデルは改変内容を説明するための資料であり、全モデルで修正後に一致すべき唯一の正解ではありません。

## 改訂版の評価結果

AT・AM・AGは各モデル・各条件15回の再実験であり、旧10試行への追加ではありません。
BW・KIVAのZero-shotも各15回実施し、原ログは計120試行分です。
BW・KIVAのRecord-guidedは記録生成失敗を各15試行分として計上し、LLM修正は実施していません。

| 対象 | Zero-shot 初回 | Record-guided 初回 | Zero-shot 10回以内 | Record-guided 10回以内 |
|---|---:|---:|---:|---:|
| 記録を生成できた3モデル | 30/45 | 44/45 | 45/45 | 45/45 |
| 全5モデル | 42/75 | 44/75 | 59/75 | 45/75 |

記録を生成できた3モデルの成功までの平均反復回数は、それぞれ1.64回と1.02回でした。
合成成功は、修正後モデルと制御可能性の下で要求を満たす制御器が得られたことを表します。
修正の妥当性、必要機能の維持、人間への支援効果を保証するものではありません。
LLMは開発者の修正作業の代行として用いており、提案アルゴリズム自体ではありません。

## ログの読み方

- 応答のみを保存したログと、入力・応答の両方を含むログがあります。
- 記録やプロンプトがログに含まれない場合、その試行の実際の入力内容を原ログだけから確認することはできません。
- AMのRecord-guided試行6は部分修正の説明であり、完全な修正後モデルは含まれていません。
- 保存されたコード候補と、実験時に採用したモデルとの一致は照合していません。
- `prompts/filled/`は準備済み入力例であり、全試行・全反復の実際の入力を収録したものではありません。
