---
title: "Raspberry Pi Pico でUSBシリアルの受信ができないとき"
category: "雑記"
tags: ["ソフトウェア"]
date: 2022-08-11 00:00:00 +09:00

permalink: /posts/20220811-raspberry-pi-pico-serial
---

## DTR と RTS を確認しましょう！！！

Arduinoシリーズ や Spresense では何も指定せずに受信できていましたが、 Raspberry Pi Pico では指定しないと受信できないようです。

```cs
new SerialPort()
{
    BaudRate = 115200,
    NewLine = "\r\n",
    ReadTimeout = 10000,
    DtrEnable = true,
    RtsEnable = true,
}
```

ちなみにデバッガのブレークポイントとかで受信が詰まるとPico自体がフリーズしてしまうようです。  
なんてこったい。
