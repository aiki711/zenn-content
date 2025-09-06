---
title: "BigFiveを統合してLLMのロールプレイ能力向上に関する論文を一緒に読みましょう！"
emoji: "🐳"
type: "idea" # tech: 技術記事 / idea: アイデア
topics: []
published: false
---


# ORCA: Big Five を統合して LLM のロールプレイ能力を底上げする
> この記事は，「自分の理解を深めたい」という気持ちで書いています．読者のみなさんと**同じ目線**で，一緒に理解を育てていくスタイルです．僕の理解が及ばない部分があれば，優しく教えていただけると幸いです！



# TL;DR
**ORCA**は、ユーザの **Big Five（各6サブ次元=計35次元）** に基づく**性格特性**を、データ拡張と指示調整を通じて **LLM のロールプレイ**に組み込む枠組み。4段構成：①**性格推定**→②**データ拡張（プロフィール/潜在知識/心理活動）**→③**PCIP（性格条件付き指示プロンプト）**でのデータ化→④**PTIT/PSIT**での学習。評価用に **OrcaBench** も新設し、**OrcaData**で学習したモデルがベンチで優位と報告。:contentReference[oaicite:0]{index=0} :contentReference[oaicite:1]{index=1} :contentReference[oaicite:2]{index=2}



# 何が新しい？（貢献）
- **心理学の統合で“キャラ作り”を強化**：LLM を用いてユーザの**Big Five**を推定し（**5次元×各6サブ次元**＝35）、**連続スコア**と**レポート**を得る。これを用い、**プロフィール/潜在知識/心理活動**まで含めた入出力を設計。:contentReference[oaicite:3]{index=3} :contentReference[oaicite:4]{index=4} :contentReference[oaicite:5]{index=5}  
- **二段の学習法**：  
  - **PTIT（explicit）**：性格レポートを**テキストとして前置**し LoRA で調整。:contentReference[oaicite:6]{index=6}  
  - **PSIT（implicit）**：**スコア→説明文**へ写像する **PTSI** を挟み、**連続値**を扱えるように。:contentReference[oaicite:7]{index=7}  
- **評価基盤の整備**：**OrcaBench** を構築（重複しない25ユーザ / 3,758投稿、画像1,782枚）。**重なり（BLEU/ROUGE）・関連（CPR/PTR/PKR）・性格一致（PSS）**で評価。:contentReference[oaicite:8]{index=8} :contentReference[oaicite:9]{index=9}



# データと前処理
- **収集**：X（旧Twitter）から**500ユーザ×直近200投稿**を収集（公開ポストに限定）。画像は**VLM でキャプション化**し多モーダル化。:contentReference[oaicite:10]{index=10}  
- **性格推定**：投稿を10件ずつ分割し、**ゼロショット**でサブ次元を0/1採点→平均して**35次元の連続スコア**を算出。**要約プロンプト**で**性格レポート**も生成。:contentReference[oaicite:11]{index=11}  
- **データ拡張**：  
  1) **プロフィール**（創作だが性格記述は含めない）、  
  2) 投稿背後の**潜在知識（Potential Knowledge）**、  
  3) **心理活動**（ポスト時の内的動機）をLLMで生成・品質判定。:contentReference[oaicite:12]{index=12}



# PCIP：性格条件付きプロンプト設計
- 入力を **I=(instruction, profile, personality, knowledge)**、出力を **O=(activities, text, media)** というタプルで整備。**関連が薄い要素は空欄**のまま学習させ、差異も学ばせる。:contentReference[oaicite:13]{index=13}  
- 実例（抜粋）は Fig.2 に掲載（プロフィール・Big Five レポート・潜在知識を条件に、**心理活動→本分**を生成）。:contentReference[oaicite:14]{index=14}



# 学習：PTIT / PSIT
- **PTIT（明示）**：性格**レポート文**をそのまま条件に使い、**LoRA**で微調整（数式定義あり）。:contentReference[oaicite:15]{index=15}  
- **PSIT（暗黙）**：**スコア列**は LLM 埋め込みとギャップが大きいため、**PTSI** で**説明文**に解釈させてから条件化。:contentReference[oaicite:16]{index=16}



