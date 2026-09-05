---
title: "パラメータ"
emoji: "⚔️"
type: "tech"
topics: ["gamedev", "typescript", "react"]
published: false
---

# パラメータ

ゲームはキャラクタ作成から開幕します。  
前章で既述の通り、キャラクタ作成は主に、ポイントを振り分けて、能力値・技能値と呼ばれるパラメータを設定していくところから始まります。  
まず `domains/` 内に `Character/` を配置し、 `Parameters.ts` を新規作成します。  
そしてこの `Parameters.ts` に何を定義していくのか、ドキュメントから仕様 (ルール) を確認しましょう。

## 1. パラメータの仕様

キャラクタ作成は、CP (キャラクタポイント) 10点を各能力値や技能値に振り分けて行います。

能力値には、**ST(筋力)**、**DX(敏捷力)**、**IN(知力)**、**HT(生命力)** の4つがあります。  
また、本来のテーブルトークRPGは、登攀だとか、鍵開けだとか、色々な場面で判定に使う「技能 (技能値) 」を設定するのですが、本プロジェクトは戦闘ロジック限定なので、技能総数をだいぶ減らしています。

能力値は 10 を基準値、技能値はいずれかの能力値を基準値とし、高く設定するためにはより多くのCPを消費します。消費CPと能力値・技能値の増加は、下記表に対応します。

##### 消費CPと能力値・技能値の対応表

| 能力値・技能値 | 消費CP |
| ------- | ---- |
| 基準値に等しい | 0CP  |
| 基準値+1   | 1CP  |
| 基準値+2   | 2CP  |
| 基準値+3   | 4CP  |
| 基準値+4   | 8CP  |

##### 技能表

| 技能   | 基準 | 説明                      |
| ---- | -- | ----------------------- |
| 武術   | ST | 重装備で近接戦闘を行うための技能。       |
| 怪力   | ST | 加ダメージ量の算出に影響する技能。       |
| 剣術   | DX | 軽装備で近接戦闘を行うための技能。       |
| 運動   | DX | 回避判定の目標値の算出に影響する技能。     |
| 青の魔法 | IN | 回復・防御・補助などを行う魔法を扱える技能。  |
| 赤の魔法 | IN | 炎や雷を操り攻撃・補助を行う魔法を扱える技能。  |
| 緑の魔法 | IN | 風や木を操り攻撃・防御を行う魔法を扱える技能。 |
| 鍛錬   | HT | Hp (耐久値) の算出に影響する技能。   |

前章のおさらいですが、これらは下記のようにコードで定義します。

```TypeScript
// src/domains/Character/Parameters.ts

// CP
const POINT_STEP = [0, 1, 2, 4, 8] as const

type Point = typeof POINT_STEP[number]

// パラメータのキー
const PARAMETER_KEYS = [
  '筋力', '敏捷力', '知力', '生命力', // 能力値
  '武術', '怪力', '剣術', '運動', '青の魔法', '赤の魔法', '緑の魔法', '鍛錬' // 技能
] as const

type ParameterKey = typeof PARAMETER_KEYS[number]

// パラメータの定義
const PARAMETERS: Record<ParameterKey, { base: ParameterKey | null }> = {
  '筋力': { base: null },
  '敏捷力': { base: null },
  '知力': { base: null },
  '生命力': { base: null },
  '武術': { base: '筋力' },
  '怪力': { base: '筋力' },
  '剣術': { base: '敏捷力' },
  '運動': { base: '敏捷力' },
  '青の魔法': { base: '知力' },
  '赤の魔法': { base: '知力' },
  '緑の魔法': { base: '知力' },
  '鍛錬': { base: '生命力' }
} as const

// パラメータ管理を司るクラス
class Parameters {
  ...
}

```

`as const` を用いて定数で各キーを宣言する。  
`typeof` を用いて各キーを型として定義する。  
という流れは、データモデリングの基本となります。

## 2. パラメータの設計

パラメータ管理を司るクラス `class Parameters` の役割は、下記に挙げられます。

- それぞれのパラメータのキーで、配分されたCPの値をセットしたり、取得したりできる
- それぞれのパラメータのキーで、能力値・技能値を取得できる
- 配分されたCPの合計を取得できる

