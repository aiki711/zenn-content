---
title: "BERTモデルを日本語レビューでファインチューニングする：Amazonレビュー × 東北BERT v3"
emoji: "😎"
type: "tech"
topics: ["AI","NLP","BERT","Fine-tuning","Sentiment"]
published: true
---

# TL;DR
- **東北BERT v3（SentencePiece）** を使って日本語Amazonレビュー（3クラス：pos/neu/neg）を**最小構成**で学習・評価  
- 出力は **Accuracy / Precision / Recall / F1（Macro）/ Confusion Matrix**。誤分類の高確信例も自動で抽出  
- HPC（OpenPBS / A40）でも**そのまま再現**できるように、キャッシュ固定・ヘッドレス描画・PBSジョブ例を同梱  
- Transformers の**バージョン差（`evaluation_strategy` vs `eval_strategy`）**にも耐える安全レシピを提示


# 背景と目的
ゼロショットの評価系に続いて（別記事参照），今回は **事前学習済みBERTのファインチューニング**で日本語の感情分類をやってみる．  
再現性を重視し，**環境→データ→前処理→学習→可視化→評価→エラー分析**が崩れない“最短動線”を意識した．HPC環境でもフォルダ汚染せずに動くよう，**作業ディレクトリ直下**に成果物とキャッシュを固定している．


# 実験設定（ざっくり）
- **データ**：SetFit/amazon_reviews_multi_ja（星1–5 → 1–2=neg, 3=neu, 4–5=pos の3値化）
- **モデル**：`cl-tohoku/bert-base-japanese-v3`（SentencePiece。**fugashi不要**）
- **評価**：Accuracy / Precision / Recall / F1（Macro）＋混同行列、誤分類トップ例のダンプ
- **出力先**：`./figs`, `./reports`, `./outputs-ja-bert`, `./.cache`, `./tmp`（すべて**カレント直下**）


# 再現手順（最短）
## 1) 環境構築（conda の例）
```bash
conda create -n nlp-sft python=3.10 -y
conda activate nlp-sft

# GPU 版 PyTorch（CUDA 12.4 の例。CPUのみなら index-url を cpu に）
python -m pip install --index-url https://download.pytorch.org/whl/cu124 torch torchvision torchaudio

# NLP一式
python -m pip install -U "transformers>=4.41" "datasets>=2.19" "accelerate>=0.30" \
  evaluate scikit-learn matplotlib sentencepiece "protobuf<5"
````

## 2) 学習スクリプト（そのまま保存して実行）

> ファイル名：`run_bert_amazon.py`

```python
import os, matplotlib
matplotlib.use("Agg")  # ヘッドレス描画

# ====== 作業ディレクトリ直下に固定 ======
BASE = os.path.abspath(os.getcwd())
os.environ.setdefault("HF_HOME",            os.path.join(BASE, ".cache", "hf"))
os.environ.setdefault("HF_HUB_CACHE",       os.path.join(BASE, ".cache", "hub"))
os.environ.setdefault("TRANSFORMERS_CACHE", os.path.join(BASE, ".cache", "transformers"))
os.environ.setdefault("HF_DATASETS_CACHE",  os.path.join(BASE, ".cache", "datasets"))
os.environ.setdefault("TMPDIR",             os.path.join(BASE, "tmp"))
for d in [os.environ["HF_HOME"], os.environ["HF_HUB_CACHE"], os.environ["TRANSFORMERS_CACHE"],
          os.environ["HF_DATASETS_CACHE"], os.environ["TMPDIR"]]:
    os.makedirs(d, exist_ok=True)

FIGS_DIR, REPORTS_DIR = os.path.join(BASE, "figs"), os.path.join(BASE, "reports")
OUT_DIR = os.path.join(BASE, "outputs-ja-bert")
os.makedirs(FIGS_DIR, exist_ok=True); os.makedirs(REPORTS_DIR, exist_ok=True)

# ====== データ ======
from datasets import load_dataset
ds = load_dataset("SetFit/amazon_reviews_multi_ja", cache_dir=os.environ["HF_DATASETS_CACHE"])

def map_label(star):
    star = int(star)
    return 0 if star <= 2 else (1 if star == 3 else 2)  # 0:NEG 1:NEU 2:POS

for split in ds:
    if "text" in ds[split].features:
        ds[split] = ds[split].rename_columns({"text":"review"})
    if "label" in ds[split].features and "star" not in ds[split].features:
        ds[split] = ds[split].rename_columns({"label":"star"})
    ds[split] = ds[split].map(lambda ex: {"label": map_label(ex["star"])})

