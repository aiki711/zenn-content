---
title: "LLMゼロショット感情推定の実験をしてみる"
emoji: "👻"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: []
published: false
---

# LLMゼロショット感情推定の実験をしてみる  
**— プロンプト → JSON → パース → 評価までの再現可能な最小構成 —**

*更新: 2025-08-17 04:44*

## TL;DR
- LLMに**JSONスキーマで出力を強制** → パーサで機械的に抽出 → 指標と混同行列で評価  
- データは小規模デブセット（3クラス）と IMDB（2クラス）を使用
- 評価は **Accuracy / Precision / Recall / F1（Macro）/ Confusion Matrix** を一発計算
- 付録として **Few-shot / CoT** のプロンプトも用意し、比較実験がすぐ始められるテンプレ設計



## 背景と目的
生成系LLMで**感情（評価的態度）**　をゼロショット分類させると、出力の**形式揺れ**と**ニュートラル判断の一貫性**が課題．
本記事では、**出力形式の統一（JSON）** と**ニュートラルの明確化**によって、実験を**壊れにくく**・**再現可能**にする最小パイプラインを紹介．



## 使ったコード（ざっくり）
```
prompts.py               # Zero-shot / Few-shot / CoT の日本語プロンプト
parser.py                # LLM応答から最初のJSONブロックを抽出・正規化
evaluate_zero_shot.py    # 指標（Accuracy/Precision/Recall/F1）と混同行列を計算
labels.json              # ラベル集合（2/3/8クラス）
run_vLLM_imdb_zero_shot.py  # vLLMでIMDB 100件をゼロショット2クラス評価
README.md                # ワークフローのREADME
```



## 実験設計

### タスク
- **3クラス**：positive / neutral / negative  
- **2クラス**：positive / negative（IMDBに合わせた設定）

### 出力スキーマ（LLMの回答は**必ず**これだけ）
```json
{{
  "label": "positive|neutral|negative",
  "confidence": 0.0_to_1.0,
  "reason": "短い根拠文"
}}
```

### プロンプト設計のポイント
- **ニュートラルの定義**を冒頭で固定（曖昧・事実のみは neutral）  
- **JSON強制**で評価を頑健化  
- **Few‑shot**は2例、**CoT**は思考→最終JSONのみ出力の流れを明示  
（後述のテンプレをそのまま使えます）



## 再現手順（3クラス：小規模デブセット）

> 手動10件で流れ確認 → 100件に拡張、の順がおすすめ

1) **プロンプト適用**（Zero‑shotの例）  
```python
from prompts import zero_shot_prompt
p = zero_shot_prompt("これは最高！また買います。")
print(p)  # LLMに与えるプロンプト
```

2) **LLMへ投げる** → 返ってきたJSONを `predictions.jsonl` に1行1件で追記  
（例）
```jsonl
{"output": "{{\"label\":\"positive\",\"confidence\":0.94,\"reason\":\"強い満足の表現\"}}"}
```

3) **評価**  
```bash
python evaluate_zero_shot.py --gold sample_dev.jsonl --pred predictions.jsonl --labels 3class
```

出力は Accuracy / Precision / Recall / F1（Macro）と混同行列．
エラー例の先頭20件も自動でプレビューする．



## 再現手順（2クラス：IMDB + vLLM）

1) **IMDBから100件をサンプリング & 推論実行**  
```bash
python run_vLLM_imdb_zero_shot.py   --model mistralai/Mistral-7B-Instruct-v0.3   --num_samples 100 --max_tokens 128 --temperature 0.0
```
- `runs_imdb_zero_shot/imdb100_predictions.jsonl` と `imdb100_gold.jsonl` が保存．

2) **評価**  
```bash
python evaluate_zero_shot.py --gold runs_imdb_zero_shot/imdb100_gold.jsonl                              --pred runs_imdb_zero_shot/imdb100_predictions.jsonl                              --labels 2class
```



## プロンプトの実体（抜粋）

### Zero‑shot（3クラス／日本語）
```text
positive / neutral / negative の3クラスで判定。
- 事実のみ・曖昧は neutral
- 明確な好意は positive
- 明確な不満は negative
- 指定の JSON スキーマで出力
```
※ Few‑shot（例示2件付き）と CoT（思考→最終JSONのみ出力）も用意．



## パース戦略（壊れない仕組み）
- 応答本文から **最初の `{ ... }` を抽出して JSON として読む**  
- `label` は **表記ゆれ補正**（pos/neg/neu → 正式表記）  
- `confidence` は 0.0–1.0 にクリップ  
- どうしてもパースできない場合は **`label=None` → 評価時は neutral にフォールバック**

> これで「余計な前置きや説明が混ざっても」評価が止まらずに進む．



## 評価指標と出力例
- **Accuracy / Precision（Macro）/ Recall（Macro）/ F1（Macro）**  
- **per-class詳細** と **Confusion Matrix** をJSONで出力  
- 直後に **誤分類の先頭20件** をダンプするので、エラー分析が始めやすい．

