---
title: "気象庁のコード表を自動で更新する"
category: "プログラミング"
tags: []
date: 2023-12-03 00:00:00 +09:00
---

こんばんは。  
これは [防災アプリ開発 Advent Calendar 2023](https://adventar.org/calendars/9301) の3日目の記事です。

## はじめに

みなさんは気象庁から配信される情報を処理する上でコードの扱いはどうしていますか？  
XML の電文であれば値が記述されているためコードを無視したりすることも可能ですが、災危通報などを表示したり、読み上げを実装するときはそうも言っていられません。  
更新のメールが来たら(or ページの更新を確認したら)内容を読んで、影響しそうだったら手元のデータもコード表を元に更新して…。  
大変ですね。

今回はそのコード表を自動で更新できるようにしよう！という趣旨の記事です。

[成果物はここににあります](https://github.com/ingen084/jma-code-dictionary)。

## 目指すもの･利用想定

開発言語は様々だと思いますが、僕の場合は C# で、Dictionary 形式で参照できると良さそうです。

```cs
AreaEpicenter[100] // 石狩地方北部
```

が、C# で開発している人ばかりではないので汎用的に利用できる形式である CSV としても出力できるようにしてみることにします。

## 自動で更新される CSV を作る

コード表は zip アーカイブ内の xls ファイルに格納されています。  
zip アーカイブは更新のたびにファイル名が変更されるので、リンクが設置されているページを読み込んだ上でコード表へのリンクを探す必要があります。

流れとしては

1. [zip ファイルへのリンクが設置されているページ](http://xml.kishou.go.jp/tec_material.html)を読み込む
1. 個別コード表のリンクを探してダウンロードする
1. zip ファイル内の xls ファイルから CSV ファイルを作成する

という感じになりそうです。

### zip ファイルのリンクを探す

リンクを探すためにはHTMLの解析が必要です。  
今回は AngleSharp を使用し、`a` タグのうち `個別コード表` の記載がある物の `href` 属性を取得してみます。

```cs
(await parser.ParseDocumentAsync(new MemoryStream(pageContent), CancellationToken.None))
    .QuerySelectorAll("a").Where(a => a.TextContent.Contains("個別コード表")).Select(a => a.GetAttribute("href")).First()
```

ワンライナー！素敵ですね。

### xls ファイルの読み込み･書き出し

そんなこんなでダウンロードしてきた zip ファイルの中を舐めて見つけた xls ファイルを片っ端から読んでいきます。

```cs
using var zipStream = await zipResponse.Content.ReadAsStreamAsync();
using var zipArchive = new ZipArchive(zipStream, ZipArchiveMode.Read, false, Encoding.GetEncoding("Shift_JIS"));

foreach (var entry in zipArchive.Entries)
{
    Console.WriteLine(entry.FullName);
    if (!entry.FullName.EndsWith(".xls"))
        continue;

    // 直接 Zip から読み込むとエラーになるので一旦メモリに展開
    using var memStream = new MemoryStream();
    using (var cStream = entry.Open())
        cStream.CopyTo(memStream);
    memStream.Seek(0, SeekOrigin.Begin);
    try
    {
        using var workbook = WorkbookFactory.Create(memStream, true);
```

xls ファイルの読み込みは仕事でも使った実績があったので NPOI というライブラリを使用しました。  
が、`PhenologicalType.xls` だけフォーマットが古かったため読み込めず、ライブラリを変更することも検討しましたが今回はそもそも変換対象から除くこととしました。ごめんね。

各ファイルでデザインやフォーマットが異なったりしているため、自動で一括変換は行わず、 `シート名` `CSVファイル名` `表の開始座標` `カラム数` という4つのパラメータを指定してCSVを出力するメソッドを作成し各出力対象ごとに定義を行いました。  
なので項目に変化がある場合は自動で更新されますが、新しく表が増えた場合は手動でコードを変更する必要があります。ちょっと詐欺ですね。

しかしアメダスの観測点のシートだけ汎用化できず個別処理となったシートがありました。  
しっかり表形式になっておらず、管理している気象台ごとに空行が差し込まれており、これを別カラムとして独立させるためだけに個別処理を行うようにしました。

|観測点名|観測点コード|...|
|:--|:--|:--|
|○○気象台管理|||
|XXX|123456|...|
|...|||
|○○気象台管理|||
|YYY|456789|...|
|...|||

## 自動実行の仕組みを作る

CSV は作成できましたが、これを自動で実行する仕組みを用意する必要があります。  
また変更履歴などもわかりやすいよう、CSV ファイル自体も git で管理したかったことなどから、更新ツールのソースコードも含んだ状態で GitHub 上にリポジトリとして作成し、GitHub Actions を定期的に実行することで更新をさせることにしました。

```yaml
name: update

on:
  schedule:
    - cron: "0 10 * * *"
  workflow_dispatch:

jobs:
  run:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v2
      - name: Setup .NET
        uses: actions/setup-dotnet@v1
        with:
          dotnet-version: 8.0.x
      - name: Run
        id: run
        shell: bash -x {0}
        run: |
```

ワークフローの定義自体はとても簡単で、

- 毎日10時0分(UTC)もしくは手動での実行をトリガーとして
- Ubuntu 環境で
- .NET 8 をセットアップして
- 今回作ったツールを動かす

といった内容になっています。

### 更新検知･自動プッシュ

こうして毎日実行できるようになりましたが、毎日ダウンロードするようではいろいろともったいないので、差分がない場合ダウンロードを行わないようにしてみます。  
まず防災情報XML同様 `Last-Modified` ヘッダーと `If-Modified-Since` ヘッダーを使って `304 Not Modified` が帰ってきたらページが更新されていないと見なします。  
普通にレスポンスが返ってきたらページが更新されたとみなしそのレスポンスを使って処理を行うことにしました。

そして更新があったときには自動でコミット･プッシュを行いリリースも作ってみることにします。

```bash
if (git diff --shortstat | grep '[0-9]'); then
```

git の差分があることを確認したら、

```bash
  git remote set-url origin https://github-actions:${GITHUB_TOKEN}@github.com/${GITHUB_REPOSITORY}
  git config --global user.email "ingen188@gmail.com"
  git config --global user.name "ingen084"
```

actions からプッシュするための設定をして

```bash
  if (git diff --name-only | grep csv); then
    CSV_UPDATED=true
  fi
```

CSV が更新されたときのみリリースを作成したいので差分にCSVが含まれていたらフラグを立てます。

```bash
  git add ../../csv ../../.cache
  git commit -m "Update `date '+%Y.%m.%d'`"
  git push
  if [ "$CSV_UPDATED" = true ]; then
    gh release create `date '+%Y.%m.%d'`
  fi
fi
```

でコミットしてプッシュ。CSVが更新されていたら gh コマンドでリリースを作成するという流れです。  
でも動作確認ちゃんとやってないので多分動きません。壊れてたら直しておきます…。

## SourceGenerator を作る

この CSV を元に、C# のコードを生成します。  
しかし、単純に KeyValue 以外の需要も多そうだったためどのような物を作るのかが定まらず、一旦僕の作っているアプリ(KyoshinEewViewer for ingen)で使うためとして作成してみることにしました。

C# には C#9 から Source Generator 、 C#10 から Incremental Source Generator というコードジェネレータが存在しており、コンパイラの動作にフックして動的に C# のコードを生成することができる機能があります。

今回の場合は特にコード内の記述に応じて生成するわけではないので古い方の Source Generator を使用しました。  
Source Generator のサンプルとして [CSV からファイルを生成する](https://github.com/dotnet/roslyn-sdk/blob/main/samples/CSharp/SourceGenerators/SourceGeneratorSamples/CsvGenerator.cs) というまんまそのままのものがあったのでその形を利用しつつ今回の目標に合わせてカスタマイズしてみました。

具体的にはそもそも Dictionary 形式にしたかったのと、1つのクラスにしたかったので内部の構造を大きく変更しています。とはいっても小さいソースコードですが。

```cs
using Microsoft.CodeAnalysis.Text;
using Microsoft.CodeAnalysis;
using System.Text;

namespace KyoshinEewViewer.CsvSourceGenerator;

[Generator(LanguageNames.CSharp)]
public partial class CsvDictionaryGenerator : ISourceGenerator
{
    public static string GenerateClassFile(StringBuilder sb, string dictionaryName, string csvText, string keyType, string keyFormat, string valueType, string valueFormat)
    {
        using var reader = new StringReader(csvText);

        sb.AppendLine($"        public static System.Collections.Generic.IReadOnlyDictionary<{keyType}, {valueType}> {dictionaryName} {{ "{{" }} get; }} = new System.Collections.Generic.Dictionary<{keyType}, {valueType}>(){{ "{{" }}");

        while (true)
        {
            var line = reader.ReadLine();
            if (line == null) break;
            var fields = line.Split(',');
            sb.AppendLine($"            {{ "{{" }} {string.Format(keyFormat, fields)}, {string.Format(valueFormat, fields)} }},");
        }

        sb.AppendLine("        };");

        return sb.ToString();
    }
    private static StringBuilder SourceFilesFromAdditionalFiles(IEnumerable<(AdditionalText file, string keyType, string keyFormat, string valueType, string valueFormat)> pathsData)
    {
        var sb = new StringBuilder();
        sb.AppendLine(@"
#nullable enable
namespace KyoshinEewViewer {
    public static class CsvDictionary {");
        foreach (var (file, keyType, keyFormat, valueType, valueFormat) in pathsData)
        {
            var className = Path.GetFileNameWithoutExtension(file.Path);
            var csvText = file.GetText()!.ToString();
            GenerateClassFile(sb, className, csvText, keyType, keyFormat, valueType, valueFormat);
        }
        sb.AppendLine("    }\r\n}");
        return sb;
    }

    private static IEnumerable<(AdditionalText, string, string, string, string)> GetLoadOptions(GeneratorExecutionContext context)
    {
        foreach (var file in context.AdditionalFiles)
        {
            if (!Path.GetExtension(file.Path).Equals(".csv", StringComparison.OrdinalIgnoreCase))
                continue;

            context.AnalyzerConfigOptions.GetOptions(file).TryGetValue("build_metadata.Additionalfiles.KeyType", out var keyType);
            context.AnalyzerConfigOptions.GetOptions(file).TryGetValue("build_metadata.Additionalfiles.KeyFormat", out var keyFormat);
            context.AnalyzerConfigOptions.GetOptions(file).TryGetValue("build_metadata.Additionalfiles.ValueType", out var valueType);
            context.AnalyzerConfigOptions.GetOptions(file).TryGetValue("build_metadata.Additionalfiles.ValueFormat", out var valueFormat);

            if (keyType == null)
                throw new Exception("KeyType is not defined.");
            if (keyFormat == null)
                throw new Exception("KeyFormat is not defined.");

            if (valueType == null)
                throw new Exception("ValueType is not defined.");
            if (valueFormat == null)
                throw new Exception("ValueFormat is not defined.");

            yield return (file, keyType, keyFormat, valueType, valueFormat);
        }
    }

    public void Execute(GeneratorExecutionContext context)
        => context.AddSource($"CsvDictionary.g.cs", SourceText.From(SourceFilesFromAdditionalFiles(GetLoadOptions(context)).ToString(), Encoding.UTF8));

    public void Initialize(GeneratorInitializationContext context)
    {
    }
}
```

今回作成した物は特に生成された気象庁の CSV ファイル限定というわけではなく汎用的に利用できる物ですが、テキトーに作ったのもありコンテンツにカンマが入っていると死にます(死にます)。  
なのでちゃんと使いたい方はライブラリを使うか、仕様とにらめっこしながらパーサを自分で実装できるとよいでしょう。

### 仕様･利用例

使用するときは csproj を直接編集します。  
まずは変換パラメータをコンパイラに認識させるために変換パラメータの記載を行います。

```xml
<ItemGroup>
    <CompilerVisibleItemMetadata Include="AdditionalFiles" MetadataName="KeyType" />
    <CompilerVisibleItemMetadata Include="AdditionalFiles" MetadataName="KeyFormat" />
    <CompilerVisibleItemMetadata Include="AdditionalFiles" MetadataName="ValueType" />
    <CompilerVisibleItemMetadata Include="AdditionalFiles" MetadataName="ValueFormat" />
</ItemGroup>
```

その上で変換対象となる CSV ファイルを `AdditionalFiles` を使用して指定します。  
csv のリポジトリを submodule として追加しておくと更新も楽そうです。

```xml
<ItemGroup>
    <Folder Include="csv\" />
    <AdditionalFiles Include="..\..\jma-code-dictionary\csv\AdditionalCommentEarthquake.csv"
                     Link="csv\AdditionalCommentEarthquake.csv"
                     KeyType="int" KeyFormat="{0}"
                     ValueType="string" ValueFormat="&quot;{1}&quot;" />
</ItemGroup>
```

今回作った Source Generator は `AdditionalFiles` の属性でファイルごとに設定を行えるようにしています。

コード生成はキーに `KeyType` 、 値に `ValueType` の型が指定された静的な Dictionary を初期化します。  
初期化に合わせて CSV 上の項目が記載されますが、そのときのコード上の表記が `KeyFormat` 、 `ValueFormat` で整形されます。

上の例からは以下のようなコードが生成されます。

```cs
public static System.Collections.Generic.IReadOnlyDictionary<int, string> AdditionalCommentEarthquake { get; } = new System.Collections.Generic.Dictionary<int, string>(){
    { 0101, "今後若干の海面変動があるかもしれません。" },
    { 0102, "今後若干の海面変動があるかもしれませんが、被害の心配はありません。" },
    { 0103, "今後もしばらく海面変動が続くと思われます。" },
    { 0104, "今後もしばらく海面変動が続くと思われますので、海水浴や磯釣り等を行う際は注意してください。" },
    { 0105, "今後もしばらく海面変動が続くと思われますので、磯釣り等を行う際は注意してください。" },
    以下略
```

コード生成の都合本当に C# のコードとして解釈することができるので、値を ValueTuple にして複数のカラムを含めることもできます。

```xml
<AdditionalFiles Include="..\..\jma-code-dictionary\csv\AreaForecastLocalEEW.csv"
                  Link="csv\AreaForecastLocalEEW.csv"
                  KeyType="int" KeyFormat="{0}"
                  ValueType="(string Name, string NameKana)" ValueFormat="(&quot;{1}&quot;, &quot;{2}&quot;)" />
```

このように記述すると、

```cs
public static System.Collections.Generic.IReadOnlyDictionary<int, (string Name, string NameKana)> AreaForecastLocalEEW { get; } = new System.Collections.Generic.Dictionary<int, (string Name, string NameKana)>(){
    { 9011, ("北海道道央", "ほっかいどうどうおう") },
    { 9012, ("北海道道南", "ほっかいどうどうなん") },
    { 9013, ("北海道道北", "ほっかいどうどうほく") },
    { 9014, ("北海道道東", "ほっかいどうどうとう") },
    以下略
```

このような生成も可能になっています。  
まだ需要を感じていないので作っていませんが、同一の CSV ファイルを元に複数の Dictionary を生成できる機能を追加してみてもいいかもしれません。

## さいごに

需要が読みきれず完全に自動化するというのは達成できませんでした…。  
どのように C# から参照できるデータとして入れてるのがいいのかが思いつきませんでした。  
案外 SQLite に固めてしまうとかも使いやすいのかもしれませんね。欲しい方がいたら Issue とかでご連絡いただけると追加できると思います。

頭の方にも記載してありますが、[成果物はここににあります](https://github.com/ingen084/jma-code-dictionary)ので、どんどん使っていただけたらと思います。  
参考にしてオリジナルの仕組みを構築してみてもよいかと思います。

今年のアドベントカレンダーは2回(+会社のに1回)書く予定です。  
次に投稿する9日目の記事はマップを描画するにあたっての現時点での工夫をすべて話してしまいたいと思いますのでよろしくお願いいたします。
