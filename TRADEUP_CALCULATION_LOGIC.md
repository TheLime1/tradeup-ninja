# CS:GO Tradeup Calculation Logic Documentation

This document comprehensively explains all the mathematical and logical operations used in the Tradeup Ninja application for calculating CS:GO weapon trade-ups.

---

## Table of Contents
1. [Overview](#overview)
2. [Core Concepts](#core-concepts)
3. [Float Calculation](#float-calculation)
4. [Outcome Calculation](#outcome-calculation)
5. [Price Calculation](#price-calculation)
6. [Trade-Up Summary Calculation](#trade-up-summary-calculation)
7. [Simulation Logic](#simulation-logic)
8. [Helper Functions](#helper-functions)

---

## Overview

A CS:GO trade-up contract allows players to exchange 10 weapons of the same rarity for 1 weapon of the next higher rarity. The outcome weapon is randomly selected from the next rarity tier of the collections represented in the input items.

### Key Files
- `src/app/tradeup-search/tradeup-search.utils.ts` - Main trade-up calculation logic
- `src/app/tradeup-search/tradeup-shared-utils.ts` - Shared utility functions (price, float indexing)
- `src/app/tradeup-simulation/tradeup-simulation.worker.ts` - Simulation worker logic
- `server/utils.ts` - Server-side utilities for price determination

---

## Core Concepts

### Weapon Rarity
Weapons in CS:GO have 6 rarity levels:
```typescript
enum WeaponRarity {
  'Consumer' = 1,      // White
  'Industrial' = 2,    // Light Blue
  'MilSpec' = 3,       // Blue
  'Restricted' = 4,    // Purple
  'Classified' = 5,    // Pink/Magenta
  'Covert' = 6         // Red
}
```

### Weapon Wear (Exterior)
Each weapon skin has a float value (0 to 1) that determines its exterior condition:
```typescript
enum WeaponWear {
  'FactoryNew' = 0,      // 0.00 - 0.07
  'MinimalWear' = 1,     // 0.07 - 0.15
  'FieldTested' = 2,     // 0.15 - 0.38
  'WellWorn' = 3,        // 0.38 - 0.45
  'BattleScarred' = 4    // 0.45 - 1.00
}
```

---

## Float Calculation

### 1. Float Index Determination

**Location:** `src/app/tradeup-search/tradeup-shared-utils.ts`

```typescript
function getFloatIndexForPrice(float: number): number
```

**Logic:**
Converts a float value to a wear index (0-4) for price lookup:

```
float (fixed to 7 decimals)
├─ if < 0.07  → 0 (Factory New)
├─ if < 0.15  → 1 (Minimal Wear)
├─ if < 0.38  → 2 (Field-Tested)
├─ if < 0.45  → 3 (Well-Worn)
└─ if ≥ 0.45  → 4 (Battle-Scarred)
```

**Important Notes:**
- Float is fixed to 7 decimal places to avoid precision errors
- Example: `0.0699996` should be treated as FN (0.07), not MW
- Returns `-1` if float is not provided (invalid case)

### 2. Average Float Calculation

**Location:** `src/app/tradeup-search/tradeup-search.utils.ts`

```typescript
function getAverageFloat(items: TradeupItemWithFloat[]): number
```

**Formula:**
```
averageFloat = sum(item.float for item in items) / count(items)
```

**Implementation:**
Uses lodash's `meanBy` function to calculate the arithmetic mean of all input item floats.

### 3. Outcome Float Calculation

**Location:** `src/app/tradeup-search/tradeup-search.utils.ts`

```typescript
function getOutcomeFloat(avgFloat: number, min: number, max: number): number
```

**Formula:**
```
outcomeFloat = avgFloat × (max - min) + min
```

**Explanation:**
- `avgFloat`: Average float of all 10 input items
- `min`: Minimum float value for the outcome skin (from skin data)
- `max`: Maximum float value for the outcome skin (from skin data)
- The formula scales the average input float to the outcome skin's float range

**Example:**
```
Input average float: 0.05
Outcome skin range: [0.10, 0.50]
Result: 0.05 × (0.50 - 0.10) + 0.10 = 0.05 × 0.40 + 0.10 = 0.12
```

### 4. Input Item Float by Condition

**Location:** `src/app/tradeup-search/tradeup-shared-utils.ts`

```typescript
function getInputItemFloatByCondition(
  conditionIndex: WeaponWear, 
  skin: Weapon, 
  difficulty: number
): number
```

**Formula:**
```
currentRangeMinimum = max(skin.min, FloatRange[conditionIndex].min)
currentRangeMaximum = min(skin.max, FloatRange[conditionIndex].max)
addingValue = (currentRangeMaximum - currentRangeMinimum) × difficulty
inputFloat = currentRangeMinimum + addingValue
```

**Explanation:**
- Takes into account skin-specific float restrictions
- `difficulty` (0-1): Controls where in the range the float falls
  - 0 = Hardest to find (minimum float)
  - 1 = Easiest to find (maximum float)
  - 0.5 = Middle of the range

**Example:**
```
Condition: Factory New (0.00 - 0.07)
Skin range: [0.06, 0.80]
Difficulty: 0.5

currentRangeMinimum = max(0.06, 0.00) = 0.06
currentRangeMaximum = min(0.80, 0.07) = 0.07
addingValue = (0.07 - 0.06) × 0.5 = 0.005
inputFloat = 0.06 + 0.005 = 0.065
```

---

## Outcome Calculation

### 1. Get Possible Outcomes

**Location:** `src/app/tradeup-search/tradeup-search.utils.ts`

```typescript
function getOutcomes(
  items: TradeupItemWithFloat[],
  structCollections: StructuredCollectionWithItems[]
): TradeupOutcome[]
```

**Process:**
1. Calculate average float of all input items
2. For each input item:
   - Find its collection
   - Get all skins from the next rarity tier in that collection
   - Calculate outcome float for each possible outcome
   - Add to outcomes list
3. Consolidate duplicate outcomes and count occurrences
4. Calculate odds for each unique outcome

**Algorithm:**
```
allOutputs = []
avgFloat = calculateAverage(inputItems)

for each inputItem in inputItems:
  collection = findCollection(inputItem)
  nextRaritySkins = collection.items[inputItem.rarity + 1]
  
  for each outputSkin in nextRaritySkins:
    outcomeFloat = getOutcomeFloat(avgFloat, outputSkin.min, outputSkin.max)
    allOutputs.add({ item: outputSkin, float: outcomeFloat, outputCount: 1 })

// Consolidate duplicates
uniqueOutputs = []
for each output in allOutputs:
  existing = find(uniqueOutputs, matches output by name, variation, and float)
  if existing:
    existing.outputCount++
  else:
    uniqueOutputs.add(output)

// Calculate odds
totalOutcomeCount = sum(output.outputCount for output in uniqueOutputs)
for each output in uniqueOutputs:
  output.odds = output.outputCount / totalOutcomeCount
```

### 2. Calculate Outcome Chance

**Location:** `src/app/tradeup-search/tradeup-search.utils.ts`

```typescript
function calculateOutcomeChance(
  outcomeItemCount: number, 
  totalOutcomeCount: number
): number
```

**Formula:**
```
outcomeChance = outcomeItemCount / totalOutcomeCount
```

**Explanation:**
- `outcomeItemCount`: Number of ways to get this specific outcome
- `totalOutcomeCount`: Total number of all possible outcomes
- Result is rounded to 4 decimal places (e.g., 0.1512 = 15.12%)

**Example:**
```
Input: 7 skins from Collection A, 3 skins from Collection B
Collection A has 5 possible outcomes at next rarity
Collection B has 3 possible outcomes at next rarity

Total outcomes = 7 × 5 + 3 × 3 = 35 + 9 = 44

If a specific skin appears 7 times (from Collection A):
Chance = 7 / 44 = 0.1591 (15.91%)
```

---

## Price Calculation

### 1. Get Price for Item

**Location:** `src/app/tradeup-search/tradeup-shared-utils.ts`

```typescript
function getPrice(
  item: Weapon, 
  float: number, 
  isStattrak?: boolean, 
  withoutTax?: boolean
): number
```

**Process:**
1. Convert float to price index (0-4)
2. Select appropriate price array (normal or StatTrak™)
3. Get price at the index
4. Apply Steam tax if `withoutTax = true`

**Formula (with tax removal):**
```
priceIndex = getFloatIndexForPrice(float)
basePrice = isStattrak ? item.price.stattrak[priceIndex] : item.price.normal[priceIndex]
finalPrice = withoutTax ? basePrice × 0.87 : basePrice
```

**Steam Tax:**
- Steam takes 13% commission (5% to Steam, 8% to CS:GO)
- Price after tax = `basePrice × 0.87` (keeping 87%)

### 2. Get Safe Price

**Location:** `server/utils.ts`

```typescript
function getPrice(prices: Prices): number
```

**Logic:**
Determines the most reliable price to use based on market stability:

```
if prices.unstable_reason exists:
  if unstable_reason == 'LOW_SALES_WEEK':
    return prices.median / prices.safe_ts.last_7d > 1.5 
           ? prices.avg 
           : prices.safe_ts.last_7d
  
  if unstable_reason == 'LOW_SALES_MONTH':
    weekPriceDiff = prices.median / prices.safe_ts.last_7d
    monthPriceDiff = prices.median / prices.safe_ts.last_30d
    
    if weekPriceDiff > 1.5 AND monthPriceDiff > 1.5:
      return prices.avg
    else:
      return weekPriceDiff > monthPriceDiff 
             ? prices.safe_ts.last_30d 
             : prices.safe_ts.last_7d
  
  default:
    return prices.safe

else (stable prices):
  return prices.median / prices.safe > 1.5 
         ? prices.avg 
         : prices.safe
```

**Explanation:**
- `safe`: Price calculated to resist market manipulation
- `avg`: Average price
- `median`: Median price
- `safe_ts.last_7d`: Safe price over last 7 days
- `safe_ts.last_30d`: Safe price over last 30 days

The 1.5x threshold indicates potential market manipulation or insufficient data.

---

## Trade-Up Summary Calculation

**Location:** `src/app/tradeup-search/tradeup-search.utils.ts`

```typescript
function calculateTradeUp(
  items: TradeupItemWithFloat[],
  stattrak: boolean,
  structCollections: StructuredCollectionWithItems[],
  withoutSteamTax: boolean,
  ignoreEmptyPrice: boolean,
  customOutcome?: TradeupOutcome
): TradeupSummary
```

### 1. Calculate Total Cost

```
totalCost = 0
for each item in inputItems:
  itemPrice = getPrice(item, item.float, stattrak)
  totalCost += itemPrice
```

### 2. Calculate Expected Value (EV)

**Formula:**
```
EV = Σ(outcomePrice × outcomeOdds) for all outcomes
```

**Implementation:**
```
expectedValue = 0
for each outcome in outcomes:
  price = getPrice(outcome.item, outcome.float, stattrak, withoutSteamTax)
  expectedValue += price × outcome.odds
```

**Example:**
```
Outcome A: $10, 60% chance → $10 × 0.60 = $6.00
Outcome B: $5, 30% chance  → $5 × 0.30 = $1.50
Outcome C: $2, 10% chance  → $2 × 0.10 = $0.20
EV = $6.00 + $1.50 + $0.20 = $7.70
```

### 3. Calculate Profit

**Formulas:**
```
profit = expectedValue - totalCost
profitPercentage = profit / totalCost
```

**Example:**
```
Total cost: $5.00
Expected value: $7.70
Profit: $7.70 - $5.00 = $2.70
Profit percentage: $2.70 / $5.00 = 0.54 (54%)
```

### 4. Track Most Expensive and Cheapest Outcomes

**Process:**
```
mostExpensivePrize = 0
cheapestPrize = MAX_VALUE

for each outcome in outcomes:
  price = getPrice(outcome)
  
  if price > mostExpensivePrize:
    mostExpensivePrize = price
    mostExpensiveOutcomeItem = outcome.item
    mostExpensiveChance = outcome.odds
    mostExpensiveItemFloat = outcome.float
  
  if price < cheapestPrize:
    cheapestPrize = price
    cheapestOutcomeItem = outcome.item
    cheapestChance = outcome.odds
    cheapestItemFloat = outcome.float
```

### 5. Calculate Outcome Summary

**Location:** `src/app/tradeup-search/tradeup.model.ts`

```typescript
class OutcomeSummary {
  outcomeItemCountBelowCost: number
  oddsBelowCost: number
  successOdds: number
}
```

**Logic:**
```
oddsBelowCost = 0
outcomeItemCountBelowCost = 0

for each outcome in outcomes:
  outcomePrice = getPrice(outcome.item, outcome.float, stattrak)
  outcomePriceForCompare = calculateWithoutTax 
                           ? outcomePrice × 0.87 
                           : outcomePrice
  
  if tradeupCost > outcomePriceForCompare:
    oddsBelowCost += outcome.odds
    outcomeItemCountBelowCost++

successOdds = 1 - oddsBelowCost
```

**Explanation:**
- `oddsBelowCost`: Probability of getting an outcome that loses money
- `successOdds`: Probability of getting an outcome that profits or breaks even
- `outcomeItemCountBelowCost`: Number of distinct outcomes that lose money

**Example:**
```
Trade-up cost: $5.00

Outcome A: $10, 20% → Profit
Outcome B: $7, 30%  → Profit
Outcome C: $3, 35%  → Loss
Outcome D: $2, 15%  → Loss

oddsBelowCost = 35% + 15% = 50%
successOdds = 1 - 0.50 = 0.50 (50%)
outcomeItemCountBelowCost = 2
```

---

## Simulation Logic

**Location:** `src/app/tradeup-simulation/tradeup-simulation.worker.ts`

### Simulation Process

```typescript
function simulateTradeups(settings: SimulationSettings)
```

**Algorithm:**
```
// Initialize
simulationSummary = new SimulationSummary()

// Create probability map
probability = {}
for i, outcome in settings.outcomes:
  probability[i] = outcome.odds

// Run simulations
simulatedItems = []
for i from 0 to settings.count:
  // Weighted random selection
  outcomeIndex = weightedRandom(probability)
  outcome = settings.outcomes[outcomeIndex]
  
  simulatedItems.add(outcome)
  calculateBasicSummary(outcome)

// Calculate final summary
calculateFinalSummary()
```

### Basic Summary Calculation

```typescript
function calculateBasicSummary(outcome: TradeupOutcome)
```

**Logic:**
```
outcomePrize = getPrice(outcome.item, outcome.float, stattrak, withoutTax=true)

if outcomePrize >= tradeupCost:
  simulationSummary.successfulOutcomes++

simulationSummary.totalSpent += tradeupCost
simulationSummary.totalReceived += outcomePrize
```

### Final Summary Calculation

```typescript
function calculateFinalSummary()
```

**Formulas:**
```
totalProfit = totalReceived - totalSpent
successRate = successfulOutcomes / totalSimulations
```

**Example:**
```
1000 simulations
Trade-up cost: $5.00 per trade-up

Results:
- 550 profitable outcomes
- Total spent: $5,000
- Total received: $6,200

successRate = 550 / 1000 = 0.55 (55%)
totalProfit = $6,200 - $5,000 = $1,200
```

### Weighted Random Selection

**Location:** `src/app/tradeup-search/tradeup-shared-utils.ts`

```typescript
function weightedRandom(prob: {[key: string]: number}): unknown
```

**Algorithm:**
```
sum = 0
rand = random(0, 1)  // Random number between 0 and 1

for key, probability in prob:
  sum += probability
  if rand <= sum:
    return key
```

**Example:**
```
Probabilities:
{
  0: 0.20,  // 20% chance
  1: 0.30,  // 30% chance
  2: 0.50   // 50% chance
}

Random = 0.65

Iteration 1: sum = 0.20, 0.65 > 0.20 → continue
Iteration 2: sum = 0.50, 0.65 > 0.50 → continue
Iteration 3: sum = 1.00, 0.65 ≤ 1.00 → return 2
```

---

## Helper Functions

### 1. Get Volume

**Location:** `src/app/tradeup-search/tradeup-shared-utils.ts`

```typescript
function getVolume(item: Weapon, float: number, isStattrak?: boolean): number
```

**Logic:**
```
volumeIndex = getFloatIndexForPrice(float)
return isStattrak ? item.volume.stattrak[volumeIndex] 
                  : item.volume.normal[volumeIndex]
```

Volume represents the number of items sold in the last 24 hours on the Steam Community Market.

### 2. Get Rarity Name

**Location:** `src/app/tradeup-search/tradeup-shared-utils.ts`

```typescript
function getRarityName(rarity: number): string
```

**Mapping:**
```
1 → 'Consumer'
2 → 'Industrial'
3 → 'Mil-spec'
4 → 'Restricted'
5 → 'Classified'
6 → 'Covert'
default → '???'
```

### 3. Convert Time

**Location:** `src/app/tradeup-search/tradeup-shared-utils.ts`

```typescript
function msToMinutesSeconds(ms: number): string
```

**Formula:**
```
minutes = floor(ms / 60000)
seconds = round((ms % 60000) / 1000)
result = "{minutes}:{seconds < 10 ? '0' : ''}{seconds}"
```

### 4. Encode/Decode Skin ID

**Location:** `server/utils.ts`

```typescript
function encodeSkinToId(name: string, wearIndex: number, stattrak?: boolean): string
function decodeIdToSkin(id: string): DecodedSkinInfo
```

**Encode Logic:**
```
content = stattrak ? "ST--{wearIndex}--{name}" : "{wearIndex}--{name}"
id = base64_encode(content)
```

**Decode Logic:**
```
content = base64_decode(id).split("--")
if content.length == 3:
  isStattrak = true
  content = content[1:]  // Remove 'ST' element
else:
  isStattrak = false

return {
  stattrak: isStattrak,
  wearIndex: int(content[0]),
  name: content[1]
}
```

**Example:**
```
Encode: "AK-47 | Redline", wearIndex=0, stattrak=true
→ "ST--0--AK-47 | Redline"
→ Base64: "U1QtLTAtLUFLLTQ3IHwgUmVkbGluZQ=="

Decode: "U1QtLTAtLUFLLTQ3IHwgUmVkbGluZQ=="
→ "ST--0--AK-47 | Redline"
→ Split: ["ST", "0", "AK-47 | Redline"]
→ { stattrak: true, wearIndex: 0, name: "AK-47 | Redline" }
```

---

## Mathematical Summary

### Complete Trade-Up Calculation Flow

```
1. INPUT PROCESSING
   ├─ For each of 10 input items:
   │  ├─ Get float value
   │  ├─ Get item price based on float
   │  └─ Sum total cost

2. AVERAGE FLOAT
   └─ avgFloat = Σ(item.float) / 10

3. OUTCOME GENERATION
   ├─ For each input item:
   │  ├─ Find collection
   │  ├─ Get next rarity skins
   │  └─ For each possible outcome:
   │     └─ outcomeFloat = avgFloat × (skinMax - skinMin) + skinMin
   │
   └─ Consolidate duplicates and count occurrences

4. PROBABILITY CALCULATION
   └─ For each unique outcome:
      └─ odds = outcomeCount / totalOutcomeCount

5. EXPECTED VALUE
   └─ EV = Σ(outcomePrice × outcomeOdds)

6. PROFIT CALCULATION
   ├─ profit = EV - totalCost
   └─ profitPercentage = profit / totalCost

7. SUCCESS ODDS
   ├─ oddsBelowCost = Σ(odds where outcomePrice < totalCost)
   └─ successOdds = 1 - oddsBelowCost

8. SUMMARY
   └─ Return {
        cost,
        expectedValue,
        profit,
        profitPercentage,
        successOdds,
        outcomes: [ { item, float, odds, price } ],
        mostExpensive: { item, price, odds, float },
        cheapest: { item, price, odds, float }
      }
```

---

## Key Formulas Reference

### Float Calculations
```
avgFloat = Σ(inputFloat) / count(inputs)
outcomeFloat = avgFloat × (skinMax - skinMin) + skinMin
inputFloat = rangeMin + (rangeMax - rangeMin) × difficulty
```

### Probability Calculations
```
outcomeOdds = outcomeCount / totalOutcomeCount
oddsBelowCost = Σ(odds where price < cost)
successOdds = 1 - oddsBelowCost
```

### Financial Calculations
```
expectedValue = Σ(outcomePrice × outcomeOdds)
profit = expectedValue - totalCost
profitPercentage = profit / totalCost
priceAfterTax = price × 0.87
```

### Simulation Calculations
```
successRate = successfulOutcomes / totalSimulations
totalProfit = totalReceived - totalSpent
```

---

## Important Constants

```typescript
// Steam tax (5% Steam + 8% CS:GO = 13% total)
STEAM_TAX_MULTIPLIER = 0.87

// Float ranges by wear
FLOAT_RANGES = {
  FactoryNew: [0.00, 0.07],
  MinimalWear: [0.07, 0.15],
  FieldTested: [0.15, 0.38],
  WellWorn: [0.38, 0.45],
  BattleScarred: [0.45, 1.00]
}

// Price stability threshold
PRICE_STABILITY_THRESHOLD = 1.5

// Float precision for calculations
FLOAT_PRECISION = 7  // decimal places
```

---

## Edge Cases and Special Handling

### 1. Missing Price Data
```
if price is undefined or null:
  if ignoreEmptyPrice:
    price = 0
    item.noSCMPrice = true
  else:
    return empty TradeupSummary (invalid)
```

### 2. Skin Float Restrictions
```
if skin.min > wearRange.min:
  useMin = skin.min
else:
  useMin = wearRange.min

if skin.max < wearRange.max:
  useMax = skin.max
else:
  useMax = wearRange.max
```

### 3. Skin Variations
```
// Handle skins with variations (e.g., Doppler Phases, Emerald)
if skinHasVariation:
  matchBy = (name AND variation)
else:
  matchBy = (name only)
```

### 4. StatTrak™ Handling
```
if stattrak required:
  if not weapon.stattrak:
    skip weapon  // Cannot trade-up non-stattrak to get stattrak
  else:
    use weapon.price.stattrak[wearIndex]
else:
  use weapon.price.normal[wearIndex]
```

---

## Performance Considerations

1. **Web Workers**: Trade-up search runs in a separate thread to avoid blocking the UI
2. **Caching**: Price data is cached and synced daily
3. **IndexedDB**: Trade-up results are stored in browser's IndexedDB
4. **Lazy Loading**: Collections and items are loaded on-demand
5. **Debouncing**: Search inputs are debounced to reduce calculations

---

## Validation Rules

### Input Validation
```
✓ Exactly 10 items required for trade-up
✓ All items must be same rarity
✓ All items must be same type (all normal or all StatTrak™)
✓ Each item must have valid float (0 ≤ float ≤ 1)
✓ Each item must have price data (unless ignoreEmptyPrice is true)
```

### Outcome Validation
```
✓ Outcome rarity must be exactly inputRarity + 1
✓ Outcome must be from collections present in input items
✓ Outcome float must be within skin's min/max range
```

### Settings Validation
```
✓ 0 ≤ difficulty ≤ 1
✓ maxCost > 0
✓ 0 ≤ minSuccess ≤ maxSuccess ≤ 100
✓ minVolume ≥ 0
✓ minEVPercent can be any value (including negative)
```

---

## References

### Source Files
- Trade-up calculation: `src/app/tradeup-search/tradeup-search.utils.ts`
- Shared utilities: `src/app/tradeup-search/tradeup-shared-utils.ts`
- Simulation worker: `src/app/tradeup-simulation/tradeup-simulation.worker.ts`
- Models: `src/app/tradeup-search/tradeup.model.ts`
- Server utilities: `server/utils.ts`

### External Dependencies
- Lodash: Used for array operations (meanBy, cloneDeep)
- RxJS: Used for reactive data streams
- IndexedDB: Used for persistent storage

---

*This documentation represents the complete trade-up calculation logic as of the current codebase version.*
