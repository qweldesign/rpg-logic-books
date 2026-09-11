---
title: "装備"
emoji: "⚔️"
type: "tech"
topics: ["gamedev", "typescript", "react"]
published: false
---

# 装備

装備は、武器・盾・鎧などキャラクタの装備を整えて、戦闘時に扱うダメージや命中・回避判定の目標値といった数値を備えるドメインです。  
`Character/` 内に `Equipments.ts` を新規作成します。

## 1. 装備の仕様

### 1-1. 武器

武器には、性能値、攻撃型、両手武器か否か、準備が必要な武器か否か、必要筋力という5つの属性があります。

#### 性能値 (基本ダメージ)

武器の致傷力を示します。ルール上では「基本ダメージ」という表記で `1d+2` とか `2d-1` とかいう表記でダメージ (打撃の強さ) を表します。  
戦闘では、武器の性能値に本人の筋力の影響を加算した値を、ダメージステップに則って、ダメージに置き換えます。

##### ダメージステップ

| 性能値 | 基本ダメージ |
| --- | ------ |
| 4   | 1d     |
| 5   | 1d+1   |
| 6   | 1d+2   |
| 7   | 2d-1   |
| 8   | 2d     |
| 9   | 2d+1   |
| 10  | 2d+2   |
| 11  | 3d-1   |
| 12  | 3d     |
| 13  | 3d+1   |
| 14  | 3d+2   |
| 15  | 4d-1   |
| 16  | 4d     |

#### 攻撃型

RPG.LOGIC の武器には、「叩」「切」「刺」の3タイプの「攻撃型」と呼ばれる属性があります。  
「叩」は相手の防具による影響を受けにくく、「刺」は影響を受けやすい特性を定義します。  
これは、堅い鎧に対しレイピアのような刺突攻撃は貫通しにくい物理法則を再現しています。

#### 両手武器

RPG.LOGIC の防御手段は、武器による「受け」、盾による「止め」、身体を左右に振って回避する「よけ」の3通りの方法がありますが、両手武器では「止め」を行えません。  
両手武器の方が致傷力が大きい上、「受け」も有利に行えます。ただし、魔法による攻撃の多くは射撃であるため、「受け」では防ぐことができず、「止め」か「避け」でのみ防ぐことができます。  
ゲーム・デザインとしてここで、【片手武器の戦士】 < 【両手武器の戦士】 < 【魔法使い】 という、ジャンケンのような戦術的な相性関係が生じます。

#### 準備

戦斧や鉾槍のような重量のある武器は、攻撃や「受け」での防御の都度、手元に武器を引き戻す「準備」という行為が必要になります。この間、武器は攻撃にも「受け」にも使えません。  
戦斧は盾を別途使えますが、鉾槍は両手武器のため、一度攻撃を繰り出すと、その後の防御に懸念が残ります。その分、性能値は他の武器よりも高めに設定されています。

#### 必要筋力

武器・盾・鎧に共通して、その装備を扱うために必要な筋力を示します。  
ST(筋力)がその値に満たない場合は、その装備を扱えません。

##### 武器表

| 武器  | 性能値 | 攻撃型 | 用法  | 必要筋力 | 備考                   |
| --- | --- | --- | --- | ---- | -------------------- |
| 細剣  | 4   | 刺   | 片手  | 10   | 標準的な「刺」型武器           |
| 長剣  | 5   | 切   | 片手  | 10   | 標準的な「切」型武器           |
| 戦棍  | 6   | 叩   | 片手  | 10   | 標準的な「叩」型武器           |
| 戦斧  | 8   | 切   | 片手  | 12   | 致傷力は大きいが、攻撃毎に「準備」が必要 |
| 長槍  | 5   | 刺   | 両手  | 11   | 両手使用の「刺」型武器          |
| 大剣  | 6   | 切   | 両手  | 11   | 両手使用の「切」型武器          |
| 長杖  | 8   | 叩   | 両手  | 11   | 両手使用の「叩」型武器          |
| 鉾槍  | 12  | 切   | 両手  | 13   | 両手使用、かつ攻撃毎に「準備」が必要   |

武器は、下記のようにコードで定義します。