# 評価（OrcaBench）
- **手順**：①PCIP で各モデルに生成させる→②**重なり**（BLEU/ROUGE）・**関連**（CPR/PTR/PKR）・**性格一致**（PSS：コサイン類似）でスコア化。:contentReference[oaicite:17]{index=17}  
- **ベースライン**：Llama 3.1（8B/70B）や **DeepSeek** の **PCIP推論**、および **PTIT/PSIT** 学習モデルを比較。**CoT 的**な中間出力も試す。:contentReference[oaicite:18]{index=18} :contentReference[oaicite:19]{index=19}



# 主な結果

## 1) PCIP（推論のみ）のアブレーション
- **プロフィール除去（CPA）で CPR=7.60**まで低下→**プロフィールは一貫性維持に必須**。:contentReference[oaicite:20]{index=20}  
- **性格特性除去（PTA）で PTR=18.09、PSSも −3.06**→**Big Five の明示が効く**。:contentReference[oaicite:21]{index=21}  
- **潜在知識除去（PKA）で BLEU=18.46/ROUGE-l=8.07**→**話題誘導の鍵は潜在知識**。:contentReference[oaicite:22]{index=22}  
- **心理活動/画像を省く（WPM）**と **PSS +1.42**（PCIP系で）＝一方で**解釈性**は下がるためトレードオフ。:contentReference[oaicite:23]{index=23}

## 2) PTIT/PSIT（学習あり）
- **BLEU/ROUGE と PSS が大幅向上**：例）**PCIP→PTIT**で **BLEU 29.94→55.85, ROUGE-l 18.37→38.76, PSS 91.65→98.11**。**PSIT**も同水準。:contentReference[oaicite:24]{index=24}  
- **心理活動付きでも PSS ほぼ不変**（PTIT vs PTIT-WPM：差0.04）→**学習により“心理活動⇔性格”の対応を獲得**。:contentReference[oaicite:25]{index=25}  
- **スケーリング**：**PTIT-70B**は 8B より高得点（2エポック学習）。:contentReference[oaicite:26]{index=26}



# 実装ノート
- **学習器**：データ構築は **Llama-3.1-70B**、学習は **Llama-3.1-8B**（画像キャプションは **cogVLM2**）。:contentReference[oaicite:27]{index=27}  
- **LoRA 設定**：`r=8, α=32, lr=5e-5, cosine scheduler, cutoff=8192, batch=4 (accum 2), epochs=5（70Bは2）`。:contentReference[oaicite:28]{index=28}



# 限界と注意点
- **ベンチの限界**：SNS上では**神経症傾向（Neuroticism）**が現れにくく、**スコア差が判別困難**。:contentReference[oaicite:29]{index=29}  
- **暗黙モデリングの課題**：**連続スコアの融合**は未踏。PSIT/PTSIは**一案**に過ぎず、より適切な埋め込み/射影が今後の課題。:contentReference[oaicite:30]{index=30}  
- **倫理**：**ロールプレイは越権行為やジェイルブレイクを誘発**し得るため、**モデレーション**が必要。また**性格推定はプライバシー懸念**がある。:contentReference[oaicite:31]{index=31}



# 使いどころのヒント（運用）
- **まずは PCIP で効果測定**→効けば **PTIT/PSIT** で常設化。  
- **設計の勘所**：**プロフィール=一貫性**, **潜在知識=話題整合**, **Big Five=振る舞い様式**。  
- **心理活動**は**解釈性**を上げるが、速度や PSS とのトレードオフに注意（学習すれば両立可）。:contentReference[oaicite:32]{index=32}



# 参考（論文情報）

- **タイトル**：*ORCA: ENHANCING ROLE-PLAYING ABILITIES OFLARGE LANGUAGE MODELS BY INTEGRATING PER-SONALITY TRAITS*
- **著者**：Yuxuan Huang
- **OpenView.net**： [2406.13960v1](https://openreview.net/forum?id=HPuLU6q7xq)