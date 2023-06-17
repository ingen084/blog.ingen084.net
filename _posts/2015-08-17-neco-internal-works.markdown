---
title: "NECOの処理　ニコニコにログインしてから生放送に接続・コメント投稿するまで"
category: "雑記"
tags: ["過去の投稿", "NECO"]
date: 2015-08-17 05:41:12 +09:00

permalink: /posts/20150817-neco-internal-works
---

> この投稿は過去のWordPressからの投稿を移植したものです。  
> あくまでこんな事もあった･過去こんな考え方をしていた程度でお受け止めください。  
> 画像が表示されていない部分が存在します。ご了承下さい。

全部記憶だから多分間違ってるけど一応動きます(白目

あくまでこれはNECO2.4で処理しているものなので、このAPIが最新・最善であるとは限りません。

指摘などあればコメントでお願いします。m(_ _)m

あと途中で口調が変わりますが気にしないでください(吐血  

## ログイン

ログインはブラウザで使用するフォームと同じURLを使用しています。  

```
Type: POST  
URL: https://secure.nicovideo.jp/secure/login?site=nicolive  
Content: next_url -> {}(文字列なし) mail -> {メールアドレス} password -> {パスワード}
```

これを叩くと結果的に生放送トップページ(多分)にリダイレクトされます。その時のクッキーを保存し、今後APIを呼び出す際に利用します。  
ログイン出来ない可能性がある場合(ユーザーにメアド・PWを打たせる場合)はなんかもうひとつAPI叩いてた気がしますがNECOには必要なかったので忘れました(ぉぃ

## 生放送の情報を取得する

### リクエスト

`Type: GET  
URL: http://live.nicovideo.jp/api/getplayerstatus/生放送ID`  
URLの例:  
`http://live.nicovideo.jp/api/getplayerstatus/lv231835386`  
又、Nsenなど生放送番号以外のURLが存在する場合はそのURLでも情報が取得できるようです。例:  
`http://live.nicovideo.jp/api/getplayerstatus/nsen/toho`

### 返り値

<a href="http://live.nicovideo.jp/api/getplayerstatus/nsen/toho" target="_blank">サンプル(Nsen東方chの情報)</a>  
例:(適当に情報を消しています)  
```xml
<getplayerstatus status="ok" time="1439782445">  
    <stream>  
        <id>lv231835386</id>  
        <title>Nsen &#8211; 東方チャンネル</title>  
        <description>  
            名曲だらけの東方音楽に酔いしれろ!アレンジ、ヴォーカルカバー、リミックス。そんな新たな息吹を吹き込むアーティスト達の動画が集まっています  
        </description>  
        <provider\_type>official</provider\_type>  
        <default_community/>  
        <international>13</international>  
        <is\_owner>0</is\_owner>  
        <owner\_id>394</owner\_id>  
        <owner_name/>  
        <is\_reserved>0</is\_reserved>  
        <is\_niconico\_enquete\_enabled>1</is\_niconico\_enquete\_enabled>  
        <watch\_count>387</watch\_count>  
        <comment\_count>1707</comment\_count>  
        <base\_time>1439751000</base\_time>  
        <open\_time>1439751000</open\_time>  
        <start\_time>1439751600</start\_time>  
        <end\_time>1439838600</end\_time>  
        <is\_rerun\_stream>0</is\_rerun\_stream>  
        <is\_archiveplayserver>1</is\_archiveplayserver>  
        <bourbon_url>  
            https://account.nicovideo.jp/premium/register?sec=nicolive\_crowded&sub=watch\_crowded\_0\_official\_lv231835386\_onair  
        </bourbon_url>  
        <full_video>  
            https://account.nicovideo.jp/premium/register?sec=nicolive\_crowded&sub=watch\_crowded\_0\_official\_lv231835386\_onair  
        </full_video>  
        <after_video/>  
        <before_video/>  
        <kickout_video>  
            https://account.nicovideo.jp/premium/register?sec=nicolive\_oidashi&sub=watchplayer\_oidashialert\_0\_official\_lv231835386\_onair  
        </kickout_video>  
        <twitter\_tag>#niconsen</twitter\_tag>  
        <danjo\_comment\_mode>0</danjo\_comment\_mode>  
        <infinity\_mode>0</infinity\_mode>  
        <archive>0</archive>  
        <press>  
            <display\_lines>-1</display\_lines>  
            <display\_time>-1</display\_time>  
            <style_conf/>  
        </press>  
        <plugin_delay/>  
        <plugin_url/>  
        <plugin_urls/>  
        <allow\_netduetto>0</allow\_netduetto>  
        <nd\_token>\*****</nd\_token>  
        <ng\_scoring>0</ng\_scoring>  
        <is\_nonarchive\_timeshift\_enabled>0</is\_nonarchive\_timeshift\_enabled>  
        <is\_timeshift\_reserved>0</is\_timeshift\_reserved>  
        <header\_comment>0</header\_comment>  
        <footer\_comment>0</footer\_comment>  
        <split\_bottom>0</split\_bottom>  
        <split\_top>0</split\_top>  
        <background\_comment>0</background\_comment>  
        <font_scale/>  
        <comment\_lock>0</comment\_lock>  
        <telop>  
            <enable>0</enable>  
        </telop>  
        <contents_list>  
            <contents id="main" disableAudio="0" disableVideo="0" start_time="1439782233" duration="232" title="愚者に創られた闇の娘　～パパは変態紳士ver～">smile:sm19930598</contents>  
        </contents_list>  
        <picture\_url>http://nl.simg.jp/img/a35/103321.39a510.jpg</picture\_url>  
        <thumb\_url>http://live.nicovideo.jp/thumb/73628.60&#215;60.jpg</thumb\_url>  
        <is\_priority\_prefecture/>  
    </stream>  
    <user>  
        <user\_id>26405447</user\_id>  
        <nickname>ingen084</nickname>  
        <is\_premium>0</is\_premium>  
        <userAge>16</userAge>  
        <userSex>1</userSex>  
        <userDomain>jp</userDomain>  
        <userPrefecture>27</userPrefecture>  
        <userLanguage>ja-jp</userLanguage>  
        <room\_label>フロア</room\_label>  
        <room\_seetno>97</room\_seetno>  
        <twitter_info>  
            <status>enabled</status>  
            <screen\_name>ingen084</screen\_name>  
            <followers\_count>1303</followers\_count>  
            <is\_vip>0</is\_vip>  
            <profile\_image\_url>  
                http://pbs.twimg.com/profile\_images/503606475223076866/rtPUJ8U7\_normal.png  
            </profile\_image\_url>  
            <after\_auth>0</after\_auth>  
            <tweet\_token>\*****</tweet\_token>  
        </twitter_info>  
    </user>  
    <ms>  
        <addr>omsg103.live.nicovideo.jp</addr>  
        <port>2832</port>  
        <thread>1458908869</thread>  
    </ms>  
</getplayerstatus>
```

ルートノード(getplayerstatus)のstatus属性にはリクエストが成功した場合にはok、失敗した場合にはfailが代入されます。  
失敗した場合には中にerror要素が生成され、その中のcode要素に理由が記載されます。(closed=放送は終了した　など&#8230;)  
user要素の中には自身のユーザー情報が入っている。

#### stream要素(生放送情報)の解説<small>NECOが使用しているもの＋重要なもののみ</small>

`id` 生放送ID、NsenなどID以外のURLからアクセスした場合でも取得できる。  
`title` 生放送のタイトル  
`description` 生放送の説明  
`provider_type` 生放送の種類、公式放送なら`official`  
`is_owner` 自分が放送主かどうか  
`owner_id` 放送主のユーザーID？  
`owner_name` 放送主名、公式放送などでない場合は空  
`watch_count``comment_count` 視聴数、コメント数、初回読み込み時に表示するためだけのもの。更新は後述するHeartbeatのAPIから取得する。

#### ms要素(生放送コメントサーバー情報)の解説<small>後述する生放送のリアルタイムコメント取得に必要になります。</small>

`addr` コメントサーバーのアドレス(ドメイン)  
`port` コメントサーバーのポート番号  
`thread` コメントサーバーのスレッドID

その他の情報については自分でXMLを見てくだしあ(丸投げ)

## コメントサーバーに接続

前述のAPIから取得したコメントサーバーの情報を使用する。  
コメントサーバーのアドレス・ポートにTCPで接続する  
以下のパケットを送信する  
```xml
<thread thread="{スレッドID}" version="20061206" res_from="{最初に過去のコメントを送信する数(負の値)}">\0
```
Note： 過去のコメント数を0(送ってこない)にする場合、-0にするのが理想。  
\0はバイトコード0の文字を表す。  
例：  
```xml
<thread thread="1122334" version="20061206" res_from="-100">\0
```
パケットの送信が成功すれば、サーバー側からこのようなパケットが送信されてきます。  
```xml
<thread ticket="{チケット}" server\_time="{サーバー時刻}" last\_res="{送信される過去のコメント数？(NECOで使用してないので不明)}">
```
その後過去のコメントが指定数送られ(-0を指定した場合やそもそもコメントがない場合は送られない)、コメントサーバーとの接続に成功します。  
チケット・サーバー時刻はコメントを投稿する際に必要になるのでとっておきましょう。

### コメントの受信

コメントサーバーに接続していると、誰か/自分がコメントを投稿するたびにコメントが送信されてきます。  
```xml
<chat anonymity="{184か}" no="{コメントの番号}" date="{コメントが投稿されたリアル時間？}" mail="{コマンド}" premium="{プレミアムID}"　thread="{スレッドID}" user_id="{ユーザーID}" vpos="{コメントが投稿された生放送の時間}" score="{NGスコア}">{コメント}</chat>\0
```

#### 属性の解説

`anonymity` 184の場合は1それ以外は0?  
`no` コメント番号、ユーザー放送ではこの番号に欠番があればその番号のコメントをNGに引っかかったコメントとして扱う。公式生放送の場合は無い。  
`date` コメントが投稿されたリアルでの時間だったはず・・・(使用していないので曖昧)  
`mail` コマンド。下赤大文字＋184の場合は`184 shita red big`のようになる。多分。  
`premium` プレミアムID。古い情報だが以下が確認されている。(出典不明)  
```
0 = 一般  
1 = プレミアム  
2 = アラート (地震情報とかはこれだったはず)  
3 = 放送主  
4 = 運営  
5 = BSP  
6 = iPhone  
7 = iPhone2
```
`thread` そのコメントが投稿されたスレッドID  
`user_id` ユーザーID、184の場合は匿名  
`vpos` コメントが投稿された生放送での時間(ここから生放送開始からの時間を算出することもできる)  
`score` そのコメントを投稿した時点でのそのユーザーのNGスコア

chat要素の中のテキストがコメント実体。&gt;や、&lt;等のXMLに干渉する文字はエンコードされているので注意

### コメントの投稿

コメントを投稿するためにはPostKey(一度のみ使用可能)、vpos(放送開始時間(秒)*100)が必要である。

#### vposの算出

(C#でのコードを貼っておきます)  
```cs
long serverTimeSpan = \_serverTime &#8211; \_baseTime;  
string vpos = (serverTimeSpan * 100).ToString();
```
_serverTimeはコメントサーバーから送られてくるchatやthreadのdate、Heartbeatの返り値のtimeなどに書かれているので該当する情報を受信するたびに変数の値を更新してください。  
\_baseTimeは生放送情報を取得した際の返り値にあるstream要素の中のbase\_time要素の内容をそのまま使用します。

#### PostKeyの取得

```
Type: GET  
http://ow.live.nicovideo.jp/api/getpostkey?thread={コメントサーバーのスレッドID}&block_no=0
```
Note: block_noについては公式放送の場合0でいいが、ユーザー生放送の場合コメントidが100増えると1つ加算される話を聞いたことがある(現在の仕様は不明、最後のコメントが53番だった場合は0、238番だった場合は2)

##### 返り値

`postkey=***`
Note: プレーンテキスト。取得に失敗した場合はpostkey=以降の文字がない。

#### コメントの送信

```xml
<chat thread="{スレッドID}" ticket="{チケット}" vpos="{vpos}" postkey="{PostKey}" mail="{コマンド}" user_id="{自分のユーザーID}" premium="">{投稿したいコメント}</chat>\0
```
`thread` コメントサーバーのスレッドID  
`ticket` コメントサーバーから送られてきたチケット  
`vpos` さっき算出したvpos  
`postkey` さっき取得したPostKeyのpostkey=以降の文字  
`mail` コマンド。`184 red`など。  
`user_id` 自分のユーザーID。多分正直に入力しないとコメントが投稿できない。184コマンドが入力されていれば勝手にサーバー側で184処理される。  
`premium` 空白でok

投稿したいコメント(要素の中身)はXMLエンコードして入れることを推奨。

このパケットをコメントサーバーに送信します。

##### 返り値

パケットが送信できたらサーバーからコメントの投稿結果が帰ってきます。(主要な属性しか書けなくて申し訳ない。)  
```xml
<chat_result status="{コメント投稿要求の返答}" />
```
`status` 上記の通り。こちらで確認しているstatusは以下のとおり。  
```
0 = 投稿に成功した  
1 = 投稿に失敗した(短時間に同じ内容のコメントを投稿しようとした、パラメータが間違っている、他)  
4 = 投稿に失敗した(ごく短時間にコメントを連投しようとした、パラメータが間違っている、他)
```
4の状態で一定時間内にさらにコメントを投稿しようとすると一定時間(1分前後？)コメント投稿規制を食らう？(出典不明、要検証)  
投稿が成功した場合直後のパケットにて自身のコメントが通常のコメントとしてサーバーから流れてくる。

## Heartbeat

コメントサーバーに接続した状態で3分以上通信がないと自動的に切断され、再接続しない限り以降コメントが送られることがなくなります。  
また、きちんと接続されていてもサーバー上では視聴していないことにされ、荒らし防止の為？にPostkeyが取得できなくなります。(つまりコメントが投稿できなくなる)  
それを回避するために一定間隔でHeartbeatを呼ぶ必要があります。  
```
Type: POST  
URL: http://ow.live.nicovideo.jp/api/heartbeat  
Content: lang -> ja-jp  
locale -> JP  
seat_locale -> JP  
screen -> ScreenNormal  
v -> {生放送ID}  
datarate -> 0
```

もしかしたら`lang``locale``seat_locale``screen``datarate`は必要ないかもしれません。  
`lang` 言語。ja-jpでいいと思う・・・  
`locale` 位置。日本ならJPで。  
`seat_locale` 正直良くわからない。一応JPで動く。  
`screen` フルスクだったりしたら変える必要がある？一応botなら画面モードとか無いのでScreenNormal。  
`v` 生放送ID。別の視聴URLがある公式放送でもURLではなくIDを入力する必要がある。  
`datarate` 多分レート入力しないといけないんだと思うけどbotなら0で問題ないかと。

なんか3時間ぐらいかかったし自分でも何書いてあるかわからん・・・