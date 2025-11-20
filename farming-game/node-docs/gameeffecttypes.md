# Game Effect Types Guide

This document provides comprehensive information about all game effect types in the Farming Game, including their purposes, when to use each type, implementation details, and examples.

## Overview

Game effect types define what happens when players draw cards from the Farmer's Fate deck. Each card has an `effectType` from the `GameEffectType` enum and associated `effectDetails` that specify the parameters for that effect.

## Effect Type Categories

### 1. Cash Effects
Effects that modify player cash directly.

#### `GainCash`
**Purpose:** Give player cash reward
**When to use:** Positive events, bonuses, windfalls
**Implementation:** Immediate cash addition
**Example:** "Found $500 in an old jacket"
```json
{
  "amount": 500
}
```

#### `PayCash`
**Purpose:** Force player to pay cash
**When to use:** Expenses, penalties, taxes
**Implementation:** Immediate cash deduction (can cause bankruptcy)
**Example:** "Pay $200 property tax"
```json
{
  "amount": 200
}
```

#### `Expense`
**Purpose:** Variable expense based on assets
**When to use:** Maintenance costs, repairs
**Implementation:** Calculated based on asset counts
**Example:** "Equipment maintenance: $50 per tractor"

### 2. Asset-Related Effects

#### `ImmediateOptionalBuyAsset`
**Purpose:** Force immediate buy/pass decision for assets
**When to use:** High-value opportunities that shouldn't be saved
**Implementation:** Modal dialog with 30-second timeout
**Key Features:**
- Human players get countdown timer with warnings
- AI players decide based on profile + strategic factors
- Auto-pass on timeout or disconnect
- Full payment required (no down payment option)

**Example:** "Uncle Bert's Legacy" - Buy 10 Hay for $10,000
```json
{
  "asset": "Hay",
  "quantity": 10,
  "cost": 10000
}
```

**AI Decision Factors:**
- Current asset portfolio (avoids over-concentration)
- Game phase (early/mid/late strategy)
- Cost per unit analysis
- Profile-based thresholds

#### `OptionalBuyAsset`
**Purpose:** Add asset purchase option to player's hand
**When to use:** Strategic opportunities that can be timed
**Implementation:** Card stays in hand, player can exercise later
**Key Difference:** No immediate decision required

**Example:** "Land Auction" - Option to buy 5 acres for $2500
```json
{
  "asset": "Land",
  "quantity": 5,
  "cost": 2500
}
```

### 3. Harvest and Production Effects

#### `DoubleYieldForCrop`
**Purpose:** Temporarily double harvest yield for specific crop
**When to use:** Weather events, fertilizer bonuses
**Implementation:** Multiplies yield by 2 for specified crop type

#### `HarvestBonusPerAcre`
**Purpose:** Bonus cash per acre of specific crop
**When to use:** Premium prices, government subsidies
**Implementation:** Additional payment per acre harvested

### 4. Movement and Location Effects

#### `GoToTile`
**Purpose:** Move player to specific tile
**When to use:** Transportation events, shortcuts
**Implementation:** Immediate position change

#### `MoveAndHarvestIfAsset`
**Purpose:** Move to tile and harvest if player owns assets there
**When to use:** Travel opportunities with production bonuses

### 5. Disaster and Penalty Effects

#### `MtStHelensDisaster`
**Purpose:** Volcanic eruption damage simulation
**When to use:** Rare catastrophic events
**Implementation:** Complex asset destruction logic

#### `SlaughterCowsWithoutCompensation`
**Purpose:** Livestock disease outbreak
**When to use:** Agricultural disasters
**Implementation:** Removes cattle assets without payment

### 6. Social and Multi-Player Effects

#### `CollectFromOthersIfAsset`
**Purpose:** Collect money from other players based on their assets
**When to use:** Market manipulation, taxation
**Implementation:** Iterates through all other players

#### `PayIfNoAssetDistribute`
**Purpose:** Pay if lacking asset, distribute to others who have it
**When to use:** Regulatory penalties, insurance requirements

## Implementation Guidelines

### For New Effect Types

1. **Add to GameEffectType enum** in both shared-types and Prisma schema
2. **Define effect details structure** with clear TypeScript interface
3. **Implement in CardEffectService** with proper validation
4. **Add Zod schema** for runtime validation
5. **Update documentation** in this guide
6. **Add unit tests** for edge cases

### Validation Rules

- All effect details must be validated at runtime
- Cash amounts must be positive integers
- Asset quantities must be reasonable (1-100 range typically)
- Asset types must exist in AssetType enum
- Effect combinations should be logical

### Error Handling

- Invalid effect details should throw descriptive errors
- Missing required fields should be caught early
- Asset purchases should validate player has sufficient cash
- Network timeouts should auto-resolve decisions safely

## AI Behavior Considerations

Different AI profiles should respond differently to effect types:

### Gambler AI
- Takes risks on uncertain payoffs
- More likely to buy assets impulsively
- Accepts higher risk-reward ratios

### Expansionist AI
- Focuses on asset acquisition
- Prioritizes productive assets (Hay, Corn, etc.)
- Maintains moderate cash buffer for opportunities

### Cautious AI
- Preserves cash reserves
- Only buys when value is exceptional
- Avoids risky investments

### Aggressive AI
- Competes aggressively for assets
- Takes calculated risks
- Builds portfolio quickly

### Balanced AI
- Makes rational economic decisions
- Considers opportunity costs
- Adapts strategy by game phase

## Testing Checklist

For each effect type implementation:

- [ ] Human player flow works correctly
- [ ] AI players make reasonable decisions
- [ ] Timeout/disconnect handling works
- [ ] Invalid inputs are rejected gracefully
- [ ] Game state updates correctly
- [ ] Logging is comprehensive
- [ ] Edge cases are handled
- [ ] Performance impact is minimal

## Future Enhancements

Potential new effect types to consider:

1. **Auction Effects**: Multi-player bidding mechanics
2. **Loan Effects**: Interest rate modifications
3. **Weather Effects**: Multi-turn harvest modifiers
4. **Insurance Effects**: Risk mitigation mechanics
5. **Alliance Effects**: Temporary partnerships
6. **Sabotage Effects**: Competitive disruption
