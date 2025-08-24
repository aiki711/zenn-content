---
title: "LoRAでMistral-7Bを映画レビュー感情分類にファインチューニングした話"
emoji: "😸"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["LoRA","LLM","PEFT","fine-tuning/ファインチューニング","TRL"]
published: True
---
s

# LoRA で Mistral-7B を映画レビュー感情分類にファインチューニングした話

## 要旨

* **ベースモデル**: `mistralai/Mistral-7B-Instruct-v0.3`
* **タスク**: IMDb（2クラス：positive / negative）
* **方法**: PEFT の **LoRA** + TRL の **SFTTrainer**
* **環境**: NVIDIA A40（46GB）, PyTorch + Transformers/TRL/PEFT
* **出力形式**: モデル出力を **JSON（label / confidence / reason）** に強制 → 評価と分析を自動化
* **ざっくり結果**: 100件の検証で **Acc ≈ 0.94**（`report_lora_quality.json` より）

---

# なぜ LoRA？

フル微調整は 7B でも重い＆遅い．**LoRA** なら，注意機構など一部行列の低ランク更新のみ学習して，**数十MB〜数百MBのアダプタ**だけを保存できます．
今回の学習ログでは，**学習可能パラメータ 41.9M（全体の \~0.58%）** に抑制できました．



# セットアップ（再現可能なバージョン）

```bash
pip install -U "transformers>=4.41.0" "datasets>=2.18.0" \
  "accelerate>=0.30.0" "peft>=0.11.1" "bitsandbytes>=0.43.0" "trl>=0.9.6"

# HF 認証（必要なら）
huggingface-cli login
export HF_TOKEN=...   # あるいは環境に設定済み
```



# 学習データの設計：出力を最初から “構造化”

今回のキモは，**学習時から “JSON を出力させる指示” を一貫して与える**ことです．
モデルに **「label / confidence / reason」** を必ず返させる **JSON 指示プロンプト**を仕込み，SFT でも同じフォーマットで教えました．
これにより，**推論 → メトリクス計算 → CoT（理由）品質分析**までを **機械可読**に繋げられます．



# 学習（Supervised Fine-Tuning; SFT）

スクリプト: `train_lora_sft.py`（PEFT + TRL）

主なハイパーパラメータ（抜粋・既定値はスクリプト内）:

* LoRA: `r`（ランク）, `alpha`, `dropout`，対象モジュールは Mistral の `q_proj / k_proj / v_proj / o_proj / gate / up / down` を想定
* 最大学習長 `max_seq_len`，学習率，エポック数，バッチサイズ，`bf16`，（任意で）`4bit` 量子化読み込み など

実行例:

```bash
python train_lora_sft.py \
  --base_model mistralai/Mistral-7B-Instruct-v0.3 \
  --out_dir runs_imdb_A40/lora_mistral_7b_instruct_v0_3 \
  --num_train 8000 \
  --epochs 1 \
  --lr 2e-4 \
  --r 16 --alpha 32 --dropout 0.05 \
  --bf16
```

学習ログ例（抜粋）:

```
trainable params: 41,943,040 || all params: 7,289,966,592 || trainable%: 0.5754
{'train_runtime': ~3403s, 'train_steps': 500, 'train_loss': ~3.36}
Saving LoRA adapter…
Done. Adapter saved to runs_imdb_A40/lora_mistral_7b_instruct_v0_3
```

---

# 推論 & 評価を “構造化 JSON” で一気通貫

## 推論

スクリプト: `infer_lora_imdb.py`

* LoRA アダプタをベースモデルに適用（`PeftModel.from_pretrained`）
* **JSON を必ず出す**プロンプト（system+user）で生成
* 生成テキストから **最初の JSON ブロックだけ**を抽出（`parser.py`）

出力（行ごとの JSONL）:

* `imdb_lora_predictions.jsonl` … `{"output": "<モデルの生出力>"}`
* `imdb_lora_gold.jsonl` … `{"text": "...", "label": "positive|negative"}`

実行例:

