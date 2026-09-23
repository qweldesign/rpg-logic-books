---
title: "Formation: 戦闘配置の管理"
emoji: "⚔️"
type: "tech"
topics: ["gamedev", "typescript", "react"]
published: false
---

# Formation: 戦闘配置の管理

戦闘ユニットの配置を司るクラスを実装します。
`Combat/` 内に、 `Formation/index.ts` を新規作成し、ここに戦闘ユニットを配置するクラスのコードを記述していきます。

## 戦闘配置の設計

### 戦闘配置の管理

`Formation` の第一の要件は、戦闘ユニット全体の配置の管理です。  
戦闘配置の仕様としては、ドキュメントの通り、`front` (前衛) か `back` (後衛) かになりますが、前衛は `left` (左翼)、`center` (中央)、`right` (右翼) のいずれかの値を取ることにします。

`back` (後衛) に立つ場合、他ユニットと干渉し合うことはありませんが、`front` (前衛) に立つ場合、他の味方ユニットが既にその位置に立っている場合、重なって立つことはできません。

よって、`Formation` は、`front` (前衛) と `back` (後衛) とで管理を分けます。

```TypeScript

type BackFormation = Record<number, Unit | null>
type FrontFormation = Record<Position, Unit | null>
type Formation = { back: BackFormation, front: FrontFormation }

```

このように `front` (前衛) と `back` (後衛) とで管理を分けた上で、`player` 側と `enemy` 側で `Formation` から静的にアクセスして配置を取得できるようにコードを組みます。

```TypeScript
// ユニットの配置を司るクラス
class Formation {

  ...

  // Store[Side][Position] でユニットへ静的アクセスできる
  private getFormation(side: Side): { back: BackFormation, front: FrontFormation } {
    const back = BACK_VALUES[side].reduce<BackFormation>((acc, value) => {
      acc[value] = this.back.get(value) ?? null
      return acc
    }, {} as BackFormation)

    const front = FRONT_VALUES.reduce<FrontFormation>((acc, pos) => {
      acc[pos] = this.front.get(`${side}-${pos}`) ?? null
      return acc
    }, {} as FrontFormation)

    return {
      back, front
    }
  }

  get player(): Formation {
    return this.getFormation('player')
  }

  get enemy(): Formation {
    return this.getFormation('enemy')
  }
}

```

### ターゲット取得関数の実装

`Formation` の第二の要件として、行動時のターゲット群を取得できるようにします。  
ターゲット群は、`allies` (味方)、`enemies` (敵)、`meleeTargets` (近接攻撃対象) の3パターンを用意しておきます。

## 戦闘配置の実装

```TypeScript
// src/domains/Combat/Formation/index.ts

const BACK_VALUES = { player: [1, 2, 3, 4], enemy: [5, 6, 7, 8] } as const
const FRONT_VALUES: Position[] = ['left', 'center', 'right'] as const

type BackFormation = Record<number, Unit | null>
type FrontFormation = Record<Position, Unit | null>
type Formation = { back: BackFormation, front: FrontFormation }

// ユニットの配置を司るクラス
class CombatFormation {
  public actor: Unit // CSSマーク用
  private units: Unit[]
  private back: Map<number, Unit | null>
  private front: Map<string, Unit | null>

  constructor(actor: Unit, units: Unit[]) {
    this.actor = actor
    this.units = units
    this.back = new Map<number, Unit | null>()
    this.front = new Map<string, Unit | null>()
    // Front 初期化
    SIDE_KEYS.forEach(side => {
      POSITION_KEYS.slice(1).forEach(position => {
        this.front.set(`${side}-${position}`, null)
      })
    })
    // ユニット配置 (Back 初期化含む)
    units.forEach((unit, i) => {
      if (unit.position === 'back') {
        this.back.set(i + 1, unit)
      } else {
        this.back.set(i + 1, null)
        this.front.set(`${unit.side}-${unit.position}`, unit)
      }
    })
  }

  // Store[Side][Position] でユニットへ静的アクセスできる
  private getFormation(side: Side): { back: BackFormation, front: FrontFormation } {
    const back = BACK_VALUES[side].reduce<BackFormation>((acc, value) => {
      acc[value] = this.back.get(value) ?? null
      return acc
    }, {} as BackFormation)

    const front = FRONT_VALUES.reduce<FrontFormation>((acc, pos) => {
      acc[pos] = this.front.get(`${side}-${pos}`) ?? null
      return acc
    }, {} as FrontFormation)

    return {
      back, front
    }
  }

  get player(): Formation {
    return this.getFormation('player')
  }

  get enemy(): Formation {
    return this.getFormation('enemy')
  }

  // 味方対象取得
  getAllies(): Unit[] {
    return this.units.filter(unit => unit.side === this.actor.side && !unit.health.unconscious)
  }

  // 敵対象取得
  getEnemies(): Unit[] {
    return this.units.filter(unit => unit.side !== this.actor.side && !unit.health.unconscious)
  }

  // 近接攻撃対象取得
  getMeleeTargets(): Unit[] {
    const enemies = this.getEnemies()
    const reachable = (() => {
      switch (this.actor.position) {
        case 'left':
          return enemies.filter(unit => {
            return (unit.position === 'center' || unit.position === 'right') && !unit.health.unconscious
          })

        case 'center':
          return enemies.filter(unit => {
            return unit.position !== 'back' && !unit.health.unconscious
          })

        case 'right':
          return enemies.filter(unit => {
            return (unit.position === 'left' || unit.position === 'center') && !unit.health.unconscious
          })

        default: // case 'back':
          return []
      }
    })()

    // 前衛にいてかつ近接攻撃対象がいない場合, 全ての敵を対象として取得できる
    if (this.actor.position !== 'back' && reachable.length === 0) return enemies
    return reachable
  }
}

```

---

次章からは、`Action` (行動) の実装を進めます。
