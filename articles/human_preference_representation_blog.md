---
title: "表現操作で人間の好みにLLMを合わせるRLなし低コスト整列に関する論文を一緒に読みましょう！"
emoji: "🎛️"
type: "idea" # tech: 技術記事 / idea: アイデア
topics: ["LLM", "RLHF", "DPO", "Representation", "Engineering"]
published: True
---


# RAHF: 表現操作で“人間の好み”にLLMを合わせる──RLなし低コスト整列
> この記事は，「自分の理解を深めたい」という気持ちで書いています．読者のみなさんと**同じ目線**で，一緒に理解を育てていくスタイルです．僕の理解が及ばない部分があれば，優しく教えていただけると幸いです！


# TL;DR
- **RAHF** は，**表現工学（RepE）**の発想で，LLM内部の**活性パターンの差**（好ましい/好ましくない）を抽出し，**LoRAでその差に合わせて表現を微調整**する整列手法．**RL不要**・**実装が簡潔**・**計算軽量**が売り．
- 2系統：①**RAHF-SCIT**＝**単一モデル**を**対照命令（good/bad）**で整列，②**RAHF-Dual**＝**好ましい/好ましくない**応答で**2モデル**を別々に整列．どちらも**活性の差ベクトル**で最終モデルを調整する．
- **Open LLM Leaderboard（6タスク）/AlpacaEval/MT-Bench** の自動評価＋**人手評価**で，**RLHF**や**DPO/HIR**などのRLフリー系と比較し，広く優位または同等以上．



# 背景

* **“表現（内部活性）”という視点の台頭**
  最近は，LLM の**内部表現を直接いじる**ことで，ふるまいを**低コスト・局所的**に制御する
  **Representation Engineering（RepE）**や**Activation Steering**が注目されている．
  直感はシンプルで，**同じ入力でも“望ましい刺激”と“望ましくない刺激”で脳内（モデル内）の活性パターンが異なる**なら，
  **その“差”の方向にモデルを少し寄せれば，望ましいふるまいに近づくはず**，という発想．

* **従来の整列（Alignment）の主流＝RLHF は重くて不安定**
  人間の好み（選好）に合わせる代表手法は RLHF（報酬モデル→PPO）だが
  1. 報酬モデルの訓練と安定化が難しい
  2. 反復学習で**計算コストが高い**
  3. 報酬過適合や報酬ハッキングで**挙動がブレやすい**

* **RLフリー整列（DPO/HIR 等）にも弱点**
  DPO のような**RL不要**の手法は実装が簡単で人気ですが，
  1. **ペアラベルのノイズ**や**データ分布の偏り**に影響を受けやすい
  2. 目的関数が**表層的なトークン確率**に縛られ，**内部表現の“何を変えたか”が不透明**



# 提案

* **ポジショニング（どこが新しいか）**
  * **目的が“表現の方向合わせ”**：DPO のように**出力確率だけ**を押し上げるのではなく，
    **“望ましい状態の内部活性”** そのものにモデルを合わせ込む．
  * **低コスト＆シンプル**：LoRA の追加だけで済み，学習も安定しやすい．
  * **一般能力の維持**：表現を**局所的に微調整**することで，汎用ベンチ（MMLU/TruthfulQA など）も
    極端に落とさず，むしろ改善するケースが示されている．

* **なぜ“差分ベクトル”が効くのか（直観）**
  望ましい応答のときモデル内で強く立ち上がるニューロン群（特徴）と，
  望ましくない応答で立つ群は**統計的に別の方向**を向くことが多い．
  この**方向差**を「好みステアリング方向」と見なし，**少量の LoRA でそこへ寄せる**と，
  **大規模 RL を回さずに“好み”の側へ**ふるまいを押し出せる．

> **「RLHF は重い」「DPO は表層的になりやすい」** というギャップに対し，
**内部表現の“差”を学んで LoRA で合わせ込む**という，
**低コストで説明可能性のある整列ルート**を提示したのが RAHF です．


