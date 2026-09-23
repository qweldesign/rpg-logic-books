---
title: "Action: 行動の管理"
emoji: "⚔️"
type: "tech"
topics: ["gamedev", "typescript", "react"]
published: false
---

# Action: 行動の管理

戦闘ユニットの行動の管理を司るクラスと、そのための型定義を実装していきます。  
`Combat/` 内に、 `Action/index.ts` と `Action/type.ts` を新規作成し、ここに戦闘ユニットの行動の管理のコードと型定義を記述していきます。

## 行動の設計

戦闘ユニットの行動の設計とは、**行動決定のためのコマンド入力から、状態更新を経てログ出力まで、ターンとログの同期設計**に直結する、本プロジェクトの核とも言える箇所です。  
行動決定のためのコマンド入力は、`type ActionRequest` のプロパティ (`key`, `options`, `target`) を引数に取って実行関数が処理を進め、状態更新を経て、`type ActionResult` の配列をログに渡し、ログを再生して次のターンへ移るように設計します。

戦闘ユニットの行動の管理を司るクラス `Action` は、主に下記のようなメンバ変数・メソッドを持ちます。  
尚、メンバ変数 `state` は、後で実装する全ての戦闘状態を管理するクラス `Combat` を格納し、前章で実装した `Formation` や `Unit` にアクセスできます。

### Action.excecute (実行関数)

先述の `type ActionRequest` のプロパティ (`key`, `options`, `target`) を引数に取って、処理を進める関数です。  
コマンドごとの各処理は、`Action.Effects` に委譲します。

### Action.Availability (行動可否判定)

コマンドごとの行動可否判定を管理し、実行不可のコマンドボタンをUI上で非表示にします。

### Action.Effects (行動実行 (状態変更))

コマンドごとの行動実行メソッドを持ち、状態変更を行った後、ログ出力のための `type ActionResult` の配列をログに渡します。

### Action.unlockd

コマンド入力パレットのロック状態を操作します。  
React側の `UseEffect` でこの値の変更を検知する仕組みで、この箇所だけがUIに直結しています。

## 行動の実装

まずは型定義からです。

```TypeScript
// src/domains/Combat/Action/types.ts

const ACTION_KEYS = ['ready', 'attack', 'feint', 'defense', 'move', 'recovery', 'standup', 'wait'] as const

const ACTION_LABELS: Record<ActionKey, string> = {
  ready: '準備',
  attack: '攻撃',
  feint: '牽制',
  defense: '全力防御',
  move: '移動',
  recovery: '回復',
  standup: '立ち上がり',
  wait: '待機'
} as const

const POSITION_LABELS: Record<Position, string> = {
  back: '後方',
  left: '左翼',
  center: '中央',
  right: '右翼'
} as const

const FULL_POWER_KEYS = ['none', 'dmg', 'level', 'feint', 'double', 'ready'] as const

const FULL_POWER_OPTIONS: Record<FullPower, { label: string }> = {
  none: { label: '通常攻撃' },
  dmg: { label: 'ダメージ安定' },
  level: { label: '技能値+4' },
  feint: { label: '牽制即攻撃' },
  double: { label: '2回攻撃' },
  ready: { label: '準備即攻撃' }
} as const

// 行動キー
type ActionKey = typeof ACTION_KEYS[number]

// 全力攻撃オプション
type FullPower = typeof FULL_POWER_KEYS[number]

// 行動オプション
type ActionOptions = {
  position?: Position
  fullPower?: FullPower
}

// 行動キーとオプションの組み合わせ
type ActionRequest =
  | { key: 'ready', options: {} }
  | { key: 'attack', options: { fullPower: FullPower }, target: Unit }
  | { key: 'feint', options: {}, target: Unit }
  | { key: 'defense', options: {} }
  | { key: 'move', options: { position: Position } }
  | { key: 'recovery', options: {} }
  | { key: 'standup', options: {} }
  | { key: 'wait', options: {} }

// 攻撃判定結果
type AttackResult = Judge & {
  ready: boolean // 攻撃後の武器の準備状態
}

// 防御判定結果
type DefenseResult = Judge & {
  type: DefenseType
  ready: boolean // 防御後の武器の準備状態
}

// ダメージ判定結果
type DmgResult = Judge

// 牽制の判定結果
type FeintResult = Score & {
  target: Unit
}

// 行動実行後の判定結果の定義
type ActionResult =
  | { type: 'attack', judge: AttackResult }
  | { type: 'defense', judge: DefenseResult }
  | { type: 'dmg', judge: DmgResult }
  | { type: 'feint', judge: FeintResult }
  | { type: 'recovery', judge: Judge }
  | { type: 'knockedDown', judge: Judge }
  | { type: 'dead', judge: Judge }

```

次に、行動の管理を司るクラス `Action` を実装します。  
実際には、まだ `Availability`、`Effects` などのドメインは未実装なので、このままではコードは動きませんが、完成イメージの共有になります。 

