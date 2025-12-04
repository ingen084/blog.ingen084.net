---
title: "KyoshinEewViewer for ingen のワークフロー機能を使いこなそう！"
category: "プログラミング"
tags: ["設計"]
date: 2025-12-04 09:00:00 +09:00
---

この記事は [防災アプリ開発 Advent Calendar 2025](https://adventar.org/calendars/11308) の4日目の記事です。

昨日に引き続き拙作のアプリの話をさせてください！

## はじめに

僕は [KyoshinEewViewer for ingen](https://svs.ingen084.net/kyoshineewviewer/) というアプリを趣味で開発しています。

KyoshinEewViewer for ingen には、地震や津波などの災害情報を受信した際に、自動的にさまざまなアクションを実行できる「ワークフロー」機能が搭載されています。本記事では、ワークフロー機能の基本から実用的な活用例まで解説します。

なお、この記事は2025年12月時点での仕様となります。今後の開発に応じて機能の追加や削除･変更を行う可能性がありますのでご了承ください。

最新の情報は [ワークフローガイド](https://github.com/ingen084/KyoshinEewViewerIngen/blob/develop/workflow-guide.md) をご確認ください。

## ワークフローとは

ワークフローは、「特定のイベントが発生したとき」に「指定したアクションを自動実行する」仕組みです。例えば、緊急地震速報を受信したらVOICEVOXで読み上げる、震度5以上の地震情報が来たらWebhookで外部サービスに通知する、といった形です。

```text
イベント発生（地震情報受信など）
    ↓
トリガーが条件をチェック
    ↓
条件に合致した場合、アクションを実行
```

## トリガー一覧

### 汎用トリガー

| トリガー | 説明 |
|---------|------|
| 何もしない | テスト実行専用。設定画面の「テスト実行」ボタンでのみ動作 |
| すべて | すべてのイベントで動作 |
| アプリケーション起動時 | 起動完了時に1度だけ実行 |
| アプリケーション更新存在時 | 更新版の検知時に動作 |
| 揺れ検知 | 強震モニタで揺れを検知した際 |
| 緊急地震速報 ※ | 緊急地震速報の受信時 |
| 地震情報 | 気象庁からの地震情報受信時 |
| 津波情報 | 津波に関する情報受信時 |
| 災危通報（QZSS） | 災危通報受信時 |

※ 警報関連のトリガーの利用には DM-D.S.S の `緊急地震（警報）` 区分の契約が必要です。

## アクション一覧

| アクション | 説明 | 主な設定 |
|----------|------|---------|
| 何もしない | 何も実行しない。一時無効化用 | - |
| 通知送信 | 画面上に通知を表示 | タイトル、本文（テンプレート可） |
| 音声再生 | 音声ファイルを再生 | ファイルパス、音量、再生完了まで待機 |
| VOICEVOX読み上げ | VOICEVOXで音声読み上げ | テキスト（テンプレート可）、再生完了まで待機 |
| ウィンドウ最前面表示 | メインウィンドウを最前面に | - |
| タブ切り替え | イベント発生元のタブに切り替え | - |
| 待機 | 指定時間待機 | 待機時間（ミリ秒） |
| ログ出力 | アプリログに出力（デバッグ用） | テキスト（テンプレート可） |
| Webhook送信 | URLにPOSTリクエスト送信 | URL |
| ファイル実行 | 外部プログラムを実行 | ファイルパス、引数、作業ディレクトリ |
| 複数アクション実行 | 複数アクションをまとめて実行 | 並列/順次、子アクション |

## テンプレートの基本

一部アクションではScribanテンプレートを使用して動的に文字列を生成できます。

### 基本構文

{% raw %}
```text
{{変数名}}                          # 変数の参照
{{if 条件}}...{{end}}               # 条件分岐
{{if 条件}}...{{else}}...{{end}}    # if-else
{{配列 | array.join "・"}}           # 配列の結合
{{数値 | math.format "F1"}}         # 数値フォーマット
{{日時 | date.to_string "%H:%M"}}   # 日時フォーマット
```
{% endraw %}

### アプリ内蔵のデフォルトテンプレート

以下は、アプリに組み込まれている通知用テンプレートです。そのまま使用することも、カスタマイズのベースにすることもできます。ぜひ参考にしてください。

#### 緊急地震速報（通知）

**タイトル：**

{% raw %}
```text
{{if IsCancelled; "[取消] "; end
if IsReplay; "[リプレイ] "; end
"緊急地震速報"
if IsFinal; "(最終"; else; "("; end
SerialNo | math.format "D2"
"報)"}}
```
{% endraw %}

**本文：**

{% raw %}
```text
{{IntensityLongName
if IsIntensityOver; "以上"; end

if EpicenterPlaceName
	$"/{EpicenterPlaceName}"
end

if Magnitude
	$"/M{Magnitude | math.format "F1"}"
end

if Depth && !IsTemporaryEpicenter
	$"/{Depth}km"
end

if IsWarning && WarningAreaNames
	$" 【{WarningAreaNames | array.join "・"}】"
end}}
```
{% endraw %}

**出力例**:

- タイトル: `緊急地震速報(03報)`
- 本文: `震度5弱以上/石川県能登地方/M7.3/10km 【石川県・富山県】`

#### 地震情報（通知）

**タイトル：**

{% raw %}
```text
{{if IsCancelled; "[取消] "; end
if IsTrainingOrTest; "[訓練/試験] "; end
LatestInformationName}}
```
{% endraw %}

**本文：**

{% raw %}
```text
{{
# 基本震度情報
$"最大{MaxIntensityLongName}"

# 震源情報（震源のみ報または詳細震度適用時のみ）
if Hypocenter && (IsHypocenterOnly || IsDetailIntensityApplied)
	$"/{Hypocenter.OccurrenceAt | date.to_string "%H:%M"}"
	$"/{Hypocenter.PlaceName}"

	# 深さ情報
	if !Hypocenter.IsNoDepthData
		if Hypocenter.IsVeryShallow
			"/ごく浅い"
		else
			$"/{Hypocenter.Depth}km"
		end
	end

	# マグニチュード情報
	if Hypocenter.MagnitudeAlternativeText
		$"/{Hypocenter.MagnitudeAlternativeText}"
	else
		$"/M{Hypocenter.Magnitude | math.format "F1"}"
	end
end}}
```
{% endraw %}

**出力例**:

- タイトル: `震源・震度に関する情報`
- 本文: `最大震度4/14:23/石川県能登地方/10km/M5.2`

#### 津波情報（通知）

**タイトル：**

{% raw %}
```text
{{if TsunamiInfo && TsunamiInfo.SpecialState; "[" + TsunamiInfo.SpecialState + "] "; end
if Level == "None"
	"津波解除"
else if Level == "Forecast"
	"津波予報"
else if Level == "Advisory"
	"津波注意報"
else if Level == "Warning"
	"津波警報"
else if Level == "MajorWarning"
	"大津波警報"
else
	"津波情報"
end}}
```
{% endraw %}

**本文：**

{% raw %}
```text
{{
case Level
	when "None"
		LevelString = "津波なし"
	when "Forecast"
		LevelString = "津波予報"
	when "Advisory"
		LevelString = "津波注意報"
	when "Warning"
		LevelString = "津波警報"
	when "MajorWarning"
		LevelString = "大津波警報"
	else
		LevelString = "津波情報"
end
case PreviousLevel
	when "None"
		PreviousLevelString = "津波なし"
	when "Forecast"
		PreviousLevelString = "津波予報"
	when "Advisory"
		PreviousLevelString = "津波注意報"
	when "Warning"
		PreviousLevelString = "津波警報"
	when "MajorWarning"
		PreviousLevelString = "大津波警報"
	else
		PreviousLevelString = "津波情報"
end

# 発表・解除・更新状態
if PreviousLevel != Level
	if PreviousLevel == "None"
		LevelString + "が発表されました。"
	else if Level == "None"
		if PreviousLevel == "Forecast"
			"津波予報の期限が切れました。"
		else
			PreviousLevelString + "は解除されました。"
		end
	else
		PreviousLevelString + " は " + LevelString + " に切り替えられました。"
	end
end}}

{{# 対象地域情報
if TsunamiInfo && Level != "None"
	if TsunamiInfo.MajorWarningAreas
		$" 【{TsunamiInfo.MajorWarningAreas | array.map "Name" | array.join "・"}】"
	else if TsunamiInfo.WarningAreas
		$" 【{TsunamiInfo.WarningAreas | array.map "Name" | array.join "・"}】"
	else if TsunamiInfo.AdvisoryAreas
		$" ({TsunamiInfo.AdvisoryAreas | array.map "Name" | array.join "・"})"
	end
end}}
```
{% endraw %}

**出力例**:

- タイトル: `大津波警報`
- 本文: `大津波警報が発表されました。 【岩手県・宮城県・福島県】`

## 実用ワークフロー例

### 例1: 緊急地震速報の通知と読み上げ

震度3以上の緊急地震速報を受信したら、通知を表示して音声で読み上げる。

| 項目 | 設定 |
|-----|------|
| トリガー | 緊急地震速報 |
| 条件 | 新規発表、震度フィルター: 震度3 |
| アクション | 複数アクション実行（並列） |

**子アクション：**

- 通知送信
  - タイトル: {% raw %}`緊急地震速報({{SerialNo | math.format "D2"}}報)`{% endraw %}
  - 本文: {% raw %}`{{IntensityLongName}}{{if IsIntensityOver}}以上{{end}} {{EpicenterPlaceName}}`{% endraw %}
- VOICEVOX読み上げ: {% raw %}`最大{{IntensityLongName}}{{if IsIntensityOver}}以上{{end}}。第1報、緊急地震速報`{% endraw %}

### 例2: 震度5弱以上で警報対応

震度5弱以上の緊急地震速報では、ウィンドウを最前面に表示し、警報音を鳴らす。

| 項目 | 設定 |
|-----|------|
| トリガー | 緊急地震速報 |
| 条件 | 新報、震度フィルター: 震度5弱 |
| アクション | 複数アクション実行（並列） |

**子アクション：**

- ウィンドウ最前面表示
- タブ切り替え
- 音声再生: `C:\Sounds\alarm.wav`

### 例3: 緊急地震速報の詳細読み上げ

キャンセル報、最終報、リプレイ状態に対応した詳細な読み上げを行う。

| 項目 | 設定 |
|-----|------|
| トリガー | 緊急地震速報 |
| 条件 | 新報 |
| アクション | 複数アクション実行（順次） |

**子アクション：**

- ウィンドウ最前面表示
- タブ切り替え
- VOICEVOX読み上げ（待機: ON）:

  {% raw %}
  ```text
  {{if IsReplay}}リプレイ中です{{end}}
  {{if IsCancelled}}
  緊急地震速報は取り消されました
  {{else if EventSubType == "Final"}}
  緊急地震速報最終報
  {{else}}
  緊急地震速報
  {{end}}
  {{if !IsCancelled}}
  最大{{IntensityLongName}}{{if IsIntensityOver}}以上{{end}}
  {{if EpicenterPlaceName}}{{EpicenterPlaceName}}{{end}}
  {{end}}
  ```
  {% endraw %}

**出力例**: `緊急地震速報 最大震度5弱以上 石川県能登地方`

### 例4: 地震情報の詳細読み上げ

震度に応じた表現を変えて自然な読み上げを実現します。大規模噴火情報にも対応しています。

| 項目 | 設定 |
|-----|------|
| トリガー | 地震情報 |
| 条件 | 震源・震度に関する情報 |
| アクション | VOICEVOX読み上げ |

**テンプレート：**

{% raw %}
```text
{{
if IsVolcano
  "大規模噴火情報 "
  Hypocenter.PlaceName
  " "
  VolcanoName
  "で、大規模な噴火が発生しました。"
  FreeFormComment
else
  LatestInformationName
  " 最大震度 "
  case MaxIntensity
    when "Unknown"
      "不明の"
    when "Int0"
      "観測なしの"
    when "Int1"
      "1の"
    when "Int2"
      "2の"
    when "Int3"
      "3の"
    when "Int4"
      "4のやや強い"
    when "Int5Lower"
      "5弱の強い"
    when "Int5Upper"
      "5強の強い"
    when "Int6Lower"
      "6弱の非常に強い"
    when "Int6Upper"
      "6強の非常に強い"
    when "Int7"
      "7の大"
    else
      Level
  end
  "地震が発生しました"
end
}}
```
{% endraw %}

**出力例**: `震源・震度に関する情報 最大震度 4のやや強い地震が発生しました`

### 例5: 津波警報の段階的対応

津波レベルに応じた音声読み上げと対象地域の案内を行う。

| 項目 | 設定 |
|-----|------|
| トリガー | 津波情報 |
| 条件 | 注意報以上、発表・レベル上昇 |
| アクション | 複数アクション実行（順次） |

**子アクション：**

- ウィンドウ最前面表示
- タブ切り替え
- VOICEVOX読み上げ（待機: ON）:

  {% raw %}
  ```text
  {{if TsunamiInfo && TsunamiInfo.SpecialState}}{{TsunamiInfo.SpecialState}}です{{end}}
  {{if Level == "None"}}
  {{if PreviousLevel != "None"}}津波警報等はすべて解除されました{{else}}津波の心配はありません{{end}}
  {{else if Level == "Forecast"}}
  津波予報が発表されました
  {{else if Level == "Advisory"}}
  津波注意報が発表されました。海岸に近づかないでください。
  {{else if Level == "Warning"}}
  津波警報が発表されました。沿岸部から離れてください。
  {{else if Level == "MajorWarning"}}
  大津波警報が発表されました。直ちに高台へ避難してください。
  {{end}}
  {{if TsunamiInfo && Level != "None"}}
  {{if TsunamiInfo.MajorWarningAreas}}
  大津波警報の対象地域は{{TsunamiInfo.MajorWarningAreas | array.map "Name" | array.join "、"}}です
  {{else if TsunamiInfo.WarningAreas}}
  津波警報の対象地域は{{TsunamiInfo.WarningAreas | array.map "Name" | array.join "、"}}です
  {{else if TsunamiInfo.AdvisoryAreas}}
  津波注意報の対象地域は{{TsunamiInfo.AdvisoryAreas | array.map "Name" | array.join "、"}}です
  {{end}}
  {{end}}
  ```
  {% endraw %}

**出力例**: `大津波警報が発表されました。直ちに高台へ避難してください。 大津波警報の対象地域は岩手県、宮城県、福島県です`

### 例6: 外部スクリプトとの連携

地震情報を受信したらPowerShellスクリプトを実行する。

| 項目 | 設定 |
|-----|------|
| トリガー | 地震情報 |
| 条件 | 震度4以上 |
| アクション | ファイル実行 |

**設定：**

- ファイルパス: `C:\Scripts\earthquake-notify.ps1`
- シェル実行: ON

**PowerShellスクリプト例（earthquake-notify.ps1）：**

```powershell
$eventData = $env:KEVI_EVENT_DATA | ConvertFrom-Json
$message = "地震発生: $($eventData.Hypocenter.PlaceName) 最大震度$($eventData.MaxIntensityLongName)"
# Slack, Teams, LINE等への通知処理...
```

実行されるプロセスには `KEVI_EVENT_DATA` 環境変数が設定され、イベント情報のJSONが格納されます。

**緊急地震速報などの情報を活用される場合は、強震モニタや DM-D.S.S などの利用規約･利用条件に抵触しないか十分に確認の上ご利用ください**。

### 例7: 連続読み上げ

複数の情報を連続で読み上げる場合、**順次実行**を使うことで音声が順番に再生されます。

VOICEVOX での読み上げは音声生成ごとにその結果を一時データとして保存させるため、同じ文言になる可能性の高いパーツごとにアクションを分離させることで、生成時間の短縮が見込めます。

| 項目 | 設定 |
|-----|------|
| トリガー | 地震情報 |
| 条件 | 震源・震度に関する情報 |
| アクション | 複数アクション実行（順次） |

**子アクション：**

VOICEVOX読み上げ（待機: ON）で、下記内容を1行ごとにアクションとして分離する。

{% raw %}
```text
{{if IsTrainingOrTest}}これは訓練もしくは試験です{{end}}
{{if IsCancelled}}地震情報は取り消されました{{else}}{{if IsTest}}ワークフローのテストです{{end}}{{end}}
{{if !IsCancelled && Hypocenter && Hypocenter.PlaceName}}{{Hypocenter.PlaceName}}{{if IsVolcano}} {{VolcanoName}}{{end}}で{{end}}
{{if IsVolcano}}大規模な噴火が発生しました{{else}}{{if !IsCancelled}}最大{{MaxIntensityLongName}}の地震が発生しました{{end}}{{end}}
{{if !IsVolcano && !IsCancelled && Hypocenter && !Hypocenter.IsNoDepthData}}{{if Hypocenter.IsVeryShallow}}深さはごく浅い{{else}}深さは{{Hypocenter.Depth}}キロ{{end}}{{end}}
{{if !IsVolcano && !IsCancelled && Hypocenter}}マグニチュードは{{Hypocenter.Magnitude | math.format "F1"}}です{{end}}
{{if !IsCancelled && Comment}}{{Comment}}{{end}}
```
{% endraw %}

#### 順次実行による最適化の仕組み

順次実行では、先んじて各アクションの **音声生成** が全アクション分先行して並列実行されます。これにより、実際の再生時には既に音声データが準備されているため、シームレスな連続再生が実現できます。

```mermaid
gantt
    title 順次実行の動作例
    dateFormat X
    axisFormat %s秒

    section 効果音
    再生      :z1, 0, 2

    section 短い音声
    生成      :a1, 0, 2
    再生      :a2, after a1, 4

    section 長い文章
    生成      :b1, 0, 5
    再生      :b2, after b1, 9

    section 短い音声
    生成      :c1, 0, 1
    再生      :c2, after b2, 11
```

**ポイント：**

- 音声生成は全アクション分が**並列**で開始される
- 読み上げは生成済みの音声を**順番に**再生する
- 再生すべきタイミングでまだ生成が終わっていない場合は、生成完了を待機する
- 音声Bの生成に時間がかかっても、音声Aの再生中もしくは直後に生成が完了するため待ち時間が発生しない
- 結果として、読み上げ間の遅延がほぼゼロになる

## まとめ

ワークフロー機能を活用することで、災害情報の受信から通知・アクションまでを自動化できます。

- **トリガー条件を適切に設定** - 必要な情報だけを対象にすることで、過剰な通知を防げます
- **複数アクションの並列/順次を使い分け** - 独立したアクションは並列で高速化、順序が重要なアクションは順次で確実に実行
- **テンプレートで動的なメッセージ** - Scribanテンプレートで状況に応じた通知内容を生成
- **VOICEVOXキャッシュを活用** - 定型文のキャッシュで音声読み上げの遅延を最小化

## 今後に向けて

ワークフロー機能自体は、僕がみなさんの需要を把握し切れていないためとりあえず万能な基盤を作って自由に触ってもらおうというのと、各機能で音声再生や通知などの呼出処理が煩雑になっており何か省略できる仕組みはないかというところから生まれた機能です。

内部の実装を無理矢理UIに落とし込んだ結構苦しい機能ではあったのですが、アンケートなどでも高く評価してくださる方がいらっしゃったりで、とてもありがたい限りです。  
今後はこういった形で使い方を紹介する機会を用意しながらも、インストールする度にわざわざ設定するのは大変だと思いますので、ワークフロー関連の機能としては

- 読み上げなどのよく使われる機能のデフォルト実装化
- ワークフローのシェア機能
- トリガー、アクションのUIの統一

などを実装していきたいと思います。

実装までは時間がかかってしまうと思いますが、長い目で見ていただけるとありがたいです。

(OSS なので、もしやる気のある方がいらっしゃいましたら貢献していただけるととてもうれしいです！)
