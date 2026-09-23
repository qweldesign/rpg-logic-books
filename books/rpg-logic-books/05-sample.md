---
title: "Sample: サンプル・キャラクタ生成"
emoji: "⚔️"
type: "tech"
topics: ["gamedev", "typescript", "react"]
published: false
---

# Sample: サンプル・キャラクタ生成

キャラクタの実装の仕上げとして、サンプル・キャラクタ生成を実装します。  
一定のシード値を元に、ある程度バランスの取れたパラメータ配分と装備選択を自動で行うアルゴリズムを作成します。  
ランダムなキャラクタの生成を実装した後、ランダムなパーティ編成の生成も実装します。  
`domains/` 内に `Sample/` を配置し、 `index.ts` を新規作成し、ここにサンプル・キャラクタ生成コードを記述していきましょう。

## サンプル・キャラクタの設計

### シード値からの生成

サンプル・キャラクタ生成は、毎回完全にランダムなキャラクタ・パーティ編成を生成しても良いのですが、その仕様だとデバッグが困難になるため、一定のシード値を元に再現可能な一律の内容を生成する仕様とします。

シード値は `0～63` の範囲に収め、1つの整数から、p: 大分類4タイプ × c: 小分類8タイプ × g: 性別相当2タイプ にそれぞれ剰余・除算の組み合わせで分解していきます。

```TypeScript
  // シード値を分化して, 乱数を生成する
  const s = seed % DEFAULT_SIZE
  const p = (s + Math.floor(s / 16)) % 4 // 4タイプによる大分類
  const c = Math.floor(s / 4) % 8 // 8タイプによる小分類
  const g = Math.floor(s / 32) % 2 // 性別相当

```

#### 4タイプの大分類

RPG.LOGIC のゲーム・デザインとしては、ST(筋力)、DX(敏捷力)、IN(知力) のいずれか (つまり、HT(生命力)を除く能力値) にCPを `4` 点注ぎ、能力値を `13` に設定したキャラクタは、何らかの形で活躍できます。  
また、ST(筋力) と IN(知力) の両方が高いと、前衛で武器を振って戦うことも、後衛で魔法を扱って援護することもできる万能キャラクタとなり、パーティに1人入れると面白いでしょう。

よって、サンプル・キャラクタ生成アルゴリズムとしては、最初に4つのキャラクタ・タイプに分類します。  
ST(筋力)重視の重戦士タイプ、DX(敏捷力)重視の軽戦士タイプ、IN(知力)重視の魔術師タイプ、ST(筋力)とIN(知力)のバランスの取れた魔戦士タイプの4つとし、パーティ編成もこの4タイプが1名ずつ加入するようにアルゴリズムを設計すれば良いでしょう。

#### 8タイプの小分類

最大能力値が決まったら、残りの能力値の配分パターンは無数にある訳でもありませんので、8パターンぐらいでも十分なバリエーションが得られます。

#### 性別相当 (2タイプ)

重戦士、魔術師、魔戦士の各タイプでST(筋力)とIN(知力)のバリエーションは網羅的に確保しているので、性別相当の分類で、DX(敏捷力)、HT(生命力)を差別化します。

#### パーティ編成

パーティ編成は、重戦士、軽戦士、魔術師、魔戦士の各タイプが1名ずつ選ばれるようにします。

## サンプル・キャラクタの実装