```bash
python infer_lora_imdb.py \
  --base_model mistralai/Mistral-7B-Instruct-v0.3 \
  --adapter_dir runs_imdb_A40/lora_mistral_7b_instruct_v0_3 \
  --out_dir runs_imdb_A40/lora_eval_mistral_7b_instruct_v0_3 \
  --num_samples 100 \
  --bf16 \
  --max_new_tokens 64 \
  --temperature 0.0
```

## 評価（精度・再現率・適合率・F1・混同行列）

スクリプト: `evaluate_zero_shot.py`

* 予測の **JSON（label/confidence/reason）** をパース
* ラベルは **`positive|negative`** を正規化
* **Precision/Recall/F1（macro）** と混同行列を算出

実行例:

```bash
python evaluate_zero_shot.py \
  --gold runs_imdb_A40/lora_eval_mistral_7b_instruct_v0_3/imdb_lora_gold.jsonl \
  --pred runs_imdb_A40/lora_eval_mistral_7b_instruct_v0_3/imdb_lora_predictions.jsonl \
  --labels 2class \
  --report_out runs_imdb_A40/report_lora.json \
  --preview_out runs_imdb_A40/preview_lora.json
```

## 理由（reason）と自信度（confidence）の品質チェック

スクリプト: `analyze_cot_quality.py`

* **解析対象**: gold と pred（生出力）
* **指標**: 解析成功率，Accuracy，平均 confidence，平均 reason 文字数，**confidence と正解の相関**（Pearson）

実行例:

```bash
python analyze_cot_quality.py \
  --gold runs_imdb_A40/lora_eval_mistral_7b_instruct_v0_3/imdb_lora_gold.jsonl \
  --pred runs_imdb_A40/lora_eval_mistral_7b_instruct_v0_3/imdb_lora_predictions.jsonl \
  --out  runs_imdb_A40/report_lora_quality.json
```

> `report_lora_quality.json` 例（抜粋）
> `{"accuracy": 0.94, "parse_success_rate": 1.0, "avg_confidence": ..., "avg_reason_length": ..., ...}`



# 結果（サマリ）

* **LoRA（100件検証）**: `report_lora_quality.json` より **Accuracy ≈ 0.94**
* `report_lora.json` には **Precision / Recall / F1（macro）** と **混同行列** を収録
* `preview_lora.json` で **誤り事例の先頭20件** を即確認できます

> 注: 以前，`positive` の指標が全ゼロになる問題がありました．
> 原因は **ラベルの正規化／クラス不均衡サンプリング／出力パース**のどれか（または複合）でした．
> **修正点**
>
> * 予測は **JSON を強制**（自由文をやめる）
> * ラベルは **`normalize_label`** で統一
> * 検証サンプルは **shuffle または層化**（両クラスが確実に含まれる）
>   これで per-class の数値が正しく出るようになりました．



# よく遭遇したエラーと対処

## 1) `SFTConfig.__init__() got an unexpected keyword argument 'max_seq_length'`

* **原因**: TRL の API バージョン差異
* **対処**: `trl>=0.9.6` に合わせて引数を整理（古い版では `SFTTrainer` の引数名も違う）

## 2) `SFTTrainer.__init__() got an unexpected keyword argument 'tokenizer'`

* **原因**: こちらも TRL の版差．古い API は `tokenizer` を直接受けない
* **対処**: `dataset_text_field` 経由に変更 or TRL をアップデート

## 3) `adapter_config.json` の 404（HF Hub で見つからない）

* **原因**: 推論時の `--adapter_dir` が **ローカル出力**ではなく **Hub リポジトリ名**になっていた
* **対処**: 学習で出力された **ローカルディレクトリ**（例: `runs_imdb_A40/lora_mistral_7b_instruct_v0_3`）を渡す

## 4) `KeyError: 'OUT'`（PBS 後処理）

* **原因**: PBS スクリプト末尾の Python ワンライナーで `os.environ["OUT"]` を **未定義のまま参照**
* **対処**: PBS 冒頭で `export OUT=...` を必ず指定 or

  ```python
  OUT = os.getenv("OUT", ".")  # 既定をカレントなどに
  ```

## 5) `avg_confidence` / `avg_reason_length` / `corr_confidence_correct` が `null`