```TypeScript
// src/domains/Character/Equipments.ts

// 武器のキー
const WEAPON_KEYS = [
  '細剣', '長剣', '戦棍', '戦斧', '長槍', '大剣', '長杖', '鉾槍'
] as const

type WeaponKey = typeof WEAPON_KEYS[number]

// 武器の定義
type Weapon = {
  dmgBase: number // 性能値
  dmgType: 0 | 1 | 2 // 攻撃型 (0: 叩, 1: 切, 2: 刺) 
  twoHanded: boolean // 両手が必要か
  ready: boolean // 準備が必要か
  requiredST: number // 必要筋力
}

const WEAPONS: Record<WeaponKey, Weapon> = {
  '細剣': { dmgBase: 4, dmgType: 2, twoHanded: false, ready: false, requiredST: 10 },
  '長剣': { dmgBase: 5, dmgType: 1, twoHanded: false, ready: false, requiredST: 10 },
  '戦棍': { dmgBase: 6, dmgType: 0, twoHanded: false, ready: false, requiredST: 10 },
  '戦斧': { dmgBase: 8, dmgType: 1, twoHanded: false, ready: true, requiredST: 12 },
  '長槍': { dmgBase: 5, dmgType: 2, twoHanded: true, ready: false, requiredST: 11 },
  '大剣': { dmgBase: 6, dmgType: 1, twoHanded: true, ready: false, requiredST: 11 },
  '長杖': { dmgBase: 8, dmgType: 0, twoHanded: true, ready: false, requiredST: 11 },
  '鉾槍': { dmgBase: 12, dmgType: 1, twoHanded: true, ready: true, requiredST: 13 }
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
  { name: '1d', dmgDice: 1, dmgMod: 0 }, // 4
  { name: '1d+1', dmgDice: 1, dmgMod: 1 }, // 5
  { name: '1d+2', dmgDice: 1, dmgMod: 2 }, // 6
  { name: '2d-1', dmgDice: 2, dmgMod: -1 }, // 7
  { name: '2d', dmgDice: 2, dmgMod: 0 }, // 8
  { name: '2d+1', dmgDice: 2, dmgMod: 1 }, // 9
  { name: '2d+2', dmgDice: 2, dmgMod: 2 }, // 10
  { name: '3d-1', dmgDice: 3, dmgMod: -1 }, // 11
  { name: '3d', dmgDice: 3, dmgMod: 0 }, // 12
  { name: '3d+1', dmgDice: 3, dmgMod: 1 }, // 13
  { name: '3d+2', dmgDice: 3, dmgMod: 2 }, // 14
  { name: '4d-1', dmgDice: 4, dmgMod: -1 }, // 15
  { name: '4d', dmgDice: 4, dmgMod: 0 } // 16
] as const

```

### 1-2. 盾

盾はルール上、大きさの違いで回避判定への防御補正が異なります。  
また大型盾は、常に自身の攻撃時の命中判定でペナルティが課されます。  
これらは、`isLarge` 属性 (大型盾か) だけを定義し、防御補正やペナルティの数値は戦闘ロジックで定義する設計にします。

盾は、下記のようにコードで定義します。

```TypeScript
// 盾のキー
const SHIELD_KEYS = [
  '小盾', '大盾'
] as const

type ShieldKey = typeof SHIELD_KEYS[number]

// 盾の定義
type Shield = {
  isLarge: boolean // 大型盾か
  requiredST: number // 必要筋力
}

const SHIELDS: Record<ShieldKey, Shield> = {
  '小盾': { isLarge: false, requiredST: 10 },
  '大盾': { isLarge: true, requiredST: 12 }
} as const

```

### 1-3. 服・鎧

服・鎧は、ルール上は「ダメージ抵抗」という表記で、防護力と重量をセットで表します。  
他に、`isChain` 属性 (環状構造か) を定義し、「刺」の攻撃型に対してDRを半分 (端数切捨) として扱うかどうかを設定します。

服・鎧は、下記のようにコードで定義します。