```TypeScript
// src/domains/Sample/index.ts

/**
 * サンプル・キャラクタ生成アルゴリズム
 * 
 * 1. シード値 0～63 を生成
 * 
 * 2. 重戦士・軽戦士・魔術師・魔戦士に分岐
 *   最大能力値を, ST (筋力), DX (敏捷力), IN (知力) のうちで決定し, 3-1. へ進むか,
 *   最大能力値を決定せずに 3-2. へ進む
 * 
 * 3-1. 残りの能力値を以下のパターンで割り振る
 *   a. [0, 0, 1] (5)
 *   b. [0, 1, 0] (5)
 *   c. [1, 0, 0] (5)
 *   d. [0, 1, 1] (6)
 *   e. [1, 0, 1] (6)
 *   f. [1, 1, 0] (6)
 *   g. [0, 2, 1] (7)
 *   h. [2, 0, 1] (7)
 * 
 * 3-2. 魔戦士タイプの割り振りパターン (固定)
 *   a. [2, 0, 4, 0] (6)
 *   b. [2, 0, 4, 1] (7)
 *   c. [4, 0, 2, 0] (6)
 *   d. [4, 0, 2, 1] (7)
 *   e. [2, 0, 4, 0] (6)
 *   f. [2, 1, 4, 0] (7)
 *   g. [4, 0, 2, 0] (6)
 *   h. [4, 1, 2, 0] (7)
 * 
 * 4-1. HT (生命力) を +1 する/しない (男性イメージ)
 * 
 * 4-2. 4-1. をしなかった場合, 可能なら DX (敏捷力) を +1 する (女性イメージ)
 * 
 */

const DEFAULT_SIZE = 64 // シード値の範囲

const TABLE_1: Point[][] = [
  [0, 0, 1],
  [0, 1, 0],
  [1, 0, 0],
  [0, 1, 1],
  [1, 0, 1],
  [1, 1, 0],
  [0, 2, 1],
  [2, 0, 1]
]

const TABLE_2: Point[][] = [
  [2, 0, 4, 0],
  [2, 0, 4, 1],
  [4, 0, 2, 0],
  [4, 0, 2, 1],
  [2, 0, 4, 0],
  [2, 1, 4, 0],
  [4, 0, 2, 0],
  [4, 1, 2, 0]
]

function makeAbilityValues(p: number, c: number): Point[] {
  let result: Point[] = [0, 0, 0, 0]

  // 最大能力値を, ST (筋力), DX (敏捷力), IN (知力) のうちで決定する
  if (p < 3) {
    result[p] = 4
    
    // 残りの能力値を割り振る
    const table = TABLE_1[c]
    if (p === 0) {
      result[1] = table[0]
      result[2] = table[1]
      result[3] = table[2]
    } else if (p === 1) {
      result[0] = table[0]
      result[2] = table[1]
      result[3] = table[2]
    } else {
      result[0] = table[0]
      result[1] = table[1]
      result[3] = table[2]
    }
  } else {
    // 魔戦士タイプの割り振りパターン (固定)
    result = TABLE_2[c].slice()
  }

  return result
}

// 名前 (NPC用)
const NPC_LIST: string[] = [
  'アーロン', 'アイゼア', 'アンドリュー', 'イアン', 'エリック', 'オーウェン',
  'ギャレット', 'クーパー', 'ケヴィン', 'コール', 'サム',
  'ジェイデン', 'ジェレミア', 'ショーン', 'ジョセフ', 'スティーブン',
  'ダニエル', 'チェイス', 'ディビッド', 'ティモシー', 'トーマス', 'ドミニク',
  'ニコラス', 'ネイサン', 'パーカー', 'パトリック', 'ブライアン',
  'マシュー', 'メイソン', 'ライアン', 'リチャード', 'ルイス',
  'アシュリン', 'アビー', 'アリアナ', 'アリシア', 'イザベラ', 'エマ',
  'オードリー', 'オリビア', 'キャロライン', 'クレア', 'グレース',
  'ケイト', 'ジェシカ', 'シエラ', 'シドニー', 'シャーロット',
  'ステファニー', 'ゾーィ', 'ディスティニー', 'トリニティ', 'ナタリー', '二コール',
  'ブルック', 'ペイジ', 'マヤ', 'マリア', 'ミア',
  'ミッシェル', 'メリッサ', 'リア', 'リリー', 'レイチェル'
]

// サンプル・キャラクタ生成クラス
class Sample extends Character {
  constructor(id: number, seed: number, total: number) {
    // シード値を分化して, 乱数を生成する
    const s = seed % DEFAULT_SIZE
    const p = (s + Math.floor(s / 16)) % 4 // 4タイプによる大分類
    const c = Math.floor(s / 4) % 8 // 8タイプによる小分類
    const g = Math.floor(s / 32) % 2 // 性別相当

    // ID, 名前, 能力値を決定し, 初期化
    const model: CharacterModel = {
      id,
      name: NPC_LIST[s],
      points: makeAbilityValues(p, c),
      equipments: []
    }
    super(model)

    // 能力値を修正
    this.modifyAbilities(g, total)

    // 技能をセット
    this.setSkills(p, s, total)

    // 装備をセット
    this.setEquips(s, total)
  }

  // 可能ならパラメータを step する
  step(name: ParameterKey, total: number) {
    super.step(name)
    if (this.total > total) super.step(name, -1) // 総計を超えたら戻す
  }

  // 能力値を修正
  // HT (生命力) または DX (敏捷力) を +1 する/しない
  modifyAbilities(g: number, total: number) {
    if (g === 0) {
      // HT (生命力) を +1 する
      this.step('生命力', total)
    } else {
      // 可能なら DX (敏捷力) を +1 する
      this.step('敏捷力', total)
    }
  }

  /**
   * 技能修得アルゴリズム
   * 
   * 1. 主技能へ2CPずつ配分
   *   a. 重戦士: 武術
   *   b. 軽戦士: 剣術
   *   c. 魔術師: 魔法2系譜
   *   d. 魔戦士: 魔法1系譜 + 武術
   * 
   * 2. ST, DX が奇数の場合, 優先的に配分 (戦闘能力値が端数切捨のため)
   * 
   * 3. 主技能+副技能の配列をループ
   * 
   * 4. まだCPの余りが生じている場合は, 「鍛錬」「運動」「怪力」に配分
   * 
   */
  setSkills(p: number, s: number, total: number) {
    // 修得すべき技能の配列を作成
    const skills: ParameterKey[] = []
    const spells = ['青の魔法', '赤の魔法', '緑の魔法'] as ParameterKey[]

    // 主技能を配列に追加
    if (p === 0) skills.push('武術')
    if (p === 1) skills.push('剣術')
    if (p === 2) {
      const selected1 = spells.find((_, i) => i === (s + 1) % 3)!
      const selected2 = spells.find((_, i) => i === (s + 2) % 3)!
      skills.push(selected1, selected2)
    }
    if (p === 3) {
      if (this.get('筋力') < this.get('知力')) {
        skills.push(spells[s % 3], '武術')
      } else {
        skills.push('武術', spells[s % 3])
      }
    }

    // 主技能を2周 step
    for (let i = 0; i < 2; i++) {
      skills.forEach(skill => this.step(skill, total))
    }

    // 「運動」が奇数なら, DX (敏捷力) または 「運動」の step を試みる
    if (this.getLevel('運動') % 2) {
      if (this.get('敏捷力') < 2) {
        // DX (敏捷力) への消費が 2 未満なら, DX を step
        this.step('敏捷力', total)
      } else {
        // そうでなければ, 「運動」を step
        this.step('運動', total)
      }
    }
    // 「怪力」が奇数なら, ST (筋力) または 「怪力」の step を試みる
    if (this.getLevel('怪力') % 2) {
      if (this.get('筋力') < 2) {
        // ST (筋力) への消費が 2 未満なら, ST を step
        this.step('筋力', total)
      } else {
        // そうでなければ, 「怪力」を step
        this.step('怪力', total)
      }
    }

    // 副技能を追加
    skills.push('鍛錬')
    if (p === 1) skills.push('運動')
    if (p === 0) skills.push('怪力')

    // 主技能+副技能の修得ループ
    let count = 0 // 安全装置
    while (this.total < total && count < 10) {
      skills.forEach(skill => this.step(skill, total))

      // 2周ループしてもCPの余りが生じた場合
      if (count > 1) {
        (['鍛錬', '運動', '怪力'] as ParameterKey[]).forEach(skill => this.step(skill, total))
      }
      count++
    }
  }

  /**
   * 装備選択アルゴリズム
   * 
   * 1. 必要筋力に応じて武器を選択 (ランダム)
   * 
   * 2. 武器に応じて盾をセット (大盾を扱える場合, ランダムで分岐)
   * 
   * 3. 必要筋力に応じて服・鎧を選択 (STで固定)
   * 
   */
  setEquips(s: number, total: number) {
    // 筋力を取得
    const st = this.getLevel('筋力')

    // 筋力とCP総計に応じた武器一覧
    const weapons = [
      ['小剣', '棍棒', '長杖'],
      ['長剣', '戦棍', '長槍'],
      ['長剣', '戦棍', '長槍', '長剣', '戦斧', '鉾槍'],
      ['長剣', '戦棍', '長槍', '大剣', '戦斧', '鉾槍']
    ]

    // 武器をセット
    if (st <= 10) this.weapon = weapons[0][s % weapons[0].length] as WeaponKey
    if (st >= 11) this.weapon = weapons[1][s % weapons[1].length] as WeaponKey
    if (st >= 13 && total < 12) this.weapon = weapons[2][s % weapons[2].length] as WeaponKey
    if (st >= 13 && total >= 12) this.weapon = weapons[3][s % weapons[3].length] as WeaponKey

    // 盾をセット
    if (!this.weapon.needsTwoHanded) {
      if (st >= 13 && Math.floor(s / 6) % 2 === 0 && total >= 12) this.shield = '大盾'
      else this.shield = '小盾'
    } else {
      this.shield = '装備無し'
    }

    // 服・鎧をセット
    if (st <= 10) this.armor = '革服'
    if (st === 11 || st > 11 && total < 12) this.armor = '革鎧'
    if (st >= 12 && total >= 12) this.armor = 'チェインメイル'
    if (st >= 13 && total >= 16) this.armor = 'プレイトメイル' 
  }
}

// サンプル・キャラクタ群生成関数
// size: 生成数 (範囲内から等間隔にサンプリング)
// total: point 総計
function createSamples(size: number = 4, total: number = 10) {
  const samples = []
  const step = DEFAULT_SIZE / size // 生成数に応じたステップ
  const mod = Math.random() * step // シード値の修正値 (4人PTなら, 0～15の範囲で適用)
  for (let n = 0; n < size; n++) {
    const id = n + 1 // 1からカウント
    const seed = Math.floor(n * step + mod) % DEFAULT_SIZE
    const sample = new Sample(id, seed, total)
    samples.push(sample)
  }
  return samples
}

```

