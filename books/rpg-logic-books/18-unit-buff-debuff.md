---
title: "Unit/Buff&Debuff: バフ・デバフ効果の管理"
emoji: "⚔️"
type: "tech"
topics: ["gamedev", "typescript", "react"]
published: false
---

# Unit/Buff&Debuff: バフ・デバフ効果の管理

魔法の実装に際して、バフ・デバフ効果を管理するクラスを実装していきます。  
`Combat/Unit/` 内に、 `StatusBuff.ts` と `StatusDebuff.ts` を新規作成し、ここにバフ・デバフ効果を管理するクラスのコードを記述していきます。

## バフ・デバフ効果の管理の設計

`StatusBuff.ts` と `StatusDebuff.ts` は、バフ・デバフの効果そのものではなく、持続ターンの管理が主となります。  
それぞれの効果は簡単に静的アクセスできるように、`getter` を設定します。

### バフ効果の算出

バフ効果は10ターン持続するルールのため、10ターンごとにそれぞれバフ効果が `+1` 得られるように算出式を記述します。  
このうち、`Level`、`Dmg`、`Ev` は効果が累積しますが、`DR` は累積しません。
ダメージ修正 (`Dmg`) と回避値 (`Ev`) は、それぞれ技能値の半分が有効なので、技能値に端数が生じた場合はバフの初期値として加算するようにします。  
（ただし、ダメージ修正に対するバフ効果を得られる魔法は、今回該当するものがありません。実装のみです。）

### デバフ効果の優先順位

狂戦士、幻惑、恐慌の状態異常は、効果の内容が競合するため、効果が適用される優先順位を付けておきます。  
恐慌状態が最も重篤な状態異常 (前衛から後退してしまう) であるため、狂戦士 < 幻惑 < 恐慌 としました (`getter` の設定を参照してください)。

### 効果の減衰

`nextTurn()` メソッドで、バフ・デバフの効果は、ターン毎に衰退するようにします。

### ラベルの設定

UI表示用のラベルの `getter` を設定しておきます。

## バフ・デバフ効果の管理の実装

```TypeScript
// src/domains/Combat/Unit/StatusBuff.ts

class StatusBuff {
  private dmgInitBuff: number //「怪力」端数
  private evInitBuff: number //「運動」端数
  private levelBuff: number // 命中UP (ヒロイズム)
  private dmgBuff: number // 攻撃UP (該当無し)
  private evBuff: number // 回避UP (ヘイスト)
  private drBuff: number // 防御UP (水の鎧)

  constructor(dmgBuff: number = 0, evBuff: number = 0) {
    this.dmgInitBuff = dmgBuff * 10
    this.evInitBuff = evBuff * 10
    this.levelBuff = 0
    this.dmgBuff = 0
    this.evBuff = 0
    this.drBuff = 0
  }

  // 各バフは10ターン持続
  // 毎ターンバフ変数をデクリメント
  nextTurn() {
    this.levelBuff = Math.max(this.levelBuff - 1, 0)
    this.dmgBuff = Math.max(this.dmgBuff - 1, 0)
    this.evBuff = Math.max(this.evBuff - 1, 0)
    this.drBuff = Math.max(this.drBuff - 1, 0)
  }

  // ヒロイズムの付与 (10ターン, +3まで累積)
  addLevelBuff() {
    this.levelBuff = Math.min(this.levelBuff + 10, 30)
  }

  // dmgBuff の付与 (10ターン, +3まで累積)
  addDmgBuff() {
    this.dmgBuff = Math.min(this.dmgBuff + 10, 30)
  }

  // ヘイストの付与 (10ターン, +3まで累積)
  addEvBuff() {
    this.evBuff = Math.min(this.evBuff + 10, 30)
  }

  // 水の鎧の付与 (10ターン, 累積しない)
  addDrBuff() {
    this.drBuff = 10
  }

  get level() {
    // 残ターン数の10分の1 (端数切上) が実際の効果
    return Math.ceil(this.levelBuff / 10)
  }

  get dmg() {
    // 残ターン数の10分の1 (端数切上) + 初期値(10) の半分 が実際の効果
    return Math.floor(Math.ceil((this.dmgInitBuff + this.dmgBuff) / 10) / 2)
  }

  get ev() {
    // 残ターン数の10分の1 (端数切上) + 初期値(10) の半分 が実際の効果
    return Math.floor(Math.ceil((this.evInitBuff + this.evBuff) / 10) / 2)
  }

  get dr() {
    // 残ターン数が残っていれば, 効果は一定
    return this.drBuff > 0 ? 1 : 0
  }

  // Summary 表示用ラベル取得 (該当するバフを1つ返す)
  get label(): string {
    if (this.levelBuff > 0) return '命中UP'
    if (this.dmgBuff > 0) return '攻撃UP'
    if (this.evBuff > 0) return '回避UP'
    if (this.drBuff > 0) return '防御UP'
    return ''
  }
}

```

```TypeScript
// src/domains/Combat/Unit/StatusDebuff.ts

class StatusDebuff {
  private _berserk: number // 狂戦士
  private _dazed: number // 幻惑
  private _fear: number // 恐慌
  public flashed: number // 目くらみ

  constructor() {
    this._berserk = 0
    this._dazed = 0
    this._fear = 0
    this.flashed = 0
  }

  // 毎ターン残存時間をデクリメント
  nextTurn() {
    this._berserk = Math.max(this._berserk - 1, 0)
    this._dazed = Math.max(this._dazed - 1, 0)
    this._fear = Math.max(this._fear - 1, 0)
    this.flashed = Math.max(this.flashed - 1, 0)
  }

  set berserk(duration: number) {
    this._berserk = duration
  }

  set dazed(duration: number) {
    this._dazed = duration
  }

  set fear(duration: number) {
    this._fear = duration
  }

  // 精神異常 (狂戦士・幻惑・恐慌) が競合した場合の優先順位: 狂戦士 < 幻惑 < 恐慌
  get berserk(): boolean {
    return this._berserk > 0 && !this.dazed && !this.fear
  }

  get dazed(): boolean {
    return this._dazed > 0 && !this.fear
  }

  get fear(): boolean {
    return this._fear > 0
  }

  // Summary 表示用ラベル取得 (該当するデバフを1つ返す)
  get label(): string {
    if (this.berserk) return `狂戦士(${this._berserk})`
    if (this.dazed) return `幻惑(${this._dazed})`
    if (this.fear) return `恐慌(${this._fear})`
    if (this.flashed) return '目くらみ'
    return ''
  }
}

```

---

次章では、`Spells` (魔法の効果一覧) の実装を進めます。
