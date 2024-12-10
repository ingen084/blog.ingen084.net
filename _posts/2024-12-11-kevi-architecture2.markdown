---
title: "2024年版 KEVi の設計(後半)"
category: "プログラミング"
tags: ["設計", "Avalonia", "AvaloniaUI"]
date: 2024-12-11 00:00:00 +09:00
---

これは [防災アプリ開発 Advent Calendar 2024](https://adventar.org/calendars/9939) の11日目の記事です。

[前半の記事はこちら！](2024-12-04-kevi-architecture)

---

前回に引き続き、設計についてのお話を書きます。

## 設定画面

今まで設定画面はずらずらと1つの XAML ファイルに書いていたのですが、タブの数や設定項目がさすがに多くなってきてメンテナンスに問題が生じるようになってきました。

まず第1段階として各タブ内の UI を UserControl としてファイルに分割しました。そしてそのコントロールを各タブ内のコンテンツとしてセットすることで設定画面の中身を複数のファイルに分散することができるようになりました。  
しかしタブの数が増えてきてコンテンツを除いてもファイルが大きくなっていたことと、合わせて ViewModel が肥大化し、保守が厳しくなってきたためもう少し汎用化することとしました。

![](../assets/img/posts/20241204-kevi-arch/settings-tab.png)

これはバージョン 0.19 でリニューアルされた設定画面ですが、有効になっている機能のタブを固定設定から上下から挟む構造になっています。  
設定ページは必ず `ISettingPage` を実装したクラスとし、各 `Series` の設定画面は上述した `SeriesController` 経由で集約し設定ウィンドウへ組み込む形としました。

![](../assets/img/posts/20241204-kevi-arch/settings-window.drawio.svg)

```cs
public interface ISettingPage
{
    public bool IsVisible { get; }
    public string? Icon { get; }
    public string Title { get; }
    public Control DisplayControl { get; }

    public ISettingPage[] SubPages { get; }
}
```

これにより、各 `Series` に変更などを加える際、設定画面の共通コードに手を入れなくて済むようになりました。

```cs
SettingPages = [
    UpdatePage,
    new BasicSettingPage<GeneralPage>("\xf53f", "外観･基本設定", []),
    new BasicSettingPage<FeaturePage>("\xf085", "機能設定", []),
    new BasicSettingPage<NotifyPage>("\xf075", "通知", []),
    new BasicSettingPage<SoundPage>("\xf028", "音声", []),
    new BasicSettingPage<WorkflowPage>("\xe289", "ワークフロー", []),
    new BasicSettingPage<VoicevoxPage>("\xf075", "VOICEVOX", []),
    ..SeriesController.EnabledSeries.SelectMany(s => s.SettingPages),
    new BasicSettingPage<DmdataPage>("\xf48b", "DM-D.S.S", []),
    new BasicSettingPage<MapPage>("\xf5a0", "地図", []),
    new BasicSettingPage<AboutPage>("\xf129", "このアプリについて", []),
    new BasicSettingPage<LicencePage>("\xf2c2", "ライセンス", []),
#if DEBUG
    new BasicSettingPage<DebugMenuPage>("\xf188", "デバッグメニュー", []),
#endif
];
```

なお、設定の実態については1つのクラスに盛り込むような形になっているのでこちらも将来的にはなんとかしたいです。  
変更を入れてしまうと設定がリセットされちゃうのでなんとかしないといけませんが…。

## 電文の汎用化

このアプリは地震情報･津波情報(+未公開の台風情報)を気象庁の発表している電文をそのまま解析して画面に表示します。  
これだけ機能を分離していると、電文の取得も一筋縄ではいきません。DM-D.S.S ではプランを選択できるからです。  
特定のプランを契約していないと利用できないようにするというのも手段の1つではありますが、他のサービスに依存はしたくないため、利用できるプランと機能として利用する必要のある情報の種類を内部で判断し、必要な物のみに絞るような仕組みにしています。

まず [気象庁防災情報XMLフォーマット形式電文の公開（PULL型）](https://xml.kishou.go.jp/xmlpull.html) と [DM-D.S.S](https://dmdata.jp/) の配信区分の最小公倍数のカテゴリを内部で定義します。

- 地震情報
- EEW予報
- EEW警報
- 津波情報
- 台風情報

まずは電文を利用する Series はこのカテゴリ内で受信したいものを subscribe します。  
subscribe された後、各情報ソースが利用できるカテゴリを宣言し、実際に利用できるソースが決定されます。

言葉で説明しても誰も読まないと思うので、メモも兼ねて各フローチャートを書いてみます。

![](../assets/img/posts/20241204-kevi-arch/telegram-arch.svg)

これらの仕組みにより生電文を配信するサービスが増えても汎用的に処理することができるようになります。  
…が、実際個人で契約できるサービスとしては全く増えていないところが厳しいですね。

## ワークフロー機能について

<blockquote class="twitter-tweet"><p lang="ja" dir="ltr">考えています <a href="https://t.co/jpXodYTPA4">pic.twitter.com/jpXodYTPA4</a></p>&mdash; ingen084 (@ingen084) <a href="https://twitter.com/ingen084/status/1244022118414610434?ref_src=twsrc%5Etfw">March 28, 2020</a></blockquote> <script async src="https://platform.twitter.com/widgets.js" charset="utf-8"></script>

ワークフロー機能については4年前よりゆっくり懐で温めていた機能でした。  
抽象的な Event クラスを流し、抽象的な Trigger でフィルタリングし、抽象的な Action クラスを実行する仕組みです。

いままで実現していなかった理由は主に僕の技術不足だったのですが、Trigger･Action を設定ファイルのシリアライズとUIへのバインディングにそのまま利用してしまうという手法で実現しています。

### 揺れ検知イベント

```cs
public class ShakeDetectedEvent(DateTime time, KyoshinEvent evt, bool isReplay) : WorkflowEvent("KyoshinShakeDetected")
{
    public DateTime EventedAt { get; } = time;
    public DateTime FirstEventedAt { get; } = evt.CreatedAt;
    public KyoshinEventLevel Level { get; } = evt.Level;
    public Guid KyoshinEventId { get; } = evt.Id;
    public string[] Regions { get; } = evt.Points.Select(p => p.Region).Distinct().ToArray();
    public bool IsReplay { get; } = isReplay;
}
```

### 揺れ検知のトリガー

```cs
public class ShakeDetectTrigger : WorkflowTrigger
{
    public override Control DisplayControl => new ShakeDetectTriggerControl() { DataContext = this };

    private KyoshinEventLevel _level = KyoshinEventLevel.Medium;
    public KyoshinEventLevel Level
    {
        get => _level;
        set => this.RaiseAndSetIfChanged(ref _level, value);
    }

    private bool _isExact = false;
    public bool IsExact
    {
        get => _isExact;
        set => this.RaiseAndSetIfChanged(ref _isExact, value);
    }

    public override bool CheckTrigger(WorkflowEvent content)
    {
        if (content is not ShakeDetectedEvent shakeEvent)
            return false;

        if (IsExact)
            return shakeEvent.Level == Level;

        return shakeEvent.Level >= Level;
    }
}
```

### 待機アクション

```cs
public class WaitAction: WorkflowAction
{
    public override Control DisplayControl => new WaitActionControl() { DataContext = this };

    private int _waitTime = 0;
    public int WaitTime
    {
        get => _waitTime;
        set => this.RaiseAndSetIfChanged(ref _waitTime, value);
    }

    public override Task ExecuteAsync(WorkflowEvent content)
        => Task.Delay(WaitTime);
}
```

とこんな感じで C#, Avalonia, Jsonシリアライザの機能をフル活用して実装しています。

1つのトリガー、アクションをセットとしたのもがワークフローとなっており、UI構造同様、複数のトリガーやアクションをセットにしたい場合は複数指定できるトリガー･アクションを用意して設定できるようにしています。  
トリガーの複数条件はまだ開放していませんが…。

## さいごに

次回のアドベントカレンダー本当に行けるんですかね…(ドルフロ2で時間を消し飛ばしながら)  
おわびに今後のアプリの作業予定をお話ししておくと、EEW周りのリファクタと VOICEVOX の音声合成を事前にやっておく仕組みの実装を予定しています。
