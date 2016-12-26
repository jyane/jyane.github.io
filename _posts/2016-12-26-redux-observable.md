---
layout: post
title: redux-observable の処理を marble testing で簡単にテストする
date: 2016-12-26 0:00 JST
description: redux-observable を marble testing で簡単にテストする
tags:
---

## TL;DR
redux-observable の公式ドキュメントでは Redux の Store を Mock するテスト手法が紹介されており、redux-saga に比べて Test が記述しにくい印象を持ちます。
しかし、Rx の Test 手法である marble testing を用いることで簡単に非同期の Test を記述することができます。

## redux-observable とは
redux-observable とは Redux の Middleware です。
Redux は自身では非同期処理や副作用に対応できないため、これらを用いるには何らかの Middleware に頼る必要があります。
そこで、redux-observable を用いることになります。
redux-observable は RxJS でアクションを受け取りアクションを返す、Epic と呼ばれる Stream で記述されます（redux-saga の Saga のようなものです）。
RxJSは非同期にとても強く、他の非同期処理との連携なども容易に記述することができ、[NetflixもReduxの問題解決にこの Middleware を採用](https://www.youtube.com/watch?v=AslncyG8whg)しています。（そもそも redux-observable や RxJS が Netflix の人が開発しているものだったりします）

## redux-observableの例
redux-observable を用いた場合、非同期処理は以下のように記述できます。

``` js
// action.js
const fetchUser = (id) => ({
  type: 'FETCH_USER',
  id
})

const fetchUserSuccess = (payload) => ({
  type: 'FETCH_USER_SUCCESS',
  payload
})

const fetchUserFailure = (e) => ({
  type: 'FETCH_USER_FAILURE',
  e
})

// epic.js
const fetchUser = (action$) =>
  action$.ofType('FETCH_USER').mergeMap((action) =>
    api.getUser(action.id)
      .map(fetchUserSuccess)
      .catch(fetchUserFailure)
  )
```

また、非同期処理同士の連携も非常に簡単に記述できます。
例えば、あるAPIの情報を用いてさらに他のAPIへアクセスすることはフロントエンドでは頻発します。
このような処理でも以下のように簡潔に記述することができます。

``` js
// fetchUserが成功したら（FETCH_USER_SUCCESSが発行されたら）、
// fetchUserDetail でさらに詳細なユーザーの情報を取得する
const fetchUserDetail = (action$) =>
  action$.ofType('FETCH_USER_SUCCESS').mergeMap((action) =>
    api.getUserDetail(action.payload.foo)
      .map(fetchUserDetailSuccess)
      .catch(fetchUSerDetailFailure)
  )
```

## redux-observable で marble testing を行う
redux-observable の[公式ドキュメントの Test 方法](https://redux-observable.js.org/docs/recipes/WritingTests.html)は Redux の Store を mock する方法で少し面倒です。
ここでは、redux-observable の epic はアクションを取ってアクションを返すだけの簡単な Stream であることに着目して、Rxのテスト手法である marble testing を適用してみます。
marble testing は Rx の説明でよくある [”矢印と玉”](http://reactivex.io/documentation/operators/map.html) で Stream を表現することで直感的に Stream を再現するTest手法です。
marble testing の詳細については、他の方々の説明が詳しい（一番下に参考を載せています）のでここでは割愛します。
redux-observable は Observable を ActionObservable と呼ばれる独自の stream に拡張しているので、単純には適用できませんが、
ActionObservable が module として与えられているので、これを用いると簡単に ActionObservable を生成できます。
例えば以下のように記述することができます。

``` js
import ...
import { ActionObservable } from 'redux-observable';


// Test Runner に Jestを用いています。

describe('Test userEpic', () => {
  let testScheduler;
  let cold;

  beforeEach(() => {
    jest.resetModules();

    testScheduler = new TestScheduler((expected, actual) => {
      expect(expected).toEqual(actual);
    });

    cold = createCold(testScheduler);
  });

  afterEach(() => testScheduler.flush());

  it('FETCH_USERのテスト', () => {
    // jest で クライアントの挙動をまるごと mock します。
    jest.mock('src/utils/client', () => ({
      api: {
        getUser: () => Rx.Observable.of({ data: { foo: 'bar' }})
      }
    }));
    const userEpic = require('src/epics/user').default;

    // FETCH_USER のアクションが 40 ms のタイミングで流れてきた
    const input$  = cold('---a--', { a: { type: 'FETCH_USER' }});
    const expect$ =      '---b';

    // 通常の marble testing の場合は、そのまま input$ を渡します。
    // redux-observable の場合は、ここで ActionObservable を作成します
    const test$ = userEpic(new ActionObservable(input$), testScheduler);

    // FETCH_USER_SUCCESS が発行されるか
    testScheduler.expectObservable(test$).toBe(expect$, { b: { type: 'FETCH_USER_SUCCESS', payload: { foo: 'bar' }}});
  });
});
```

## まとめ
redux-observable について軽く紹介を行ったあと、Test方法について記述しました。
redux-observable に興味を持ち軽く調べただけでは、redux-saga に比べて Test が面倒に感じます。
しかし、上記のように marble testing を用いて直感的かつ簡単に Test を記述することがわかります。

## 参考

[RxJS Marble Test 入門](http://qiita.com/ovrmrw/items/62117d9f4a859e1688fa)

[RxJS(5.x)で行うテストファーストな機能開発](http://blog.mmmcorp.co.jp/blog/2016/06/25/testing-rxjs-5/)
