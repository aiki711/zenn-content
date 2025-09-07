---
title: "LLMの内部表現を読んで・操る最新研究に関する論文を一緒に読みましょう！"
emoji: "✨"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["LLM","ActSteering","Representation", "Engineering","Interpretability"]
published: false
---

# Representation Engineering入門：LLMの“内部表現”を読んで・操る最新サーベイ（課題つき）
> この記事は，「自分の理解を深めたい」という気持ちで書いています．読者のみなさんと**同じ目線**で，一緒に理解を育てていくスタイルです．僕の理解が及ばない部分があれば，優しく教えていただけると幸いです！

 

# TL;DR
- **Representation Engineering（RE）**は、LLM内部の**高次概念の表現**（truthfulness, harmfulness, style など）を**読み取り（Reading）**、必要に応じて**介入（Control）**して**ふるまいを直接制御**する枠組みの総称。機構解釈の“ボトムアップ”に対し、**“トップダウン”で高次表現を扱う**のがコア発想。:contentReference[oaicite:0]{index=0}  
- 本サーベイは RE の**目的・理論（線形表現仮説／重ね合わせ仮説）**・手法（LAT, Probing, CAA/ActAdd/ITI/Task Vectors ほか）・**評価と副作用**・**研究課題**を体系化。**安全・パーソナライゼーション・性能・真実性**にまたがる応用を横断整理。:contentReference[oaicite:1]{index=1}  
- **課題**は「標準評価の欠如」「一般化と多目標同時制御」「強い線形性仮説の未確立」「多モーダルへの展開」「倫理・デュアルユース」。実務上は**介入強度・層・トークン選択**の最適化と**流暢さ劣化の監視**がカギ。:contentReference[oaicite:2]{index=2}



# なにをサーベイしているか（俯瞰）
- **定義**：RE は「**Representation Reading**（内部表現の特定）→ **Representation Control**（表現を介した出力制御）」の2段。**対照刺激**で活性差を作り、**線形手法**で方向を抽出し、**推論時に加算・スケーリング・写像**などで介入する。:contentReference[oaicite:3]{index=3}
- **位置づけ**：機械学習の XAI/機構解釈/微調整/プロンプト設計との比較も提示（長所＝**即応性・軽量性・意味的一貫性**、短所＝**評価標準の未整備**）。:contentReference[oaicite:4]{index=4}



# 理論のベース
- **Linear Representation Hypothesis（LRH）**：高次概念が**活性空間のほぼ線形な方向**として符号化される、という作業仮説。**弱いLRH（いくつかは線形）**には経験的裏付けが厚いが、**強いLRH（すべて線形）**は未確立。**因果内積**など、方向の妥当性を測る理論化が前進中。:contentReference[oaicite:5]{index=5}
- **Superposition 仮説**：特徴数＞次元数でも**方向の重ね合わせ**で表現するため、**干渉や副作用**が起こり得る＝**多重介入や合成**が難しい理由。:contentReference[oaicite:6]{index=6}



# Representation Reading（どう“見つける”か）
- **LAT（Linear Artificial Tomography）**：対照刺激を与え、活性差の主成分や線形モデルで**概念方向**を抽出。**監督あり（線形プローブ）／なし（PCA 等）**の両系統。:contentReference[oaicite:7]{index=7}
- **Probing**：線形／非線形プローブで**概念→活性**の写像を学び、**真偽・バイアス・知識位置・構文階層**などを検査。**因果パッチ**で「その特徴をモデルが実際に使っているか」を確認する流儀も。:contentReference[oaicite:8]{index=8}



# Representation Control（どう“操る”か）
- **単一／多プロパティ制御**、**定数／動的強度**、**介入箇所**（残差流・注意・MLP・特定ニューロン・複数層）などの設計軸を整理（図表・分類）。**複数目標の同時介入は干渉しやすく、層・強度の分散配置が効く**、という経験則もまとめられる。:contentReference[oaicite:9]{index=9}
- **代表レシピのカタログ**（一部抜粋）
  - **シンプル加算系**：CAA（対照平均差）、ActAdd（単一対照ペア）、RepE（PCA差）、ITI（真偽ヘッド検出＋介入）、LSV（学習済み潜在ベクトル注入）。:contentReference[oaicite:10]{index=10}
  - **最適化・構造化系**：BiPO（選好対で直接最適化）、PaCE（疎コーディングと斜め射影）、**HPR**（ハウスホルダー反射＋回転で**ノルム一貫性**維持）、MiMiC（平均・共分散整形）、SARA/CAST/FLORAIN など。:contentReference[oaicite:11]{index=11}
  - **ダイナミック系**：DAC/ACT/RE-CONTROL/Activation Scaling（入力や生成位相に応じ**強度を自動調整**）。:contentReference[oaicite:12]{index=12}
  - **SAE連携**：SPARE/SAE-TS で**疎特徴**を足場に**副作用を抑えた介入**を設計。:contentReference[oaicite:13]{index=13}
  - **安全・移植**：Circuit Breakers/InferAligner/ConTRANS/LEGEND など、**安全方向の転写・拒否の選択的発火**。:contentReference[oaicite:14]{index=14}



