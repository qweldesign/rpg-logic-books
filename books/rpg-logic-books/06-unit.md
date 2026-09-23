---
title: "Unit: 戦闘ユニットの管理"
emoji: "⚔️"
type: "tech"
topics: ["gamedev", "typescript", "react"]
published: false
---

# Unit: 戦闘ユニットの管理

戦闘場面の実装に入る準備として、戦闘ユニットを管理するクラスを実装しておきます。  
`domains/` 内に `Combat/` を配置し、 `Unit/index.ts` を新規作成し、ここに戦闘ユニットを管理するクラスのコードを記述していきます。

## 戦闘ユニットの設計

### ユニットモデルの定義

戦闘ユニットを管理するクラス `Unit` は、前章までで作成したキャラクタを管理するクラス `Character` の各種パラメータを引き継いで初期化します。  
`Character` に下記コードを追加し、ユニットモデル (`type UnitModel`) 用データに変換する `getter` を設定します。

```TypeScript
// src/domains/Character/index.ts

// キャラクタの管理を司るクラス
class Character {

  ...

  // 戦闘モデル用データ変換
  get unitModel(): UnitModel {
    return {
      name: this.name,
      maxHp: this.maxHp,
      level: this.combatSkill.level,
      dmgMod: this.dmgMod,
      ev: this.ev,
      pre: this.getLevel('生命力'),
      mre: this.getLevel('知力'),
      equipments: this.equipments
    }
  }
}


```

そして、`type UnitModel` は下記のように型定義し、`class Unit` は `UnitModel` を引数として受け取って初期化させます。

```TypeScript
// src/domains/Combat/Unit/index.ts

// 戦闘ユニットモデル
type UnitModel = {
  name: string
  maxHp: number
  level: number
  dmgMod: number
  ev: number
  pre: number
  mre: number
  equipments: Equipments
}

// 戦闘ユニットを司るクラス
class Unit {
  public name: string
  ...

  constructor(model: UnitModel, combatId: CombatId) {
    const { name, ... } = model
    this.combatId = combatId
    this.name = name
    ...
  }
}

```

### ドメインの分割

戦闘では、様々なユニットの状態を管理します。  
`Unit` 自体は、ユニットの `ID`、`名前`、`区分 (PC側か敵側か)`、`戦闘配置` などを管理しますが、より大きな機能のまとまりは、分割したドメインに役割を委譲し、`Unit` にはそれらを統合管理する役割を与えます。  
分割したドメインには、以下の役割があります。

| ドメイン           | 役割       |
| -------------- | -------- |
| `Health`       | 負傷状態の管理  |
| `StatusBuff`   | バフ効果の管理  |
| `StatusDebuff` | デバフ効果の管理 |
| `Attack`       | 攻撃モデルの管理 |
| `Defense`      | 防御モデルの管理 |

これらの多くは、コンストラクタ引数として `Unit` 自身を受け取るようにします。  
例えば、`Health` は、ユニットが気絶したとき、戦闘配置を管理する `Unit` 自身のメンバ変数 `position` を参照します。  
また、`Attack` や `Defense` は、ユニットのバフやデバフの効果を参照し、パラメータに修正を加えます。

### ターン経過による状態の更新

ターン経過によって、バフ・デバフ効果が衰退していったり、フェイント・全力攻撃・全力防御の効果をリセットしたりします。  
それらは `Unit.nextTurn()` によって、まとめて解決します。

```TypeScript
class Unit {
  ...

  // 各種状態を更新
  nextTurn() {
    this.health.nextTurn()
    this.buff.nextTurn()
    this.debuff.nextTurn()
    this.attack.nextTurn()
    this.defense.nextTurn()
  }
}

```

### Summary表示用のラベル

戦闘画面上で、ユニットの状態 (朦朧状態、転倒、気絶、バフ効果、デバフ効果など) を表示するため、ラベルを取得する `getter` を設定します。

```TypeScript
class Unit {
  ...

  // Summary 表示用ラベル取得
  get label(): string {
    return this.health.label || this.debuff.label || this.buff.label
  }
}

```

## 戦闘ユニットの実装

実際には、まだ `Health`、`Attack`、`Defense` などのドメインは未実装なので、このままではコードは動きませんが、完成イメージの共有になります。

```TypeScript
const combatIds: number[] = [1, 2, 3, 4, 5, 6, 7, 8] as const

const SIDE_KEYS = ['player', 'enemy'] as const

const POSITION_KEYS = ['back', 'left', 'center', 'right'] as const

// 戦闘ユニットID
type CombatId = typeof combatIds[number]

// 戦闘ユニットの所属
type Side = typeof SIDE_KEYS[number]

// 戦闘ユニットの配置
type Position = typeof POSITION_KEYS[number]

// 戦闘ユニットモデル
type UnitModel = {
  name: string
  maxHp: number
  level: number
  dmgMod: number
  ev: number
  pre: number
  mre: number
  equipments: Equipments
}

// 戦闘ユニットの管理を司るクラス
class Unit {
  public combatId: CombatId
  public name: string
  public side: Side
  public position: Position
  public health: Health
  public buff: StatusBuff
  public debuff: StatusDebuff
  public attack: Attack
  public defense: Defense
  public pre: number
  public mre: number

  constructor(model: UnitModel, combatId: CombatId) {
    const { name, maxHp, pre, mre } = model
    this.combatId = combatId
    this.name = name
    this.side = combatId <= 4 ? 'player' : 'enemy'
    this.position = 'back'
    this.health = new Health(this, maxHp)
    this.buff = new StatusBuff()
    this.debuff = new StatusDebuff()
    this.attack = new Attack(this, model)
    this.defense = new Defense(this, model)
    this.pre = pre
    this.mre = mre
  }

  // 各種状態を更新
  nextTurn() {
    this.health.nextTurn()
    this.buff.nextTurn()
    this.debuff.nextTurn()
    this.attack.nextTurn()
    this.defense.nextTurn()
  }

  // Summary 表示用ラベル取得
  get label(): string {
    return this.health.label || this.debuff.label || this.buff.label
  }
}

```

---

次章では、`Formation` (戦闘配置) の実装を進めます。  
本章以降、UIに着手するのはだいぶ先になりますので、しばらく新しいデモ画面はありません。
