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
# VueUse ソースコード解説: useCounter

<!-- NOTE: 下書き -->
<!-- TODO: 作成した簡易版の動作確認 -->

useCounter のソースコード解説を行いたいと思います。

- 一番簡単なのでわかりやすい
- 解説も楽

## どういうコンポーザブルか

数値のカウントをいい感じに管理するためのものです。

- 値の増減 (inc, dec)
- リセット (reset)

### 基本的な使い方

```vue
<script setup lang="ts">
import { useCounter } from '@vueuse/core'

const { count, inc, dec, reset } = useCounter()

</script>

<template>
  <div>
    <p>カウンター: {{ count }}</p>
    <button type="button" @click="inc">増加</button>
    <button type="button" @click="dec">減少</button>
    <button type="button" @click="reset">リセット</button>
  </div>
</template>
```

- デモ: https://vueuse.org/shared/useCounter/#demo

## まずは簡略化版を作ってみる

```ts
import { ref  } from 'vue'

export function useCounter(initialValue: number = 0) {
  const count = ref(initialValue)

  const inc = () => count.value = count.value + 1
  const dec = () => count.value = count.value - 1
  const reset = () => count.value = initialValue

  return { count, inc ,dec, reset }
}
```

- 簡易版はこれだけです。
- これを徐々に拡張していって、本家の useCounter の機能に近いものを作ってみましょう

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