実運用では次の3点をまず見る：  
1) `f1_macro` の水準（ベースライン比較）  
2) ニュートラル周りの取り違え（`neutral→positive/negative` の誤り）  
3) 特定ラベルの**精度と再現率のアンバランス**（過検出/過少検出）



## 典型的な失敗と対処
- **LLMがJSON以外も返す** → パーサで最初のJSONだけ拾う．プロンプトに **「JSONのみ」** を明記
- **`label`欠落や表記揺れ** → 正規化テーブルで補正（pos/neg/neu）
- **どうしても読めない** → `neutral` にフォールバックして先へ進む（後で誤分類として確認）  
- **CoTで出力崩壊** → 「最後は**JSONのみ**」を強調。`max_tokens` を余裕ある値に



## 実験結果
```
# (例)
"accuracy": 0.96,
  "precision_macro": 0.9556650246305418,
  "recall_macro": 0.9624999999999999,
  "f1_macro": 0.9586606035551881,
  "per_class": {
    "positive": {
      "precision": 0.9827586206896551,
      "recall": 0.95,
      "f1": 0.9661016949152542
    },
    "negative": {
      "precision": 0.9285714285714286,
      "recall": 0.975,
      "f1": 0.951219512195122
    }
  }
```

- 本実験では Accuracy=0.96、F1(macro)=0.959 と高い性能を示した．
- クラス別には positive で Precision=0.983/Recall=0.95、negative で Precision=0.929/Recall=0.975 となり，**ネガティブの見落とし（FN）は少ない一方，ネガ過検出（FP）が相対的に多い**傾向が確認できた．これはモデルが**ネガ判定にやや攻め**で，**ポジ判定を慎重**に行う方針を暗に取っていることを示す．誤りは主に**ポジ→ネガ**の取り違えに由来すると推測される．対策として， 
  (i) プロンプト内でネガ判定の基準をより厳格に規定する
  (ii) 返却される信頼度に基づきネガ採用の**しきい値**を設ける
  (iii) 境界事例を含む**対称的なFew-shot**を付与する
  の3点を提案する．
- 加えて，評価サンプルが100件規模の場合，Accuracy の95%CIは概ね [0.90, 0.98] であり，**±4%程度の統計的ゆらぎ**が存在する点にも留意が必要である．
- 今後はこれらの改善を加味し，Few-shot/CoT/モデル差の横持ち比較を行い，特に negative の Precision 改善と再現性の確認を進める．



## 発展（Few‑shot / CoT / LoRA）
- 少数例を入れると `reason` の質が上がり、ニュートラルの扱いが安定化することが多い 
- CoTは理由文が良くなる一方で**出力崩壊リスク**があるため、**最後にJSONのみ**を再度強制
- LoRA微調整を検討する場合も、本記事の**同じ評価関数**を流用すれば比較が公正になる



## 付録：コマンド速攻リファレンス
```bash
# 3クラス（任意のLLM → 応答をpredictions.jsonlに格納）
python evaluate_zero_shot.py --gold sample_dev.jsonl --pred predictions.jsonl --labels 3class

# 2クラス（IMDB x vLLM）
python run_vLLM_imdb_zero_shot.py --model mistralai/Mistral-7B-Instruct-v0.3
python evaluate_zero_shot.py --gold runs_imdb_zero_shot/imdb100_gold.jsonl                              --pred runs_imdb_zero_shot/imdb100_predictions.jsonl                              --labels 2class
```



## おわりに
今回のゼロショット実験では，Accuracy 0.96／F1(macro) 0.959 という自分の想定を大きく上回る結果が得られました．正直，LLMが追加学習なしでここまで安定して判定できるとは思っておらず，**「テキストを入れて、返ってきたJSONを評価するだけ」** という手軽さにも驚かされました．プロンプトで出力形式を固定し，パーサで機械的に救済するという最小構成でも，十分に**再現可能な評価**に到達できることを確認できたのは大きな収穫です．

一方で，評価対象はIMDBの2クラス・サンプル数も限定的であり，**分布の違い（日本語レビュー，皮肉や婉曲表現，コードスイッチなど）** に対する頑健性は今後の検証課題です．クラス別の誤りを見ると，negativeの**Precisionが相対的に低い**傾向があり，ネガ過検出（FP）が主な誤り源になっていました．ここは**ネガ判定の基準を厳格化**するプロンプト修正や，**confidenceしきい値**の導入で改善できる見込みがあります．

次の一歩として，(1) Few-shot/Cotの導入と境界事例の提示，(2) モデル横断比較（Mistral/Llama/Qwen/日本語特化系），(3) 3クラス設定でのニュートラルの安定化，(4) confidenceに基づく**運用上のしきい値設計**（Recall/Precisionのトレードオフ最適化）を進めます．今回の「**プロンプト→JSON→パース→評価**」という足場を起点に，データ分布と要求精度に合わせて段階的に強化していきます．
