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

今回は [useCounter](https://vueuse.org/shared/useCounter/) を段階的に作ってみることで、その内部実装を理解していこうと思います。

## どういうコンポーザブルか

`useCounter` はリアクティブな数値のカウントをいい感じに管理するためのものです。カウントの増減、リセット、指定した値にセットなどができます。

実際の動作については公式ドキュメントに [Demo](https://vueuse.org/shared/useCounter/#demo) があるので、それを動かしてもらうのがわかりやすいと思います。

## カウントの増減、リセット

まずは最小限な実装としてカウントの増減(1 ずつ)とリセット(0 固定)だけできるようなコンポーザブルを作ってみます。

### 実装

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

特に詳細な解説も必要ないかなと思います。

これを徐々に拡張して、本家の機能に近づけてみましょう。


<details><summary>使用例</summary>

```vue
<script setup lang="ts">
import { useCounter } from '@/composables/useCounter';

const { count, inc, dec, reset } = useCounter();
</script>

<template>
  <div>
    <p>Count: {{ count }}</p>
    <button type="button" @click="inc()">
      Increment
    </button>
    <button type="button" @click="dec()">
      Decrement
    </button>
    <button type="button" @click="reset()">
      Reset
    </button>
  </div>
</template>
```

</details>

## 使用者側で増減の変化量などを指定できるようにする

次に下記の機能を追加してみましょう。

- カウントの増減の変化量を使用者側が指定できるようにする
- カウントの初期値を使用者側が指定できうようにする
- リセット時の値を指定できるようにする

### 実装

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

ポイントとしてはリセット時に初期値 (`_initialValue`) を与えられた引数 (`val`) で更新していることでしょうか。

これにより次回以降 `reset` が引数なしで使われた際に、その値にリセットされるようになります。

これは本家がそうなっているので合わせたのですが、別にそんな機能いらない場合は省いてもいいと思います。

```ts
const { count, reset } = useCounter(10)

reset() // count を 10 にリセット

reset(50) // count を 50 にリセット

reset() // count を新しい初期値の 50 にリセット
```

<details><summary>使用例</summary>

```vue
<script setup lang="ts">
import { useCounter } from '@/composables/useCounter';

const { count, inc, dec, set, reset } = useCounter(10);
</script>

<template>
  <div>
    <p>Count: {{ count }}</p>
    <button type="button" @click="inc()">
      Increment
    </button>
    <button type="button" @click="dec()">
      Decrement
    </button>
    <button type="button" @click="inc(5)">
      Increment (+5)
    </button>
    <button type="button" @click="dec(5)">
      Decrement (-5)
    </button>
    <button type="button" @click="set(100)">
      Set (100)
    </button>
    <button type="button" @click="reset()">
      Reset
    </button>
    <button type="button" @click="reset(20)">
      Reset (20)
    </button>
  </div>
</template>
```

</details>

## カウントを指定された範囲内で増減させる

無制限に増減させるのではなく、決められた範囲内でのみ増減させたい場合があるかもしれません。

今度はその機能を実装してみましょう。

### 実装

```ts
import { ref } from 'vue';

// 範囲の型。片方だけ指定したい場合もあるのでどちらもオプショナルにしている
export interface UseCounterOptions {
  min?: number;
  max?: number;
}

// 引数に範囲のオプションを追加 (デフォは空オブジェクト)
export function useCounter(initialValue: number = 0, options: UseCounterOptions = {}) {
  let _initialValue = initialValue;
  const count = ref(initialValue);

  // 最小値と最大値を取り出す。指定されてない場合はデフォルト値を設定
  const {
    // デフォは正の無限大 (Infinity)
    max = Number.POSITIVE_INFINITY,
    // デフォは負の無限大 (-Infinity)
    min = Number.NEGATIVE_INFINITY,
  } = options;

  const inc = (delta = 1) => count.value = Math.min(max, count.value + delta);
  const dec = (delta = 1) => count.value = Math.max(min, count.value - delta);
  const set = (val: number) => (count.value = Math.max(min, Math.min(max, val)));
  const reset = (val = _initialValue) => {
    _initialValue = val;
    return set(val);
  };

  return { count, inc, dec, set, reset };
}
```

ここでのポイントは範囲を指定しない場合でも問題なく動くようにデフォルトの値を設定していることだと思います。

カウントをいくら増減しても範囲を超えないように最大値には正の無限大数、最小値には負の無限大数を設定しています。

https://developer.mozilla.org/ja/docs/Web/JavaScript/Reference/Global_Objects/Infinity

:::note info
ユースケースによっては `inc`, `dec` の引数 (`delta`) に負の数を指定したい場合があるかもしれません。

その場合は、`inc` で最小値を下回らないように、`dec` で最大値を下回らないように実装する必要があります。

```diff
- const inc = (delta = 1) => count.value = Math.min(max, count.value + delta);
+ const inc = (delta = 1) => count.value = Math.max(Math.min(max, count.value + delta), min)
- const dec = (delta = 1) => count.value = Math.max(min, count.value - delta);
+ const dec = (delta = 1) => count.value = Math.min(Math.max(min, count.value - delta), max)
```

本家もこっちの実装になっています。

関連イシュー  
https://github.com/vueuse/vueuse/pull/3650
:::

<details><summary>使用例</summary>

```vue
<script setup lang="ts">
import { useCounter } from '@/composables/useCounter';

const { count, inc, dec, set, reset } = useCounter(10, { min: 0, max: 100 });
</script>

<template>
  <div>
    <p>Count: {{ count }}</p>
    <button @click="inc()">
      Increment
    </button>
    <button @click="dec()">
      Decrement
    </button>
    <button @click="inc(5)">
      Increment (+5)
    </button>
    <button @click="dec(5)">
      Decrement (-5)
    </button>
    <button @click="set(100)">
      Set (100)
    </button>
    <button @click="reset()">
      Reset
    </button>
  </div>
</template>
```

</details>

## 最後に

実はここまでの実装で本家のソースコードとほぼ同じになっています。

本家: https://github.com/vueuse/vueuse/blob/main/packages/shared/useCounter/index.ts

まだ追加できてない機能としては、カウントの値を取得するようのゲッター関数を用意したり、引数で受け取る初期値の値で `ref` を許容すると言うのがありますが、今回は省きます。気になる方はソースコードを参考に実装してみてください。

`useCounter` は VueUse の中でも簡単な実装なので、そこまでソースコードも多くないですね。今後も他のコンポーザブルを作ってみる記事を書きたいと思います！

