# シングルプロセス・シングルスレッドとコールバック

コンピューティング。特に並列、並行処理をするプログラミングに入ってくるとプロセス、スレッドという言葉を耳にするようになります。

JavaScriptはシングルプロセス、シングルスレッドの言語です。これは言い換えるとすべてのプログラムは直列に処理されることを意味します。シングルスレッドの言語はコールスタックも1個です。

コールスタックとは実行している関数の呼び出しの順序を司っているものです。スタックという言葉自体は関数の再帰呼び出しを誤って無限ループにしてしまった時に目にしたことがある人が多いのではないでしょうか。

```typescript
function stack(): never {
  stack();
}

stack();
```

```typescript
RangeError: Maximum call stack size exceeded
```

## ブロッキング

直列に処理されるということは、時間のかかる処理があるとその間は他の処理が実行されないことを意味します。

ブラウザでAJAX通信を実装したことがある方は多いでしょう。AJAXはリクエストを送信してからレスポンスを受信するまでの間は待ち時間になりますが、直列で処理するのであればJavaScriptはこの間も他の処理ができないことになります。これをブロッキングと言います。

JavaScriptはブラウザで発生するクリック、各種input要素からの入力、ブラウザの履歴の戻る進むなど、各種イベントをハンドリングできる言語ですが、時間のかかる処理が実行されている間はブロッキングが起こるためこれらの操作をハンドリングできなくなります。画面の描画もJavaScriptに任せていた場合はさらに画面が止まったように見えるでしょう。

```typescript
ajax('https://...');
wait(3000);

if (!ajaxDone()) {
  cancelAjax();
}
```

上記のメソッドたちはどれも実在するメソッドではありませんが、おおよその意味を理解していただければ問題ありません。これを先入観なく見ると

1. AJAXを開始する
2. 3000ms待つ
3. AJAXが終わっていなかったら
   1. AJAXを中止する

のように見えるかもしれませんが、これはその意図した動作にはなりません。実際には次のようになります。

1. AJAXをして、結果を取得する（ブロックして戻ってきたら2に進む\)
2. 3000ms待つ
3. AJAXが終わっていなかったら\(すでに終了している\)
4. AJAXを中止する

となります。もちろん`ajaxDone()`は`ajax()`の時点で結果にかかわらず終了しているため`cancelAjax()`は実行されません。

## ノンブロッキング

ブロッキングの逆の概念です。Node.jsはノンブロッキングI/Oを取り扱うことができます。

これは入出力の処理が終わるまで待たずにすぐに呼び出し元に結果を返し、追って別の方法で結果を伝える方式を指します。  
ここで指している入出力とはアプリケーションが動くマシン\(サーバー\)が主にリポジトリと呼ばれるようなファイル、リクエスト、DBなど他のデータがある場所へのアクセスを指す時に使われます。

ノンブロッキングかわかりやすい例としては次のようなものがあります。

```typescript
console.log('first');

setTimeout(() => {
  console.log('second');
}, 1000);

console.log('third');
```

`setTimeout()`は実際に存在する関数です。第2引数では指定したミリ秒後に第1引数の関数を実行します。ここでは1000を指定しているので、1000ミリ秒、つまり1秒後となります。

JavaScriptを始めて日が浅い方はこのコードに対する出力を次のように考えます。

```typescript
first
second
third
```

実際の出力は以下です。

```typescript
first
third
second
```

`setTimeout()`がノンブロッキングな関数です。この関数は実行されると第1引数の関数をいったん保留し、処理を終えます。そのため次の`console.log('third')`が実行され、1000ミリ秒後に第1引数の関数が実行され、中にある`console.log('second')`が実行されます。

1000ミリ秒は待ちすぎ、もっと短ければ意図する順番とおりに表示される。と思われるかもしれませんが、基本的に意図するとおりにはなりません。以下は第2引数を1000ミリ秒から0ミリ秒に変更した例ですが、出力される内容は変更前と変わりません。

```typescript
console.log('first');

setTimeout(() => {
  console.log('second');
}, 0);

console.log('third');

// ->
//   'first'
//   'third'
//   'second'
```

現実世界の料理に例えてみるとわかりやすいかもしれません。お米を炊いている40分間、ずっと炊飯器の前で待機する料理人はおらず、その間に別のおかずを作るでしょう。

時間はかかるものの待機が多い作業、炊飯器なら炊飯ボタンを押したら炊き上がるまでの間待たずに他の処理の実行に移ることがノンブロッキングを意味します。

## ノンブロッキングを成し遂げるための立役者

ノンブロッキングを語る上で欠かせない、必ず目にすることになるであろう縁の下の力持ちを紹介します。

### メッセージキュー

メッセージキューとはユーザーからのイベントや、ブラウザからのイベントなどを一時的に蓄えておく領域です。メッセージキューに蓄積されたイベントはコールスタックが空のときにひとつずつコールスタックに戻されます。

### コールバック

`setTimeout()`のときに説明した**いったん保留した関数**は、いわゆるコールバック関数と呼ばれます。前項で述べた、**追って別の方法で伝える**というのは、このコールバック関数のことです。

コールバック関数は、ある関数が条件を満たした時、前項の例だと1000ミリ秒後に、メッセージキューに蓄積されます。メッセージキューに蓄積されるだけなので、実際に実行されるのはコールスタックが空になるまでさらに時間がかかります。

いままで`setTimeout()`は第2引数のミリ秒だけ遅延させてコールバック関数を実行すると説明していましたが、厳密にはミリ秒経過後にメッセージキューに戻すだけで、そのコールバック関数が即座に実行されるわけではありません。

### イベントループ

イベントループは単純な無限ループです。常にコールスタックを監視しており、イベントがあればそれを実行します。普通の関数呼び出しのスタック以外にもメッセージキューが戻してきたイベントも処理します。現時点では詳しくは説明しませんが、ずっとイベントをどうにかしてくれているやつがいるなー、程度の認識でオッケーです！

## ノンブロッキングの弊害

ノンブロッキングにはたくさんいいところがあって頼れる仲間ですが、そのノンブロッキングが時として唐突に牙を剥くことがあります。こわいですねぇ。

### コールバック地獄\(Callback hell\)

コールバック界における**負の産物**です。

一般的にコールバックは、ある一定の時間を要する処理結果を後から受け取るために使われます。コールバックを採用している関数は主に次のような形をしています。

```typescript
function ajax(uri: string, callback: (res: Response) => void): void {
  // ...
}
```

この関数を使う時はこのようになります。

```typescript
ajax('https://...', (res: Response) => {
  // ...
});
```

ここで、この関数`ajax()`の結果を受けてさらに`ajax()`を使いたいとすると、このようになってしまいます。

```typescript
ajax('https://...', (res1: Response) => {
  ajax('https://...', (res2: Response) => {
    // ...
  });
});
```

インデント\(ネスト\)が深くなります。これが何度も続くと見るに堪えなくなります。

```typescript
ajax('https://...', (res1: Response) => {
  ajax('https://...', (res2: Response) => {
    ajax('https://...', (res3: Response) => {
      ajax('https://...', (res4: Response) => {
        ajax('https://...', (res5: Response) => {
          ajax('https://...', (res6: Response) => {
            // ...
          });
        });
      });
    });
  });
});
```

これは波動拳ネストとも呼ばれているようですw。このコールバック地獄を解消する画期的なクラスとして`Promise`が登場し主要なブラウザとNode.jsではビルトインオブジェクトとして使うことができます。こちらの説明については本書に専用のページがありますのでそちらをご参照ください。

{% page-ref page="promise-async-await.md" %}

