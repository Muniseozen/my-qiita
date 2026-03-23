---
title: 'Qiitaの記事をGitHubで管理する【手順まとめ】'
tags:
  - name: 'GitHub'
  - name: '手順書'
  - name: '自動化'
  - name: 'バージョン管理'
  - name: 'Qiita記事'
private: false
updated_at: ''
id: '542c3504397a48ab7246'
organization_url_name: null
slide: false
---

この記事では、**ターミナルをほぼ使わずに**
Qiitaの記事をGitHubで管理する方法を紹介します。

1. [GitHubでリポジトリを作成](#1-githubでリポジトリを作成)
2. [GitHub Desktopでリポジトリをクローン](#2-github-desktopでリポジトリをクローン)
3. [publicフォルダを作成](#3-publicフォルダを作成)
4. [Markdownで記事を書く](#4-markdownで記事を書く)
5. [GitHub Desktopでpush](#5-github-desktopでpush)
6. [Qiitaに記事が反映される](#6-qiitaに記事が反映される)

---

# はじめに

Qiitaの記事はブラウザから直接書くこともできますが、**GitHubで管理すると次のメリットがあります。**

- Markdownで記事を書ける
- GitHubで履歴管理できる
- バックアップになる
- チームレビューができる

今回は **GitHub Desktopを使って、ターミナルなしで管理する方法** を紹介します。

# 1. GitHubでリポジトリを作成

まず、Qiita記事を管理するための **GitHubリポジトリ** を作成します。

![my-Qiita.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/4257135/decc3b43-c1c3-4581-bb12-0d1ef84fbc54.png)

設定は次のようにします。

| 項目 | 日本語 | 設定 |
|---|---|---|
| Repository name | リポジトリー名 | my-qiita |
| Description | 説明 | Source repository for my Qiita articles | 
| Visibility | 公開・非公開 | Public |
| Add README | README追加 | Off |
| Add .gitignore | .gitignoreの追加 | None |
| License | ライセンス | None |

設定が完了したら **Create repository** をクリックします。

---

# 2. GitHub Desktopでリポジトリをクローン

次に、作成したリポジトリを **GitHub Desktop** を使ってローカル環境にクローンします。

GitHub Desktopを開き、以下の手順で進めます。
リポジトリ一覧から、先ほど作成した **my-qiita** を選択します。

![github-desktop-clone.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/4257135/8f753888-94a9-4eca-bd22-e9fa1d0ddae2.png)

保存先のフォルダを選択し、**Clone** をクリックします。

![スクリーンショット 2026-03-08 0.26.48.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/4257135/5b02cfc5-c6db-4b2c-a764-f0f3febba5fd.png)

これでローカル環境にリポジトリがダウンロードされます。

---

# 3. publicフォルダを作成

GitHub Desktopでクローンしたフォルダを開き、その中に Qiitaの記事を保存するための**publicフォルダ** を作成します。

![public-folder.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/4257135/b0588931-16b0-408e-8ed9-73097c2b9e18.png)

フォルダ構成は次のようになります。
my-qiita
└ public

---

# 4. Markdownファイルを作成

次に、Qiitaの記事となる **Markdownファイル** を作成します。

今回は編集しやすいように **VS Code** を使用します。

まだ **VS Codeをインストール** していない場合はこちらからダウンロードできます。

https://code.visualstudio.com/

#### VS Codeでフォルダを開く

VS Codeを起動し、GitHub Desktopでクローンした **my-qiita** フォルダを開きます。

![open-folder.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/4257135/e369d9fa-cc5d-44c1-bd8a-3a003f988a43.png)

フォルダ一覧から **my-qiita** を選択します。

![my-qiita-folder-vs.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/4257135/79ee6564-8091-4b89-ab45-95887af7616c.png)

#### Markdownファイルを作成

左側のエクスプローラーで `public` フォルダを選択し、  
**新しいファイル** を作成します。

![add-first-post-md.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/4257135/8ee5aabf-2b42-47b6-b6dc-2650c9440c92.png)

⚠️ **テンプレートを作っておくと便利** ⚠️

毎回同じフォーマットを書くのは大変なので、
template.md のようなテンプレートを用意しておくと便利です。

my-qiita
 └ public
    　 ├ template.md
    　 └ first-post.md (投稿１つ目)
      
新しい記事を書くときは `template.md` をコピーして使います。