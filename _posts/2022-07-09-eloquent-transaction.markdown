---
title: "Eloquentのトランザクションには気をつけよう！"
category: "雑記"
tags: ["ソフトウェア", "PHP"]
date: 2022-07-09 00:00:00 +09:00

permalink: /posts/20220709-eloquent-transaction
---

10ヶ月ぐらい前、仕事でとても詰まったポイントがあったので共有します。  
結局は先入観によるものが原因だったのですが、引っかかる人は結構いるんじゃないかなと思います。

## 結論

**トランザクションはちゃんと `COMMIT` or `ROLLBACK` しよう！**

## あらまし

Laravel の artisan でレコードを舐めながら外部にリクエストを送信し、状態に応じてレコードを更新するバッチを作成しました。  
コードとしてはこんな感じです。※擬似コードなのでずいぶん適当です。

```php
foreach ($ids as $id) {
    $connection = getConnection();
    $connection->beginTransaction();
    try {
        // 現在の状態を取得しつつlock
        $record = $connection->table('records')
            ->where('id', $id)
            ->lockForUpdate()
            ->first();
        
        // 外部システムから情報を引っ張ってくる
        $result = $this->apiClient->getStatus($record->getOtherId());

        // 問題なければ次のレコードへ
        if ($result->getStatus() === $record->status) {
            continue;
        }
        
        // 問題があるので修正
        $count = $connection->table('records')
            ->where('id', $id)
            ->update([
                'status' => $result->getStatus(),
            ]);
        
        // コミット
        $connection->commit();
    } catch (\Throwable $e) {
        // 問題が起きたらロールバック
        $connection->rollback();
    }
}
```

## 発生したこと

バッチ自体は動作しているが、レコードが更新されない。

## 原因

カンの良い方はコード時点で違和感を覚えたとおもいます。  
そうです、トランザクション中にも関わらず `continue;` で次のループのトランザクションを開始しようとしてしまっています。

MySQL ではトランザクションは1つしか開始することが出来ず、 `BEGIN`(`START TRANSACTION`) がトランザクション中に実行された場合、暗黙的にコミットされます。  
そのため単なる SQL の wrapper として動作するのであれば問題ないはずです(実際そう思って調査をしていた)。

ところが、 Eloquent はトランザクションのネストに対応させるため、トランザクション中に新たにトランザクションを開始しようとすると `SAVEPOINT` が発行されます。  
この状態でコミットしようとしても Eloquent 内部でのトランザクションの扱いが終了するだけでトランザクションは終了しません。

そして一連のバッチ処理が走り終わったあと、アプリの終了時にトランザクションが残っていた場合、 Eloquent は `ROLLBACK` を発行します。  
この時点で**最初に `continue;` をした以降のレコードがロールバックされてしまう**、というかたちになっていました。

## 原因特定の話

この現象が発生しマネージャもろもろ大変お騒がせしたのですが、この原因の特定方法についてご紹介します。

**パケットを見ました**

簡単ですね！やはりこういうケースは通信を見るのが一番早い…。
