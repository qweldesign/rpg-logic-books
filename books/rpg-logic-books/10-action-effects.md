---
title: "Action/Effects: 行動実行 (状態変更)"
emoji: "⚔️"
type: "tech"
topics: ["gamedev", "typescript", "react"]
published: false
---

# Action/Effects: 行動実行 (状態変更)

戦闘ユニットの行動実行 (状態変更) を司るクラスを実装していきます。  
前章で作成した `Action/` 内に、 `Effects.ts` を新規作成し、ここに戦闘ユニットの行動実行 (状態変更) のコードを記述していきます。

## 行動実行 (状態変更) の設計

行動実行による状態変更には、本プロジェクトの主題である「複雑なRPG戦闘ロジック」の大部分が詰め込まれています。  
基本的に `Effects` の各メソッドは、判定を伴う (全力防御・移動以外) 場合、`type ActionResult` の配列を返すようにします。

## ダイスによる判定処理

ダイスによる判定処理の管理は `Action/resolver` に委譲し、コードをスッキリとさせます。  
ただし、本著で `Action/resolver` を実装するのはしばらく先になりますので、テストで動かしたい場合、判定を伴わない行動である「全力防御」と「移動」で試すようにしてください。

### 攻撃の基本ルーティン

RPG.LOGIC の攻撃は、攻撃 (命中) 判定 → 防御 (回避) 判定 → ダメージ判定と、順を追って解決します。  
攻撃判定は比較的まだシンプルですが、防御判定は「受け」「止め」「よけ」の3パターンがある上、防御を行う武器によっては武器を非準備状態に更新したり、「受け」「止め」の試行回数をカウントしたり、全力防御の場合、異なる防御方法を2回試みることができたり、結構複雑です。  
また、ダメージ判定も、朦朧状態・転倒・気絶・死亡と、ダメージによる影響で更に判定が行われるため、同じく複雑です。
よって、防御判定は `tryDefend()` に、ダメージ判定は `resolveDmg()` に、それぞれ独立したメソッドとして処理を書くことにします。

### 全力攻撃 (2回攻撃・牽制即攻撃) の実装

RPG.LOGIC では、全力攻撃として、2回攻撃・牽制即攻撃というオプションを選択できます。
これらをコードの保守性を維持しつつ実装するためには、上記の攻撃の基本ルーティンを `attackRoutine()` として定義し、それをラップする `attack()` を別途定義します。
`attack()` は、全力攻撃のオプションを引数として受け取って、2回攻撃の場合は、`attackRoutine()` を2回実行する、と言う形の設計になります。

## 行動実行 (状態変更) の実装

実際には、魔法の実装を始めるタイミングで、`Action/resolver` と共に、魔法の効果ごとにメソッドが増えていきますが、基本的な戦闘行動のみの場合は、実装は下記になります。

