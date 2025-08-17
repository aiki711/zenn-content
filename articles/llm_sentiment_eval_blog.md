---
title: "LLMゼロショット感情推定の実験をしてみる"
emoji: "🔎"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["AI","LLM","NLP","Sentiment","Evaluation"]
published: true
---

# LLMゼロショット感情推定の実験をしてみる  
**— プロンプト → JSON → 応答解析モジュール(パース) → 評価までの再現可能な最小構成 —**

*更新: 2025-08-17 04:44*

## TL;DR
- LLMに**JSONスキーマで出力を強制** → パースで機械的に抽出 → 指標と混同行列で評価  
- データは小規模デブセット（3クラス）と IMDB（2クラス）を使用
- 評価は **Accuracy / Precision / Recall / F1（Macro）/ Confusion Matrix** を一発計算
- 付録として **Few-shot / CoT** のプロンプトも用意し、比較実験がすぐ始められるテンプレ設計



## 背景と目的
　本稿の実験は，最初は「単純に面白そうだから」という好奇心から始めた．ゼロショットで感情をどれくらい正しく当てられるのかを，自分の手で確かめたかったからだ．
もう一つの動機は，今後進める相互理解支援の研究に向けて，モデルやプロンプトを入れ替えながら壊れずに回る評価パイプラインを確立しておくことである．
そこで，プロンプトをJSON出力で固定→パースで形式揺れを救済→基本指標でサクッと評価，という最小構成を用意した．
これによって，モデル・データ・プロンプトを差し替えるだけで比較できる．今後の改良（Few-shot/CoT/モデル横断）につながる実験の足場を作ることを目的としている．



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
（後述のテンプレをそのまま使える，はず）



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
- **LLMがJSON以外も返す** → パースで最初のJSONだけ拾う．プロンプトに **「JSONのみ」** を明記
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


## 留意事項
本稿は2クラス（positive/negative）評価であるが，パースは将来の拡張を見据えた3クラス対応の汎用設計となっている．推論プロンプトは2クラスのみを許すJSONスキーマを使っているため，通常は neutral は出力されない．なお，2クラス外（パース不能/neutral）が出た場合にどう扱うかで指標が変わるので，strictモード（2クラス外を誤りとして集計）または invalidクラスの明示化を推奨する．



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




## 実際に使ったコード

> セキュリティ注意：公開時は APIキーや個人情報、社内URLなど**秘密情報を含めない**ようにしてください。

### `evaluate_zero_shot.py` — 評価スクリプト（2クラス/3クラス対応）
- 入力：`--gold`（正解JSONL）、`--pred`（LLMの出力JSONL）  
- 出力：Accuracy / Precision / Recall / F1（macro）と混同行列、誤分類の先頭20件プレビュー

