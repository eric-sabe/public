# Fair Market Value (FMV) Calculation System

## Overview

The Fair Market Value (FMV) calculation system provides sophisticated economic analysis for all marketplace transactions in the farming game. It determines the "true" economic worth of assets by analyzing their income potential, risk factors, and market conditions.

FMV is used for:
- **Marketplace listings**: Suggesting appropriate asking prices
- **AI decision making**: Evaluating purchase/sell opportunities
- **Player guidance**: Showing economic value when considering OTB exercise

## Core Calculation Framework

### Base Formula
```
FMV = Base_Productivity × Competition_Factor × Time_Value × Scarcity × Operating_Expense_Risk × Equipment_Usage
```

### Key Components
1. **Base Productivity**: Expected earnings from harvests/income
2. **Competition Factor**: Market saturation effects (0.3-1.0)
3. **Time Value**: Asset worth increases with game progression (0.3-1.1)
4. **Scarcity**: OTB card rarity premium (1.0-1.2)
5. **Operating Expense Risk**: Card-based expenses/income (0.2-1.5)
6. **Equipment Usage Probability**: Likelihood of actual utilization (0.4-1.0)

---

## 1. Base Productivity Calculation

### Harvestable Assets (Hay, Grain, Fruit, Cows)
```
Base_Productivity = Base_Earnings_Per_Harvest × Quantity × Expected_Harvests
```

**Base Earnings Per Harvest:**
- Hay: $1,000 per acre
- Grain: $2,000 per acre
- Fruit: $5,000 per acre
- Cows: $500 per head

**Expected Harvests Calculation:**
```
Harvest_Probability = Harvestable_Spaces / Total_Board_Spaces × Movement_Efficiency
Expected_Harvests = Turns_Remaining × Harvest_Probability
```

**Movement Efficiency:**
- Hay: 0.75 (common, clustered)
- Grain: 0.8 (moderately common)
- Fruit: 0.75 (premium, targeted)
- Cows: 0.65 (rare spaces)

**Board Composition (40 spaces):**
- Hay Spaces: 14 (35% of harvest spaces)
- Grain Spaces: 10 (25% of harvest spaces)
- Fruit Spaces: 12 (30% of harvest spaces)
- Livestock Spaces: 4 (10% of harvest spaces)

### Equipment Assets (Tractors, Harvesters)
Equipment doesn't generate direct income but enables harvesting:
- Base_Productivity = $0 (direct earnings)
- Value comes from enabling other assets to harvest

---

## 2. Competition Factor

**Reduces earnings when many players own similar assets:**
```
Competition_Factor = 1 / (1 + Competition_Ratio)
Competition_Ratio = Total_Similar_Assets / (Player_Count × 10)
```

**Examples:**
- 0 similar assets: Competition_Factor = 1.0 (no discount)
- 20 similar assets in 4-player game: Competition_Factor = 0.67 (33% discount)
- 50 similar assets in 4-player game: Competition_Factor = 0.5 (50% discount)

**Minimum Competition Factor:** 0.3 (earnings never reduced below 30%)

---

## 3. Time Value Multiplier

**Assets become more valuable as the game progresses:**
```
Time_Value = 0.3 + (Turns_Remaining / 100 × 0.8)
```

**Time Value Range:**
- Early game (90 turns left): 1.02× (assets very valuable)
- Mid game (50 turns left): 0.7× (moderate value)
- Late game (10 turns left): 0.38× (assets lose value rapidly)

---

## 4. Scarcity Multiplier

**OTB card rarity creates premium pricing:**
```
Availability_Ratio = OTB_Cards_In_Deck / Total_OTB_Cards
Scarcity_Factor = (1 - Availability_Ratio) × 0.2
Scarcity_Multiplier = 1 + Scarcity_Factor
```

