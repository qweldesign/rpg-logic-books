---
title: "Combat: ターンの管理"
emoji: "⚔️"
type: "tech"
topics: ["gamedev", "typescript", "react"]
published: false
---

# Combat: ターンの管理

ターンおよび全ての戦闘状態を管理するクラスを実装していきます。  
`Combat/` 内に、 `index.ts` を新規作成し、ここにターンと全ての戦闘状態を管理するクラスのコードを記述していきます。

## ターンの設計

`Combat` の役割はターン進行ですが、ターン進行の役割は、下記6つに分割し、それぞれのメソッドとしました。

### Combat.playLog()

UI側でログが再生し終わるのを待ちます。この関数の中身はUI側で定義します。

### Combat.advanceTurn()

ターンをカウントし、次の行動者にターンを移す機能を果たします。  
このとき、倒れているユニットのターンをパスします。

### Combat.startTurn()

 新しいターンの開始処理を行い、前章までで実装してきた `Formation`、`Log`、`Action` を初期化します。

### Combat.runOpeningActions()

ターン開幕時の自動実行の処理が終わるのを待ちます。

### Combat.waitForCommand()

コマンド入力を待機し、コマンド入力後に状態を更新し、ログの再生を待って次のターンへ再帰します。

### Combat.checkResult()

勝敗判定を行います。ちなみに RPG.LOGIC では、勝利・敗北条件は全滅ではなく、前衛ユニットの崩壊です。

## ターンの実装

```TypeScript
// src/domains/Combat/index.ts

// 決着 (未決着は null)
type CombatResult = 'win' | 'lose' | null

// 全ての情報を集約・管理するクラス
class Combat {
  public round: number // 経過時間
  public turnIndex: number // 行動順
  public units: Unit[]
  public formation: Formation | null
  public action: Action | null
  public logs: Log[]
  public playLog: () => Promise<void> // Combat 本体から受け取り, ActionStore から呼び出す
  public result: CombatResult // 勝敗判定

  constructor(models: UnitModel[], playLog: () => Promise<void>) {
    this.round = 1 // 1からカウント
    this.turnIndex = 0 // 開幕前は 0, 開幕と同時に 1 になる
    this.units = models.map((model, i) => {
      return new Unit(model, i + 1) // combatIdは1からカウント
    })
    this.formation = null
    this.action = null
    this.logs = []
    this.playLog = playLog
    this.result = null
  }

  get actor() {
    return this.units[this.turnIndex - 1]
  }

  // 次のターンへ進む
  async nextTurn() {
    if (this.checkResult()) {
      await this.playLog()
      return
    }
    this.advanceTurn()
    await this.startTurn()
    await this.runOpeningActions()
    await this.waitForCommand()
  }

  // 勝敗判定
  // 決着していれば result, log へ反映して true を返す
  private checkResult(): boolean {
    if (this.round <= 1) return false
    const result = this.judgeResult()
    if (!result) return false
    this.result = result
    this.logs[0]?.receiveResult(result)
    return true
  }

  // turnIndex / round を進める
  // 倒れているユニットのターンをパス 
  private advanceTurn(): void {
    let isAlive = false
    while (!isAlive) {
      this.turnIndex++
      if (this.turnIndex > this.units.length) {
        this.round++
        this.turnIndex -= this.units.length
      }
      isAlive = !this.actor.health.unconscious
    }
  }

  // 新しいターンの開始処理
  // formation, log, action 初期化
  private async startTurn(): Promise<void> {
    this.formation = new Formation(this.actor, this.units)
    // 前ターンのログを, その行動者の履歴として保持 (Summaryの行動ラベル表示用)
    if (this.logs[0]) this.logs[0].actor.history = this.logs[0]
    this.logs.unshift(new Log(this.actor))
    await this.playLog()
    this.action = new Action(this)
  }

  // 開幕時の自動実行
  // 朦朧回復・立ち上がりの完了を待つ
  private async runOpeningActions(): Promise<void> {
    await this.action!.ready
  }

  // コマンド入力待機 → 状態更新 → 次のターンへ再帰
  private async waitForCommand(): Promise<void> {
    await this.action!.promise.then(() => {
      this.actor.nextTurn()
      this.debug()
      this.nextTurn()
    })
  }

  // 勝敗判定
  // 前衛に生存者 (気絶していない者) が1人もいない陣営があれば, その陣営の敗北とする
  // 開幕直後 (round === 1, 全員が最初の1巡を終えるまで) は判定対象外
  private judgeResult(): CombatResult {
    const hasFrontAlive = (side: Side) => this.units.some(unit => (
      unit.side === side && unit.position !== 'back' && !unit.health.unconscious
    ))
    if (!hasFrontAlive('player')) return 'lose'
    if (!hasFrontAlive('enemy')) return 'win'
    return null
  }

  debug() {
    const { round, turnIndex, units } = this
    console.log({ round, turnIndex, units })
  }
}

```

---

本章までで「第2フェーズ: 戦闘の実装・前編 (ターンとログの実装)」は終わりです。  
`Unit/Attack`、`Unit/Defense`、`Action/resolver` などのドメインは未実装のため、下記デモ版では「全力防御」と「移動」のみが実装され、実際にログ出力とターン進行が行われているのを確認できます。
ここまでの デモ \[03\] とソースコード (v0.3.0) は、下記URLをご参考ください。

- [DEMO 03 | ターンとログの実装 | RPG.LOGIC](https://rpg-logic.dev/demo/03/)
- [GitHub - RPG.LOGIC v0.3.0](https://github.com/qweldesign/rpg-logic/releases/tag/v0.3.0)

---

次フェーズでは、戦闘行動の実装として、`Unit/Attack`、`Unit/Defense`、`Action/resolver` などのドメインを実装していきますが、実際の制作ではドメインごとの実装ではなく、各種コマンドごとに各々ドメインの実装を少しずつ進めてきました。  
本フェーズと次フェーズの間の途中までの実装内容として、攻撃の基本実装 (攻撃判定 → 防御判定 → ダメージ判定 → ダメージ効果 → ダメージ効果適用) までを完了した デモ \[04\] とソースコード (v0.4.0) は、下記URLをご参考ください。

- [DEMO 04 | 攻撃の基本実装 | RPG.LOGIC](https://rpg-logic.dev/demo/04/)
- [GitHub - RPG.LOGIC v0.4.0](https://github.com/qweldesign/rpg-logic/releases/tag/v0.4.0)
