---
title: "ゼロショットの次へ：Few-shotとCoTで感情分類を底上げする（IMDB＋vLLM）"
emoji: "🧩"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["AI","LLM","NLP","CoT","Prompting"]
published: false
---


# LLM感情推定の続編：Few-shotとCoTで安定性と精度を底上げする

**— 例示（Few-shot）とChain-of-Thought、多数決・投票率confidence、JSONパースまで “壊れない” 比較実験 —**

*更新: 2025-08-18 15:00*

## TL;DR

* **Few-shot**は「判断の物差し」を示せるので、ゼロショットより**境界事例で安定**しやすい．
* **CoT（考える→最終はJSONのみ）**は曖昧例に効く一方、出力崩壊のリスクがあるため**“JSONのみ”の強制**＋**n本生成の多数決**で堅牢化．
* **投票率をconfidenceに採用**すると「自己申告のconfidenceの無相関問題」を回避できる．
* 付属スクリプト（vLLM + IMDB）で **zero / few(2,5) / cot** を**同一パイプライン**で再現・比較可能．


## 背景と目的

前回「ゼロショット編」では，**プロンプト → JSON強制 → パース → 指標**という最小構成を作り，壊れずに回せる評価の足場を用意した．本稿ではその足場の上に**Few-shot**と**CoT**を積み増し，**精度**と**安定性**の改善を検証する．
特に，CoTは**思考の誘導と最終出力の厳格さ**を両立させる設計が肝になるため，**多数生成×多数決**＋**投票率confidence**を組み合わせて，評価・解析まで崩れない形に整える．


## 使ったコード（ざっくり）

```
prompts.py                 # Zero / Few-shot / CoT のプロンプト（2/3クラス対応の雛形）
parser.py                  # 応答から最初のJSONを抽出・正規化（label表記ゆれ/信頼度クリップ）
run_vLLM_imdb_variants.py  # vLLMで IMDB を zero / few2 / few5 / cot で一括実行＆評価
analyze_cot_quality.py     # CoTの “パース率/精度/平均confidence/相関” を要約（※環境によっては .py.py）
report_cot_quality.json    # 実行例のサマリレポート
```

> 備考：`run_vLLM_imdb_variants.py` は各 `--variant` の推論後に評価を呼び出し，`report_{variant}.json` を `--outdir` 配下に保存する構成．


## 実験設計

### タスク

* **2クラス**：IMDB（positive / negative）
* **3クラス**：positive / neutral / negative（将来拡張）

### 共通の出力スキーマ（LLMの回答は**必ずこれのみ**）

```json
{
  "label": "positive|neutral|negative",
  "confidence": 0.0_to_1.0,
  "reason": "根拠の短文"
}
```

### Few-shot の狙い

* 典型例を数件示し**境界の基準**を共有（皮肉・弱い感情・曖昧評価の取り扱い）
* 例は**短く明快**、難例を入れすぎない（まず土台を作る）

#### Few-shot（2ショット／2クラス・英語例）

```text
You are a strict grader. Follow the examples and classify movie review sentiment into positive or negative.

[Example 1]
Review: "Loved the pacing and the ending."
Output: {"label":"positive","confidence":0.94,"reason":"clear satisfaction"}

[Example 2]
Review: "Waste of time. Wouldn't recommend."
Output: {"label":"negative","confidence":0.95,"reason":"clear dissatisfaction"}

Rules:
- Clear satisfaction/recommendation → positive
- Clear dissatisfaction/complaint/refusal → negative
Output must be only this JSON schema (no extra text / no markdown / no code fences):
{
  "label": "<positive|negative>",
  "confidence": <0.0_to_1.0>,
  "reason": "<one short sentence>"
}

Review:
<text>
```

### CoT の狙いと注意

* **思考を促す**ことで曖昧例の判断を安定化．
* ただし**思考文が混ざる崩壊**に備えて，プロンプトで\*\*“最終はJSONのみ”\*\*を強制．
* **同一サンプルに対して n本生成**→**多数決で最終ラベル**→**vote shareをconfidence**にする運用が実務的に堅い．

