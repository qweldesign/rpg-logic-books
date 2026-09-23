---
title: "Equipments: 装備管理"
emoji: "⚔️"
type: "tech"
topics: ["gamedev", "typescript", "react"]
published: false
---

# Equipments: 装備管理

装備は、武器・盾・鎧などキャラクタの装備を整えて、戦闘時に扱うダメージや命中・回避判定の目標値といった数値を備えるドメインです。  
`Character/` 内に `Equipments.ts` を新規作成します。

## 装備の要件

### 1. 武器

武器には、性能値、攻撃型、両手武器か否か、準備が必要な武器か否か、必要筋力という5つの属性があります。

#### 性能値 (基本ダメージ)

ルール上では「基本ダメージ」という表記で `1d+2` とか `2d-1` とかいう表記でダメージ (打撃力) を表します。  
戦闘では、武器の性能値に本人の筋力の影響を加算した値を、ダメージステップに則って、ダメージに置き換えるため、システム上では「基本ダメージ」を直接扱わず、「性能値」として数値型の値を扱います。

##### ダメージステップ

| 性能値 | 基本ダメージ |
| --- | ------ |
| 2   | 1d     |
| 3   | 1d+1   |
| 4   | 1d+2   |
| 5   | 2d-1   |
| 6   | 2d     |
| 7   | 2d+1   |
| 8   | 2d+2   |
| 9   | 3d-1   |
| 10  | 3d     |
| 11  | 3d+1   |
| 12  | 3d+2   |

#### 攻撃型

RPG.LOGIC の武器には、「叩」「切」「刺」の3タイプの「攻撃型」と呼ばれる属性があります。  
「叩」は相手の防具による影響を受けにくく、「刺」は影響を受けやすい特性を定義します。  
これは堅い鎧に対して、レイピアのような刺突攻撃は貫通しにくい、という物理法則を再現しています。

#### 両手武器

RPG.LOGIC の防御手段は、武器による「受け」、盾による「止め」、身体を左右に振って回避する「よけ」の3通りの方法がありますが、両手武器では「止め」を行えません。  
両手武器の方が性能値が大きい上、「受け」も有利に行えます。ただし、魔法による攻撃の多くは射撃であるため、「受け」では防ぐことができず、「止め」か「よけ」でのみ防ぐことができます。  
ゲーム・デザインとしてここで、【片手武器の戦士】 < 【両手武器の戦士】 < 【魔術師】 という、ジャンケンのような戦術的な相性関係が生じます。

#### 準備

棍棒や戦斧のような重量のある武器は、攻撃や「受け」での防御の都度、手元に武器を引き戻す「準備」という行為が必要になります。この間、武器は攻撃にも「受け」にも使えません。  
棍棒や戦斧は盾を別途使えますが、鉾槍は両手武器のため、一度攻撃を繰り出すと、その後の防御に懸念が残ります。その分、性能値は他の武器よりも高めに設定されています。

#### 必要筋力

武器・盾・鎧に共通して、その装備を扱うために必要な筋力を示します。  
ST(筋力)がその値に満たない場合は、その装備を扱えません。

##### 武器表

| 武器  | 性能値 | 攻撃型 | 用法  | 準備  | 必要ST | 金額  |
| --- | --- | --- | --- | --- | ---- | --- |
| 短剣  | 2   | 切   | 片手  | 不要  | -    | 10  |
| 小剣  | 2   | 刺   | 片手  | 不要  | -    | 20  |
| 長剣  | 3   | 切   | 片手  | 不要  | 11   | 40  |
| 大剣  | 4   | 切   | 両手  | 不要  | 13   | 80  |
| 棍棒  | 5   | 叩   | 片手  | 必要  | -    | 10  |
| 戦棍  | 6   | 叩   | 片手  | 必要  | 11   | 20  |
| 戦斧  | 5   | 切   | 片手  | 必要  | 13   | 40  |
| 長杖  | 4   | 叩   | 両手  | 不要  | -    | 10  |
| 長槍  | 3   | 刺   | 両手  | 不要  | 11   | 20  |
| 鉾槍  | 7   | 切   | 両手  | 必要  | 13   | 40  |

武器は、下記のようにコードで定義します。