# ====== EDA（長さ分布） ======
import numpy as np, matplotlib.pyplot as plt
lengths = [len(ex["review"]) for ex in ds["train"]]
plt.figure(); plt.hist(lengths, bins=50)
plt.title("Review length distribution (train)"); plt.xlabel("chars"); plt.ylabel("count")
plt.tight_layout(); plt.savefig(os.path.join(FIGS_DIR, "length_hist.png"), dpi=150); plt.close()

# ====== トークナイズ ======
from transformers import AutoTokenizer, DataCollatorWithPadding
MODEL_NAME = "cl-tohoku/bert-base-japanese-v3"
tokenizer = AutoTokenizer.from_pretrained(MODEL_NAME, use_fast=False, word_tokenizer_type="basic")  # MeCab経路を使わない

def tok(batch):
    return tokenizer(batch["review"], truncation=True, padding=False, max_length=256)

remove_cols = [c for c in ds["train"].column_names if c not in ["label"]]
tokenized = ds.map(tok, batched=True, remove_columns=remove_cols)
collator = DataCollatorWithPadding(tokenizer=tokenizer)

if "validation" not in tokenized:
    sp = tokenized["train"].train_test_split(test_size=0.2, seed=42)
    tokenized = {"train": sp["train"], "validation": sp["test"],
                 "test": ds["test"].map(tok, batched=True, remove_columns=remove_cols)}

# ====== 学習 ======
from transformers import AutoModelForSequenceClassification, TrainingArguments, Trainer
import evaluate

id2label = {0:"NEG",1:"NEU",2:"POS"}; label2id = {v:k for k,v in id2label.items()}
model = AutoModelForSequenceClassification.from_pretrained(MODEL_NAME, num_labels=3, id2label=id2label, label2id=label2id)

acc = evaluate.load("accuracy"); prec = evaluate.load("precision"); rec = evaluate.load("recall"); f1 = evaluate.load("f1")
def compute_metrics(eval_pred):
    import numpy as np
    logits, labels = eval_pred
    preds = np.argmax(logits, axis=-1)
    return {
        "accuracy": acc.compute(predictions=preds, references=labels)["accuracy"],
        "precision_macro": prec.compute(predictions=preds, references=labels, average="macro")["precision"],
        "recall_macro": rec.compute(predictions=preds, references=labels, average="macro")["recall"],
        "f1_macro": f1.compute(predictions=preds, references=labels, average="macro")["f1"],
    }

# ← Transformers版の違いに対応（evaluation_strategy / eval_strategy）
import inspect
ta_params = set(inspect.signature(TrainingArguments.__init__).parameters)
eval_key = "evaluation_strategy" if "evaluation_strategy" in ta_params else ("eval_strategy" if "eval_strategy" in ta_params else None)
save_key = "save_strategy" if "save_strategy" in ta_params else None

kwargs = dict(
    output_dir=OUT_DIR, per_device_train_batch_size=16, per_device_eval_batch_size=16,
    learning_rate=2e-5, num_train_epochs=3, warmup_ratio=0.1, weight_decay=0.01,
    logging_steps=50, seed=42, dataloader_num_workers=2,
)
if eval_key: kwargs[eval_key] = "epoch"
if save_key: kwargs[save_key] = "epoch"
else:
    kwargs["save_steps"] = 500
    if eval_key:
        kwargs[eval_key] = "steps"
        if "eval_steps" in ta_params: kwargs["eval_steps"] = 500
if "report_to" in ta_params: kwargs["report_to"] = "none"
if ("load_best_model_at_end" in ta_params) and eval_key and ((save_key and kwargs.get(save_key)==kwargs.get(eval_key)) or (not save_key and kwargs.get(eval_key)=="steps")):
    kwargs["load_best_model_at_end"] = True
    if "metric_for_best_model" in ta_params: kwargs["metric_for_best_model"] = "f1_macro"
    if "greater_is_better" in ta_params: kwargs["greater_is_better"] = True

args = TrainingArguments(**kwargs)

trainer = Trainer(
    model=model, args=args,
    train_dataset=tokenized["train"], eval_dataset=tokenized["validation"],
    tokenizer=tokenizer, data_collator=collator, compute_metrics=compute_metrics,
)
trainer.train()

