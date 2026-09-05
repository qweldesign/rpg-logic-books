---
title: "キャラクタ"
emoji: "⚔️"
type: "tech"
topics: ["gamedev", "typescript", "react"]
published: false
---

# キャラクタ

キャラクタは、パラメータと装備を統合管理し、戦闘時に扱うダメージや命中・回避判定の目標値といった数値を算出します。  
`Character/` 内の `index.ts` を追記していきます。

## 1-1. キャラクタの設計

`class Character` は、メンバ変数 `parameters` で `class Parameters` を、`equipments` で `class Equipments` をそれぞれ管理します。  
これらのメンバ変数は、カプセル化するのが理想です。  
よって、`class Character` に定義していくメソッドは、`class Parameters` と `class Equipments` の各メソッドを `class Character` 内で使えるようマージしたような内容になっていきます。

加えて、キャラクタ・シートの装備欄には、武器の攻撃目標値 (技能値) 、防御目標値 (「受け: P-Ev」) 、盾の防御目標値 (「止め」: B-Ev) 、人物の防御目標値 (「よけ」: D-Ev) を記載できるように、それらの値をパラメータと装備の両方からデータを取得し、算出します。 

## 1-2. キャラクタの実装

```TypeScript
// src/domains/Character/index.ts

class Character {
  public id: number
  public name: string
  private parameters: Parameters
  private equipments: Equipments

  constructor(model: CharacterModel) {
    this.id = model.id
    this.name = model.name
    this.parameters = new Parameters(model.abilities)
    model.skills.forEach(([name, point]) => this.set(name, point))
    this.equipments = new Equipments(...model.equipments)
  }

  //
  // class Parameters のメソッドを継承
  //
  set(name: ParameterKey, point: Point) {
    this.parameters.set(name, point)
  }

  step(name: ParameterKey, size: number = 1) {
    this.parameters.step(name, size)
  }

  get(name: ParameterKey): Point {
    return this.parameters.get(name)
  }

  getLevel(name: ParameterKey): number {
    return this.parameters.getLevel(name)
  }

 get total(): number {
    return this.parameters.total
  }

  get maxHp() {
    return this.parameters.maxHp
  }

  get dmgMod() {
    return this.parameters.dmgMod
  }

  get ev() {
    return this.parameters.ev
  }

  getParam(name: ParameterKey): Parameter {
    return this.parameters.getParam(name)
  }

  get params() {
    return this.parameters.params
  }

  get skills() {
    return this.parameters.skills
  }

  // 「武術」か「剣術」のうち高い方を返す
  get combatSkill(): Parameter {
    const warrior = this.getParam('武術')
    const fencer = this.getParam('剣術')
    return (warrior.level >= fencer.level) ? warrior : fencer
  }

  // 主技能 (level が最も高いか Point 消費が最も多い技能) を取得
  // 無ければ「武術」を返す
  get mainSkill(): Parameter {
    if (this.skills.length) {
      const sorted = this.skills.sort((a, b) => {
        return b.level === a.level ? b.point - a.point : b.level - a.level
      })
      return sorted[0]
    } else {
      return {
        name: '武術', base: '筋力', point: 0, level: this.getLevel('筋力')
      }
    }
  }

  //
  // class Equipments のメソッドを継承
  //
  set weapon(weaponKey: WeaponKey) {
    this.equipments.weapon = weaponKey
  }

  set shield(shieldKey: ShieldKey | null) {
    this.equipments.shield = shieldKey
  }

  set armor(armorKey: ArmorKey) {
    this.equipments.armor = armorKey
  }

  get weapon(): Weapon & { name: WeaponKey } {
    return this.equipments.weapon
  }

  get shield(): Shield & { name: ShieldKey } | null {
    return this.equipments.shield
  }

  get armor(): Armor & { name: ArmorKey } {
    return this.equipments.armor
  }

  getDmg(): Dmg {
    return this.equipments.getDmg(this.dmgMod)
  }

  getDmgName(): string {
    return this.equipments.getDmgName(this.dmgMod)
  }

  getDRName(): string {
    return this.equipments.getDRName()
  }

  // 「よけ」: Ev - 服・鎧の重量 (DR)
  get dev(): number {
    return this.ev - this.armor.dr
  }

  // 「受け」: Ev + 武器の防御補正 (片手持ち: +1, 両手持ち: +3)
  get pev(): number {
    return this.ev + (this.weapon.twoHanded ? 3 : 1)
  }

  // 「止め」: Ev + 盾の防御補正 (小盾: +2, 大盾: +4)
  get bev(): number {
    if (!this.shield) return 0
    return this.ev + (this.shield.isLarge ? 4 : 2)
  }
}

```

「**キャラクタ**」ドメインの実装はこれだけです。  
React側で編成画面UIを組み立てれば、編成画面のキャラクタ・シートが完成します。

---

ここまでの デモ \[03\] と ソースコード (v0.3.0) は、下記URLをご参考ください。

- [RPG.LOGIC DEMO - 03](https://rpg-logic.dev/demo/03/)
- [GitHub - RPG.LOGIC DEMO v0.3.0](https://github.com/qweldesign/rpg-logic-demo/releases/tag/v0.3.0)