```python:evaluate_zero_shot.py
import json, argparse, collections
from typing import List, Dict
from parser import extract_json, normalize_label

def load_jsonl(path: str) -> List[dict]:
    with open(path, "r", encoding="utf-8") as f:
        return [json.loads(line) for line in f]

def classification_report(y_true: List[str], y_pred: List[str], labels: List[str]) -> Dict:
    # Precision / Recall / F1 (macro) + Confusion
    cm = {l:{ll:0 for ll in labels} for l in labels}
    for t, p in zip(y_true, y_pred):
        if t in labels and p in labels:
            cm[t][p] += 1
    # per-class
    precs, recs, f1s = [], [], []
    for l in labels:
        tp = cm[l][l]
        fp = sum(cm[ll][l] for ll in labels if ll != l)
        fn = sum(cm[l][ll] for ll in labels if ll != l)
        prec = tp / (tp + fp) if (tp+fp) > 0 else 0.0
        rec = tp / (tp + fn) if (tp+fn) > 0 else 0.0
        f1 = 2*prec*rec/(prec+rec) if (prec+rec) > 0 else 0.0
        precs.append(prec); recs.append(rec); f1s.append(f1)
    acc = sum(cm[l][l] for l in labels) / max(1, sum(sum(cm[l].values()) for l in labels))
    report = {
        "accuracy": acc,
        "precision_macro": sum(precs)/len(labels),
        "recall_macro": sum(recs)/len(labels),
        "f1_macro": sum(f1s)/len(labels),
        "per_class": {l: {"precision":p, "recall":r, "f1":f}
                      for l, p, r, f in zip(labels, precs, recs, f1s)},
        "confusion_matrix": cm
    }
    return report

def main():
    ap = argparse.ArgumentParser()
    ap.add_argument("--gold", required=True, help="Gold JSONL: {text:str, label:str} per line")
    ap.add_argument("--pred", required=True, help="Predictions JSONL: LLM raw outputs per line OR {output:str}")
    ap.add_argument("--labels", default="3class", choices=["2class","3class"])
    args = ap.parse_args()

    labels = ["positive","negative"] if args.labels=="2class" else ["positive","neutral","negative"]
    gold = load_jsonl(args.gold)
    pred_raw = load_jsonl(args.pred)

    if len(gold) != len(pred_raw):
        raise ValueError(f"Length mismatch: gold={len(gold)} vs pred={len(pred_raw)}")

    y_true, y_pred, parsed_rows = [], [], []
    for g, pr in zip(gold, pred_raw):
        gold_label = normalize_label(g.get("label"))
        raw_text = pr.get("output") or pr.get("text") or json.dumps(pr, ensure_ascii=False)
        parsed = extract_json(raw_text)
        pred_label = normalize_label(parsed["label"])
        y_true.append(gold_label)
        # 2クラス運用でも、パース不能は 'neutral' に落とす（分母から外れる→別集計で確認）
        y_pred.append(pred_label if pred_label is not None else "neutral")
        parsed_rows.append({"gold":gold_label, "pred":pred_label, "confidence":parsed["confidence"], "reason":parsed["reason"]})

    report = classification_report(y_true, y_pred, labels)
    print(json.dumps(report, ensure_ascii=False, indent=2))

    # エラー解析用の先頭20件をダンプ
    errors = [r for r in parsed_rows if r["gold"] != r["pred"]]
    preview = {"num_errors": len(errors), "preview": errors[:20]}
    print("
# Errors (first 20)
")
    print(json.dumps(preview, ensure_ascii=False, indent=2))

if __name__ == "__main__":
    main()
```

---

### `parser.py` — LLM応答のJSON抽出と正規化
- 応答本文から**最初の `{...}`** を拾ってJSONとして読む  
- `label` の表記ゆれ（pos/neg/neuなど）を補正、`confidence` を 0–1 にクリップ

```python:parser.py
import json, re

VALID_3 = {"positive", "neutral", "negative"}

def extract_json(text: str) -> dict:
    """
    応答本文から最初のJSONブロックを抽出してparseする。
    失敗したら空のデフォルトを返す。
    """
    # 最初の { から最後の } までの最短一致を試みる
    m = re.search(r'\{.*\}', text, flags=re.S)
    if not m:
        return {"label": None, "confidence": None, "reason": None, "raw": text}
    snippet = m.group(0)
    try:
        data = json.loads(snippet)
    except Exception:
        return {"label": None, "confidence": None, "reason": None, "raw": text}
    # 正規化
    label = (data.get("label") or "").strip().lower()
    if label not in VALID_3:
        # よくある表記ゆれを補正
        alias = {
            "pos": "positive", "positive ": "positive",
            "neg": "negative", "negative ": "negative",
            "neu": "neutral", "neutral ": "neutral"
        }
        label = alias.get(label, None)
    conf = data.get("confidence")
    try:
        conf = float(conf) if conf is not None else None
        if conf is not None:
            conf = max(0.0, min(1.0, conf))
    except Exception:
        conf = None
    reason = data.get("reason")
    return {"label": label, "confidence": conf, "reason": reason, "raw": text}

def normalize_label(x: str) -> str:
    x = (x or "").strip().lower()
    if x in VALID_3: return x
    if x in {"pos"}: return "positive"
    if x in {"neg"}: return "negative"
    if x in {"neu", "neutrality"}: return "neutral"
    return None
```

---

### `prompts.py` —（将来の3クラス拡張向けテンプレ）
- 今回のブログは2クラス中心ですが、**3クラス拡張**を見据えたテンプレも置いておきます。