「**サンプル・キャラクタ**」ドメインの実装はここまでです。

---

本章までで「第1フェーズ: キャラクタの実装」は終わりです。次章からいよいよ戦闘場面の実装に入っていきます。

下記デモ版では、React側で編成画面に「CP選択機能」と「ソート機能」を入れてみました。  
ここまでの デモ \[01\] とソースコード (v0.1.0) は、下記URLをご参考ください。

- [DEMO 01 | キャラクタの実装 | RPG.LOGIC](https://rpg-logic.dev/demo/01/)
- [GitHub - RPG.LOGIC v0.1.0](https://github.com/qweldesign/rpg-logic/releases/tag/v0.1.0)

---

実際の実装では、本章の後に「編成画面の作成」を行い、キャラクタ作成やパーティ編成が行えるページを追加しています。  
ただし、この実装はReactによるUI実装が主となり、ドメインモデルを解説する本著の主題とは逸れるため、割愛させて頂いております。  
「編成画面の作成」を含む デモ \[02\] とソースコード (v0.2.0) は、下記URLをご参考ください。

- [DEMO 02 | 編成画面の作成 | RPG.LOGIC](https://rpg-logic.dev/demo/02/)
- [GitHub - RPG.LOGIC v0.2.0](https://github.com/qweldesign/rpg-logic/releases/tag/v0.2.0)
