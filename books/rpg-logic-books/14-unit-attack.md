---
title: "Unit/Attack: 攻撃モデルの管理"
emoji: "⚔️"
type: "tech"
topics: ["gamedev", "typescript", "react"]
published: false
---

# Unit/Attack: 攻撃モデルの管理

攻撃に関するパラメータを管理するクラスを実装していきます。  
`Combat/Unit/` 内に、 `Attack.ts` を新規作成し、ここに攻撃に関するパラメータを管理するクラスのコードを記述していきます。

## 攻撃モデルの設計



## 攻撃モデルの実装

```TypeScript
// src/domains/Combat/Unit/Attack.ts

const DMG_RATE = [1, 1.5, 2]

// 牽制の定義
type Feint = {
  currentTurn: boolean // true: 牽制を行ったターン (まだ適用されない), false: 次ターン以降 (適用可能)
  target: Unit
  score: number
}

// 攻撃に関するパラメータを管理するクラス
class Attack {
  // 設定値
  private self: Unit
  public name: string
  public level: number // 大盾による修正込み
  public dmg: Dmg
  public dmgName: string
  public needsReady: boolean
  // 状態値
  public feint: Feint | null
  public ready: boolean // 準備の可否

  constructor(self: Unit, model: UnitModel) {
    this.self = self
    this.name = model.equipments.weapon.name
    this.level = model.level
    this.dmg = model.equipments.getDmg(model.dmgMod)
    this.dmgName = model.equipments.getDmgName(model.dmgMod)
    this.needsReady = model.equipments.weapon.needsReady
    this.feint = null
    this.ready = true
  }

  nextTurn() {
    if (this.feint && this.feint.currentTurn) {
      // 牽制を行ったターンが終わったので, 次ターンに適用可能な状態としてマークする
      this.feint.currentTurn = false
    } else if (this.feint) {
      // 適用されないまま次のターンを迎えたので, 牽制を破棄する
      this.feint = null
    }
  }

  // 攻撃 (命中判定) の目標値を取得
  // 各種自身の状況による修正値 (バフ, デバフ) を含める
  // 各種戦闘の状況による修正値 (全力攻撃オプションによる修正) を含めない
  get target(): number {
    let target = this.level
    target += this.self.buff.level // 命中UPバフ
    target += this.self.debuff.flashed > 0 ? -2 : 0 // 目くらみ
    return Math.max(4, target)
  }

  // 攻撃 (命中判定) の目標値を取得
  // 各種状況による修正値 (バフ, デバフ, 全力攻撃オプションによる修正) を含める
  getTarget(fullPower: FullPower): number {
    const fullPowerMod = fullPower === 'level' ? 4 : 0
    return Math.max(4, this.target + fullPowerMod)
  }

  // 攻撃型によるダメージ倍率を取得
  getDmgRate(): number {
    return DMG_RATE[this.dmg.dmgType]
  }

  // 攻撃 (ダメージ判定) のためのパラメータを取得
  getDmgParams(dr: number, isChain: boolean, fullPower: FullPower): { count: number, mod: number, rate: number } {
    let count = this.dmg.dmgDice
    count -= fullPower === 'dmg' ? 1 : 0 //「ダメージ安定」
    let mod = this.dmg.dmgMod - (this.dmg.dmgType === 2 && isChain ? Math.floor(dr / 2) : dr)
    mod += this.self.buff.dmg // 攻撃UPバフ
    mod += fullPower === 'dmg' ? 6 : 0 //「ダメージ安定」
    const rate = this.getDmgRate()
    return { count, mod, rate }
  }

  // 攻撃 (ダメージ判定) の期待値を取得
  getExpectedDmg(dr: number, isChain: boolean, fullPower: FullPower) {
    const { count, mod, rate } = this.getDmgParams(dr, isChain, fullPower)
    return Math.max(0, Math.floor((count * 3.5 + mod) * rate))
  }
}

```

---

次章では、`Unit/Defense` (防御モデルの管理) の実装を進めます。
