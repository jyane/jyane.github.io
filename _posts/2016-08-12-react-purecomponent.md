---
layout: post
title: React v15.3 で新たに追加された PureComponentについて
date: 2016-08-12 0:00 JST
description: React v15.3 で新たに追加された PureComponentについて
tags:
---

React v15.3 から新たなトップレベルAPIとして PureComponent が追加されました。

<strong style="color: #d00000">2016年8月12日現在、公式 Document にも未だ記載されていない内容のため、誤りが含まれる可能性があります。</strong>

## TL;DR
React PureComponent はデフォルトの `shouldComponentUpdate` を変更した Component である。
shallow な比較によって Update するかしないかを決定する。

## PureComponent について
PureComponent は React v15.3 から追加されたトップレベルAPIです。

[Add React.PureComponent #7195](https://github.com/facebook/react/pull/7195)

### 使い方
通常の Component と変わりません。

``` js
import React, { PureComponent } from 'react';

class Pure extends PureComponent {
  render() {
    return (
      <p>pure</p>
    );
  }
}
```

### メリット
React は仮想DOMを用いることで、他の View をレンダリングするライブラリと比べて比較的高速に動作するとされています。
しかし、レンダリングする数が多くなってくると、React であってもパフォーマンスチューニングが必要となる局面が出てきます。

React のパフォーマンスを向上させる時、多くの場合 `shouldComponentUpdate` を書き換えることが基本的な戦略となります。

React は Component を Update するか決定する過程で、Component の `shouldComponentUpdate` を実行し、Update する必要があるか判断します。
Reactは `shouldComponentUpdate` で `true` を返すと Update する必要があると判断し、`false` を返すと Update をする必要がないと判断します。

PureComponent はこの `shouldComponentUpdate` のデフォルトの挙動を変更した Component です。
今まで持っていた props と新しく与えられた props を shallow な比較をして `true` か `false` を返します。

[該当部分のコード](https://github.com/spicyj/react/blob/aab1fd6e6af43aacb36f2e2006d3fc9245e064ec/src/renderers/shared/stack/reconciler/ReactCompositeComponent.js#L874-L876)

これによって今までと比較して、Update が実行されにくくなります。
shallow な比較をして変化がないと考えられた Component は Update されなくなります。

### デメリット
shallow な比較を行う計算コストはそこまで低くない（`shouldComponentUpdate`はやたら呼び出される）ため、全ての Component をとにかく置き換えると逆に遅くなることがあります。

### 注意
名前から勘違いしてしまいそうですが、stateless ではありません。
内部に state を持つことも出来ます。
また通常の Component を同じように lifeCycle に関連する関数を上書きすることもできます（`shouldComponentUpdate`を含む）。
stateless な Component を扱いたい場合には今まで通り、Functional Stateless Component を用いることになります。