```TypeScript
// src/domains/Combat/Action/Effects.ts

// 行動実行 (状態変更) を司るクラス / Action.execute から呼び出される
class CombatActionEffects {
  private state: State
  private formation: Formation

  constructor(state: State) {
    this.state = state
    this.formation = state.formation!
  }

  //「準備」実行
  ready(): ActionResult[] {
    this.state.actor.attack.ready = true
    return []
  }

  //「攻撃」「全力攻撃」実行
  attack(target: Unit, fullPower: FullPower): ActionResult[] {
    const actor = this.state.actor
    const results: ActionResult[] = []

    // 次のターンまで能動防御 (受け・止め・よけ) 不可
    if (fullPower !== 'none') actor.defense.isFullAttackTurn = true

    if (fullPower === 'feint') {
      // 「牽制即攻撃」: 牽制を即座に適用した上で, そのまま攻撃する
      results.push(...this.feint(target, true))
      results.push(...this.attackRoutine(target, fullPower))
    } else if (fullPower === 'double') {
      // 「2回攻撃」: 対象が気絶しなければ, 続けてもう1回攻撃する
      results.push(...this.attackRoutine(target, fullPower))
      if (!target.health.unconscious) {
        results.push(...this.attackRoutine(target, fullPower))
      }
    } else {
      // 通常攻撃, および全力攻撃オプション「ダメージ安定」「技能値+4」
      results.push(...this.attackRoutine(target, fullPower))
    }

    return results
  }

  // 攻撃1回分の判定・効果適用 (判定結果に基づき, HPへのダメージ反映と朦朧・転倒・気絶までを処理する)
  private attackRoutine(target: Unit, fullPower: FullPower): ActionResult[] {
    const results: ActionResult[] = []
    const actor = this.state.actor

    // 攻撃判定
    const attackJudge = judgeAttack(actor, fullPower)
    // 武器の準備状態を更新 (準備の要る武器の場合, 攻撃後は非準備状態になる)
    actor.attack.ready = !actor.attack.needsReady
    results.push({ type: 'attack', judge: { ...attackJudge, ready: actor.attack.ready } })
    if (!attackJudge.success) return results // 攻撃失敗時はここで処理を止める

    // 防御判定
    const canDefend = !attackJudge.critical && target.defense.canDefend
    const defenseResults = this.tryDefend(target, canDefend, () => judgeDefense(actor, target))
    for (const defenseResult of defenseResults) {
      results.push(defenseResult)
      if (defenseResult.type === 'defense' && defenseResult.judge.success) {
        return results // 防御に成功した場合はここで処理を止める
      }
    }

    // ダメージ判定
    const dmgJudge = rollDmg(actor, target, fullPower, attackJudge.critical)
    results.push(...this.resolveDmg(dmgJudge, target))

    return results
  }

  // 防御試行
  private tryDefend(target: Unit, canDefend: boolean, getDefenseJudges: () => Omit<DefenseResult, 'ready'>[]): defenseResults: ActionResult[] {
    const defenseResults: ActionResult[] = []
    const defenseJudges = getDefenseJudges()

    // 防御不能攻撃 (クリティカル) または対象が全力攻撃ターンの場合は空の結果を返す
    if (!canDefend) return defenseResults

    // 「受け」「止め」試行回数を加算
    for (const defenseJudge of defenseJudges) {
      if (defenseJudge.type === 'parry') {
        // 武器の準備状態を更新 (準備の要る武器の場合, 攻撃後は非準備状態になる)
        target.attack.ready = !target.attack.needsReady
        target.defense.parryCount++
      } else if (defenseJudge.type === 'block') {
        target.defense.blockCount++
      }

      // 判定結果をpush (ログ表示用に target も含めること)
      defenseResults.push({ type: 'defense', judge: { ...defenseJudge, ready: target.attack.ready, target } })

      // 防御に成功したら処理を抜ける
      if (defenseJudge.success) break
    }

    return defenseResults
  }

  // ダメージ効果
  private resolveDmg(dmgJudge: DmgResult, target: Unit): ActionResult[] {
    const results: ActionResult[] = []

    // 判定結果をpush (ログ表示用に target も含めること)
    results.push({ type: 'dmg', judge: { ...dmgJudge, target } })

    if (!dmgJudge.success) return results // ダメージが通らなかった時はここで処理を止める

    // ダメージ効果
    target.health.injury += dmgJudge.roll

    // 気絶・死亡判定
    // 気絶への状態遷移は Health に委譲
    if (target.health.unconscious) {
      const deadJudge = judgeEndurance(target)
      results.push({ type: 'dead', judge: deadJudge })
      if (!deadJudge.success) {
        target.health.dead = true // 死亡
      }
      return results // 以降のログ出力を止める
    }

    // 朦朧状態・転倒判定
    // 朦朧状態への状態遷移は Health に委譲
    if (target.health.stunned) {
      const knockedDownJudge = judgeEndurance(target)
      results.push({ type: 'knockedDown', judge: knockedDownJudge })
      if (!knockedDownJudge.success) {
        target.health.prone = true // 転倒
      }
    }

    return results
  }

  //「牽制」実行
  feint(target: Unit, isImmediate: boolean = false): ActionResult[] {
    const actor = this.state.actor
    const feintJudge = judgeFeint(actor, target)
    if (isImmediate && actor.attack.feint && feintJudge.success) {
      // 全力攻撃の牽制で, かつ前ターンに牽制を実行していた場合は, 効果の高い方を適用
      const prevScore = actor.attack.feint.score
      actor.attack.feint = { currentTurn: !isImmediate, target, score: Math.max(prevScore, feintJudge.score) }
    } else if (feintJudge.success) {
      actor.attack.feint = { currentTurn: !isImmediate, target, score: feintJudge.score }
    }
    return [{ type: 'feint', judge: feintJudge }]
  }

  //「全力防御」実行
  defense() {
    this.state.actor.defense.isFullDefenseTurn = true
  }

  //「移動」実行
  move(position: Position) {
    this.state.actor.position = position
  }

  // 朦朧状態からの「回復」実行 (自動実行)
  recovery(): ActionResult[] {
    const recoveryJudge = judgeEndurance(this.state.actor)
    if (recoveryJudge.success) {
      this.state.actor.health.stunned = false // 回復
    }
    return [{ type: 'recovery', judge: recoveryJudge }]
  }

  // 転倒状態からの「立ち上がり」実行 (自動実行)
  standup() {
    this.state.actor.health.standupTurn = true // 立ち上がり
  }

  //「待機」実行
  wait() {
    // 状態変更なし
  }
}

```

---

次章では、`Log` (ログの管理) の実装を進めます。
