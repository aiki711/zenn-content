---
title: "ペルソナ方向ベクトルの抽出と制御・監視に関する論文を一緒に読みましょう！"
emoji: "🍣"
type: "idea" # tech: 技術記事 / idea: アイデア
topics: ["LLM","Personality","Communication","Psychology","AI Ethics"]
published: false
---

# Persona Vectors: Monitoring and Controlling Character Traits in Language Models を噛み砕く

> この記事は，「自分の理解を深めたい」という気持ちで書いています．読者のみなさんと**同じ目線**で，一緒に理解を育てていくスタイルです．僕の理解が及ばない部分があれば，優しく教えていただけると幸いです！


# TL;DR（最初に結論）

* LLM の内部表現には，「悪意（evil）」「迎合（sycophancy）」「ハルシネーション傾向」などの“人格特性”に対応する**線形方向（= persona vectors）**が潜在しています。
* その方向を使うと，

  1. 生成開始 **前** にプロンプトで人格が崩れる兆候を **監視**，
  2. 微調整（fine-tuning）で望ましくない人格の **シフトを測定・予測**，
  3. 生成時の **事後ステアリング** や学習中の **予防ステアリング** による **制御**
     が可能になります。
* さらに学習データを **“危険度”でふるいにかける（投影差）** こともできます。



# はじめに - なぜ「人格特性に対応する方向ベクトル」に着目したのか

* LLM は「アシスタント」という人格（persona）を模して応答しますが、
  * プロンプトの影響や
  * 微調整や継続学習の副作用によって，時に **悪意（攻撃性）**・**迎合（過度な同意）**・**ハルシネーション傾向** などに **滑る（drift）** など
ときどき望ましくない振る舞いに逸脱します。

* 著者らは「悪意（evil）」「迎合（sycophancy）」「ハルシネーション傾向」などの特性に対応する\*\*線形方向（persona vectors）\*\*をモデルの活性空間から抽出し、
  1. こうした人格の揺らぎを**観測**でき、
  2. 学習（微調整）で生じる人格変化を**予測**・**制御**できる
という仮説を立てた．

* 本論文は，**線形方向ベースの実用的フレームワーク** を提示している。



# 何をした論文か（貢献）

* 任意の自然言語で記述された“特性”から，**自動**で persona vector を抽出するパイプライン。
* **監視**：生成直前のプロンプト活性を特性方向に射影し，「今から逸脱しそうか」をスコア化。
* **制御**：

  * 生成時に $\pm \alpha v_\ell$ を加える **事後ステアリング**（悪化/抑制），
  * 学習過程に方向を入れて偏りを打ち消す **予防ステアリング**（能力劣化を抑えやすい）。
* **データ審査**：特性方向への **投影差（projection difference）** により，

  * データセット単位・サンプル単位で「人格シフトを招きやすい」危険データを **事前にフラグ** 可能。

微調整後の意図的・非意図的な人格変化は、該当する persona vector 方向への活性シフトと強く相関します。
こうしたシフトは、生成時の**事後介入**で緩和でき、さらに学習過程での**予防的ステアリング**でそもそも回避することも可能です。
さらに persona vectors を使うと、**データセット単位／サンプル単位**で“望ましくない人格変化を招きやすい学習データ”を**事前にフラグ**できます。
ベクトル抽出は自動化されており、**自然言語の特性記述さえあれば**任意の人格特性に適用できます。



# 手法の全体像

![パイプライン](/images/llm_persinality_reserch_blog/figure1.png "Persona vectors and their applications.")

1. **特性記述**（例：「evil: 攻撃性・悪意を示す応答傾向」）
2. **対照プロンプト生成**：特性を促す/抑える *system prompt* と評価用 *question* を自動生成
3. **多数応答生成 & ジャッジ**：各応答に“特性スコア”を付与（LLM ジャッジ等）
4. **活性差から方向抽出**：応答トークンの残差活性平均の差 = persona vector
5. **用途**：監視／制御（事後・予防）／データ審査（投影差）



# 数式メモ（最小限）

**記法**：層 $\ell$ の残差活性を $h_\ell(x,t)$（トークン $t$）とし，応答トークンで平均：
$$
\bar{h}_\ell(x)=\frac{1}{T}\sum_{t=1}^T h_\ell(x,t)
$$