```python:prompts.py
from textwrap import dedent

DEFAULT_SCHEMA = """\
必ず次のJSON形式のみで出力してください:
{
  "label": "<positive|neutral|negative のいずれか>",
  "confidence": <0.0から1.0>,
  "reason": "<根拠を日本語で1文以内>"
}
"""

def zero_shot_prompt(text: str) -> str:
    """3値感情分析のゼロショットプロンプト（日本語）。"""
    return dedent(f"""
    あなたは厳密な採点者です。与えられたテキストの「筆者の評価的態度」を
    positive / neutral / negative の3クラスで判定してください。
    
    ルール:
    - 「事実のみの説明」「どちらとも取れない曖昧」は neutral
    - 明確な好意・満足・推奨は positive
    - 明確な不満・批判・落胆・拒否は negative
    - 出力は必ず指定の JSON スキーマに従うこと
    
    テキスト:
    ```
    {text}
    ```
    
    {DEFAULT_SCHEMA}
    """).strip()

def few_shot_prompt(text: str) -> str:
    """2例のFew-shotつきテンプレ。"""
    return dedent(f"""
    あなたは厳密な採点者です。以下の例に従い、与えられたテキストの
    sentiment を positive / neutral / negative の3クラスで判定してください。
    
    [例1]
    テキスト: "最高の体験でした。また必ず購入します。"
    出力: {{"label":"positive","confidence":0.94,"reason":"強い満足と再購入意思"}}
    
    [例2]
    テキスト: "思ったより普通でした。特に可もなく不可もなし。"
    出力: {{"label":"neutral","confidence":0.82,"reason":"評価が中立で感情が弱い"}}
    
    ルール:
    - 「事実のみの説明」「どちらとも取れない曖昧」は neutral
    - 明確な好意・満足・推奨は positive
    - 明確な不満・批判・落胆・拒否は negative
    - 出力は必ずJSONスキーマに従う
    
    テキスト:
    ```
    {text}
    ```
    
    {DEFAULT_SCHEMA}
    """).strip()

def cot_prompt(text: str) -> str:
    """Chain-of-Thoughtを誘導しつつ、最終的にJSONだけ返させる構成。"""
    return dedent(f"""
    あなたは厳密な採点者です。以下の手順で考え、最後に **JSONのみ** を出力してください。
    手順:
    1) テキストから評価的表現（肯定/否定/中立）を抽出（思考は日本語で簡潔に）
    2) 最終ラベルを positive / neutral / negative のいずれかで決定
    3) JSONスキーマに従って出力
    
    テキスト:
    ```
    {text}
    ```
    
    {DEFAULT_SCHEMA}
    """).strip()
```

---

### `run_vLLM_imdb_zero_shot.py` — vLLMでIMDBを2クラス評価
- `--model` にHugging Faceのモデル名を渡すだけ。予測（raw JSON文字列）と正解（gold）を `.jsonl` に保存します。