**OTB Deck Composition (28 total cards):**
- Hay: 5 cards (17.9% availability) → Scarcity_Multiplier = 1.064
- Grain: 5 cards (17.9% availability) → Scarcity_Multiplier = 1.064
- Fruit: 6 cards (21.4% availability) → Scarcity_Multiplier = 1.137
- Cows: 6 cards (21.4% availability) → Scarcity_Multiplier = 1.137
- Tractor: 3 cards (10.7% availability) → Scarcity_Multiplier = 1.157
- Harvester: 3 cards (10.7% availability) → Scarcity_Multiplier = 1.157

---

## 5. Operating Expense Risk

**Card-based expenses and income opportunities:**
```
Expected_Net_Impact = Σ(Expense × Probability) - Σ(Income × Probability)
Net_Impact_Ratio = Expected_Net_Impact / Expected_Base_Earnings
Risk_Factor = (1 - Total_Probability) + (Total_Probability × (1 - Net_Impact_Ratio))
```

### Operating Expense Cards (20-card deck)
- **Grain-specific**: Fertilizer Bill ($100/acre, 2 cards), Wire Worm ($100/acre, 1 card)
- **Equipment-specific**: Custom Hire ($2,000 penalty if no equipment, 2 cards each)
- **General**: Equipment Breakdown ($500, 2 cards), Equipment in Shop ($1,000, ? cards)
- **Flat fees**: Fuel ($1,000), Electric ($500), Parts ($500), etc.

### Farmer's Fate Cards (25-card deck)
- **Tractor Hire Bill**: $3,000 penalty if no Tractor (2 cards)
- **Harvester Custom Hire**: +$2,000 income from each player without Harvester (1 card)

**Risk Factor Range:** 0.2 - 1.5 (can increase or decrease FMV)

---

## 6. Equipment Usage Probability

**Equipment only provides value when actually used:**
```
Base_Usage = 1 / Player_Count
Equipment_Bonus = 1.1 (versatility advantage)
Position_Penalty = 1 - (Player_Index × 0.1)
Usage_Probability = Base_Usage × Equipment_Bonus × Position_Penalty
```

**Usage Probability Range:** 0.4 - 1.0
- First player in turn order: ~90% usage probability
- Last player in turn order: ~40% usage probability

---

## Real Gameplay Examples

### Example 1: 10 Grain Acres (4-player game, turn 25)
```
Game State: 75 turns remaining, 4 players, mid-game competition

Base Earnings: $2,000/acre × 10 acres × 10.8 expected harvests = $216,000
Competition Factor: 0.8 (moderate market saturation)
Time Value: 0.9 (75 turns remaining)
Scarcity: 1.064 (5 Grain OTB cards)
OpEx Risk: 0.85 (Fertilizer/Wire Worm expenses)
Equipment Usage: 1.0 (not equipment-dependent)

FMV = $216,000 × 0.8 × 0.9 × 1.064 × 0.85 × 1.0 = $132,000
Suggested Price: $118,800 (10% conservative discount)
```

### Example 2: Harvester (4-player game, turn 25)
```
Game State: 75 turns remaining, 4 players, 3 without harvesters

Base Earnings: $0 (equipment doesn't harvest directly)
Competition Factor: 1.0 (equipment scarcity)
Time Value: 0.9 (75 turns remaining)
Scarcity: 1.157 (3 Harvester OTB cards)
OpEx Risk: 1.15 (Custom Hire income opportunity)
Equipment Usage: 0.6 (competition for harvest cards)

FMV = $0 × 1.0 × 0.9 × 1.157 × 1.15 × 0.6 = $0
Wait - equipment has no base earnings! This needs adjustment...

Actual Equipment FMV = Equipment_Cost × Usage_Probability × Risk_Adjustment
Equipment_Cost = $25,000 (typical marketplace price)
FMV = $25,000 × 0.6 × 1.15 = $17,250
```

### Example 3: OTB Exercise Decision (10 Hay Acres)
```
OTB Cost: $10,000
FMV if exercised: $85,000 (calculated as above)
Net Profit: $75,000

Decision: EXERCISE - FMV significantly exceeds OTB cost
```

