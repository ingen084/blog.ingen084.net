---
title: "MonoGameでのWpfFontPipelineの不具合について"
category: "雑記"
tags: ["過去の投稿", "C#"]
date: 2016-03-23 04:13:30 +09:00

permalink: /posts/20160308-aterm
---

> この投稿は過去のWordPressからの投稿を移植したものです。  
> あくまでこんな事もあった･過去こんな考え方をしていた程度でお受け止めください。  
> 画像が表示されていない部分が存在します。ご了承下さい。

XNA(Monogame)のスプライトフォント生成にものすごく便利なWpfFontPipelineですが、生成されるフォントが必ず太字になる不具合を見つけたのでここに書き記しておきます。  

渡されたスタイルのフラグをand演算をしてフォントの太さなどを決めているようなのですが、  

```
// フォントスタイルの変換  
var fontWeight = ((input.Style & FontDescriptionStyle.Bold) == FontDescriptionStyle.Bold) ? FontWeights.Bold : FontWeights.Regular;
```

何故かわかりませんがMonoGameではスタイルのフラグが  

```
Bold=0b  
Italic=1b  
Regular=10b
```

のように定義されていて、(0とand演算すると絶対に0になっちゃいます)  
たとえRegularが指定されていてもフォントが太字になってしまいます。  
修正は簡単  

```
// フォントスタイルの変換  
var fontWeight = ((input.Style & FontDescriptionStyle.Regular) == FontDescriptionStyle.Regular) ? FontWeights.Regular : FontWeights.Bold;
```

BoldとRegularを反対にするだけです。

生成済みdllは要望があれば配布します？
