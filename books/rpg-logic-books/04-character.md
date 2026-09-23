---
title: "Character: キャラクタ管理"
emoji: "⚔️"
type: "tech"
topics: ["gamedev", "typescript", "react"]
published: false
---

# Character: キャラクタ管理

キャラクタは、パラメータと装備を統合管理し、戦闘時に扱うダメージや命中・回避判定の目標値といった数値を算出します。  
`Character/` 内の `index.ts` を追記していきます。

## キャラクタの設計

`class Character` は、メンバ変数 `parameters` で `class Parameters` を、`equipments` で `class Equipments` をそれぞれ管理します。

### カプセル化

これまでの実装では、`parameters` と `equipments` を `public` なメンバ変数として定義していました。  
これは、まず動くものを優先して作ったためですが、このままではUI側のコードから `Character` を素通りして中の実装に直接アクセスできてしまいます。  
そのためこの2つのメンバ変数は `private` にし、必要な操作はすべて `Character` 自身のメソッド・プロパティ経由で行うようにします。

```TypeScript

class Character {
  public id: number
  public name: string
  private parameters: Parameters
  private equipments: Equipments

  ...
}

```

これにより、UI側は `character.getLevel('筋力')` や `character.weapon` のように、`Character` が公開しているインターフェースのみを使うことになります。

よって、`Character` に定義していくメソッドは、`Parameters` と `Equipments` の各メソッドを `Character` 内で使えるようマージしたような内容になっていきます。

### ドメインをまたぐ算出ロジック

キャラクタ・シートには、パラメータ単体でも装備単体でもなく、**両方の情報を組み合わせて初めて求まる値**がいくつかあります。  
こうしたロジックは `Parameters` にも `Equipments` にも置き場がないため、両方を知っている `Character` に実装します。 

## キャラクタの実装

```TypeScript
// src/domains/Character/index.ts

// キャラクタの管理を司るクラス
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
    this.equipments = model.equipments.length ? new Equipments(...model.equipments) : new Equipments()
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
    const result = (warrior.level >= fencer.level) ? warrior : fencer
    const hasLargeShield = this.shield.size > 1 ? 2 : 0
    return { ...result, level: result.level - hasLargeShield}
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

  set shield(shieldKey: ShieldKey) {
    this.equipments.shield = shieldKey
  }

  set armor(armorKey: ArmorKey) {
    this.equipments.armor = armorKey
  }

  get weapon(): Weapon & { name: WeaponKey } {
    return this.equipments.weapon
  }

  get shield(): Shield & { name: ShieldKey } {
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
    return this.ev + (this.weapon.needsTwoHanded ? 3 : 1)
  }

  // 「止め」: Ev + 盾の防御補正 (小盾: +2, 大盾: +4)
  get bev(): number {
    if (!this.shield.size) return 0
    return this.ev + (this.shield.size ? 4 : 2)
  }
}

```

「**キャラクタ**」ドメインの実装はこれだけです。  
React側で編成画面UIを組み立てれば、編成画面のキャラクタ・シートが完成します。

---

次章では、本章で作成したメソッドを活用して、サンプル・キャラクタ生成を実装していきたいと思います。
ここまでの デモ \[00-03\] は、下記URLをご参考ください。

- [DEMO 00-03 | キャラクタの実装 | RPG.LOGIC](https://rpg-logic.dev/demo/00/03/)