これを実装するには、`class Parameters` にプライベートメンバ変数 `points` を持たせ、パラメータのキーと配分されたCPのペアの値を `Map` で管理するのが最適です。  
`Map` を用いる理由は、設定したパラメータの存在確認や、未設定のキーの削除が容易いからです。

まず `points` へ、各パラメータのキーとCPのペアの値をセット・取得するメソッド群を定義してみましょう。

また、いずれキャラクタ作成画面では、計12種の能力値・技能値を設定できるUIを作成するため、それらのCP配分の配列をコンストラクタの引数として受け取り、上記の機能を満たす `class Parameters` を初期化させたいと思います。  
これは、デバッグの際も、`[4, 0, 0, 2] // 筋力, 敏捷力, 知力, 生命力の順` のような配列をコンストラクタの引数として渡すだけで、サンプル・キャラクタを手軽に作成することができます。

さらに、キャラクタ作成画面では `1` ずつパラメータをインクリメント・デクリメントさせるため、`increase`、`decrease`、`step` の各メソッドも役に立つでしょう。

## 3. パラメータの実装

```TypeScript
// パラメータ管理を司るクラス
class Parameters {
  private points: Map<ParameterKey, Point>

  // CPの配列を Point の Map に変換
  constructor(points: Point[]) {
    this.points = new Map()
    points.forEach((p, i) => this.set(PARAMETER_KEYS[i], p))
  }

  // name と point を指定し, パラメータをセット
  // point: 0 を指定した場合は, パラメータを削除
  set(name: ParameterKey, point: Point) {
    if (point > 0) {
      this.points.set(name, point)
    } else {
      this.unset(name)
    }
  }

  // name を指定し, パラメータを削除
  private unset(name: ParameterKey) {
    this.points.delete(name)
  }

  // name を指定し, POINT_STEP に則りパラメータを減らす
  private decrease(name: ParameterKey, size: number = 1) {
    for (let i = 0; i < size; i++) {
      const point = this.get(name)
      if (!point) return // Map に無ければ無視
      const index = POINT_STEP.indexOf(point as Point)
      const result = index > 0 ? POINT_STEP[index - 1] : 0 // 最小値(0)であれば削除
      this.set(name, result)
    }
  }

  // name を指定し, POINT_STEP に則りパラメータを増やす
  private increase(name: ParameterKey, size: number = 1) {
    for (let i = 0; i < size; i++) {
      const point = this.get(name)
      const index = POINT_STEP.indexOf(point as Point)
      const result = index < POINT_STEP.length - 1 ? POINT_STEP[index + 1] : point // 最大値であれば無視
      this.set(name, result)
    }
  }

  // name と size を指定し, POINT_STEP に則りパラメータを増減
  // Map に無ければ追加し, 最小値(0)になれば削除する
  step(name: ParameterKey, size: number = 1) {
    if (size > 0) {
      this.increase(name, Math.abs(size))
    } else {
      this.decrease(name, Math.abs(size))
    }
  }

  // name を指定し, point (CP) を取得
  get(name: ParameterKey): Point {
    return this.points.get(name) ?? 0
  }

  // name を指定し, level (能力値・技能値) を算出
  getLevel(name: ParameterKey): number {
    const base = PARAMETERS[name].base
    const baseValue = base !== null ? this.getLevel(base) : 10
    const point = this.get(name)
    return baseValue + POINT_STEP.indexOf(point)
  }

  // point 総計を算出
  get total(): number {
    let total = 0
    for (const p of this.points.values()) total += p ?? 0
    return total
  }
}

```

ドキュメントに則って、パラメータの設定後、戦闘能力値である Dmg(ダメージ)、EV(よけ)、Hp(耐久値) を算出します。

- Dmg(ダメージ修正) = ST(筋力) または「怪力」 / 2 (端数切捨) - 5
- EV(よけ) = DX(敏捷力) または「運動」 / 2 (端数切捨) + 5
- Hp(耐久値) = HT(生命力) または「鍛錬」 × 2 - 10

`getLevel` メソッドを使用して、戦闘能力値を算出できるようにしましょう。

