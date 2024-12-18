---
title: VueUse の useCounter を作ってみる
tags:
  - Vue.js
private: true
updated_at: '2024-12-06T20:07:13+09:00'
id: cbdd44309615b535c9f7
organization_url_name: null
slide: false
ignorePublish: false
---
## はじめに

[VueUse](https://vueuse.org/) にはたくさんの便利なコンポーザブルがあります。

使ったことがある人は多いと思いますが、その内部実装を見たことはありますでしょうか?

今回は [useCounter](https://vueuse.org/shared/useCounter/) を段階的に作ってみることで、その内部実装を解説していきたいと思います。

対象読者：

- VueUse を使ったことがあるが、内部実装を深く理解したことがない人
- Vue のコンポーザブルの設計に興味がある人
- JavaScript/TypeScript にある程度馴染みがある人

## どういうコンポーザブルか

`useCounter` はリアクティブな数値のカウントを簡単かつ柔軟に管理するためのものです。カウントの増減、リセット、指定した値にセットなどができます。

「クリックされた回数をカウントしたい」「カート内の商品数を管理したい」などの場面で、`useCounter` は非常に便利です。

実際の動作は公式ドキュメントの [デモ](https://vueuse.org/shared/useCounter/#demo) で確認できます。

## カウントの増減とリセット機能を実装する

まずは最小限な実装として、カウントの増減とリセットだけできるようなコンポーザブルを用意します。

### 実装

```ts
import { ref } from 'vue'

export function useCounter() {
  // カウント (ref を使ってリアクティブに管理)
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

最初の実装はこれだけです。

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

もう少し使い勝手を良くしてみましょう。

### 新しく追加する機能

- カウントの増減の変化量を使用者側が指定できるようにする
- カウントの初期値を使用者側が指定できるようにする
- リセット時の値を指定できるようにする

### 実装

```ts
import { ref } from 'vue';

// 引数で初期値 (initialValue) を受け取る
export function useCounter(initialValue: number = 0) {
  // 初期値を let で管理することで更新可能にする
  let _initialValue = initialValue
  // カウントの初期値を設定
  const count = ref(initialValue);

  // 変化量 (delta) を引数で受け取りその分増減させる
  const inc = (delta = 1) => count.value = count.value + delta;
  const dec = (delta = 1) => count.value = count.value - delta;
  // 指定した値にセットさせる関数を追加
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

やっていることはシンプルですね。各関数で引数を受け取れるようにしました。

初期値 (`_initialValue`) は `let` で管理することで、リセット時に更新できるようにしています。

これにより、リセット時の値を状況に応じて動的に変化させることができます。

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

無制限に増減させるのではなく、特定の範囲内でのみ増減させたい場合があるかもしれません。

今度はその機能を実装してみましょう。

この実装が完了するとだいぶ本家の実装に近づいてきます。

### 新しく追加する機能

- カウントの増減の範囲を使用者側が指定できるようにする

### 実装

```ts
import { ref } from 'vue';

// 範囲の型。片方だけ指定したい場合もあるのでどちらもオプショナルにしている
export interface UseCounterOptions {
  min?: number; // カウントの最小値
  max?: number; // カウントの最大値
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

  // 最大値を超えないように増加
  const inc = (delta = 1) => count.value = Math.min(max, count.value + delta);
  // 最小値を超えないように減少
  const dec = (delta = 1) => count.value = Math.max(min, count.value - delta);
  // 範囲を超えないように値をセット
  const set = (val: number) => (count.value = Math.max(min, Math.min(max, val)));
  const reset = (val = _initialValue) => {
    _initialValue = val;
    return set(val);
  };

  return { count, inc, dec, set, reset };
}
```

ここでのポイントは範囲を指定しない場合でも問題なく動くように、デフォルトの値を設定していることだと思います。

`Infinity` をデフォルト値にすることで、特定の範囲を設定しない場合でも制限なくカウントが動作するようにしています。

https://developer.mozilla.org/ja/docs/Web/JavaScript/Reference/Global_Objects/Infinity

:::note info
増減の変化量 `delta` に負の数を指定する場合

ユースケースによっては `inc`, `dec` の引数 (`delta`) に負の数を指定したい場合があるかもしれません。

その場合は `inc` で最小値を下回らないように、`dec` で最大値を下回らないように実装する必要があります。

```diff
- const inc = (delta = 1) => count.value = Math.min(max, count.value + delta);
+ const inc = (delta = 1) => count.value = Math.max(Math.min(max, count.value + delta), min)
- const dec = (delta = 1) => count.value = Math.max(min, count.value - delta);
+ const dec = (delta = 1) => count.value = Math.min(Math.max(min, count.value - delta), max)
```

実は本家もこっちの実装になっています。
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

ここまでの実装と[本家のソースコード](https://github.com/vueuse/vueuse/blob/main/packages/shared/useCounter/index.ts)を比べてみましょう。ほとんど同じになっていると思います。

まだ追加できてない機能としては、カウントの値を取得する用のゲッター関数を用意したり、引数で受け取る初期値の値で `ref` を許容するというのがありますが今回は省きます。

気になる方はぜひ実装してみてください。

本記事では、`useCounter` の基本的な実装から、柔軟なオプション機能の追加までを段階的に解説しました。

今後も VueUse のコンポーザブルを 1 から作成する方法について解説していきたいと思います！