```TypeScript
// src/domains/Character/Equipments.ts

// 武器のキー
const WEAPON_KEYS = [
  '装備無し', '短剣', '小剣', '長剣', ...
] as const

type WeaponKey = typeof WEAPON_KEYS[number]

// 武器の定義
type Weapon = {
  dmgBase: number // 性能値
  dmgType: 0 | 1 | 2 // 攻撃型 (0: 叩, 1: 切, 2: 刺)
  needsTwoHanded: boolean // 両手が必要か
  needsReady: boolean // 準備が必要か
  requiredST?: number // 必要筋力
  gold: number // 金額
}

const WEAPONS: Record<WeaponKey, Weapon> = {
  '装備無し': { dmgBase: 2, dmgType: 0, needsTwoHanded: false, needsReady: false, gold: 0 },
  '短剣': { dmgBase: 2, dmgType: 1, needsTwoHanded: false, needsReady: false, gold: 10 },
  '小剣': { dmgBase: 2, dmgType: 2, needsTwoHanded: false, needsReady: false, gold: 20 },
  '長剣': { dmgBase: 3, dmgType: 1, needsTwoHanded: false, needsReady: false, requiredST: 11, gold: 40 },
  ...
} as const

// ダメージの定義
type Dmg = {
  name: string // ダメージ表記
  dmgDice: number // ダメージダイスの数
  dmgMod: number // ダメージ修正値
  dmgType: 0 | 1 | 2 // 攻撃型 (0: 叩, 1: 切, 2: 刺) 
}

// ダメージステップ
const DMG_STEP: Omit<Dmg, 'dmgType'>[] = [
  { name: '1d-2', dmgDice: 1, dmgMod: -2 }, // 0
  { name: '1d-1', dmgDice: 1, dmgMod: -1 }, // 1
  { name: '1d', dmgDice: 1, dmgMod: 0 }, // 2
  { name: '1d+1', dmgDice: 1, dmgMod: 1 }, // 3
  ...
] as const

```

### 2. 盾

盾はルール上、大きさの違いで回避判定への防御補正が異なります。  
また大型盾は、常に自身の攻撃時の命中判定でペナルティが課されます。  
これらは、`size` 属性 (大きさ) だけを定義し、防御補正やペナルティの数値は戦闘ロジックで定義する設計にします。

盾は、下記のようにコードで定義します。

```TypeScript
// 盾のキー
const SHIELD_KEYS = [
  '小盾', '大盾'
] as const

type ShieldKey = typeof SHIELD_KEYS[number]

// 盾の定義
type Shield = {
  size: number // 大きさ
  requiredST?: number // 必要筋力
  gold: number // 金額
}

const SHIELDS: Record<ShieldKey, Shield> = {
  '装備無し': { size: 0, gold: 0 },
  '小盾': { size: 1, gold: 10 },
  '大盾': { size: 2, requiredST: 13, gold: 40 }
} as const

```

### 3. 服・鎧

服・鎧は、ルール上は「ダメージ抵抗」という表記で、防護力と重量をセットで表します。  
他に、`isChain` 属性 (繊維構造または環状構造か) を定義し、「刺」の攻撃型に対してDRを半分 (端数切捨) として扱うかどうかを設定します。

服・鎧は、下記のようにコードで定義します。

```TypeScript
// 服・鎧のキー
const ARMOR_KEYS = [
  '服', '革服', '革鎧', 'チェインメイル', 'プレイトメイル'
] as const

type ArmorKey = typeof ARMOR_KEYS[number]

// 服・鎧の定義
type Armor = {
  dr: number // ダメージ抵抗
  isChain: boolean // 環状構造か
  requiredST?: number // 必要筋力
  gold: number // 金額
}

const ARMORS: Record<ArmorKey, Armor> = {
  '服': { dr: 1, isChain: true, gold: 0 },
  '革服': { dr: 1, isChain: false, gold: 20 },
  '革鎧': { dr: 2, isChain: false, requiredST: 11, gold: 40 },
  'チェインメイル': { dr: 3, isChain: true, requiredST: 12, gold: 80 },
  'プレイトメイル': { dr: 4, isChain: false, requiredST: 13, gold: 160 }
} as const

```

## 装備の設計

装備管理を司るクラス `class Equipments` の役割は、下記に挙げられます。

- コンストラクタの引数に、初期の武器・盾・鎧のキーを取り、`class Equipments` を初期化する
- それぞれの武器・盾・鎧のキーで、装備をセットしたり、取得したりできる
- 武器は、性能値に本人の筋力の影響を加算して、ダメージオブジェクト (ダメージダイスの数、修正値、攻撃型など) を返すことができる

武器・盾・鎧の `setter` / `getter` を設定していきます。
このとき、武器と盾は「何も装着していない状態」が必要になるので、その際に `null` を設定するとTypeScriptの型エラー問題が発生するため、上記コードの通り、「装備無し」という項目を設けています。

また、キャラクタ・シートへのレンダリングのため、ダメージやダメージ抵抗の詳細な表記を返すメソッドもあると良いでしょう。

## 装備の実装

