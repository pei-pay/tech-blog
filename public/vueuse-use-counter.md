---
title: vueuse-use-counter
tags:
  - 'Vue.js'
private: true
updated_at: ''
id: null
organization_url_name: null
slide: false
ignorePublish: true
---
# VueUse の useCounter を作ってみる

[VueUse](https://vueuse.org/) にはたくさんの便利なコンポーザブルがあります。

使ったことがある人は多いと思いますが、その内部実装を見たことはありますでしょうか?

今回は [useCounter](https://vueuse.org/shared/useCounter/) を段階的に作ってみようと思います。

## どういうコンポーザブルか

リアクティブな数値のカウントをいい感じに管理するためのものです。カウントの増減、リセット、指定した値にセットなどができます。

動作についてはドキュメントの [Demo](https://vueuse.org/shared/useCounter/#demo) を触ってもらうのが一番わかりやすいと思います。

## まずは値の増減、リセットだけ

```ts
import { ref } from 'vue'

export function useCounter() {
  // カウント
  const count = ref(0)

  // カウントを増加 (increment)
  const inc = () => count.value = count.value + 1
  // カウントを減少 (decrement)
  const dec = () => count.value = count.value - 1
  // リセット
  const reset = () => count.value = 0

  return { count, inc ,dec, reset }
}
```

これについては特に解説は必要ないかなと思います。

ソースコードは少ないですが、これで値の増減(1ずつ)と初期値(0)へのリセットができます。

これを徐々に拡張して、本家の機能に近づけてみましょう。

## 増減の変化量やカウントの初期値を指定できるようにする

```ts
import { ref } from 'vue';

// 引数で初期値 (initialValue) を受け取る
export function useCounter(initialValue: number = 0) {
  // 初期値を let で管理することで更新可能にしている
  let _initialValue = initialValue
  // カウントの初期値を設定
  const count = ref(initialValue);

  // 変化量 (delta) を引数で受け取りその分増減させる
  const inc = (delta = 1) => count.value = count.value + delta;
  const dec = (delta = 1) => count.value = count.value - delta;
  // 指定した値にセットさせる関数に追加
  const set = (val: number) => count.value = val;
  // 指定した値にリセットできるように引数を受け取るよう変更 (デフォは初期値)
  const reset = (val = _initialValue) => {
    // 初期値を更新。次回以降のデフォ値もその値になる
    _initialValue = val;
    return set(val);
  };

  // set も利用できるようリターンに追加
  return { count, inc, dec, set, reset };
}
```

ポイントとしてはリセット時に初期値 (_initialValue) を与えられた引数 (`val`) で更新していることでしょうか。

これにより次回以降 `reset` が引数なしで使われた際に、その値にリセットされるようになります。

```ts
const { count, reset } = useCounter(10)

reset() // count を 10 にリセット

reset(50) // count を 50 にリセット

reset() // count を新しい初期値の 50 にリセット
```

## カウントを指定された範囲内で上限させる

```ts
import { ref } from 'vue'

export interface UseCounterOptions {
  min?: number
  max?: number
}

export function useCounter(initialValue: number = 0, options: UseCounterOptions = {}) {
  const count = ref(initialValue)

  const {
    max = Number.POSITIVE_INFINITY,
    min = Number.NEGATIVE_INFINITY,
  } = options

  const inc = () => count.value = Math.max(Math.min(max, count.value + 1), min)
  const dec = () => count.value = Math.min(Math.max(min, count.value - 1), max)
  // TODO: initialValue が範囲外だった場合は?
  const reset = () => count.value = initialValue

  return { count, inc, dec, reset }
}
```

- オプションで最小値と最大値を受け取るようにする
<!-- TODO: 工夫している部分を解説 -->
- 範囲内で増減するように工夫する


## 指定した値の増減、指定した値にリセット