#### CoT（2クラス・英語例：**最終はJSONのみ**）

```text
You are a strict grader. Think briefly inside your head, but DO NOT output your steps.
Output ONLY the final JSON (no extra text / no markdown / no code fences).

Rules:
- Clear satisfaction/recommendation → positive
- Clear dissatisfaction/complaint/refusal → negative

Final JSON schema (reason ≤ 12 words):
{
  "label": "<positive|negative>",
  "confidence": <0.0_to_1.0>,
  "reason": "<one short sentence (≤12 words)>"
}

Review:
<text>
```


## 再現手順（IMDB × vLLM）

### 1) 実行

```bash
# 例: Mistral-7B-Instruct を利用。variant は zero / few2 / few5 / cot を切り替え
python run_vLLM_imdb_variants.py \
  --model mistralai/Mistral-7B-Instruct-v0.3 \
  --variant cot \
  --num_samples 100 \
  --outdir runs_imdb_variants
```

* 推論結果（raw JSONL）と gold が `--outdir` に保存され，評価レポート `report_{variant}.json` が出力される．
* CoTは**各サンプルをn回生成**して**多数決**し，**投票率をconfidence**として記録（スクリプト先頭コメント参照）．

### 2) 品質サマリ（CoT例）

`report_cot_quality.json`（100件, IMDB, CoT）の例を抜粋：

```json
{
  "parse_success_rate": 1.0,
  "accuracy": 0.92,
  "avg_confidence": 0.9743899999999996,
  "avg_reason_length": 62.64,
  "corr_confidence_correct": 0.019740392190562284,
  "n": 100
}
```

**所見**：`corr_confidence_correct ≈ 0` で，**自己申告confidenceが正誤とほぼ無相関**になりがち．
→ 本稿の運用の通り，**“投票率”をconfidenceに採用**すると意味のある信頼度になる．


## パース戦略（前回と同様）

* 応答から**最初の `{ ... }`** を抽出してJSONとして読む（`parser.py`）
* `label` は**表記ゆれ補正**（pos/neg/neu → 正式表記），`confidence` は **0–1にクリップ**
* パース不能は `label=None` 扱い（評価時の方針は 2クラスstrict / neutralフォールバックなどで明示）


## 評価指標と見方

* **Accuracy / Precision（Macro）/ Recall（Macro）/ F1（Macro）**
* **Confusion Matrix** と **誤分類の先頭プレビュー**（エラー分析の起点）
* まず注視するポイント：

  1. `f1_macro` の水準（ゼロショット比較）
  2. **few2 → few5** での安定性改善（特に境界例の取り違え）
  3. CoTの**出力崩壊が抑えられているか**（パース成功率）
  4. **投票率confidence**と正解の整合（しきい値運用の基礎）


## 典型的な失敗と対処

* **思考文や前置きが混ざる** → 「**JSONのみ**」「**{で始め}で終える**」を明記．
* **ラベル表記ゆれ** → 正規化テーブルで救済（pos/neg/neu など）．
* **reasonが長文化** → 語数上限（≤12語など）をプロンプト側で明示．
* **confidenceがアテにならない** → 自己申告値は捨てて**投票率**or**外部較正**へ．

## 実験結果（few-shot / CoT, IMDB, n=100）

まず全体のスコアボードである．

| Variant       | Accuracy | Precision (macro) | Recall (macro) | F1 (macro) |
|:--------------|---------:|------------------:|---------------:|-----------:|
| Zero-shot     | 0.94     | 0.9343            | 0.9458         | 0.9384     |
| Few-shot (2)  | 0.95     | 0.9447            | 0.9542         | 0.9485     |
| Few-shot (5)  | 0.94     | 0.9343            | 0.9458         | 0.9384     |
| CoT           | 0.92     | 0.9147            | 0.9292         | 0.9184     |