# ====== 可視化・評価・誤差分析 ======
hist = trainer.state.log_history
train_loss = [h["loss"] for h in hist if "loss" in h and "epoch" in h]
eval_loss  = [h["eval_loss"] for h in hist if "eval_loss" in h]
et, ee = [h["epoch"] for h in hist if "loss" in h and "epoch" in h],[h["epoch"] for h in hist if "eval_loss" in h]
plt.figure(); plt.plot(et, train_loss, label="train loss"); plt.plot(ee, eval_loss, label="eval loss")
plt.legend(); plt.xlabel("epoch"); plt.title("Loss curves"); plt.tight_layout()
plt.savefig(os.path.join(FIGS_DIR, "loss_curves.png"), dpi=150); plt.close()

test_metrics = trainer.evaluate(tokenized["test"])
print("=== TEST METRICS ==="); print(test_metrics)

import numpy as np
from sklearn.metrics import confusion_matrix, classification_report
preds = trainer.predict(tokenized["test"])
y_true = preds.label_ids; y_pred = preds.predictions.argmax(axis=-1)
report = classification_report(y_true, y_pred, target_names=["NEG","NEU","POS"])
cm = confusion_matrix(y_true, y_pred)
with open(os.path.join(REPORTS_DIR, "classification_report.txt"), "w", encoding="utf-8") as f:
    f.write(str(test_metrics) + "\n\n" + report)

plt.figure(); plt.imshow(cm, interpolation="nearest"); plt.title("Confusion Matrix"); plt.colorbar()
plt.xticks([0,1,2], ["NEG","NEU","POS"]); plt.yticks([0,1,2], ["NEG","NEU","POS"])
plt.tight_layout(); plt.savefig(os.path.join(FIGS_DIR, "confusion_matrix.png"), dpi=150); plt.close()

# 高確信で誤った例を5件
probs = preds.predictions - preds.predictions.max(axis=1, keepdims=True)
probs = np.exp(probs)/np.exp(probs).sum(axis=1, keepdims=True)
conf = probs.max(axis=1); wrong = np.where(y_true!=y_pred)[0]
top_wrong = wrong[np.argsort(conf[wrong])[::-1][:5]]
for i in top_wrong:
    print(f"[gt={['NEG','NEU','POS'][y_true[i]]}, pred={['NEG','NEU','POS'][y_pred[i]]}, conf={conf[i]:.2f}]")
```

## 3) 実行

```bash
python run_bert_amazon.py
# ./figs, ./reports, ./outputs-ja-bert, ./.cache, ./tmp が作成されます
```


# 代表的な失敗と対処

* **`fugashi` が無い/MeCab系で落ちる**
  → `cl-tohoku/bert-base-japanese-v3` を使い，`AutoTokenizer(..., use_fast=False, word_tokenizer_type="basic")` で **MeCab経路を明示回避**．`sentencepiece` のみでOK．
* **`protobuf` の ImportError**
  → `python -m pip install -U "protobuf<5"`（Transformers互換のため `<5` 推奨）．
* **Transformers の引数名違いで落ちる**（`evaluation_strategy` vs `eval_strategy`）
  → 上記スクリプトの **引数自動検出**で吸収．固定で書く場合は，環境に合わせて**どちらか**を指定．
* **フォルダがバラける**
  → スクリプト冒頭の **環境変数と保存先の固定**を確認（`.cache/` と `tmp/` をカレント直下）．
* **ヘッドレスで `plt.show()` が詰まる**
  → `matplotlib.use("Agg")` ＋ `savefig` へ．


# HPC（OpenPBS）での実行例

最小のジョブスクリプト（環境名・キュー名・GPU資源名はクラスターに合わせて変更）

```bash
#!/bin/bash
# ====== OpenPBS/PBS Pro (pbs_version 20.x) ======
# Job metadata
#PBS -N sft_bert_ja
#PBS -q GPU-1
# Resource request (OpenPBS 20.x 'select' syntax)
# 1 chunk: 8 CPU cores, 1 GPU, 32GB RAM
# If your site names the GPU resource differently, change ngpus=1 to gpu=1 or gres=1 accordingly.
#PBS -l select=1:ncpus=8:ngpus=1:mem=32gb
# Place policy (optional): pack tasks on a single node, exclusively
#PBS -l place=pack:excl
# Walltime
#PBS -l walltime=12:00:00
# Join stdout/stderr
#PBS -j oe

set -euo pipefail