# 手法（RAHFの3ステップ）
![Figure2](/images/human_role_prefarence_representation_blog/figure2.png)
- **好み整列のRepE化**：LLMを**好み指示**で刺激して**活性差**を抽出し，その差に**LoRA**で合わせ込む“**表現編集による整列**”を確立．
- **2つの整列前段**  
  1. まず **“好み”をわかるモデル（または設定）** を用意（前段）：
     * **SCIT**：同一モデルに good/bad の**対照命令**で好みを教え込む
     * **Dual**：**好ましい応答**と**好ましくない応答**で**2モデル**を別々に SFT
  2. つぎに**同じ応答**に **good/bad 刺激**を与えて**各層の活性差（ステアリング方向）** を測る
  3. **LoRA の極少パラメータ**で**活性をその差方向へ寄せる**（MSE 整列）．
     これにより，**RL なし・小規模更新**で，好みに沿った挙動へ**安定に寄せる**ことを狙う

## 1) 「好み」をモデルに教える（前段整列）
- **RAHF-SCIT**（Single-Model, Contrastive Instruction Tuning）  
  good/bad の**対照命令**で同一モデルを学習し，**好ましい/好ましくない**を**相対確率**で近づけ/遠ざける：  

  $$
  \mathcal{L}=-\sum \Big[\;P^+ + \log \frac{e^{P^+}}{e^{P^+}+e^{P^-}}\;\Big],\ \ P^\pm=\log\pi(r\mid p^\pm,q)
  $$

- **RAHF-Dual**（Dual-Model, Supervised）  
  同じクエリに対し，**好ましい応答**で**好モデル**，**好ましくない応答**で**悪モデル**を**別々にSFT**：  

  $$
  \theta_h^*=\arg\max\sum\log\pi(r_h\mid q),\quad \theta_l^*=\arg\max\sum\log\pi(r_l\mid q)
  $$

## 2) 活性パターンの収集（差ベクトル）
- **同一の応答 $r$** に **good/bad 刺激 $p^+,p^-$** を連結して入力し，**各層の隠れ状態**を抽出．**好み条件の差**を**層ごと**に計算：  

  $$
  v_l = A_{p^+,\pi,l}-A_{p^-,\pi,l}
  $$

  （SCITは**同一モデル**，Dualは**好/悪モデル**で取得）

## 3) LoRAで“差”に合わせる（最終モデル構築）
- **LoRA出力**を**元の活性**に足して**差ベクトル方向**へ**MSE整列**：  

  $$
  \mathcal{L}_{\text{Align}}=\left\|A_{p,\pi_{\text{LoRA}},l}-\big(A_{p,\pi_{\text{base}},l}+\alpha v_l\big)\right\|_2^2
  $$

  係数 $\alpha$ は介入強度，**対象層（例：10,20,2）** で操作．

## 4) 推論時の使い方

最終モデルは **LoRAを合成（または適用）した1モデル**として振る舞うので，**特別な“good/bad命令”や補助プロンプトは不要**．通常のプロンプトで望ましい側へ挙動が寄った応答が返る，という位置づけ（RL は用いない）．



# 実験設定（抜粋）
- **データ**：UltraFeedback（好みペア），ベースは **LLaMA2-7B** を Helpful&Harmless でSFTしたモデル．
- **比較**：**Preferred-SFT / HIR / DPO / RLHF-PPO**．
- **評価**：Open LLM Leaderboard（Arc/HellaSwag/MMLU/TruthfulQA/Winogrande/GSM8k），**AlpacaEval（GPT-4判定）**，**MT-Bench**，**人手評価**．



# 主な結果

## Open LLM Leaderboard
![Figure3](/images/human_role_prefarence_representation_blog/figure3.png)
- **RAHF-SCIT** が**平均+2.33pt**向上（Base比）．**TruthfulQA** で両RAHFが全ベースラインを上回る．

