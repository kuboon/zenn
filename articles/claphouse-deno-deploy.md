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

## WebSocket
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

## WebAudio
早くも Deno 関係ない話になりますが、 WebAudio による再生部分はこんな感じにしてみました。
拍手や笑い声の効果音って、通常は大勢の音素材をそのまま使うところですが、そこが今回のプロジェクトのこだわりポイントでして、音データには1発分の音しか入っておらず、それを WebAudio でリアルタイムに重ねて再生しています。
ついでに、ランダムに音量と再生速度を変化させることで、たくさんの音が重なった時のバリエーションを出しています。
```ts:lib/sound.ts
export function play(tag: string) {
  if (!context || context.state !== "running") return;
  const file = sample(list[tag].files);
  if (!buffers[file]) return;
  const gain = context.createGain();
  gain.connect(context.destination);
  gain.gain.value = Math.random() * 0.5 + 0.5;

  const source = context.createBufferSource();
  source.buffer = buffers[file];
  source.playbackRate.value = Math.random() + 0.5;

  source.connect(gain);
  source.start(context.currentTime);
}
```

最後の1行、 `start(context.currentTime)` これはめちゃくちゃ重要。この指定により AudioContext の現在再生中のタイミングに効果音を追加することができます。この指定をしないと、音声データの末尾に追加されていくので重なってくれません。

## useToggle
画面上に2つ並んでいるトグルボタンはそれぞれ React 要素です。
普通の発想だと、 Toggle の中で useState() によって setter を取得しますが、今回は、
ブラウザによって接続が切られた時などにトグルがオフになるようにしたかったんですよね。
コンポーネントの外側で setter/getter を使いたい場合は、 global State を持って messaging で状態更新する React saga のようなアーキテクチャを使うのが標準的だと思います。
ですが今回はそこまで大袈裟にやらずになんとかしたいな、ということで、自作の hook っぽいものから React Function Component を返すようにしています。
実装はちょっと長いので https://github.com/kuboon/claphouse/blob/main/components/useToggle.tsx を参照いただくとして、使う側はこんな感じ。

```ts
export function WireToggle() {
  const { Toggle, setIsOn } = useToggle(true);
  if (ws) {
    ws.onclose = () => {
      log("disconnected");
      setIsOn(false);
    };
  }
  return <Toggle onClick={onClick}>📶</Toggle>;
}
function onClick(newVal: boolean) {
  if (newVal) {
    wsConnect(ws.url);
  } else {
    ws.close();
  }
}
```

ドヤりたい内容は以上となります。

https://github.com/kuboon/claphouse
