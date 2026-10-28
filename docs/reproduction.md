# 実行と集計

## 改変MTSA

[論文の改変MTSA配布先](https://drive.google.com/file/d/13OWOGnleB78oLaWvDHn5dj-BesxPz4dt/view?usp=sharing)
と [MTSA本家](https://mtsa.dc.uba.ar/) を参照する。
配布リンクは従来のREADMEに示されたもので、今回の整備でバイナリを変更していない。
論文実験に用いたバイナリの正確な版・SHA-256・Java版は本資料だけでは確定できていない。

1. MTSAで `models/unrealizable/` の対象モデルを開く。
2. ファイル末尾の合成対象controller宣言を選択して合成する。
3. 改変版の診断・介入・再合成により得られる記録と合成結果を確認する。
4. 環境モデルを具体化した場合は、その修正モデルについて改めて合成する。

改変MTSAが合成失敗後に自動介入する設定なら、その修復後の成功を
LLM出力モデル単独の成功と混同しない。対象宣言と自動修復の有無を確認する。
参照モデルは改変の説明用であり、修正後モデルが一致すべき唯一の正解ではない。

## RQ1集計

```sh
python3 scripts/analyze_rq1_iterations.py
python3 scripts/validate_artifacts.py
```

いずれもPython標準ライブラリのみ。前者は実験時の観測値を再計算する。
後者はログ・記録・コードの参照とハッシュ、試行数、集計の整合を確認し、
合成や意味的妥当性を判定するものではない。

## 図の再生成

`scripts/plot_rq1_iterations.py` はnumpy・reportlab、
`scripts/plot_rq2_scalability.py` はnumpy・pandas・openpyxl・reportlabを使用する。
作図用スクリプトと元集計表を収録するが、作図ライブラリの版は固定していない。
不明な版を推測して記載していない。
