---
title: "Action/roll: ダイスによる判定"
emoji: "⚔️"
type: "tech"
topics: ["gamedev", "typescript", "react"]
published: false
---

# Action/roll: ダイスによる判定

ダイスによる判定を行う関数群を実装していきます。  
`Combat/Action/` 内に、 `roll.ts` を新規作成し、ここにダイスによる判定を行う関数群のコードを記述していきます。

## ダイスによる判定の設計



## ダイスによる判定の実装

```TypeScript
// src/domains/Combat/Action/roll.ts

// ロール定義
type Roll = {
  roll: number // 出目
}

// 判定定義
type Judge = Roll & {
  success: boolean // 成功/失敗
  critical: boolean // クリティカル
}

// 成功度定義
type Score = Roll & {
  success: boolean
  score: number // 成功度
}

// ダイスを振った出目を取得
function getRoll(count: number = 3, mod: number = 0, sides: number = 6): number {
  return Array.from<number>({ length: count }).reduce(sum => {
    return sum + Math.ceil(Math.random() * sides)
  }, 0) + mod
}

// 判定結果 (Judge型) を返す
function judge(target: number): Judge {
  const roll = getRoll()
  const criticalTarget = Math.max(4, Math.min((target - 10), 6)) // クリティカル
  const fumbleTarget = Math.max(17, Math.min((target + 1), 18)) // ファンブル
  const success = roll <= criticalTarget || (roll <= target && roll < 17)
  const critical = roll <= criticalTarget || roll >= fumbleTarget
  return { roll, success, critical }
}

// 成功度結果 (Score型) を返す
function score(target: number): Score {
  const roll = getRoll()
  const success = roll < target
  const score = target - roll
  return { roll, success, score}
}

```

---

次章では、`Action/resolver` (判定結果の管理) の実装を進めます。