特性“有り/無し”の集合 $X^+,X^-$ に対し，**persona vector**：
$$
 v_\ell=\mathbb{E}_{x\in X^+}[\bar{h}_\ell(x)]-\mathbb{E}_{x\in X^-}[\bar{h}_\ell(x)]
$$

**監視スコア（生成前）**：直前プロンプトのラストトークン活性 $\tilde h_\ell(p)$ に対し
$$
 s(p)=\langle \tilde h_\ell(p),\, v_\ell\rangle
$$

**事後ステアリング（生成時）**：
$$
 h'_\ell(x,t)=h_\ell(x,t)+\alpha\,v_\ell \quad(\text{抑制は }\alpha<0)
$$

**投影差（データ審査）**：候補データ応答とベースモデル自然応答の投影差
$$
 \Delta P=\langle \mathbb{E}[\bar{h}_\ell(\text{candidate})],v_\ell\rangle-\langle \mathbb{E}[\bar{h}_\ell(\text{base})],v_\ell\rangle
$$



## Persona Vector 抽出（詳細）

![抽出フロー](/images/llm_personality_reserch_blog/figure2.png "Automated pipeline for persona vector extraction.")

1. **対照条件の用意**：特性を「促す／抑える」2種の system prompt を自動生成。
2. **応答生成**：同一質問に対して両条件で多数の応答を作成。
3. **ジャッジ**：各応答に LLM 判定で特性スコア（連続 or バイナリ）を付与。
4. **活性収集**：対象層の残差活性を**応答トークン平均**で集計。
5. **方向推定**：上の式から $v_\ell$ を算出し、検証で最も効いた層 $\ell$ を採用。

> **実務メモ**
>
> * Hook 名（`resid_pre` / `post_attention` など）は実装依存。
> * まずは**応答トークン平均**が安定。
> * 単層よりも**小さめの係数で複数層**をステアした方が副作用が少ない傾向。



## 制御（Steering）

![事後/予防ステアリング](/images/llm_personality_reserch_blog/figure3.png)

### 事後ステアリング（生成時）

* 生成ループ中に $h_\ell \leftarrow h_\ell + \alpha v_\ell$ を適用。
* **強める**：$\alpha>0$、**弱める**：$\alpha<0$。
* **利点**：外付けで即適用・撤去が容易。
* **注意**：$|\alpha|$ を大きくしすぎると\*\*一般能力（例：MMLU）\*\*が落ちやすい。

### 予防ステアリング（学習中）

* 学習時に $+\beta v_\ell$ を加えて“偏り圧”を相殺、または
  **正則化** $\mathcal{L}_{\text{persona}}=\lambda\langle \bar{h}_\ell, v_\ell\rangle^2$ を損失に追加。
* **目的**：**人格シフトを小さく**保ちつつ、**能力劣化を抑える**。



## 監視（Monitoring）

![監視概念図](/images/llm_personality_reserch_blog/figure5.png)

* **生成前**：ユーザ最終プロンプトの**最後のトークン活性**を $v_\ell$ に射影してスコア化。
* **運用**：しきい値超えで安全化モード（出力の緩和・言い換え・人間確認）を発火。
* 補足：このスコアは後続の**特性スコアと高い相関**（概ね $r\approx0.75\text{–}0.83$）が報告される。



## 人格シフトの測定（微調整の影響）

* 特性方向への**活性シフト量（finetuning shift）**は、学習後の**特性表出**と高相関（$r=0.76\text{–}0.97$）。
* 特性を狙っていない **EM-like**（例：医療の誤助言、誤ったコード／数理解法）でも**意図せぬ人格変化**を誘発し得る。



## データ審査：投影差で“危険”を弾く

![投影差の概念図](/images/llm_personality_reserch_blog/figure4.png)

* 各サンプルについて、候補応答とベース自然応答の**投影差** $\Delta P$ を計算。
* $\Delta P$ が大きいほど、その特性方向に**引っ張る危険サンプル**。
* **運用**：高 $\Delta P$ を**除外／重み減衰**して、データセット段階でのリスク低減を図る。
* 注意：自然応答の生成が必要で**計算コストが重い**ため、近似やバッチ化を検討。