## AlpacaEval（GPT-4審査）
![Figure4](/images/human_role_prefarence_representation_blog/figure4.png)
- **勝率**：**RAHF-SCIT 87.44%／RAHF-Dual 86.98%** ＞ DPO 83.68%．

##  MT-Bench
![Figure5](/images/human_role_prefarence_representation_blog/figure5.png)
- 8側面中**6側面で最高**，平均でも上位．**推論・ロールプレイ・STEM**で特に強み．

## 人手評価
![Figure6](/images/human_role_prefarence_representation_blog/figure6.png)
- **RAHF vs RLHF/DPO/HIR**：**勝>引分>負**で優勢（両方式）．

## アブレーション（RepEのみとの比較）
![Figure7](/images/human_role_prefarence_representation_blog/figure7.png)
- **LORRA / LORRA-Pref**（前段整列なしの表現編集）より，**RAHFの前段整列＋差ベクトル学習**が有意に効く．



# 直感で掴む：なぜ効く？
- **同じ応答**でも **good/bad 刺激**で**内部活性が変わる** → その**差分方向**へ**表現を少量のLoRAで寄せる**から，**挙動を低コストに整列**できる．  
- **SCIT**は**同一モデル内**で差を取るため**表現空間のズレが少ない**（Dualより有利な場面あり）．



# 限界と注意点

## 限界（論文が明記）

* **スケールの限定**：有効性の検証は**7B規模のLLM**に限られています．より大規模SOTAへの拡張は今後の課題．
* **LoRAによる追加パラメータ**：最終モデルは**差ベクトルをLoRAで近似**する設計のため，**少量ながら追加パラメータ**が入ります．将来は**差ベクトルをモデルに直接統合**してコストをさらに下げる方向を提案．

## 注意点（設計・運用で気をつけるところ）

* **どの層を操作するか**：**中間層での操作が最も有効**．上位層に近いほど**同じαでも既存表現への影響が大きく**，RAHF-DUALでは特に劣化しやすい．
* **どのトークンの表現を使うか**：RAHF-DUALは**応答の先頭64トークン**のみでLoRAを学習（後半は指示の影響が薄れて性能が落ちやすいと観察）．**長文応答**では先頭寄りの表現に重みを置く前提を理解しておく．
* **評価の前提**：自動評価の一部（AlpacaEval/MT-Bench）は**GPT-4を審査者**として用いる設計で，「人手評価と高相関」とは言え**LLM-as-judge バイアス**の余地はある．実運用では**人手評価も併用**して検証を．
* **データ品質への依存**：RAHFは“報酬なし”整列ですが，**好みペアのノイズや偏り**は依然リスク．RL（報酬学習）はノイズを報酬関数で緩和できる一方，**報酬なし整列はノイズの影響を受けやすい**という一般的な性質を踏まえ，**学習データのクリーニング**は重要．



# まとめ
- **RAHF = “好み刺激で生じる活性差”を捉え，LoRAでそこへ寄せる**．  
- **RLなし**で**実装容易**，**DPO/RLHFと競合**しうる整列手段として有望．まずは**SCIT**から試し，**差ベクトルの抽出長・層・$\alpha$** を軽くスイープ，**TruthfulQA/AlpacaEval/MT-Bench**で**自動＋人手評価**するのが実務向けの入り口．



# 参考（論文情報）

- **タイトル**：*Aligning Large Language Models with Human Preferences
through Representation Engineering*
- **著者**：Wenhao Liu, Xiaohua Wang, Muling Wu, Tianlong Li, Changze Lv
Zixuan Ling, Jianhao Zhu, Cenyuan Zhang, Xiaoqing Zheng∗, Xuanjing Huang
- **年**：2025
- **ACL Anthology**： [リンクはこちら](https://aclanthology.org/2024.acl-long.572/)