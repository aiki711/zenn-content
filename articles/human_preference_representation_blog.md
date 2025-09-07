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
- **RAHF** は、**表現工学（RepE）**の発想で、LLM内部の**活性パターンの差**（好ましい/好ましくない）を抽出し、**LoRAでその差に合わせて表現を微調整**する整列手法。**RL不要**・**実装が簡潔**・**計算軽量**が売り。
- 2系統：①**RAHF-SCIT**＝**単一モデル**を**対照命令（good/bad）**で整列、②**RAHF-Dual**＝**好ましい/好ましくない**応答で**2モデル**を別々に整列。どちらも**活性の差ベクトル**で最終モデルを調整する。
- **Open LLM Leaderboard（6タスク）/AlpacaEval/MT-Bench** の自動評価＋**人手評価**で、**RLHF**や**DPO/HIR**などのRLフリー系と比較し、広く優位または同等以上。



# 背景と位置づけ
- **従来**：RLHF は強力だが**不安定・複雑・高コスト**。一方、**DPO/HIR** 等の**RL不要**な整列は**ラベルノイズやデータ偏り**に弱い。
- **発想転換**：**神経科学的な“表現”の観点**から、**好み**に関連する**内部表現（活性）**を特定→**差分**で**ふるまい**を制御する**RepE**を、人間の多様な好み整列に拡張。



# 貢献（何が新しい？）
- **好み整列のRepE化**：LLMを**好み指示**で刺激して**活性差**を抽出し、その差に**LoRA**で合わせ込む“**表現編集による整列**”を確立。
- **2つの整列前段**  
  - **SCIT**：**対照命令**（good/bad）で**単一モデル**に好みを学習させる（対照損失）。
  - **Dual**：**好ましい/好ましくない**応答で**2モデル**を**別々に教師あり**で作る。
- **簡潔＆効果的**：**RLなし**で**汎用能力**・**TruthfulQA** 等も向上し、**人手評価**でも優位。



# 手法（RAHFの3ステップ）

## 1) 「好み」をモデルに教える（前段整列）
- **RAHF-SCIT**（Single-Model, Contrastive Instruction Tuning）  
  good/bad の**対照命令**で同一モデルを学習し、**好ましい/好ましくない**を**相対確率**で近づけ/遠ざける：  

  \[
  \mathcal{L}=-\sum \Big[\;P^+ + \log \frac{e^{P^+}}{e^{P^+}+e^{P^-}}\;\Big],\ \ P^\pm=\log\pi(r\mid p^\pm,q)
  \] 

- **RAHF-Dual**（Dual-Model, Supervised）  
  同じクエリに対し、**好ましい応答**で**好モデル**、**好ましくない応答**で**悪モデル**を**別々にSFT**：  

  \[
  \theta_h^*=\arg\max\sum\log\pi(r_h\mid q),\quad \theta_l^*=\arg\max\sum\log\pi(r_l\mid q)
  \] 

## 2) 活性パターンの収集（差ベクトル）
- **同一の応答 \(r\)** に **good/bad 刺激 \(p^+,p^-\)** を連結して入力し、**各層の隠れ状態**を抽出。**好み条件の差**を**層ごと**に計算：  

  \[
  v_l = A_{p^+,\pi,l}-A_{p^-,\pi,l}
  \]  
  （SCITは**同一モデル**、Dualは**好/悪モデル**で取得）

## 3) LoRAで“差”に合わせる（最終モデル構築）
- **LoRA出力**を**元の活性**に足して**差ベクトル方向**へ**MSE整列**：  

  \[
  \mathcal{L}_{\text{Align}}=\left\|A_{p,\pi_{\text{LoRA}},l}-\big(A_{p,\pi_{\text{base}},l}+\alpha v_l\big)\right\|_2^2
  \]  

  係数 \(\alpha\) は介入強度、**対象層（例：10,20,2）** で操作。



# 実験設定（抜粋）
- **データ**：UltraFeedback（好みペア）、ベースは **LLaMA2-7B** を Helpful&Harmless でSFTしたモデル。
- **比較**：**Preferred-SFT / HIR / DPO / RLHF-PPO**。
- **評価**：Open LLM Leaderboard（Arc/HellaSwag/MMLU/TruthfulQA/Winogrande/GSM8k）、**AlpacaEval（GPT-4判定）**、**MT-Bench**、**人手評価**。



# 主な結果

## Open LLM Leaderboard
- **RAHF-SCIT** が**平均+2.33pt**向上（Base比）。**TruthfulQA** で両RAHFが全ベースラインを上回る。

## AlpacaEval（GPT-4審査）
- **勝率**：**RAHF-SCIT 87.44%／RAHF-Dual 86.98%** ＞ DPO 83.68%。

##  MT-Bench
- 8側面中**6側面で最高**、平均でも上位。**推論・ロールプレイ・STEM**で特に強み。

## 人手評価
- **RAHF vs RLHF/DPO/HIR**：**勝>引分>負**で優勢（両方式）。

## アブレーション（RepEのみとの比較）
- **LORRA / LORRA-Pref**（前段整列なしの表現編集）より、**RAHFの前段整列＋差ベクトル学習**が有意に効く。



# 直感で掴む：なぜ効く？
- **同じ応答**でも **good/bad 刺激**で**内部活性が変わる** → その**差分方向**へ**表現を少量のLoRAで寄せる**から、**挙動を低コストに整列**できる。  
- **SCIT**は**同一モデル内**で差を取るため**表現空間のズレが少ない**（Dualより有利な場面あり）。


# 実装メモ（最小構成）
1. **前段整列**：SCIT（対照命令）or Dual（好/悪SFT）。
2. **差ベクトル抽出**：**レスポンス先頭〜一定長**の隠れ状態で \(v_l\) を作る（例：Dualは**先頭64トークン**に限定）。
3. **LoRA整列**：**中間層（例：10,20,2）**に**MSE整列**、\(\alpha\) を**小さすぎ/大きすぎ**ない範囲で探索。



# 限界と注意点
- **スケール未検証**：実証は**7B級**。より大規模SOTAへの拡張は今後の課題。
- **追加パラメータ**：LoRAにより**少量だが追加コスト**。将来は**差ベクトルの直統合**も検討余地。
- **データ依存リスク**：前段整列に使う**好みデータのノイズ**や**命令設計**次第で差ベクトル品質が揺れる（しきい値・層選択・\(\alpha\) のチューニングが重要）。



# まとめ
- **RAHF = “好み刺激で生じる活性差”を捉え、LoRAでそこへ寄せる**。  
- **RLなし**で**実装容易**、**DPO/RLHFと競合**しうる整列手段として有望。まずは**SCIT**から試し、**差ベクトルの抽出長・層・\(\alpha\)** を軽くスイープ、**TruthfulQA/AlpacaEval/MT-Bench**で**自動＋人手評価**するのが実務向けの入り口。



# 参考（論文情報）

- **タイトル**：*Aligning Large Language Models with Human Preferences
through Representation Engineering*
- **著者**：Wenhao Liu, Xiaohua Wang, Muling Wu, Tianlong Li, Changze Lv
Zixuan Ling, Jianhao Zhu, Cenyuan Zhang, Xiaoqing Zheng∗, Xuanjing Huang
- **年**：2025
- **OpenView.net**： [リンクはこちら](https://aclanthology.org/2024.acl-long.572/)