```TypeScript
// src/domains/Combat/Action/index.ts

// 行動の管理を司るクラス / Actionコンポーネントに対応
class CombatAction {
  private state: State
  public round: number
  public formation: Formation
  public unlocked: boolean // コマンドパレットのロック状態 → Actions にて検知
  public promise: Promise<void>
  public ready: Promise<void> // 開幕時の自動実行 (朦朧回復・立ち上がり) が完了したら解決
  private resolve!: () => void
  private readonly availabilityChecker: Availability
  private readonly effects: Effects

  constructor(state: State) {
    this.state = state
    this.round = state.round
    this.formation = state.formation!
    this.unlocked = true // コマンドパレットをアンロック
    this.availabilityChecker = new Availability(state)
    this.effects = new Effects(state)

    // ターン終了を Promise で State に伝え, 次のターンへ進む
    this.promise = new Promise(resolve => {
      this.resolve = resolve
    })

    if (this.actor.health.stunned) {
      // 朦朧状態の場合は「回復」を自動実行する
      this.ready = this.execute({ key: 'recovery', options: {} })
    } else if (!this.actor.health.stunned && this.actor.health.prone) {
      // 転倒状態の場合は「立ち上がり」を自動実行する
      this.ready = this.execute({ key: 'standup', options: {} })
    } else if (this.actor.position === 'back' && !this.availabilityChecker.canMove()) {
      // 後衛にいて狂戦士状態で, かつ前衛の移動先が無い場合は「待機」を自動実行する
      this.ready = this.execute({ key: 'wait', options: { status: 'berserk'} })
    } else if (this.actor.debuff.fear) {
      // 恐慌状態の場合は「後退」か「待機」を自動実行する
      this.ready = this.actor.position !== 'back'
        ? this.execute({ key: 'move', options: { position: 'back' } })
        : this.execute({ key: 'wait', options: { status: 'fear'} })
    } else {
      this.ready = Promise.resolve()
    }
  }

  get actor() {
    return this.state.actor
  }

  // 実行可否
  get availability() {
    return {
      ready: this.availabilityChecker.canReady(),
      attack: this.availabilityChecker.canAttack(),
      fullPowerAttack: this.availabilityChecker.canFullPowerAttack(),
      doubleAttack: this.availabilityChecker.canDoubleAttack(),
      feint: this.availabilityChecker.canFeint(),
      defense: this.availabilityChecker.canDefense(),
      move: POSITION_KEYS.reduce((acc, position) => {
        acc[position] = this.availabilityChecker.canMoveToPosition(position)
        return acc
      }, {} as Record<typeof POSITION_KEYS[number], boolean>),
      wait: this.availabilityChecker.canWait()
    }
  }

  // ターゲット (Formation の配置情報を元に絞り込む)
  get target() {
    const formation = this.state.formation
    return {
      all: this.state.units,
      allies: formation?.getAllies() ?? [],
      enemies: formation?.getEnemies() ?? [],
      melee: formation?.getMeleeTargets() ?? []
    }
  }

  // 実行
  // ActionRequest のプロパティ (key, options, target) を引数に取って処理を進め,
  // ActionResult の配列を Log に渡し, 再生して次のターンへ移る
  async execute (action: ActionRequest) {
    // コマンドパレットをロック (アンロックはコンストラクタで行われる)
    this.unlocked = false

    // 「集中」以外を実行した場合, 精神集中を破棄
    if (action.key !== 'cast') this.effects.cancelCastSelf()

    // 行動実行
    let results: ActionResult[] = []
    switch (action.key) {
      case 'ready':
        results = this.effects.ready()
        break

      case 'attack':
        results = this.effects.attack(action.target, action.options.fullPower)
        break

      case 'feint':
        results = this.effects.feint(action.target)
        break

      case 'defense':
        this.effects.defense()
        break

      case 'move':
        this.effects.move(action.options.position)
        break

      case 'recovery':
        results = this.effects.recovery()
        break

      case 'standup':
        this.effects.standup()
        break

      default: // case 'wait':
        this.effects.wait()
    }

    // ログを更新
    const log = this.state.logs[0]
    log.receiveResults(action, results)

    // 行動終了分岐
    // 回復成功時・立ち上がりはターンを終えず, 同じ actor の行動を続ける
    let nextTurn = true
    const recoveryResult = results.find(result => result.type === 'recovery')
    if ((action.key === 'recovery' && recoveryResult?.judge.success)
      || action.key === 'standup'
    ) {
      this.unlocked = true
      nextTurn = false
    }

    // 行動終了
    await this.state.playLog() // ログの再生完了を待つ
    if (nextTurn) {
      this.resolve()
    }
  }
}

```

---

次章では、`Action/Availability` (行動可否判定の管理) の実装を進めます。
