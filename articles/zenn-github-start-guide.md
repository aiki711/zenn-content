---
title: "Zennではじめる技術ブログ：GitHub連携と初期リポジトリ構成，最初のmd雛形"
emoji: "💬"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["zenn", "github"]
published: true
---


## TL;DR

- **Zenn × GitHub 連携**でローカル執筆→Pushだけで公開できる．([Zenn](https://zenn.dev/zenn/articles/connect-to-github?utm_source=chatgpt.com "アカウントにGitHubリポジトリを連携してZennのコンテンツを ..."))
    
- **zenn-cli** を入れて `zenn init` → `zenn new:article` → `zenn preview` の流れで執筆．([Zenn](https://zenn.dev/zenn/articles/zenn-cli-guide?utm_source=chatgpt.com "Zenn CLIで記事・本を管理する方法"))
    
- 記事の **Front Matter**（title/emoji/type/topics/published）をテンプレで固定化．([Zenn](https://zenn.dev/zenn/articles/zenn-cli-guide?utm_source=chatgpt.com "Zenn CLIで記事・本を管理する方法"))
    


## 1. なぜ Zenn + GitHub？

- Webエディタもあるけれど，**GitHub連携**すると普段のエディタで書いてPushするだけで公開でき，履歴も残るから．([Zenn](https://zenn.dev/zenn/articles/connect-to-github?utm_source=chatgpt.com "アカウントにGitHubリポジトリを連携してZennのコンテンツを ..."))
    


## 2. 事前準備（5分）

- Node.js / npm を用意　←([Node.js](https://nodejs.org/ja/download))
    1. **Node.js LTS（推奨版）** をインストール  
    Windows向けの **.msi 64-bit** を実行 → 既定のまま進めれば OK（npm も同時に入る）．
    2. **ターミナルを開き直す**（PowerShell/コマンドプロンプト/VSCodeのターミナルを完全に閉じて再起動）．
    3. **動作確認**
        ```
        node -v npm -v
        ```

- Git for Windowsをインストール （参考 [Gitのインストール方法(Windows版)](https://qiita.com/takeru-hirai/items/4fbe6593d42f9a844b1c)）
1. ローカルをGit管理にする
   作業フォルダ（例：`zenn-contents`）で：
   `git init git add . git commit -m "chore: init zenn (cli+articles)"`
   > `zenn init` 済みなら `articles/` と `books/`、`README.md`、`.gitignore` などができています。[Zenn](https://zenn.dev/zenn/articles/install-zenn-cli?utm_source=chatgpt.com)

2. GitHubで空のリポジトリ（例：`zenn-contents`）を作成（Public推奨）
   
3. リモート登録 → 初回Push
   HTTPSでつなぐ（一般的）
   `git branch -M main git remote add origin https://github.com/<USER>/<REPO>.git git push -u origin main`
   - 2021/08/13以降，**パスワードは使えません**．プロンプトの“Password”には **Personal Access Token (PAT)** を入力します（作成は Settings → Developer settings → _Personal access tokens_）．[Stack Overflow](https://stackoverflow.com/questions/68775869/message-support-for-password-authentication-was-removed?utm_source=chatgpt.com)[GitHub Docs](https://docs.github.com/en/authentication/keeping-your-account-and-data-secure/managing-your-personal-access-tokens?utm_source=chatgpt.com) 
   -  以降は `git push` だけでOK．資格情報はWindowsの**Credential Manager**に保存されます．入れ替えたい時はそこでトークンを更新．[Super User](https://superuser.com/questions/1309196/how-to-update-authentication-token-for-a-git-remote?utm_source=chatgpt.com)
    


## 3. zenn-cli を入れて初期化

ローカルの作業ディレクトリで実行：

```bash
# 1) プロジェクト作成（任意）
mkdir zenn-contents && cd zenn-contents
npm init -y

# 2) zenn-cli を dev 依存で導入
npm i -D zenn-cli

# 3) 初期化（articles / books ができる）
npx zenn init

# 4) ローカルプレビュー（http://localhost:8000）
npx zenn preview
```

- `zenn init` で **articles/** と **books/** が生成され，以後 CLI で記事を作れる．([Zenn](https://zenn.dev/zenn/articles/zenn-cli-guide?utm_source=chatgpt.com "Zenn CLIで記事・本を管理する方法"))
    
- `npx zenn init` が見つからない場合は **先に zenn-cli のインストール**が必要．([Zenn](https://zenn.dev/zenn/articles/install-zenn-cli?utm_source=chatgpt.com "Zenn CLIをインストールする"))
    


## 4. 最初の記事を作る

```bash
# スラッグとタイトルを指定して作成
npx zenn new:article --slug start-zenn --title "Zennの始め方とGitHub連携"
# => articles/start-zenn.md が生成
```

- 生成されたmdの先頭には **Front Matter** が入り，タイトル・トピック・公開設定を管理する．([Zenn](https://zenn.dev/zenn/articles/zenn-cli-guide?utm_source=chatgpt.com "Zenn CLIで記事・本を管理する方法"))
    


## 5. リポジトリ構成（初期形）

```
zenn-contents/
├─ articles/
│  └─ start-zenn.md         # 最初の記事
├─ books/                   # （書籍化する場合に使用）
├─ images/                  # 画像置き場（任意）
├─ package.json
└─ node_modules/ ...
```

> 補足：Publication（複数人運用）では `articles/` や `images/` の配置ルールがある．個人運用の基本もこの並び．([Zenn](https://zenn.dev/zenn/articles/connect-to-github-publication?utm_source=chatgpt.com "PublicationにGitHubリポジトリを連携してZennのコンテンツを ..."))


## 6. GitHub に Push → Zenn と連携

```bash
git add .
git commit -m "chore: init zenn"
git branch -M main
git remote add origin <your-repo-url>
git push -u origin main
```

**Zenn 側の手順（数クリック）**

1. Zennにログイン → アイコンメニューから **GitHub 連携 / GitHubからデプロイ** を選択
    
2. 連携したいリポジトリを選ぶ（必要権限の許可は “選択したリポジトリのみ” が安全）
    
3. 以後、該当リポジトリにPushすると Zenn が自動で取り込み・公開します。([Zenn](https://zenn.dev/zenn/articles/connect-to-github?utm_source=chatgpt.com "アカウントにGitHubリポジトリを連携してZennのコンテンツを ..."))
    


## 7. 記事の md 雛形（そのまま上書きOK）

> `articles/start-zenn.md` の中身を以下に置き換えれば、そのまま初回投稿として使える．

```md
---
title: "Zennではじめる技術ブログ：GitHub連携と初期構成、最初のmd雛形"
emoji: "📝"
type: "tech"        # tech（技術）/ idea（アイデア）
topics: ["zenn", "github", "blog"]
published: true     # 下書きは false
---

## TL;DR
- ZennとGitHubを連携して、ローカル執筆→Pushで公開．
- `zenn-cli` の `init/new:article/preview` の3コマンドだけ覚えればOK．
- 本記事の雛形を自分用テンプレにして回す．

## なにをやる？
- zenn-cli の導入と初期化
- リポジトリの構成と運用
- 最初のmdを用意（このページ自体が雛形）

## 手順メモ
1. `npm i -D zenn-cli && npx zenn init`
2. `npx zenn new:article --slug start-zenn --title "Zennの始め方とGitHub連携"`
3. `npx zenn preview` でローカル確認（http://localhost:8000）
4. GitHubにPush → Zennで連携 → 自動反映

## 運用ルール（自分用）
- Front Matterは `title/emoji/type/topics/published` を毎回埋める
- 1記事1テーマ（図は最大2枚）
- 末尾に参考リンク（一次情報）を置く

## 参考
- Zenn公式：GitHub連携（デプロイ）  
- Zenn公式：zenn-cliガイド（Front Matter・コマンド）
```

> Front Matter の各キー（`title/emoji/type/topics/published`）は Zenn 公式が定義する基本セット．**type** は `tech`（技術）/`idea`（アイデア）の2択です．([Zenn](https://zenn.dev/zenn/articles/zenn-cli-guide?utm_source=chatgpt.com "Zenn CLIで記事・本を管理する方法"))


## 8. 下書きから投稿，記事の修正

基本の流れは (1) 状態確認 → (2) 追加（ステージ）→ (3) コミット → (4) プッシュ 

コマンドでやる場合（PowerShell/ターミナル）
0. いまどこが変わってるか確認
```bash
git status
```

1. 変更をステージ（全部なら .、一部ならファイル名）
```bash
git add .
# 例: git add articles/zenn-github-start-guide.md
```

2. スナップショットとして保存（メッセージ必須）
```bash
git commit -m "docs: 初回記事を追加"
```

3. GitHubへ送る（初回は -u で追跡設定）
```bash
git push -u origin main
# 2回目以降は: git push
```

初回で名前/メール未設定なら
```bash
git config --global user.name "Your Name"
git config --global user.email "you@example.com"
```
```bash
#よく使う確認コマンド
git log --oneline -n 5   # 直近のコミット履歴
git status               # 変更/ステージ状況

#うっかり対処
git restore --staged <file>      # ステージから外す
git commit --amend               # 直前のコミットメッセージを修正
```

## 9. うまくいかない時のチェックリスト

- `npm i -D zenn-cli` を実行しているか（`npx zenn init` が見つからない場合）([Zenn](https://zenn.dev/zenn/articles/install-zenn-cli?utm_source=chatgpt.com "Zenn CLIをインストールする"))
    
- リポジトリに **articles/** があり，md が置かれているか（空だと何も出ない）([Zenn](https://zenn.dev/zenn/articles/zenn-cli-guide?utm_source=chatgpt.com "Zenn CLIで記事・本を管理する方法"))
    
- Zenn 側で **GitHub連携** を済ませたか（権限は対象リポジトリを選択）([Zenn](https://zenn.dev/zenn/articles/connect-to-github?utm_source=chatgpt.com "アカウントにGitHubリポジトリを連携してZennのコンテンツを ..."))
    


## 10. 次の一手（おすすめ）

- `images/` を作って画像もGitで管理
    
- 記事テンプレを自分用に拡張（TL;DR／要旨／気づき／参考）
    
- 必要なら Qiita に“要点版”をクロスポスト（見つけてもらいやすく）
    

## 参考
- Zenn公式：GitHub連携（デプロイ）  
- Zenn公式：zenn-cliガイド（Front Matter・コマンド）