```TypeScript
// 装備管理を司るクラス
class Equipments {
  private _weapon: WeaponKey
  private _shield: ShieldKey
  private _armor: ArmorKey

  constructor(weapon: WeaponKey = '装備無し', shield: ShieldKey = '装備無し', armor: ArmorKey = '服') {
    this._weapon = weapon
    this._shield = shield
    this._armor = armor
  }

  // 武器をセット
  set weapon(weapon: WeaponKey) {
    this._weapon = weapon
  }

  // 盾をセット
  set shield(shield: ShieldKey) {
    this._shield = shield
  }

  // 服・鎧をセット
  set armor(armor: ArmorKey) {
    this._armor = armor
  }

  // 武器を取得
  get weapon(): Weapon & { name: WeaponKey } {
    return { name: this._weapon, ...WEAPONS[this._weapon] }
  }

  // 盾を取得
  get shield(): Shield & { name: ShieldKey } {
    return { name: this._shield, ...SHIELDS[this._shield] }
  }

  // 服・鎧を取得
  get armor(): Armor & { name: ArmorKey } {
    return { name: this._armor, ...ARMORS[this._armor] }
  }

  // ダメージ修正を引数として受け取り, 武器のダメージオブジェクトを取得
  getDmg(mod: number): Dmg {
    const { dmgBase, dmgType } = this.weapon
    const totalDmg = dmgBase + mod
    const step = DMG_STEP[Math.max(0, Math.min(totalDmg - 4, DMG_STEP.length - 1))]
    return { ...step, dmgType }
  }

  // ダメージ修正を引数として受け取り, 武器のダメージ表記を取得
  getDmgName(mod: number): string {
    const { dmgDice, dmgMod } = this.getDmg(mod)
    const dmgTypeStr = ['叩', '切', '刺'][this.weapon.dmgType]
    return `${dmgDice}d${dmgMod === 0 ? '' : dmgMod > 0 ? '+' + dmgMod : dmgMod} (${dmgTypeStr})`
  }

  // 服・鎧のDR表記を取得
  getDRName(): string {
    const { dr, isChain } = this.armor
    return `${dr}${isChain ? ' (' + Math.floor(dr / 2) + ')' : ''}`
  }
}

```

ここまでで、「**装備**」ドメインの設計・実装は完了です。  
最後に、動作確認として、`Character/` 内 `index.ts` に、下記のコードを書き込んでください。  
React側で編成画面UIも更新すれば、編成画面で装備の表示まで見ることができます。

```TypeScript
// src/domains/Character/index.ts

// キャラクタ・モデル
type CharacterModel = {
  id: number
  name: string
  abilities: Point[]
  skills: [ParameterKey, Point][]
  equipments: [WeaponKey, ShieldKey, ArmorKey] | [] // ← 追加
}

// サンプル・モデル
const SAMPLE_MODELS: CharacterModel[] = [
  {
    id: 1,
    name: 'アーロン',
    abilities: [4, 0, 0, 2], // 筋力, 生命力高めの重戦士タイプ
    skills: [['武術', 2], ['怪力', 1], ['鍛錬', 1]],
    equipments: ['長剣', '小盾', '革鎧'] // ← 追加
  },
  {
    id: 2,
    name: 'ダニエル',
    abilities: [1, 4, 0, 2], // 敏捷力, 生命力高めの軽戦士タイプ
    skills: [['剣術', 2], ['運動', 1]],
    equipments: ['戦棍', '小盾', '革鎧'] // ← 追加
  },
  {
    id: 3,
    name: 'アシュリン',
    abilities: [0, 1, 4, 1], // 知力高めの魔術師タイプ
    skills: [['青の魔法', 2], ['赤の魔法', 2]],
    equipments: ['長杖', '装備無し', '革服'] // ← 追加
  },
  {
    id: 4,
    name: 'ステファニー',
    abilities: [2, 1, 4, 0], // 筋力, 知力高めの魔戦士タイプ
    skills: [['武術', 1], ['青の魔法', 2]],
    equipments: ['長剣', '小盾', '革鎧'] // ← 追加
  }
]

// キャラクタ管理を司るクラス
class Character {
  public id: number
  public name: string
  public parameters: Parameters
  public equipments: Equipments // ← 追加

  constructor(model: CharacterModel) {
    this.id = model.id
    this.name = model.name
    this.parameters = new Parameters(model.abilities)
    model.skills.forEach(([name, point]) => this.parameters.set(name, point))
    this.equipments = model.equipments.length ? new Equipments(...model.equipments) : new Equipments() // ← 追加
  }
}

const SAMPLE_CHARACTERS = SAMPLE_MODELS.map((model) => new Character(model))

```

---

次章では、パラメータと装備を統合する「**キャラクタ**」ドメインの設計・実装に入っていきたいと思います。  
ここまでの デモ \[DEMO 00-02\] は、下記URLをご参考ください。

- [DEMO 00-02 | 装備の実装 | RPG.LOGIC](https://rpg-logic.dev/demo/00/02/)