echo "===[ENV]============================================="
date
echo "PBS version: ${PBS_VERSION:-unknown}  Job ID: ${PBS_JOBID:-N/A}"
echo "Job Name: ${PBS_JOBNAME:-N/A}"
echo "Work dir: ${PBS_O_WORKDIR:-$(pwd)}"
echo "Node: $(hostname)"
echo "======================================================"

# Go to submission directory
cd "${PBS_O_WORKDIR:-$PWD}"

# --- Modules (site-specific; uncomment if needed) ---
# module load cuda/12.1
# module load anaconda/2024.02

# --- Activate environment (conda preferred; venv fallback) ---
if [ -f "$HOME/miniconda3/etc/profile.d/conda.sh" ]; then
  source "$HOME/miniconda3/etc/profile.d/conda.sh"
elif [ -f "$HOME/anaconda3/etc/profile.d/conda.sh" ]; then
  source "$HOME/anaconda3/etc/profile.d/conda.sh"
fi
conda activate nlp-sft 2>/dev/null || true
# Or, if you used venv:
# source .venv/bin/activate

# --- Per-job caches to reduce $HOME I/O ---
BASE="${PBS_O_WORKDIR:-$PWD}"
mkdir -p "$BASE/.cache/hf" "$BASE/.cache/hub" "$BASE/.cache/transformers" "$BASE/.cache/datasets" "$BASE/tmp"
export HF_HOME="$BASE/.cache/hf" HF_HUB_CACHE="$BASE/.cache/hub" TRANSFORMERS_CACHE="$BASE/.cache/transformers" HF_DATASETS_CACHE="$BASE/.cache/datasets"
export TMPDIR="$BASE/tmp"


echo "===[RUNTIME DIAGNOSTICS]=============================="
nvidia-smi || true
python - <<'PY'
import sys, torch
print("Python:", sys.version.replace("\n"," ").strip())
print("Torch:", torch.__version__, "CUDA available:", torch.cuda.is_available())
try:
    import transformers, datasets
    print("Transformers:", transformers.__version__)
    print("Datasets:", datasets.__version__)
except Exception as e:
    print("Transformers/Datasets import issue:", e)
PY
echo "======================================================"

# ---- RUN: replace with your training script & args ----
# Example (fits the fine-tuning assignment):
python run_bert_amazon.py \
  --model_name cl-tohoku/bert-base-japanese-v3 \
  --epochs 3 \
  --batch_size 16 \
  --lr 2e-5 \
  --max_length 256 \
  --seed 42

echo "Done."

```

送信：

```bash
qsub train_sft_openpbs20.pbs
```


# 結果の見方（まずここをチェック）

1. `reports/classification_report.txt`：Accuracy / Macro-F1、クラス別指標
2. `figs/confusion_matrix.png`：どのクラスで取り違えが多いか
3. 誤分類ログ（標準出力の5件）：**確信度が高いのに誤り**のサンプルを読む

# 実験結果

```
              precision    recall  f1-score   support

         NEG       0.89      0.90      0.89      3000
         NEU       0.52      0.46      0.49      1000
         POS       0.67      0.73      0.70      1000

    accuracy                           0.78      5000
   macro avg       0.69      0.69      0.69      5000
weighted avg       0.77      0.78      0.77      5000
```

* 全体：Accuracy 0.78．多数派（NEG=60%）に“寄せるだけ”の 0.60 は十分超えていて，学習は効いている．

* NEG：P/R/F1 ≈ 0.89/0.90/0.89 と堅い（取りこぼし少）．

* POS：F1=0.70 でまずまず．

* NEU：F1=0.49 と弱い．Precision 0.52／Recall 0.46 → 「実際にNEUなのを半分以上取り逃す」し，「NEUと判定したものの約半分は他クラス」。

* 不均衡の影響：サポートが 3000/1000/1000（3:1:1）．weighted avg が Accuracy と近いのは，多数派NEGの出来が全体を押し上げているサイン．macro avg=0.69 が“バランス評価”．


# おわりに

ゼロショットに比べ，BERTのタスク微調整は **より安定した再現性** と **現場運用に近い学習/評価の型**を提供する．
一方で，バージョン差・形態素依存・HPC事情などの**非本質的な沼**が潜むのも事実．この記事ではそれらを回避して，**動くプログラムにする**ことを目標にしました．
次は mBERT や軽量モデル（DistilBERT）との**速度・精度トレードオフ**，少量追加学習（LoRA等）での**改善幅**を検証していきたい．