# 応用の4象限
- **Truthfulness（幻覚抑制）**：truth 方向の増幅や選択的介入で**嘘の尤度を下げる**。:contentReference[oaicite:15]{index=15}
- **Security（有害出力防止／赤組）**：**安全方向**の抽出・注入、**回路遮断**、**概念転写**。逆に**安全除去の実証**もあり**デュアルユース**性が高い。:contentReference[oaicite:16]{index=16}
- **Personalization**：**スタイル／価値観**の多様化を**微小介入で実現**。**複数ベクトルの合成**は要注意。:contentReference[oaicite:17]{index=17}
- **Performance**：Task Vectors/SCANS/EAST などで**狙い撃ちの性能改善**（CoT, コード, 選択的探索など）。:contentReference[oaicite:18]{index=18}



# 評価：何を測るべきか（落とし穴つき）
- **開放生成での有効性**：多肢選択だけだと効いて見えても、**自由生成では崩れる**ことがある。**トークン確率・因果検証**も合わせる評価パイプラインを推奨。:contentReference[oaicite:19]{index=19}
- **流暢さ（副作用）**：介入が強すぎると**意味連続性の崩壊・確率分布の歪み**が起こるため、**強度・層・トークン選択**の調整が必須。:contentReference[oaicite:20]{index=20}



# 研究課題（Open Problems）
1. **標準評価の欠如**：データセット・指標・設定がバラバラ。**開放生成＋尤度＋因果検証**の標準化が必要。:contentReference[oaicite:21]{index=21}  
2. **理論の未整備**：LRH の適用範囲、**最適層・強度**の理論導出、**干渉解析**が未成熟。:contentReference[oaicite:22]{index=22}  
3. **一般化と反作用**：**反ステア（逆効果）例**や**O.O.D.**で効かないケースの体系理解。:contentReference[oaicite:23]{index=23}  
4. **マルチモーダル展開**：**トークナイズ／量子化の非連続性**、**方向の意味解釈**、**非線形相互作用**への対処。:contentReference[oaicite:24]{index=24}  
5. **倫理・デュアルユース**：**安全ガードの剥離**・**属性偏りの増幅**の危険。**検知・不可逆化**の研究が重要。:contentReference[oaicite:25]{index=25}



# 実務Tips（とりあえず動かす）
- **Reading**：小規模でも良いので**対照刺激**で差を作り、**中間層×先頭側トークン**で方向抽出（PCA or 線形プローブ）。:contentReference[oaicite:26]{index=26}  
- **Control**：まずは**単層・小強度の加算**（CAA/ActAdd）→**ダイナミック強度**（DAC/Scaling）→**多目標は層分散**で干渉低減。:contentReference[oaicite:27]{index=27}  
- **評価**：目的指標＋**流暢さ（困惑度/エントロピー等）**＋**トークン尤度**＋**因果パッチ**。**多肢→自由生成**の順で最終確認。:contentReference[oaicite:28]{index=28}  
- **安全**：Guard 方向は**挿入も除去も可能**＝**運用ポリシーと監査**をセットで。:contentReference[oaicite:29]{index=29}



# まとめ
- **RE は“LLMの内部表現を直接いじる”実践的な上位概念**。**軽量・即効**で、**整列・安全・個人化・性能**に効くが、**評価と理論、複合制御**は未成熟。  
- **当面の最適解**は：**小さく読む→中間層に薄く効かせる→副作用を多視点で検査**。**標準化された評価パイプライン**が次の鍵。:contentReference[oaicite:30]{index=30}



# 参考（論文情報）

- **タイトル**：*Representation Engineering for Large-Language Models: Survey and
Research Challenges*
- **著者**：Lukasz Bartoszcze, Sarthak Munshi, Bryan Sukidi, Jennifer Yen, Zejia Yang, David Williams-King, Linh Le, Kosi Asuzu, Carsten Maple
- **年**：2025
- **arXiv**： [2502.17601v1](https://arxiv.org/abs/2502.17601v1)