---

## Code Implementation

### Core Files
- **`packages/backend/src/services/market-analysis.service.ts`**: Complete FMV calculation engine
- **`packages/backend/src/services/marketplace.service.ts`**: FMV analysis API endpoints
- **`packages/backend/src/services/ai/strategies/cautious-farmer-strategy.ts`**: AI integration

### Key Methods
- `analyzeMarketValue()`: Main FMV calculation entry point
- `getAssetProductivityValue()`: Core productivity calculation
- `calculateCompetitionFactor()`: Market saturation analysis
- `getTimeValueMultiplier()`: Game progression adjustment
- `calculateOtbScarcityMultiplier()`: OTB card rarity premium
- `calculateOperatingExpenseRisk()`: Card-based risk/reward analysis
- `calculateEquipmentUsageProbability()`: Equipment utilization modeling

### Integration Points
- **Marketplace Modal**: Shows FMV for OTB exercise decisions
- **AI Strategies**: Use FMV for buy/sell decisions
- **Backend API**: `analyzeOtbCardFMV()` and `analyzeMarketValue()` endpoints

---

## Strategic Implications

### For Players
- **OTB Exercise**: Compare FMV vs. OTB cost to decide exercise timing
- **Market Timing**: Assets more valuable early game, sell before time value drops
- **Equipment Investment**: Consider both utility and card-based income risks
- **Competition Awareness**: Assets lose value as more players enter the market

### For AI
- **Conservative Strategy**: Only buy when FMV significantly exceeds asking price
- **Risk Assessment**: Factor in operating expense exposure
- **Equipment Valuation**: Account for usage probability and custom hire opportunities

### Economic Realism
- **Time Value**: Assets depreciate as game progresses (realistic)
- **Scarcity**: Rare OTB cards command premium (market dynamics)
- **Risk Premium**: Operating expenses create uncertainty (farming reality)
- **Competition**: Oversupply reduces prices (supply/demand)

---

## Future Enhancements

### Potential Adjustments
- **Board Analysis**: Dynamic harvest space calculation based on actual board state
- **Player Position**: More sophisticated turn order modeling
- **Card Synergies**: Combined effects of multiple cards
- **Market History**: Price trends based on transaction history
- **Player Strategy**: Adjust FMV based on observed player behavior

### Balancing Considerations
- **Time Value Curve**: Adjust the 100-turn baseline as needed
- **Risk Premiums**: Tune operating expense impact levels
- **Equipment Bonuses**: Balance custom hire income vs. penalty costs
- **Competition Sensitivity**: Adjust market saturation curves

---

## Testing & Validation

### Unit Tests
- Individual component calculations
- Edge cases (late game, high competition, equipment scarcity)
- Card probability distributions

### Integration Tests
- Full FMV calculations with realistic game states
- AI decision making with FMV inputs
- Frontend display accuracy

### Gameplay Validation
- Economic balance across different game scenarios
- Player satisfaction with FMV guidance
- AI competitiveness with enhanced valuation

---

## Quick Reference

### FMV Ranges (Typical Values)
- **Grain (10 acres)**: $80,000 - $150,000
- **Fruit (5 acres)**: $40,000 - $80,000
- **Hay (10 acres)**: $50,000 - $90,000
- **Cows (10 head)**: $15,000 - $30,000
- **Tractor**: $8,000 - $15,000
- **Harvester**: $12,000 - $20,000

### Key Multipliers
- **Early Game**: 0.9-1.1× (assets valuable)
- **Mid Game**: 0.6-0.8× (moderate value)
- **Late Game**: 0.3-0.5× (assets depreciated)
- **High Competition**: 0.3-0.7× (market saturated)
- **Equipment Risk**: 0.4-1.2× (depends on penalties/benefits)

This document serves as the authoritative reference for FMV calculations. All changes to the system should be documented here first, then implemented in code.