```TypeScript
// 服・鎧のキー
const ARMOR_KEYS = [
  '革服', '革鎧', 'チェインメイル', 'プレイトメイル'
] as const

type ArmorKey = typeof ARMOR_KEYS[number]

// 服・鎧の定義
type Armor = {
  dr: number // ダメージ抵抗
  isChain: boolean // 環状構造か
  requiredST: number // 必要筋力
}

const ARMORS: Record<ArmorKey, Armor> = {
  '革服': { dr: 1, isChain: false, requiredST: 10 },
  '革鎧': { dr: 2, isChain: false, requiredST: 11 },
  'チェインメイル': { dr: 3, isChain: true, requiredST: 12 },
  'プレイトメイル': { dr: 4, isChain: false, requiredST: 13 }
} as const

```

## 2. 装備の設計

装備管理を司るクラス `class Equipments` の役割は、下記に挙げられます。

- コンストラクタの引数に、初期の武器・盾・鎧のキーを取り、`class Equipments` を初期化する
- それぞれの武器・盾・鎧のキーで、装備をセットしたり、取得したりできる
- 武器は、性能値に本人の筋力の影響を加算して、ダメージオブジェクト (ダメージダイスの数、修正値、攻撃型など) を返すことができる

武器・盾・鎧の `setter` / `getter` を設定していきますが、盾だけ `null` を許容する (装備しない場合がある) ので注意が必要です。

また、キャラクタ・シートへのレンダリングのため、ダメージやダメージ抵抗の詳細な表記を返すメソッドもあると良いでしょう。

## 3. 装備の実装

```TypeScript
// 装備管理を司るクラス
class Equipments {
  private _weapon: WeaponKey
  private _shield: ShieldKey | null
  private _armor: ArmorKey

  constructor(weapon: WeaponKey = '細剣', shield: ShieldKey | null = '小盾', armor: ArmorKey = '革服') {
    this._weapon = weapon
    this._shield = shield
    this._armor = armor
  }

  // 武器をセット
  set weapon(weapon: WeaponKey) {
    this._weapon = weapon
  }

  // 盾をセット
  set shield(shield: ShieldKey | null) {
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
  get shield(): Shield & { name: ShieldKey } | null {
    return this._shield ? { name: this._shield, ...SHIELDS[this._shield] } : null
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
  equipments: [WeaponKey, ShieldKey | null, ArmorKey] // ← 追加
}

// サンプル・モデル
const SAMPLE_MODELS: CharacterModel[] = [
  {
    id: 1,
    name: 'アーロン',
    abilities: [4, 0, 0, 2], // 筋力, 生命力高めの重戦士タイプ
    skills: [['武術', 2], ['怪力', 1], ['鍛錬', 1]],
    equipments: ['戦斧', '大盾', 'プレイトメイル'] // ← 追加
  },
  {
    id: 2,
    name: 'ダニエル',
    abilities: [1, 4, 0, 2], // 敏捷力, 生命力高めの軽戦士タイプ
    skills: [['剣術', 2], ['運動', 1]],
    equipments: ['大剣', null, '革鎧'] // ← 追加
  },
  {
    id: 3,
    name: 'アシュリン',
    abilities: [0, 1, 4, 1], // 知力高めの魔術師タイプ
    skills: [['赤の魔法', 2], ['緑の魔法', 2]],
    equipments: ['戦棍', '小盾', '革服'] // ← 追加
  },
  {
    id: 4,
    name: 'ステファニー',
    abilities: [2, 0, 4, 1], // 筋力, 知力高めの魔法戦士タイプ
    skills: [['武術', 1], ['青の魔法', 2]],
    equipments: ['細剣', '小盾', 'チェインメイル'] // ← 追加
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
    this.equipments = new Equipments(...model.equipments) // ← 追加
  }
}

export const SAMPLE_CHARACTERS = SAMPLE_MODELS.map((model) => new Character(model))

```

---

次章では、パラメータと装備を統合する「**キャラクタ**」ドメインの設計・実装に入っていきたいと思います。  
ここまでの デモ \[02\] と ソースコード (v0.2.0) は、下記URLをご参考ください。

- [RPG.LOGIC DEMO - 02](https://rpg-logic.dev/demo/02/)
- [GitHub - RPG.LOGIC DEMO v0.2.0](https://github.com/qweldesign/rpg-logic-demo/releases/tag/v0.2.0)
