---
title: "C#でBUFRを読んでみる(ヘッダーのみ)"
category: "プログラミング"
tags: ["設計"]
date: 2024-12-22 00:00:00 +09:00
---

これは [防災アプリ開発 Advent Calendar 2024](https://adventar.org/calendars/9939) の17日目の記事です。  
といいつつ遅れてます…すみません。さすがにキツかった…。

と言いつつ原因は他のところにあります。10万ぶち込んで5凸しかできてない愚かな人類がここにいます。

## BUFR とは？

アドベントカレンダー経由で見た方はご存じの方も多いかもしれませんが、パッと聞かれても答えられる人は相当なマニアだと思います。

正直なところ今回の記事の内容自体すべて[とてもわかりやすく解説しているドキュメント](https://github.com/etoyoda/bufrconv/blob/master/BUFR.md)をC#に移植しただけなのですが、せっかくなのでみなさんも入門しましょう。

細かい解説や歴史は↑のドキュメントにあるので読んでいただくとして、専ら気象庁における使われ方としては 『XML だとでかくなりすぎるのでやむを得ずバイナリとして配信しているデータ』 だと思います(個人の意見)。

## 全体を見てみる

BUFR は6つのセクション(節)に分かれているようです。

1. マジックナンバー･電文長定義
1. 電文情報
1. 任意情報
1. データ構造
1. データ本体
1. マジックナンバー(おわり)

ファイル単位のフォーマットではなく単一のストリームがベースとなっているためマジックナンバーを検出して処理を開始したり、繰り返しを含めたデータの構造を事前に定義しておき、本文の部分でそのデータ構造が(区切りなしで)入っているというところが時代を感じますね。

## パーサーを書いてみよう

実際にファイルを読んでみましょう。  
今回は dmdata からダウンロードしたファイルを読み込んでみます。  
C#でバイナリデータを扱うときは `Span<byte>` を使うのがおすすめです。簡単に配列の一部を切り出したりできます。

```cs
var file = (await File.ReadAllBytesAsync(@"C:\Users\ingen\Downloads\IXAC41_RJTD_20240818155600_fd09b8c.bin")).AsSpan();
```

まずはマジックナンバー `BUFR` の位置を探します。  

```cs
var offset = file.IndexOf("BUFR"u8);
if (offset < 0)
    throw new Exception("Not a BUFR file");
offset += 4;
```

`IndexOf` は遅そうに見えますが、しっかり Span を受け入れられるようになっているのできっと早いはずです。  
末尾の `u8` については、これを付けることで文字列ではなく UTF-8 のバイト列( `ReadOnlySpan` )として扱われるようになります。

### 数値を読み込む

C# でバイト列を数値に変換するときはどうしますか？  
ビットシフトでANDしますか？それとも `BitConverter` でしょうか。  
しかし、これらは実行環境のエンディアンに依存するためあまりきれいにはできません。

そこでいつの間にか追加されていた便利なクラス `BinaryPrimitives` を使ってガシガシ読んでいきましょう。  
ポイントとしては、ビッグエンディアンなことと、Int32は4バイトなので3バイトの数値を読みたい場合は頭にゼロを付ける必要があることです。

```cs
Console.WriteLine("第0節");
var totalLength = BinaryPrimitives.ReadInt32BigEndian([0, .. file[offset..(offset + 3)]]);
Console.WriteLine("電文長: " + totalLength);
var edition = file[offset + 3];
Console.WriteLine("版番号: " + edition);
offset += 4;
```

### 第1節を読む

ここまでくればあとはドキュメントに沿って読むだけです。  
僕が試しに組んでみたコードを貼ってみます。

#### Edition3

```cs
public class Edition3IdentificationSection : IIdentificationSection
{
    public byte MasterTableNumber { get; }
    public byte SubCenterNumber { get; }
    public byte SubCenterSubNumber { get; }
    public byte UpdateSequenceNumber { get; }
    public bool HasOptionalSection { get; }
    public byte CategoryNumber { get; }
    public byte SubCategoryNumber { get; }
    public byte MasterTableVersionNumber { get; }
    public byte LocalTableVersionNumber { get; }
    public byte ReferenceYear { get; }
    public byte ReferenceMonth { get; }
    public byte ReferenceDay { get; }
    public byte ReferenceHour { get; }
    public byte ReferenceMinute { get; }

    public DateTime ReferenceDateTime => new(
        ReferenceYear + (ReferenceYear > 80 ? 1900 : 2000),
        ReferenceMonth,
        ReferenceDay,
        ReferenceHour,
        ReferenceMinute,
        0
    );

    public Edition3IdentificationSection(ReadOnlySpan<byte> sectionBytes)
    {
        if (sectionBytes.Length < 17)
            throw new Exception("識別節の長さが短すぎます");
        MasterTableNumber = sectionBytes[3];
        SubCenterNumber = sectionBytes[5];
        SubCenterSubNumber = sectionBytes[4];
        UpdateSequenceNumber = sectionBytes[6];
        HasOptionalSection = sectionBytes[7] == 0b10000000;
        CategoryNumber = sectionBytes[8];
        SubCategoryNumber = sectionBytes[9];
        MasterTableVersionNumber = sectionBytes[10];
        LocalTableVersionNumber = sectionBytes[11];
        ReferenceYear = sectionBytes[12];
        ReferenceMonth = sectionBytes[13];
        ReferenceDay = sectionBytes[14];
        ReferenceHour = sectionBytes[15];
        ReferenceMinute = sectionBytes[16];
    }
}
```

#### Edition4

```cs
public class Edition4IdentificationSection : IIdentificationSection
{
    public byte MasterTableNumber { get; }
    public ushort SubCenterNumber { get; }
    public ushort SubCenterSubNumber { get; }
    public byte UpdateSequenceNumber { get; }
    public bool HasOptionalSection { get; }
    public byte CategoryNumber { get; }
    public byte InternationalSubCategoryNumber { get; }
    public byte SubCenterSubCategoryNumber { get; }
    public byte MasterTableVersionNumber { get; }
    public byte LocalTableVersionNumber { get; }
    public ushort ReferenceYear { get; }
    public byte ReferenceMonth { get; }
    public byte ReferenceDay { get; }
    public byte ReferenceHour { get; }
    public byte ReferenceMinute { get; }
    public byte ReferenceSecond { get; }
    public DateTime ReferenceDateTime => new(
        ReferenceYear,
        ReferenceMonth,
        ReferenceDay,
        ReferenceHour,
        ReferenceMinute,
        ReferenceSecond
    );
    public Edition4IdentificationSection(ReadOnlySpan<byte> sectionBytes)
    {
        if (sectionBytes.Length < 22)
            throw new Exception("識別節の長さが短すぎます");
        MasterTableNumber = sectionBytes[3];
        SubCenterNumber = BinaryPrimitives.ReadUInt16BigEndian(sectionBytes[4..6]);
        SubCenterSubNumber = BinaryPrimitives.ReadUInt16BigEndian(sectionBytes[6..8]);
        UpdateSequenceNumber = sectionBytes[8];
        HasOptionalSection = sectionBytes[9] == 0b10000000;
        CategoryNumber = sectionBytes[10];
        InternationalSubCategoryNumber = sectionBytes[11];
        SubCenterSubCategoryNumber = sectionBytes[12];
        MasterTableVersionNumber = sectionBytes[13];
        LocalTableVersionNumber = sectionBytes[14];
        ReferenceYear = BinaryPrimitives.ReadUInt16BigEndian(sectionBytes[15..17]);
        ReferenceMonth = sectionBytes[17];
        ReferenceDay = sectionBytes[18];
        ReferenceHour = sectionBytes[19];
        ReferenceMinute = sectionBytes[20];
        ReferenceSecond = sectionBytes[21];
    }
}
```

#### これらのクラスを呼ぶ

そして先に取得しておいた版番号で分岐して読ませればOKです。  
第2節は仕様がよくわからなかったので存在する場合は一旦例外を投げることにしました。

```cs
Console.WriteLine("\n第1節");
if (edition == 3)
{
    BinaryPrimitives.TryReadInt32BigEndian([0, .. file[offset..(offset + 3)]], out var indentificationSectionLength);
    if (indentificationSectionLength < 17)
        throw new Exception("Invalid identification section length");
    Console.WriteLine("第1節長さ: " + indentificationSectionLength);
    var identificationSection = file[offset..(offset + indentificationSectionLength)];
    var idSection = new BUFR.Edition3IdentificationSection(identificationSection);

    if (idSection.HasOptionalSection)
        throw new NotImplementedException();
    offset += indentificationSectionLength;
}
else if (edition == 4)
{
    BinaryPrimitives.TryReadInt32BigEndian([0, .. file[offset..(offset + 3)]], out var indentificationSectionLength);
    if (indentificationSectionLength < 17)
        throw new Exception("Invalid identification section length");
    Console.WriteLine("第1節長さ: " + indentificationSectionLength);
    var identificationSection = file[offset..(offset + indentificationSectionLength)];
    var idSection = new BUFR.Edition4IdentificationSection(identificationSection);

    if (idSection.HasOptionalSection)
        throw new NotImplementedException();
    offset += indentificationSectionLength;
}
else
    throw new Exception("Unsupported edition");
```

### 第3節を読む

第3節は第4節のデータ構造を定義します。  
なかなか複雑な仕様をしていますが、まずは第3節で使用されている識別子列を簡単に扱えるようにクラスを用意しておきましょう。

```cs
public readonly struct Descriptor(ushort value)
{
    public byte F { get; } = (byte)(value >> 14);
    public byte X { get; } = (byte)((value & (0b00111111 << 8)) >> 8);
    public byte Y { get; } = (byte)(value & 0xFF);
}
```

これを使って読み込んでみます。

```cs
Console.WriteLine("\n第3節");
var dataDescriptionSectionLength = BinaryPrimitives.ReadInt32BigEndian([0, .. file[offset..(offset + 3)]]);
Console.WriteLine("第3節長さ: " + dataDescriptionSectionLength);

var dataDescriptionSection = file[offset..(offset + dataDescriptionSectionLength)];
var dataSubsetCount = BinaryPrimitives.ReadUInt16BigEndian(dataDescriptionSection[4..6]);
Console.WriteLine("データサブセットの数: " + dataSubsetCount);
Console.WriteLine($"フラグ: {dataDescriptionSection[6]} 観測資料?:{(dataDescriptionSection[6] & 0x80) > 0} 圧縮?:{(dataDescriptionSection[6] & 0x40) > 0}");
for (var i = 0; i < (dataDescriptionSectionLength - 7) / 2; i++)
{
    var localOffset = 7 + i * 2;
    var descriptor = BinaryPrimitives.ReadUInt16BigEndian(dataDescriptionSection[localOffset..(localOffset + 2)]);
    Console.WriteLine($"識別子列{i}: {descriptor >> 14} {(descriptor & (0b00111111 << 8)) >> 8:00} {descriptor & 0xFF:000}");
}
offset += dataDescriptionSectionLength;
```

## 第4節も読みたかった

そしてこの読み込めたデータを元に第4節を読みたかったのですが、時間が無かったのと良い感じのインターフェイスが思いつかなかったので断念しました。とりあえずはここまでです。

なぜパーサーの実装をしているのかというと、近いうちに KEVi に推計震度分布図を表示する機能を実装するつもりでした。  
電文から直接画面に描画する予定ですのでお待ちください。  
たぶんめっちゃ処理に工夫が必要なのでまたブログの記事にしたいと思います。

## さいごに

めっちゃ遅れました！！！！！！！ごめんなさい！！！！  
その上内容が薄いという…。