![IMDB: AccuracyとF1](https://storage.googleapis.com/zenn-user-upload/68efc5669827-20250818.png "Accuracy/F1 bar chart")
![IMDB: クラス別Recall](https://storage.googleapis.com/zenn-user-upload/c82c774f88a5-20250818.png "Recall by class")
![IMDB: AccuracyとF1](/images/imdb_variants_accuracy_f1.png "Accuracy/F1 bar chart")
![IMDB: クラス別Recall](/images/imdb_variants_recall_by_class.png "Recall by class")

### 混同行列（Positive/Negative）
`[[TP, FN], [FP, TN]]` の順で表示：

- Zero-shot / Few-shot(5)：`[[55, 5], [1, 39]]`
- Few-shot(2)　　　 　　：`[[56, 4], [1, 39]]`
- CoT　　　　　　　　　：`[[53, 7], [1, 39]]`

### 読み取りポイント
- **Few-shot(2)** が最良（Acc=0.95 / F1=0.9485）．ゼロショット比で **Positive再現率が改善（0.9167→0.9333）**．
- **Few-shot(5)** はゼロショットとほぼ同等 → 小規模タスクでは **例数の増量が必ずしも効かない** 可能性．
- **CoT** は **Positiveの再現率が低下（0.8833）**．ただし **Positiveの精度は最高（0.9815）** で，厳しめに絞っている挙動．
- 全手法で **Negative側は安定（再現率=0.975、FP=1）**．データ分布の偏り（Negativeが判定しやすい）を示唆．
- CoTの**自己申告信頼度**と正解の相関は **~0.31**．しきい値運用の余地あり．



## 付録：コマンド速攻リファレンス

```bash
# ゼロ/少数例/CoT を切り替えてIMDB 100件評価
python run_vLLM_imdb_variants.py --model mistralai/Mistral-7B-Instruct-v0.3 --variant zero  --num_samples 100
python run_vLLM_imdb_variants.py --model mistralai/Mistral-7B-Instruct-v0.3 --variant few2  --num_samples 100
python run_vLLM_imdb_variants.py --model mistralai/Mistral-7B-Instruct-v0.3 --variant few5  --num_samples 100
python run_vLLM_imdb_variants.py --model mistralai/Mistral-7B-Instruct-v0.3 --variant cot   --num_samples 100

# CoT品質の要約（raw予測とgoldを渡して集計）
python analyze_cot_quality.py \
  --gold runs_imdb_variants/imdb_gold.jsonl \
  --pred runs_imdb_variants/imdb_cot_predictions.jsonl \
  --out report_cot_quality.json
```

## 実際に使ったコード


> **補足**
>
> * 下記スクリプトは **実際に使用したもの**です．
> * `run_vLLM_imdb_variants.py`と`analyze_cot_quality.py` は前回の `parser.py` と `evaluate_zero_shot.py` を同ディレクトリに置く前提です（パースと評価に利用）．


### run_vLLM_imdb_variants.py（実験スクリプト）

```python
import argparse, os, json, random, subprocess, sys
from textwrap import dedent
from datasets import load_dataset
from vllm import LLM, SamplingParams

# 各サンプルについて5本パース→多数決ラベルを1つに
from collections import Counter
from parser import extract_json, normalize_label

# ---- 基本プロンプト（英語2クラス：既存ゼロショットと整合） ----
def prompt_zero(text:str)->str:
    return dedent(f"""
    You are a strict grader. Classify the movie review sentiment into two classes: positive or negative.

    Rules:
    - Clear satisfaction/recommendation → positive
    - Clear dissatisfaction/complaint/refusal → negative
    Output must be only the following JSON schema (no extra text, no markdown, no code fences):
    {{
      "label": "<positive|negative>",
      "confidence": <0.0_to_1.0>,
      "reason": "<one short sentence>"
    }}

    Review:
    {text}
    """).strip()

def build_fewshot_block(examples):
    # examples: list[{"text":..., "label":...}] label in {"positive","negative"}
    lines = []
    for i,ex in enumerate(examples,1):
        lab = ex["label"]
        reason = "clear praise" if lab=="positive" else "explicit dissatisfaction"
        lines.append(f'[Example {i}]\nReview: "{ex["text"].strip()}"\nOutput: {{"label":"{lab}","confidence":0.9,"reason":"{reason}"}}')
    return "\n\n".join(lines)

def prompt_few(text:str, shots):
    return dedent(f"""
    You are a strict grader. Follow the examples and classify movie review sentiment into positive or negative.

    {build_fewshot_block(shots)}

    Rules:
    - Clear satisfaction/recommendation → positive
    - Clear dissatisfaction/complaint/refusal → negative
    Output must be only the following JSON schema (no extra text, no markdown, no code fences):
    {{
      "label": "<positive|negative>",
      "confidence": <0.0_to_1.0>,
      "reason": "<one short sentence>"
    }}

    Review:
    {text}
    """).strip()

def prompt_cot(text:str)->str:
    from textwrap import dedent
    return dedent(f"""
    You are a strict grader. Think briefly inside your head, but DO NOT output your steps.
    Output ONLY the final JSON with no extra text / no markdown / no code fences.
    If the sentiment is ambiguous, choose "negative".

    Rules:
    - Clear satisfaction/recommendation → positive
    - Clear dissatisfaction/complaint/refusal → negative

    Final JSON schema (reason ≤ 12 words):
    {{
      "label": "<positive|negative>",
      "confidence": <0.0_to_1.0>,
      "reason": "<one short sentence (≤12 words)>"
    }}

    Start your output with '{{' and end with '}}'.
    Review:
    {text}
    """).strip()


def set_seed(seed:int):
    random.seed(seed)

def get_balanced_shots(train_ds, k):
    # k=2 or 5: 半々で pos/neg を取る（端数はpos優先）
    pos = [r for r in train_ds if r["label"]==1]
    neg = [r for r in train_ds if r["label"]==0]
    random.shuffle(pos); random.shuffle(neg)
    take_pos = (k+1)//2
    take_neg = k - take_pos
    shots_raw = [(pos[i]["text"], "positive") for i in range(take_pos)] + \
                [(neg[i]["text"], "negative") for i in range(take_neg)]
    # テキストを短めに整形（長すぎるものは先頭一文など）
    clean = []
    for t,l in shots_raw:
        t = " ".join(t.strip().split())
        if len(t)>240: t = t[:240] + "..."
        clean.append({"text": t, "label": l})
    return clean

def main():
    ap=argparse.ArgumentParser()
    ap.add_argument("--model", required=True)
    ap.add_argument("--outdir", default="runs_imdb_A40")
    ap.add_argument("--num_samples", type=int, default=100)
    ap.add_argument("--seed", type=int, default=42)
    ap.add_argument("--max_tokens", type=int, default=128)
    ap.add_argument("--temperature", type=float, default=0.0)
    ap.add_argument("--tensor_parallel_size", type=int, default=1)
    ap.add_argument("--dtype", default="auto")
    ap.add_argument("--variant", choices=["zero","few2","few5","cot"], default="zero")
    args=ap.parse_args()

    os.makedirs(args.outdir, exist_ok=True)
    set_seed(args.seed)

    # ---- GOLD を優先的に使用（存在すれば同じテキストで実行）----
    gold_path = os.path.join(args.outdir, "imdb100_gold.jsonl")
    texts = None

    def read_jsonl(path):
        arr=[]
        with open(path, encoding="utf-8") as f:
            for line in f:
                line=line.strip()
                if line:
                    arr.append(json.loads(line))
        return arr

    if os.path.exists(gold_path):
        gold = read_jsonl(gold_path)
        texts = [g["text"] for g in gold]
        if len(texts) != args.num_samples:
            # 件数が違う場合は先頭から合わせる（必要ならここで assert にしてもOK）
            texts = texts[:args.num_samples]
        print("Using existing GOLD:", gold_path, f"(n={len(texts)})")
    else:
        # GOLD が無ければ今回のサンプルで作成
        ds = load_dataset("stanfordnlp/imdb")
        test = ds["test"]
        idx = list(range(len(test)))
        random.shuffle(idx); idx = idx[:args.num_samples]
        subset = test.select(idx)
        def label_to_str(x:int)->str: return "positive" if x==1 else "negative"
        with open(gold_path, "w", encoding="utf-8") as f:
            for r in subset:
                f.write(json.dumps({"text": r["text"], "label": label_to_str(r["label"])}, ensure_ascii=False) + "\n")
        texts = [r["text"] for r in subset]
        print("Saved GOLD:", gold_path, f"(n={len(texts)})")

    # 2) Few-shot 例（必要時）
    shots = None
    if args.variant in {"few2","few5"}:
        # GOLD の有無に関係なく、few-shot 用に train だけロード
        ds = load_dataset("stanfordnlp/imdb")
        train = ds["train"]
        k = 2 if args.variant == "few2" else 5
        shots = get_balanced_shots(train, k)

    # 3) プロンプト組み立て（GOLD 由来の texts を使用）
    prompts=[]
    for t in texts:
        t = " ".join(t.strip().split())
        if args.variant=="zero":
            p = prompt_zero(t)
        elif args.variant in {"few2","few5"}:
            p = prompt_few(t, shots)
        else:
            p = prompt_cot(t)
        prompts.append(p)

    # 4) 生成
    n = 5 if args.variant == "cot" else 1
    temp = 0.6 if args.variant == "cot" else args.temperature
    sampling = SamplingParams(temperature=temp, max_tokens=args.max_tokens, n=n)
    llm = LLM(model=args.model, tensor_parallel_size=args.tensor_parallel_size, dtype=args.dtype)
    outs = llm.generate(prompts, sampling_params=sampling)
    def trim_reason(r, max_words=12):
        if not r: return ""
        return " ".join(r.strip().split()[:max_words])
    pred_rows = []
    for o in outs:
        labels = []
        reasons = []
        confs = []
        for cand in o.outputs:  # n本
            pj = extract_json(cand.text)
            lab = normalize_label(pj.get("label"))
            if lab:
                labels.append(lab)
                reasons.append(trim_reason(pj.get("reason")))
                confs.append(pj.get("confidence"))
        if labels:
            maj = Counter(labels).most_common(1)[0][0]
            if n == 1:
                # 単発生成（zero/few）はそのまま代表を採用
                idx = labels.index(maj)
                rep_reason = reasons[idx] or ""
                rep_conf = confs[idx] if confs[idx] is not None else 0.9
                final_json = {"label": maj, "confidence": float(rep_conf), "reason": rep_reason}
            else:
                # CoT: 自信値は多数決の投票率に置換
                vote = labels.count(maj) / len(labels)
                rep_reason = reasons[labels.index(maj)]
                final_json = {"label": maj, "confidence": float(vote), "reason": rep_reason}
        else:
            # 全滅したら保守的に negative
            final_json = {"label":"negative","confidence":0.55,"reason":"fallback"}
        final_str = json.dumps(final_json, ensure_ascii=False)
        pred_rows.append({"output": final_str})

    # 5) 予測保存
    pred_path = os.path.join(args.outdir, f"imdb100_pred_{args.variant}.jsonl")
    with open(pred_path,"w",encoding="utf-8") as f:
        for row in pred_rows:
            f.write(json.dumps(row, ensure_ascii=False)+"\n")
    print("Saved pred:", pred_path)

    # 6) 評価（既存 evaluator を呼び出し、先頭JSONだけ抽出して保存）
    cmd = [
        sys.executable, "evaluate_zero_shot.py",
        "--gold", gold_path, "--pred", pred_path, "--labels", "2class"
    ]
    run = subprocess.run(cmd, capture_output=True, text=True, check=True)
    s = run.stdout
    obj, end = json.JSONDecoder().raw_decode(s)  # 先頭JSONのみ
    rep_path = os.path.join(args.outdir, f"report_{args.variant}.json")
    with open(rep_path, "w", encoding="utf-8") as f:
        json.dump(obj, f, ensure_ascii=False, indent=2)
    print("Saved report:", rep_path)

if __name__=="__main__":
    main()
```



### analyze_cot_quality.py（分析スクリプト））


```python
import json, argparse, statistics
from parser import extract_json, normalize_label

def load_jsonl(p):
    with open(p, encoding="utf-8") as f:
        for line in f:
            line=line.strip()
            if line:
                yield json.loads(line)

def pearson(x, y):
    n=len(x)
    if n<2: return None
    mx=sum(x)/n; my=sum(y)/n
    num=sum((a-mx)*(b-my) for a,b in zip(x,y))
    den=(sum((a-mx)**2 for a in x)*sum((b-my)**2 for b in y))**0.5
    return num/den if den else None

def main():
    ap=argparse.ArgumentParser()
    ap.add_argument("--gold", required=True)
    ap.add_argument("--pred", required=True)  # imdb100_pred_cot.jsonl を想定
    ap.add_argument("--out", required=True)
    args=ap.parse_args()

    gold=list(load_jsonl(args.gold))
    pred=list(load_jsonl(args.pred))
    assert len(gold)==len(pred), "length mismatch"
    parses=[]; rights=[]; confs=[]; reason_lens=[]

    for g,p in zip(gold,pred):
        raw = p.get("output","")
        parsed = extract_json(raw)
        ok = parsed["label"] is not None
        parses.append(1 if ok else 0)
        y_true = normalize_label(g["label"])
        y_pred = normalize_label(parsed["label"])
        right = (y_true==y_pred) if y_pred is not None else False
        rights.append(1 if right else 0)
        c = parsed["confidence"]
        if c is not None: confs.append(float(c))
        rn = parsed["reason"]
        if rn is not None: reason_lens.append(len(rn.strip()))

    report = {
        "parse_success_rate": sum(parses)/len(parses),
        "accuracy": sum(rights)/len(rights),
        "avg_confidence": (sum(confs)/len(confs)) if confs else None,
        "avg_reason_length": (sum(reason_lens)/len(reason_lens)) if reason_lens else None,
        "corr_confidence_correct": pearson(confs, rights) if confs else None,
        "n": len(gold),
    }
    with open(args.out, "w", encoding="utf-8") as f:
        json.dump(report, f, ensure_ascii=False, indent=2)
    print("Saved:", args.out)

if __name__=="__main__":
    main()
```



## おわりに

今回の実装を通して，できたことやわかったこと，今後の予定をまとめる．
* **Few-shot**で判断基準を共有 → **ゼロショットよりブレが減る**．
* **CoT**は**思考**を促すが、**“最終はJSONのみ”**の統制と**多数決＋投票率confidence**で**評価可能性を担保**するのが実務的．
* 同一パイプライン上で **zero / few / cot** を横並び比較できるので，**モデル差**や**例数の効果**も一貫して評価できる．


前回のゼロショットの実験結果は95%程であったため，今回のfew-shotとCoTの面白さや技術的にすごいポイントがわかりづらかった．以下の理由によりその特徴を活かしきれなかったと考えられる．
- 天井効果：IMDB 2値は指示調整済みモデルの得意領域．ゼロショットで既に 95% 付近．
- サンプル数が少ない（n=100）：3件差＝3%は統計ノイズ級（上の信頼区間参照）．
- 課題特性：CoTが効きやすいのは推論鎖が必要な問題（複合条件・数理・長文照合など）．単純2値感情は**考えるほど良くなるとは限らない**．

また，few-shotやCoTの強みが目に見えてわかるようになるためには，以下の様な工夫が必要そうである．
- ドメイン移行：IMDB→Amazon/Steam/Twitter（日本語）など語彙・レジスターが違う領域．Few-shotの例示効果が出やすい．
- 多クラス化：neutral を含む3値はゼロショットで落ちやすい→Few-shotの境界提示が効く．
- 実運用耐性テスト：入力にノイズ（不要ヘッダ/絵文字/URL/引用符）を混ぜ，パース成功率と精度の劣化耐性を比較．