# あなたの研究への示唆（感情推定・パーソナライズ文体）

* **感情やスタイル**を“特性”として定義すれば、同様の手順で**日本語モデル**にも方向抽出が可能（要：評価文＆ジャッジ）。
* **学習前スクリーニング**で“迎合的/攻撃的/虚偽”な応答を誘発しやすいサンプルを**自動検知→除外**できる。
* **デプロイ前の安全監視**として、**最後のプロンプト活性の投影**をモニタリング → 危険兆候で**応答生成を抑止/言い換え**するガードを実装可能。




## 0. 図・アセットの置き方（Zenn）

* 画像は `zenn-content/images/persona-vectors-monitoring-and-control/` に配置します。
* 参照例：

  * `![パイプライン](/images/persona-vectors-monitoring-and-control/pipeline.png)`
  * `![事後/予防ステアリング](/images/persona-vectors-monitoring-and-control/steering.png)`
  * `![投影差の概念図](/images/persona-vectors-monitoring-and-control/projection-diff.png)`



# 実装メモ（最小スニペット）

> 以降は PyTorch + Decoder-only（Llama/Qwen系）を想定した概念コードです。実際の属性名は実装に合わせて調整してください。

## 活性のフック

```python
# resid_pre をフック（例: Llama系）
handles = []
captures = {}

def hook(layer_id):
    def _hook(module, inp, out):
        # out: (batch, seq, hidden)
        captures.setdefault(layer_id, []).append(out.detach())
    return _hook

for i, block in enumerate(model.model.layers):
    h = block.register_forward_hook(hook(i))
    handles.append(h)
```

## 事後ステア（生成ループ内）

```python
alpha = -1.0  # 抑制なら負に

def steer_hook(v):  # v: (hidden,)
    def _hook(module, inp, out):
        return out + alpha * v.view(1, 1, -1)  # 全トークン同量の簡易版
    return _hook

layer_id = best_layer
steer_handle = model.model.layers[layer_id].register_forward_hook(steer_hook(v_layer))
# 生成後: steer_handle.remove()
```

## 監視スコア（生成前）

```python
with torch.no_grad():
    outputs = model(**prompt_inputs, output_hidden_states=True)
    h_last = outputs.hidden_states[layer_id][:, -1, :]  # 最終トークン活性
    s = torch.einsum("bd,d->b", h_last, v_layer)       # しきい値で判定
```

> **Tips**
>
> * 応答トークン平均 vs 最終トークン：抽出時は **応答側平均** が安定。監視は **最後のプロンプトトークン** が扱いやすい。
> * 多層同時ステアは効くが，強すぎは禁物。**小さく複層** が無難。

---

# 実験設計と評価（読み方ガイド）

* **対象特性**：悪意（evil），迎合（sycophancy），ハルシネーション傾向（hallucination）など。
* **対象モデル例**：Qwen / Llama の 7–8B 級。
* **評価指標**：

  * LLM ジャッジによる特性スコア（連続/バイナリ）。
  * 微調整後の人格変化と **方向への活性シフト量** の相関 $r$。
  * **副作用**：能力ベンチ（例：MMLU）とステア強度とのトレードオフ。
* **可視化例**：

  * `images/.../corr_shift.png`（シフト量 vs 特性スコア）
  * `images/.../mmlu_tradeoff.png`（$\alpha$ と MMLU の関係）




# 既知の限界・注意点

* **LLMジャッジ**（GPT-4.1-mini）に依存 → 人手評価や外部ベンチで妥当性チェックはしているが、評価者バイアスには注意。
* 監視の相関は**プロンプト条件の違い**で主に生じる（微妙な変化検出は弱まる）。
* 特性間の**相関**（負の/正の連動）も観測 → 単一特性だけを独立に動かすのは難しい，一方を抑えると別が強まる等の連動があり得る。
* 事後ステアリングは**能力低下**の副作用があり得る一方、予防ステアリングは**多層**でより有効。



# 参考

* 原著：*Persona Vectors: Monitoring and Controlling Character Traits in Language Models*（arXiv）
  ※リンク・正式な書誌情報はご自身の確認に基づき追記してください。

> ご意見・修正依頼歓迎です。図や数式の詳細化，Colab 最小再現の追加，安全ガードの設計ガイドなども追補可能です。
