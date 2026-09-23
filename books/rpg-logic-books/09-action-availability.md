---
title: "Action/Availability: 行動可否判定の管理"
emoji: "⚔️"
type: "tech"
topics: ["gamedev", "typescript", "react"]
published: false
---

# Action/Availability: 行動可否判定の管理

戦闘ユニットの行動可否判定の管理を司るクラスを実装していきます。  
前章で作成した `Action/` 内に、 `Availability.ts` を新規作成し、ここに戦闘ユニットの行動可否判定の管理のコードを記述していきます。

## 行動可否判定の設計

各種コマンドごとの行動可否判定メソッドを定義していきます。  
魔法の実装はまだ先になりますが、「狂戦士」と「幻惑」状態は、コマンド制限によって効果が適用されます。

## 行動可否判定の実装

```TypeScript
// src/domains/Combat/Action/Availability.ts

// 行動可否判定を司るクラス / Action.availability に対応
class CombatActionAvailability {
  private state: State

  constructor(state: State) {
    this.state = state
  }

  //「準備」実行可否取得
  // 武器が非準備状態であること, かつ幻惑状態ではないこと
  canReady(): boolean {
    return !this.state.actor.attack.ready && !this.state.actor.debuff.dazed
  }

  //「攻撃」「全力攻撃」実行可否基本条件
  // 自身が前方に配置されていること, かつ幻惑状態ではないこと (武器の準備状態は含めない)
  canAttackBase(): boolean {
    return this.state.actor.position !== 'back' && !this.state.actor.debuff.dazed
  }

  //「攻撃」実行可否取得
  // 武器が準備状態, かつ狂戦士状態ではないこと (暫定)
  canAttack(): boolean {
    return this.canAttackBase() && this.state.actor.attack.ready && !this.state.actor.debuff.berserk
  }

  //「全力攻撃」実行可否取得
  canFullPowerAttack(): boolean {
    return this.canAttackBase()
  }
  
  // 「2回攻撃」実行可否取得
  // 攻撃毎に準備を要する武器でないこと
  canDoubleAttack(): boolean {
    return this.canFullPowerAttack() && !this.state.actor.attack.needsReady
  }

  //「牽制」実行可否取得
  // 「攻撃」と同条件
  canFeint(): boolean {
    return this.canAttack()
  }

  //「全力防御」実行可否取得
  // 狂戦士状態ではないこと
  canDefense(): boolean {
    return !this.state.actor.debuff.berserk
  }

  //「移動」実行可否取得 (移動先を指定)
  // 後退: 自身が後方に配置されていないこと, かつ狂戦士状態ではないこと
  // 前進: そこへ既に他の味方ユニットが配置されていないこと
  canMoveToPosition(position: Position): boolean {
    const actor = this.state.actor
    const formation = this.state.formation
    if (!formation) return false
    if (position === 'back') {
      if (actor.debuff.berserk) return false
      return formation[actor.side].back[actor.combatId] === null ? true : false
    } else {
      return formation[actor.side].front[position] === null ? true : false
    }
  }

  //「移動」実行可否取得 (全ての移動先に対しての結果)
  canMove(): boolean {
    const actor = this.state.actor
    const formation = this.state.formation
    if (!formation) return false
    if (actor.position === 'back') {
      return POSITION_KEYS.some(position => formation[actor.side].front[position] === null ? true : false)
    } else {
      if (actor.debuff.berserk) {
        return POSITION_KEYS.some(position => formation[actor.side].front[position] === null ? true : false)
      } else {
        return true
      }
    }
  }

  //「待機」実行可否取得
  canWait(): boolean {
    return false
  }
}

```

---

次章では、`Action/Effects` (行動実行 (状態変更) の管理) の実装を進めます。
