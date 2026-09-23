---
title: "Log: ログの管理"
emoji: "⚔️"
type: "tech"
topics: ["gamedev", "typescript", "react"]
published: false
---

# Log: ログの管理

ログを管理するクラスを実装していきます。  
`Combat/` 内に、 `Logs.tsx` を新規作成し、ここにログを管理するクラスのコードを記述していきます。  
このドメインのみ、拡張子が `.tsx` になる点をご注意ください。

## ログの設計

ログは、次章で実装する予定の `Combat` のメンバ変数として保持され、前章で実装した `Action` 内で `type ActionRequest` と `type ActionResult` の配列を受け取り、UI表示用の行動履歴ラベル・結果ログを生成するのが役割です。

```TypeScript
// src/domains/Combat/Action/index.ts

class CombatAction {

  ...

  async execute (action: ActionRequest) {

    ...

    // ログを更新
    const log = this.state.logs[0] // ← state は Combat を格納している
    log.receiveResults(action, results)

    ...
  }
}

```

## ログの実装

```tsx
// src/domains/Combat/Log.tsx

let count = 0

// タイムラインへのログ表示を司るクラス / Timelineコンポーネントに対応
class CombatLog {
  public id: number
  public actor: Unit
  public messages: ReactNode[][]
  public label?: string

  // インスタンス生成時は「XXXXの行動順」を表示する
  constructor(actor: Unit) {
    this.id = count++
    this.actor = actor
    const firstMessage = (<span className="font-bold">{actor.name} の行動順</span>)
    this.messages = [[firstMessage]]
  }

  // Action コンポーネントで ActionRequest と ActionResult[] を受け取り, ラベルと結果ログを生成する
  receiveResults(request: ActionRequest, results: ActionResult[] = []) {
    this.label = this.createLabel(request, results)
    this.messages.push(this.createMessages(request, results))
  }

  // ラベル生成 (Summary履歴用)
  private createLabel(request: ActionRequest, results: ActionResult[]): string {
    switch (request.key) {
      case 'attack':
        const attackLabel = request.options.fullPower !== 'none' ? '全力攻撃' : ACTION_LABELS[request.key]
        return `${attackLabel}:${this.createAttackResultLabel(request, results)}`

      case 'feint':
        return `${ACTION_LABELS[request.key]}:${this.createFeintResultLabel(results)}`

      case 'move':
        return `${ACTION_LABELS[request.key]}:${POSITION_LABELS[request.options.position]}`

      default: // case 'ready': case 'defense': case 'recovery': case: 'standup': case 'wait':
        return ACTION_LABELS[request.key]
    }
  }

  // 攻撃の成否ラベルを生成
  // 攻撃(成功) → 防御(失敗) → ダメージ(貫通) の場合のみ「成功」を返す
  private createAttackResultLabel(request: ActionRequest, results: ActionResult[]): string {
    if (request.key !== 'attack') return ''
    let success = false
    results.forEach(result => {
      switch (result.type) {
        case 'attack':
          success = result.judge.success
          break
        case 'defense':
          success = !result.judge.success
          break
        case 'dmg':
          success = result.judge.success
          break
      }
    })
    return success ? '成功' : '失敗'
  }

  // 牽制の成否ラベルを生成 (成功時は成功度も表示)
  private createFeintResultLabel(results: ActionResult[]): string {
    let success = false
    let score = 0
    results.forEach(result => {
      if (result.type === 'feint') {
        success = result.judge.success
        score = result.judge.score
      }
    })
    return success ? `成功(${score})` : '失敗'
  }

  // 結果ログ生成
  private createMessages(request: ActionRequest, results: ActionResult[]): ReactNode[] {
    const actor = this.actor.name
    const messages: ReactNode[] = []
    switch (request.key) {
      case 'ready': {
        messages.push(<>{`${actor} は ${this.actor.attack.name} を構えた`}</>)
        break
      }
      case 'attack': case 'feint': {
        const target = request.target
        results.forEach(result => {
          switch (result.type) {
            case 'attack':
              messages.push(<>{`${actor} の ${this.actor.attack.name} による攻撃!`}</>)
              messages.push(<>{`出目は ${result.judge.roll}、${this.getResultLabel(result.judge)}`}</>)
              if (!result.judge.success && !result.judge.ready) {
                // 攻撃失敗時のみ非準備状態への変化をログに表示
                messages.push(<>{`${actor} の ${this.actor.attack.name} は非準備状態になった`}</>)
              }
              break

            case 'feint':
              messages.push(<>{`${actor} は ${target.name} に対して牽制を仕掛けた!`}</>)
              if (result.judge.success) {
                messages.push(<>{`出目は ${result.judge.roll}、牽制は成功した!`}</>)
                messages.push(<>{`次のターン, ${target.name} は防御判定に -${result.judge.score} の修正が課せられる!`}</>)
              } else {
                messages.push(<>{`出目は ${result.judge.roll}、牽制は失敗した...`}</>)
              }
              break

            default: // case 'defense': case 'dmg': case 'trip': case 'knockedDown': case 'fatal':
              this.pushDmgResolutionMessage(messages, request.target, result)
              break
          }
        })
        break
      }
      case 'defense': {
        messages.push(<>{`${actor} は 防御に専念!`}</>)
        break
      }
      case 'move': {
        messages.push(<>{`${actor} は ${POSITION_LABELS[request.options.position]} へ移動した`}</>)
        break
      }
      case 'recovery': {
        results.forEach(result => {
          if (result.type === 'recovery') {
            if (result.judge.success) messages.push(<>{`${actor} は 朦朧状態から回復した!`}</>)
            else messages.push(<>{`${actor} は 朦朧としていて何も行動できない...`}</>)
          }
        })
        break
      }
      case 'standup': {
        messages.push(<>{`${actor} は 転倒状態から立ち上がろうとしている`}</>)
        break
      }
      default: { // case 'wait':
        messages.push(<>{`${actor} は 待機している`}</>)
        break
      }
    }
    messages.push(<>&nbsp;</>)
    return messages
  }

  private getResultLabel(judge: Judge): string {
    return judge.success && judge.critical ? 'クリティカル!!'
      : judge.success && !judge.critical ? '成功!' : '失敗!'
  }

  // 防御判定以降の結果ログを追加
  private pushDmgResolutionMessage(messages: ReactNode[], target: Unit, result: ActionResult) {
    switch (result.type) {
      case 'defense':
        const defenseTypeLabel = result.judge.type === 'parry' ? '武器による受け流し'
          : result.judge.type === 'block' ? '盾による受け止め' : '回避'
        messages.push(<>{`${target.name} は ${defenseTypeLabel} を試みた!`}</>)
        messages.push(<>{`出目は ${result.judge.roll}、${this.getResultLabel(result.judge)}`}</>)
        if (result.judge.success && !result.judge.ready) {
          // 受け成功時のみ非準備状態への変化をログに表示
          messages.push(<>{`${target.name} の ${target.attack.name} は非準備状態になった`}</>)
        }
        break

      case 'dmg':
        if (result.judge.roll < 1) messages.push(<>{`ダメージは ${target.name} の鎧によって完全に止められた...`}</>)
        else if (!result.judge.critical) messages.push(<>{`${target.name} は ${result.judge.roll} 点のダメージを受けた!!`}</>)
        else messages.push(<>{`${target.name} は ${result.judge.roll} 点のダメージを受けた!!!`}</>)
        break

      case 'knockedDown':
        if (result.judge.success) messages.push(<>{`${target.name} は 朦朧状態に陥った!`}</>)
        else messages.push(<>{`${target.name} は 転倒した!!`}</>)
        break

      case 'dead':
        if (result.judge.success) messages.push(<>{`${target.name} は 気絶した...`}</>)
        else messages.push(<>{`${target.name} は 死亡した...`}</>)
        break
    }
  }
  
  // 決着ログ
  receiveResult(result: 'win' | 'lose') {
    const messages: ReactNode[] = []
    if (result === 'win') {
      messages.push(<span className="font-bold">{'敵陣営の前衛が崩れた!! 勝利!!'}</span>)
    } else {
      messages.push(<span className="font-bold">{'味方陣営の前衛が崩れた... 敗北...'}</span>)
    }
    this.messages.push(messages)
  }
}

```

---

次章では、`Combat` (全ての戦闘状態の管理) の実装を進め、ようやくターンの実装が完了します。
