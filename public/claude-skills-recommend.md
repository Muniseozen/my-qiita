---
title: 【2026年版】Claude Skills、まず入れるべきおすすめ5選
tags:
  - Claude
  - ClaudeCode
  - AI
  - 生成AI
  - SKILLS
private: false
updated_at: '2026-11-25T18:19:38+09:00'
id: be39790c577c2587957a
organization_url_name: null
slide: false
ignorePublish: false
posting_campaign_uuid: null
agreed_posting_campaign_term: false
---
# はじめに

Claude Skills(Agent Skills)、使っていますか?

一言でいうと、**Claudeに渡す「手順書」のパッケージ**です。`SKILL.md` というファイルに手順や知識を書いておくと、Claudeが必要な場面でだけ読み込んで、その手順どおりに動いてくれます。毎回長いプロンプトで説明していた定型作業が、「あれやって」の一言で済むようになる仕組みです。

ただ、公式・コミュニティ合わせてスキルが大量にあるので、「結局どれを入れればいいの?」となりがち。この記事では、**まず最初に入れるべき5つ**に絞って紹介します(フロントエンド開発+ドキュメント作成が多い人向けの選定です)。

# 前提:Skillsの使い方は2通り

**Claudeアプリ(Web/デスクトップ)の場合**

1. Settings → 機能 で「コード実行とファイル作成」をON
2. スキルを登録すると、該当する依頼のときにClaudeが自動で参照してくれる

**Claude Codeの場合**

プロジェクトの `.claude/skills/<スキル名>/SKILL.md` に置くだけ。skills CLIを使うと1コマンドで入ります。

```bash
npx skills add <リポジトリURL> --skill <スキル名>
```

# おすすめ5選

## 1. skill-creator — スキルを作るためのスキル

**何ができる?** 「こういうスキルが欲しい」と伝えると、Claudeが質問しながら `SKILL.md` を一緒に組み立ててくれます。

**なぜ最初に入れる?** スキルの書き方(フロントマター、description の書き方、ファイル分割)には作法があり、自己流で書くとClaudeがスキルを見つけてくれないことがあります。最初の1つをこれで作ると、作法ごと学べます。

```bash
npx skills add https://github.com/anthropics/skills --skill skill-creator
```

**使ってみて:** 「Qiita記事を書くときの構成ルール」をスキル化するのに使いました。質問に答えていくだけでSKILL.mdが完成するので、スキル自作のハードルが一気に下がります。

## 2. frontend-design — 「AIが作った感」のないUIを出す

**何ができる?** ClaudeにUIを作らせたときの、あの「いかにもAIが作ったテンプレ感」を減らし、配色・レイアウト・タイポグラフィの質を上げてくれます。

**なぜ入れる?** フロントエンドの人間として、AIの出すUIの「惜しさ」には悩まされてきました。これを入れてからの出力は、たたき台としてそのまま見せられるレベルに近づきます。

```bash
npx skills add https://github.com/anthropics/skills --skill frontend-design
```

## 3. ドキュメント系スキル(docx / xlsx / pptx / pdf)— Office文書の生成・編集

**何ができる?** Word・Excel・PowerPoint・PDFの読み書きを、崩れの少ない状態でやってくれる公式スキル群です。「この内容でWord作って」「このExcelに列を足して」が安定します。

**なぜ入れる?** 職務経歴書、クライアント向け資料、ちょっとした集計表——エンジニアでも文書仕事は意外と多い。この4つを入れておくと、Claudeが「文書作成アシスタント」としても機能するようになります。

```bash
npx skills add https://github.com/anthropics/skills --skill docx
# xlsx / pptx / pdf も同様に
```

## 4. find-skills — スキルを探してくれるスキル

**何ができる?** 「こういうことがしたい」と言うと、**Claudeが自分で使えそうなスキルを検索・提案してくれる**ようになります。スキルのApp Storeを持たせるイメージです。

**なぜ入れる?** スキルのエコシステムは拡大が速く、手動で追いかけるのは無理があります。探す作業自体をClaudeに任せるのが一番ラクです。

```bash
npx skills add https://github.com/vercel-labs/skills --skill find-skills
```

## 5. vercel-react-best-practices — Reactコードの質を底上げ

**何ができる?** Vercelのエンジニアリングチームがまとめた、React/Next.jsのパフォーマンスルール集(ウォーターフォール排除、バンドルサイズ、再レンダリング対策など)に沿ってコードを書く・レビューするようになります。

**なぜ入れる?** AIの書くReactコードは「動くけど無駄な再レンダリングだらけ」になりがち。このスキルを入れておくと、生成段階からベストプラクティスに寄ります。レビューの指摘も減りました。

```bash
npx skills add https://github.com/vercel-labs/agent-skills --skill vercel-react-best-practices
```

# 入れるときの注意:スキルはコードが動く

1つだけ注意を。スキルは中でスクリプトを実行できるため、**信頼できないソースのスキルを安易に入れるのは危険**です。公式(anthropics)や信頼できる組織のリポジトリから入れる、中身のSKILL.mdを一度読んでから使う、くらいの習慣はつけておきましょう。

# まとめ

| スキル | 一言でいうと |
| --- | --- |
| skill-creator | スキルを作るためのスキル。最初の1つに |
| frontend-design | 「AIが作った感」のないUI |
| docx / xlsx / pptx / pdf | Office文書の生成・編集を安定させる |
| find-skills | スキルを探す作業をClaudeに任せる |
| vercel-react-best-practices | ReactコードをVercel流ベストプラクティスに |

Skillsの本領は「自作」にあります。自分の定型作業をスキル化し始めると、Claudeがどんどん自分専用のアシスタントになっていく。まずはこの5つで感覚を掴んで、skill-creatorで最初の1つを作ってみてください。

おすすめのスキルがあれば、ぜひコメントで教えてください。