```TypeScript
class Parameters {

  ...

  // 最大Hpを取得
  get maxHp() {
    return this.getLevel('鍛錬') * 2 - 10
  }

  // ダメージ修正を取得
  get dmgMod() {
    return Math.floor(this.getLevel('怪力') / 2) - 5
  }

  //「よけ」を取得
  get ev() {
    return Math.floor(this.getLevel('運動') / 2) + 5
  }
}

```

加えて、キャラクタ・シートへのレンダリングのため、パラメータの名称・CP・値をセットで出力するメソッドがあると便利そうです。  
まずは `type Parameter ` を定義しましょう。

```TypeScript
type Parameter = {
  name: ParameterKey
  base: ParameterKey | null
  point: Point
  level: number
}

```

そしてメソッドをいくつか追加します。

```TypeScript
class Parameters {

  ...

  // name を指定し, その全ての属性を, オブジェクトに変換して取得
  getParam(name: ParameterKey): Parameter {
    return {
      name, point: this.get(name), level: this.getLevel(name), ...PARAMETERS[name]
    }
  }
  
  // 全てのパラメータとその全ての属性を, オブジェクトの配列に変換して取得 (ソート込み)
  get params(): Parameter[] {
    return [...this.points]
      .map(([name]) => this.getParam(name))
      .sort((a, b) => PARAMETER_KEYS.indexOf(a.name) - PARAMETER_KEYS.indexOf(b.name))
  }

  // 全ての技能とその全ての属性を, オブジェクトの配列に変換して取得 (ソート込み)
  get skills(): Parameter[] {
    return this.params
      .filter(p => p.base !== null)
  }
}

```

ここまでで、ひとまず「**パラメータ**」ドメインの設計・実装は完了です。  
最後に、動作確認として、`Character/` 内に `index.ts` を新規作成し、下記のコードを書き込んでください。  
`export` / `import` は適宜に行ってください。

これで、React側で編成画面UIを組み立てれば、実際に編成画面の初期構成を見ることができます。  
React側の実装は、本プロジェクトの本題とは逸れるため、割愛しますので、ソースコードをご参考ください。

```TypeScript
// src/domains/Character/index.ts

// キャラクタ・モデル
type CharacterModel = {
  id: number
  name: string
  abilities: Point[]
  skills: [ParameterKey, Point][]
}

// サンプル・モデル
const SAMPLE_MODELS: CharacterModel[] = [
  {
    id: 1,
    name: 'アーロン',
    abilities: [4, 0, 0, 2], // 筋力, 生命力高めの重戦士タイプ
    skills: [['武術', 2], ['怪力', 1], ['鍛錬', 1]]
  },
  {
    id: 2,
    name: 'ダニエル',
    abilities: [1, 4, 0, 2], // 敏捷力, 生命力高めの軽戦士タイプ
    skills: [['剣術', 2], ['運動', 1]]
  },
  {
    id: 3,
    name: 'アシュリン',
    abilities: [0, 1, 4, 1], // 知力高めの魔術師タイプ
    skills: [['赤の魔法', 2], ['緑の魔法', 2]]
  },
  {
    id: 4,
    name: 'ステファニー',
    abilities: [2, 0, 4, 1], // 筋力, 知力高めの魔法戦士タイプ
    skills: [['武術', 1], ['青の魔法', 2]]
  }
]

// キャラクタ管理を司るクラス
class Character {
  public id: number
  public name: string
  public parameters: Parameters

  constructor(model: CharacterModel) {
    this.id = model.id
    this.name = model.name
    this.parameters = new Parameters(model.abilities)
    model.skills.forEach(([name, point]) => this.parameters.set(name, point))
  }
}

export const SAMPLE_CHARACTERS = SAMPLE_MODELS.map((model) => new Character(model))

```

---

次章では、「**装備**」ドメインの設計・実装に入っていきたいと思います。  
ここまでの デモ \[01\] と ソースコード (v0.1.0) は、下記URLをご参考ください。

- [RPG.LOGIC DEMO - 01](https://rpg-logic.dev/demo/01/)
- [GitHub - RPG.LOGIC DEMO v0.1.0](https://github.com/qweldesign/rpg-logic-demo/releases/tag/v0.1.0)
