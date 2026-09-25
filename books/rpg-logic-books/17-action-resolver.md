---
title: "Action/resolver: 判定結果の管理"
emoji: "⚔️"
type: "tech"
topics: ["gamedev", "typescript", "react"]
published: false
---

# Action/resolver: 判定結果の管理

判定結果の管理を行う関数群を実装していきます。  
`Combat/Action/` 内に、 `resolver.ts` を新規作成し、ここに判定結果の管理を行う関数群のコードを記述していきます。

## 判定結果の設計

攻撃判定、防御判定、ダメージ判定の他、牽制の判定結果を返す関数、生命力の判定結果を返す関数を実装します。

### 各関数の返り値の型について

それぞれの判定は、ログ出力に用いるため、`type ActionResult` の型のパターンである `type AttackResult`、`type DefenseResult`、`type DmgResult` の型に準拠します。  
このうち防御判定は、全力防御で2回結果を返す可能性があるため、配列を返します。  
攻撃結果や防御結果で、武器の準備状態を更新する場合がありますが、判定とは直接的に関係しないため、型を `Ommit` しています。

## 判定結果の実装

実際には、魔法の実装を始めるタイミングで、`Action/Effects` と共に、魔法の効果ごとに関数が増えていきますが、基本的な戦闘行動のみの場合は、実装は下記になります。

```TypeScript
// src/domains/Combat/Action/resolver.ts

// 攻撃の判定結果を返す
function judgeAttack(actor: Unit, fullPower: FullPower): Omit<AttackResult, 'ready'> {
  const attackTarget = actor.attack.getTarget(fullPower)
  return judge(attackTarget)
}

// 防御の判定結果を配列で返す
// 可能な防御のうちで, 最も成功率の高い防御を自動選択する
// 全力防御選択中は, 最初の防御に失敗しても, 残り試行回数の範囲で別の防御を続けて試みる
// いずれかが成功すればそこで処理を終了する
function judgeDefense(actor: Unit, target: Unit): Omit<DefenseResult, 'ready'>[] {
  const defense = target.defense
  const maxAttempts = defense.isFullDefense ? 2 : 1
  const results = []

  if (defense.canBlock) {
    const blockResult = { ...judge(defense.getBlockTarget(actor)), type: 'block' as const }
    results.push(blockResult)
    if (blockResult.success) return results
  }

  if (defense.canParry && results.length < maxAttempts) {
    const parryResult = { ...judge(defense.getParryTarget(actor)), type: 'parry' as const }
    results.push(parryResult)
    if (parryResult.success) return results
  }

  if (results.length < maxAttempts) {
    const dodgeResult = { ...judge(defense.getDodgeTarget(actor)), type: 'dodge' as const }
    results.push(dodgeResult)
  }
  return results
}

// ダメージの判定結果を返す
function rollDmg(actor: Unit, target: Unit, fullPower: FullPower, isCritical: boolean = false): DmgResult {
  let { dr, isChain } = target.defense
  if (isCritical) dr = 0 // クリティカル時はダメージ貫通
  const { count, mod, rate } = actor.attack.getDmgParams(dr, isChain, fullPower)
  const roll = Math.max(0, Math.floor(getRoll(count, mod) * rate))
  return { roll, success: roll > 0, critical: roll >= 10 }
}

// 牽制の判定結果を返す (成功度がそのまま target の次の防御目標値へのペナルティになる)
function judgeFeint(actor: Unit, target: Unit): FeintResult {
  return { target, ...score(actor.attack.target) }
}

// 生命力判定の結果を返す (転倒判定・回復判定・死亡判定)
function judgeEndurance(target: Unit): Judge {
  return judge(target.pre)
}

```

---

本章までで「第3フェーズ: 戦闘の実装・中編 (戦闘行動の実装)」を終え、戦闘の基本実装を完了させることができました。  
次章以降、最終フェーズとして、魔法の実装と敵AIの実装を進めます。  
ここまでの デモ \[05\] とソースコード (v0.5.0) は、下記URLをご参考ください。

- [DEMO 05 | 戦闘行動の実装 | RPG.LOGIC](https://rpg-logic.dev/demo/05/)
- [GitHub - RPG.LOGIC v0.5.0](https://github.com/qweldesign/rpg-logic/releases/tag/v0.5.0)
