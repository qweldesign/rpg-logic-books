---
title: "Unit/Defense: 防御モデルの管理"
emoji: "⚔️"
type: "tech"
topics: ["gamedev", "typescript", "react"]
published: false
---

# Unit/Defense: 防御モデルの管理

防御に関するパラメータを管理するクラスを実装していきます。  
`Combat/Unit/` 内に、 `Defense.ts` を新規作成し、ここに防御に関するパラメータを管理するクラスのコードを記述していきます。

## 防御モデルの設計



## 防御モデルの実装

```TypeScript
// src/domains/Combat/Unit/Defense.ts

// 防御種別の定義
type DefenseType = 'parry' | 'block' | 'dodge'

type DefenseTarget = {
  type: DefenseType
  target: number
}

// 防御に関するパラメータを管理するクラス
class Defense {
  // 設定値
  private self: Unit
  public name: {shield: string | null, armor: string}
  private ev: { self: number, weapon: number, shield: number, wt: number }
  private _dr: number
  public isChain : boolean
  public drName: string
  // 状態値
  public parryCount: number //「受け」試行回数
  public blockCount: number //「止め」試行回数
  public isFullAttackTurn: boolean //「全力攻撃」実行ターン
  public isFullAttack: boolean //「全力攻撃」可否
  public isFullDefenseTurn: boolean //「全力防御」実行ターン
  public isFullDefense: boolean //「全力防御」可否

  constructor(self: Unit, model: UnitModel) {
    this.self = self
    this.name = { shield: model.equipments.shield?.name ?? null, armor: model.equipments.armor.name }
    this.ev = {
      self: model.ev,
      weapon: model.equipments.weapon.needsTwoHanded ? 3 : 1,
      shield: model.equipments.shield.size ? (model.equipments.shield.size * 2) : 0,
      wt: model.equipments.armor.dr
    }
    this._dr = model.equipments.armor.dr
    this.isChain = model.equipments.armor.isChain
    this.drName = model.equipments.getDRName()
    this.parryCount = 0
    this.blockCount = 0
    this.isFullAttackTurn = false
    this.isFullAttack = false
    this.isFullDefenseTurn = false
    this.isFullDefense = false
  }

  // 次のターンに進む際に, 「受け」「止め」試行回数と「全力攻撃」「全力防御」をリセットする
  nextTurn() {
    this.parryCount = 0
    this.blockCount = 0
    // このターンに全力攻撃を選択したなら, 全力攻撃を true に変更
    this.isFullAttack = this.isFullAttackTurn
    this.isFullAttackTurn = false
    // このターンに全力防御を選択したなら, 全力防御を true に変更
    this.isFullDefense =  this.isFullDefenseTurn
    this.isFullDefenseTurn = false
  }

  // 防御可能の可否を取得
  get canDefend() {
    return !this.isFullAttack
  }

  // 各種防御 (回避判定) の可否状況を取得
  // 「受け」
  get canParry() {
    return (this.ev.weapon > 0
      && this.self.attack.ready
      && this.parryCount < (this.isFullDefense ? 2 : 1)
    )
  }

  // 「止め」
  get canBlock() {
    return (this.ev.shield > 0
      && this.blockCount < (this.isFullDefense ? 2 : 1)
    )
  }

  // 防御 (回避判定) の目標値を取得
  // 各種自身の状況による修正値 (バフ, デバフ, 朦朧状態, 転倒) を含める
  // 各種戦闘の状況による修正値 (牽制のターゲット, 射撃による修正等) を含めない
  // 「受け」
  get parryTarget() {
    let mod = 0
    mod += this.self.buff.ev // 回避UPバフ
    mod += this.self.debuff.flashed > 0 ? -1 : 0 // 目くらみ
    if (this.self.health.stunned) mod -= 4
    else if (this.self.health.prone) mod -= 2
    return Math.max(4, this.ev.self + this.ev.weapon + mod)
  }

  // 「止め」
  get blockTarget() {
    let mod = 0
    mod += this.self.buff.ev // 回避UPバフ
    mod += this.self.debuff.flashed > 0 ? -1 : 0 // 目くらみ
    if (this.self.health.stunned) mod -= 4
    else if (this.self.health.prone) mod -= 2
    return Math.max(4, this.ev.self + this.ev.shield + mod)
  }
  
  // 「よけ」
  get dodgeTarget() {
    let mod = 0
    mod += this.self.buff.ev // 回避UPバフ
    mod += this.self.debuff.flashed > 0 ? -1 : 0 // 目くらみ
    if (this.self.health.stunned) mod -= 4
    else if (this.self.health.prone) mod -= 2
    return Math.max(4, this.ev.self - this.ev.wt + mod)
  }

  // 可能な防御のうちで, 最も成功率の高い防御種別と目標値を取得
  get target(): DefenseTarget {
    let type, target
    if (this.canBlock) {
      type = 'block' as const
      target = this.blockTarget
    } else if (this.canParry) {
      type = 'parry' as const
      target = this.parryTarget
    } else {
      type = 'dodge' as const
      target = this.dodgeTarget
    }
    return { type, target }
  }

  get dr(): number {
    return this._dr + this.self.buff.dr // 防御UPバフ
  }

  // 防御 (回避判定) の目標値を取得
  // 各種状況による修正値 (バフ, デバフ, 朦朧状態, 転倒, 牽制のターゲット, 射撃による修正等) を含める
  // 「受け」
  getParryTarget(actor: Unit, isShoot: boolean = false, extraMod: number = 0) {
    const feint = actor.attack.feint
    const feintScore = (feint && feint.target === this.self && !isShoot) ? feint.score : 0
    const shootMod = isShoot ? -4 : 0
    return Math.max(4, this.parryTarget - feintScore + shootMod + extraMod)
  }

  // 「止め」
  getBlockTarget(actor: Unit, isShoot: boolean = false, extraMod: number = 0) {
    const feint = actor.attack.feint
    const feintScore = (feint && feint.target === this.self && !isShoot) ? feint.score : 0
    const shootMod = isShoot ? -2 : 0
    return Math.max(4, this.blockTarget - feintScore + shootMod + extraMod)
  }

  // 「よけ」
  getDodgeTarget(actor: Unit, isShoot: boolean = false, extraMod: number = 0) {
    const feint = actor.attack.feint
    const feintScore = (feint && feint.target === this.self && !isShoot) ? feint.score : 0
    return Math.max(4, this.dodgeTarget - feintScore + extraMod)
  }

  // 可能な防御のうちで, 最も成功率の高い防御種別と目標値を取得
  getTarget(actor: Unit, isShoot: boolean = false, extraMod: number = 0): DefenseTarget {
    let type, target
    if (this.canBlock) {
      type = 'block' as const
      target = this.getBlockTarget(actor, isShoot, extraMod)
    } else if (this.canParry) {
      type = 'parry' as const
      target = this.getParryTarget(actor, isShoot, extraMod)
    } else {
      type = 'dodge' as const
      target = this.getDodgeTarget(actor, isShoot, extraMod)
    }
    return { type, target }
  }
}

```

---

次章では、`Action/roll` (ダイスによる判定) の実装を進めます。
