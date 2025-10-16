---
title: "Kong Gateway Certified Associate 合格体験記"
emoji: "🦍"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["kong"]
published: false
---

## はじめに

[Kong Gateway Certified Associate](https://konghq.com/academy/exam-preparation) という Kong Gateway のための資格を受けてみました。タイトルにある通り、ちゃんと合格したので書けそうな範囲でやったことや諸注意をまとめてみたいと思います。別の日本語記事に以下があるのですが、なるべく重複しないように書きたいと思います。

https://qiita.com/ipppppei/items/0bde08881d836d2e6a54

## やったこと

[Kong Academy](https://education.konghq.com/) という学習コンテンツが提供されているので基本的にはここから提供されている以下のコンテンツを行いました。

- [(KGAC-101) Kong Gateway Foundation](https://education.konghq.com/catalog/learning-paths/73798)
- [(KGAC-201) Kong Gateway Operations Learning Path](https://education.konghq.com/catalog/learning-paths/63587)

Kong Gateway Foundation の方は、一般的な API Gateway の概要や Kong Gateway の基礎的なことが学習できます。所々、詳細を知りたくなったら[ドキュメント](https://docs.jp.konghq.com/gateway/2.8.x/)を読みに行くと良いと思います。

:::message
学習後半になって気がついたのですが、Kong Academy から提供されているコンテンツは、Kong Gateway 2.8.x が対象となっています。最新版は、2025/06 現在で 3.10.x のためドキュメントを読む際はその辺りを注意すると良いでしょう。
:::

Kong Gateway Operations Learning Path の方は、Foundation に比べてもう少し高度な内容を扱います。具体的には、インストールやアップグレードの方法・各種プラグインの設定方法などです。動画で学習した後に、ブラウザで完結するハンズオン環境があるので、手を動かして学習することができます。各項目の最後にミニテストがあるのですが、動画やハンズオン環境で学習したことだけでは中々正解するのが難しいと感じました。都度ドキュメントを参照すると良いと思います。