* **原因**: 予測が **JSON ではなく自由文** → パーサが `confidence` / `reason` を取れない
* **対処**: 学習・推論とも **JSON 出力を強制**（本稿の実装で解決）

---

# vLLM でのベースライン比較（任意）

ゼロショット／少ショット（few-shot, CoT）基準の作成に `run_vLLM_imdb_variants.py` を使用し，`report_zero.json`, `report_few2.json`, `report_few5.json` を作成．LoRA の改善幅を確認しました．
（vLLM は高速ですが，**LoRA アダプタはそのままでは読めない**ため，必要なら次の「マージ」を使います）

---

# LoRA をベースにマージしてデプロイ

`merge_lora_and_export.py` で LoRA をベースモデルに **merge & save**．
vLLM 等でそのまま配布したい時に便利です．

```bash
python merge_lora_and_export.py \
  --base_model mistralai/Mistral-7B-Instruct-v0.3 \
  --adapter_dir runs_imdb_A40/lora_mistral_7b_instruct_v0_3 \
  --out_dir runs_imdb_A40/mistral7b_imdb_lora_merged \
  --bf16
```



# 実験結果（LoRA / IMDb 2クラス）

**環境**: NVIDIA A40（46GB）, bf16, Mistral-7B-Instruct-v0.3
**学習**: 1 epoch / 8,000 サンプル（SFT, LoRA r=16, α=32, dropout=0.05）

## 学習ログ（抜粋）

* 学習可能パラメータ: **41,943,040**（全体 7,289,966,592 の **0.575%**）
* 学習時間: **3,403.39 s ≒ 56.7 分**
* 1 秒あたりステップ数: **0.147 steps/s**
* 最終 `train_loss`: **3.3608**
* LoRA アダプタ保存サイズ（`adapter_model.safetensors`）: **約 160.1 MB**

## 推論・評価（100件の検証）

（`report_lora_quality.json` より）

| 指標                               |        値 |
| -------------------------------- | -------: |
| Accuracy                         | **0.94** |
| JSON 解析成功率（parse\_success\_rate） | **0.99** |
| サンプル数                            |  **100** |

> 備考: 現状の `report_lora_quality.json` では `avg_confidence` / `avg_reason_length` / `corr_confidence_correct` が `null`（未集計）です．これは **推論出力に `confidence` と `reason` を含めない設定で回した履歴**が混じっているためです．`infer_lora_imdb.py` の更新版（モデルに JSON 形式 `{label, confidence, reason}` を強制）で再実行すると，これらの指標も自動で埋まります．

## これまでの実験との比較

![Figure1](/images/lora_sentiment_eval/figure1.png "bar accuracy axis80~100 ")


## 注意点（再現時）

* `report_lora.json` が全ゼロになるケースは，**評価対象の予測/gold のラベル整形不一致**や**クラス片寄り（検証に positive が入っていない等）**，**誤ったファイルパス**が原因になりがちです．最新のスクリプトでは以下で対策済みです：

  * 予測を **JSON 強制** → `label` を正規化して評価
  * 検証セットの **層化またはシャッフル**（両クラスが必ず含まれる）
  * 失敗時は `preview_*.json` で先頭の失敗例を即確認



# まとめと今後

* LoRA + JSON 一貫設計で，**精度**だけでなく \*\*自信度（confidence）や理由（reason）\*\*まで解析できる評価系を構築．
* 版差に強いスクリプトにしておくと運用が安定します（TRL/Transformers は更新頻度が高い）．
* 次の一手:

  * **データ拡張**と **エポック増**で LoRA の上積み検証
  * **校正損失**（confidence calibration）や **reason の妥当性評価**（例: 事実一致・一貫性）
  * **別ドメイン**（ツイート，レビュー以外）への転移



## 付録：成果物のファイル一覧（例）

```
runs_imdb_A40/
  lora_mistral_7b_instruct_v0_3/
    adapter_config.json
    adapter_model.safetensors
    tokenizer.* / chat_template.jinja
  lora_eval_mistral_7b_instruct_v0_3/
    imdb_lora_predictions.jsonl
    imdb_lora_gold.jsonl
  report_lora.json              # P/R/F1/混同行列
  report_lora_quality.json      # JSON 解析成功率・信頼度・相関など
```

