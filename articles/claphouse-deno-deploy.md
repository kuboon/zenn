---
title: "ライブスピーカーに拍手を送る"
emoji: "👏"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["deno", "WebAudio", "WebSocket"]
published: true
---
# 拍手したい気持ち
オンラインライブで話を聞いている時、聴衆からのフィードバックを得る方法はいくつかありますが、シンプルに拍手だけ出来たらいいなと思いまして。

作りました、その名も claphouse
https://claphouse.kbn.one

# 使い方
イベント主催者はあらかじめ claphouse のトップページから Create Room を実行します。
そうすると uuid を含んだルームが発行されますので、ライブの開始前に聴衆にURLを配布します。
あとはおおよそ期待通りのことが起きます。拍手〜〜👏

# 技術的な話
同じ uuid を持つページを開いているブラウザを WebSocket 経由で全部つないでいます。
音の再生には WebAudio を使用し、ブラウザ上で単発の拍手を重ね合わせています。

インフラには Deno deploy を使用しています。（てってれーーー
Deno deploy の BroadcastChannel という鬼つよつよ機能のおかげで、ふつうのサーバーレスでは実現できない WebSocket 接続をあっさり実現できました。すごすぎる。。。

フレームワークには [fresh](https://github.com/lucacasonato/fresh) を採用。
README に "DO NOT USE" と書いてありますがガン無視して使いました。

aleph.js も使ってみたのですが、 Deno deploy に対応していなかったのだった。

fresh で WebSocket 繋ぐ方法はドキュメントされてませんでしたが、こうすれば動くずやろ、と勘で書いたらちゃんと動きました。凄いですねえ。

```ts:pages/ws/[uuid].ts
import { Handlers } from "https://raw.githubusercontent.com/lucacasonato/fresh/main/server.ts";

export const handler: Handlers = {
  GET(ctx) {
    const { req } = ctx;
    const { socket, response } = Deno.upgradeWebSocket(req);
    if (!socket) throw new Error("unreachable");

    const uuid = ctx.match["uuid"];
    // ...
    return response;
  },
};
```

https://github.com/kuboon/claphouse