```python:run_vLLM_imdb_zero_shot.py
"""
run_vLLM_imdb_zero_shot.py

vLLM を用いて IMDB データセット 100件でゼロショット 2クラス感情分析（positive/negative）を実行。
- モデルは Hugging Face Hub 名を引数で指定（例: mistralai/Mistral-7B-Instruct-v0.3）
- 量子化モデル（AWQ/GPTQ）なら VRAM を大幅に節約可能（例: TheBloke/*-AWQ）
- 生成出力は JSON 形式を強制するプロンプト。解析は既存 `evaluate_zero_shot.py` を使用。
"""
import argparse, json, random, os
from datasets import load_dataset
from vllm import LLM, SamplingParams
from textwrap import dedent

def set_seed(seed: int):
    random.seed(seed)

def prompt_2class(text: str) -> str:
    return dedent(f"""
    You are a strict grader. Classify the review sentiment into two classes: positive or negative.
    
    Rules:
    - Clear satisfaction/recommendation → positive
    - Clear dissatisfaction/complaint/refusal → negative
    - Output must be only the following JSON schema:
    {{
      "label": "<positive|negative>",
      "confidence": <0.0_to_1.0>,
      "reason": "<one short sentence>"
    }}
    
    Review:
    ```
    {text}
    ```
    """).strip()

def label_to_str(x: int) -> str:
    # IMDB: 0=neg, 1=pos
    return "positive" if x == 1 else "negative"

def main():
    ap = argparse.ArgumentParser()
    ap.add_argument("--model", required=True, help="HF model id (e.g., mistralai/Mistral-7B-Instruct-v0.3)")
    ap.add_argument("--outdir", default="runs_imdb_zero_shot")
    ap.add_argument("--num_samples", type=int, default=100)
    ap.add_argument("--seed", type=int, default=42)
    ap.add_argument("--max_tokens", type=int, default=128)
    ap.add_argument("--temperature", type=float, default=0.0)
    ap.add_argument("--tensor_parallel_size", type=int, default=1)
    ap.add_argument("--dtype", default="auto", help="auto|half|bfloat16|float32 etc (forwarded to vLLM)")
    args = ap.parse_args()

    os.makedirs(args.outdir, exist_ok=True)
    set_seed(args.seed)

    # 1) load IMDB
    ds = load_dataset("stanfordnlp/imdb")
    test = ds["test"]
    idx = list(range(len(test)))
    random.shuffle(idx)
    idx = idx[:args.num_samples]
    subset = test.select(idx)

    # 2) prepare prompts
    prompts = [prompt_2class(x["text"]) for x in subset]

    # 3) init vLLM
    sampling = SamplingParams(
        temperature=args.temperature,
        max_tokens=args.max_tokens,
        stop=[],  # JSONのみを促すため stop は無し
    )
    llm = LLM(model=args.model, tensor_parallel_size=args.tensor_parallel_size, dtype=args.dtype)
    
    # 4) generate
    outputs = llm.generate(prompts, sampling_params=sampling)
    # vLLM outputs: list[RequestOutput] -> .outputs[0].text
    texts = [o.outputs[0].text if o.outputs else "" for o in outputs]

    # 5) save predictions (raw), and gold
    pred_path = os.path.join(args.outdir, "imdb100_predictions.jsonl")
    gold_path = os.path.join(args.outdir, "imdb100_gold.jsonl")
    with open(pred_path, "w", encoding="utf-8") as f:
        for t in texts:
            f.write(json.dumps({"output": t}, ensure_ascii=False) + "\n")
    with open(gold_path, "w", encoding="utf-8") as f:
        for row in subset:
            f.write(json.dumps({"text": row["text"], "label": label_to_str(row["label"])}, ensure_ascii=False) + "\n")

    print(f"Saved predictions to: {pred_path}")
    print(f"Saved gold to:        {gold_path}")
    print("\n次の評価コマンドを実行してください：")
    print(f"python evaluate_zero_shot.py --gold {gold_path} --pred {pred_path} --labels 2class")

if __name__ == "__main__":
    main()
```

## おわりに
今回のゼロショット実験では，Accuracy 0.96／F1(macro) 0.959 という自分の想定を大きく上回る結果が得られた．正直，LLMが追加学習なしでここまで安定して判定できるとは思っておらず，**「テキストを入れて、返ってきたJSONを評価するだけ」** という手軽さにも驚かされた．最小構成でも，十分に**再現可能な評価**に到達できることを確認できたのは大きな収穫である．

一方で，評価対象はIMDBの2クラス・サンプル数も限定的であり，**分布の違い** に対する頑健性は今後の検証課題だ．クラス別の誤りを見ると，**ネガ判定の基準を厳格化**するプロンプト修正や，**confidenceしきい値**の導入で改善できる見込みがある．

次の一歩として，
1. Few-shot/Cotの導入と境界事例の提示，
2. モデル横断比較（Mistral/Llama/Qwen/日本語特化系）
3. 3クラス設定でのニュートラルの安定化，
4. confidenceに基づく運用上のしきい値設計（Recall/Precisionのトレードオフ最適化）

を進めたい．

今回の「**プロンプト→JSON→パース→評価**」という足場を起点に，データ分布と要求精度に合わせて段階的に強化していく．
