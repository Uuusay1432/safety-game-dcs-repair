# 評価モデルの由来と改変

全5モデルの主要変数、パラメータ、事象、要求と改変内容は
[補足B](paper-supplement/benchmark-details.md) に示す。
[宣言・要求の索引](model-source-index.md) と以下の差分から具体的な変更を確認できる。

| モデル | 参照用モデル | 実現不能入力 | 差分 | 主な改変 |
|---|---|---|---|---|
| AT(2,2) | [reference](../models/reference/AT-2-2-origin.lts) | [input](../models/unrealizable/AT-2-2.lts) | [diff](../models/diffs/AT.patch) | 着陸処理中の他の着陸要求を制限するRequestMonitorを要求に追加 |
| AM(2,5) | [reference](../models/reference/AM-2-5-origin.lts) | [input](../models/unrealizable/AM-2-5.lts) | [diff](../models/diffs/AM.patch) | 利用者間の要求処理を排他とする要求を追加 |
| AG(2,2) | [reference](../models/reference/AG-2-2-realizable.lts) | [input](../models/unrealizable/AG-2-2.lts) | [diff](../models/diffs/AG.patch) | AllowActionを制御可能集合から除外。許可事象・扉プロセス自体は保持 |
| BW(2,2) | [reference](../models/reference/BW-2-2-origin.lts) | [input](../models/unrealizable/BW-2-2.lts) | [diff](../models/diffs/BW.patch) | 外部審査者・排他要求を追加し、assignを制御不可能化 |
| KIVA | [reference](../models/reference/KIVA-N-2-robot-origin.lts) | [input](../models/unrealizable/KIVA-N-2-robot.lts) | [diff](../models/diffs/KIVA.patch) | 移動・replenish・bringの可制御性を変更し、位置競合回避の要求を追加 |

差分は公開ファイル間の機械的な差分であり、コメント・書式・補助宣言の差も含む。
環境動作はプロセス定義、要求はltl_propertyとsafety、制御可能性はcontrollable、
合成構成はEnvironmentとcontroller宣言を区別して確認する。
参照モデルは改変の説明用であり、修正後モデルが一致すべき唯一の正解とはしない。
AGの参照はN=2・しきい値2であり、N=4の別モデルとの比較ではない。

## 出典

- AT / BW: [MTSA公開モデル](https://mtsa.dc.uba.ar/)。本リポジトリの参照ファイルが具体的な比較対象である。
- AG: [DOI 10.1145/3194133.3194146](https://doi.org/10.1145/3194133.3194146)。
- AM: [IEEE document 9694217](https://ieeexplore.ieee.org/document/9694217)。
- KIVA: [DOI 10.1145/2908675.2908681](https://doi.org/10.1145/2908675.2908681)。

先行研究での利用例を含む書誌情報は [references.bib](references.bib) を参照する。
文献上のモデルと、ここに掲載した具体的な参照ファイルの版は区別する。
元モデル配布版の正確な取得日時・コミットは保存資料から確定できていない。
