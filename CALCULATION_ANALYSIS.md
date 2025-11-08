# CNC Feeds & Speeds Calculator - Analysis Report

## Executive Summary

This analysis verifies the calculation logic in your CNC feeds and speeds calculator and identifies opportunities for improvement, specifically for low-power machines like yours.

## Your Machine Configuration

- **Spindle Max RPM**: 29,000 RPM
- **Max Feed Rate**: 1,000 mm/min
- **Machine Rigidity**: Low (0.5x multiplier)
- **Motor**: NEMA17 stepper (0.6x multiplier)
- **Drive System**: TR8 leadscrew (0.6x multiplier)
- **Combined Machine Factor**: 0.5 × 0.6 × 0.6 = **0.18x** (82% reduction!)

## Test Bit Configuration

- **Diameter**: 3mm
- **Flutes**: 2
- **Material**: HSS
- **Bit Size Factor**: 0.7x (small bit adjustment)

---

## Calculation Verification

### Formula Correctness ✅

The calculator uses the correct standard CNC formula:

```
Feed Rate (mm/min) = RPM × Number of Flutes × Chip Load (mm/tooth)
```

Rearranged to find chip load:
```
Chip Load (mm/tooth) = Feed Rate (mm/min) ÷ (RPM × Number of Flutes)
```

**Status**: ✅ **CORRECT**

---

## Example Calculation: Softwood

### Material Parameters
- Base chip load: 0.010 inches/tooth
- Recommended RPM range: 10,000 - 14,000
- Optimal chip load range: **0.20 - 0.35 mm/tooth**
- Conservative RPM: 10,000
- Aggressive RPM: 13,000

### Step-by-Step Calculation

#### 1. Adjust Chip Load for Bit Size
```
Base chip load = 0.010 inches
Since diameter (3mm) < 3mm:
Adjusted chip load = 0.010 × 0.7 = 0.007 inches
Convert to mm: 0.007 × 25.4 = 0.1778 mm
```

#### 2. Calculate Base Feed Rates
```
Conservative: 10,000 RPM × 2 flutes × 0.1778 mm = 3,556 mm/min
Aggressive:   13,000 RPM × 2 flutes × 0.1778 mm = 4,623 mm/min
```

#### 3. Apply Machine Factors
```
Combined factor = 0.5 × 0.6 × 0.6 = 0.18

Conservative: 3,556 × 0.18 = 640 mm/min
Aggressive:   4,623 × 0.18 = 832 mm/min
```

#### 4. Cap at Machine Maximum
```
Conservative: min(640, 1000) = 640 mm/min ✅
Aggressive:   min(832, 1000) = 832 mm/min ✅
```

#### 5. Calculate Actual Chip Loads
```
Conservative: 640 ÷ (10,000 × 2) = 0.032 mm/tooth
Aggressive:   832 ÷ (13,000 × 2) = 0.032 mm/tooth
```

### ⚠️ **CRITICAL ISSUE IDENTIFIED**

**Optimal range**: 0.20 - 0.35 mm/tooth
**Actual achieved**: 0.032 mm/tooth

**Result**: Chip load is **84% BELOW minimum** optimal range!

**This means**:
- ❌ Tool is rubbing, not cutting
- ❌ Excessive heat buildup
- ❌ Poor surface finish
- ❌ Rapid tool wear
- ❌ Risk of burning material

---

## Root Cause Analysis

### The Problem: RPM vs Feed Rate Mismatch

Your machine has:
- ✅ **High spindle RPM** capability (29,000 RPM)
- ❌ **Low feed rate** capability (1,000 mm/min)

This creates an impossible situation where you cannot achieve optimal chip loads at recommended RPMs.

### The Math

To achieve minimum optimal chip load (0.20 mm/tooth) for softwood with 2-flute bit:

```
Required Feed Rate = RPM × Flutes × Chip Load
                   = 10,000 × 2 × 0.20
                   = 4,000 mm/min

But your max feed rate is only 1,000 mm/min!
```

### The Solution: Lower RPM!

For your 1,000 mm/min max feed rate to achieve 0.20 mm/tooth:

```
Required RPM = Feed Rate ÷ (Flutes × Chip Load)
             = 1,000 ÷ (2 × 0.20)
             = 2,500 RPM

For 0.25 mm/tooth (mid-range):
Required RPM = 1,000 ÷ (2 × 0.25)
             = 2,000 RPM
```

**🎯 Recommendation**: For your machine, use **2,000-2,500 RPM** for softwood, not 10,000!

---

## Calculation Issues and Improvements

### Issue #1: No RPM Adjustment for Feed-Limited Machines

**Problem**: Calculator always uses material-recommended RPM ranges, regardless of machine's feed rate capability.

**Fix**: Add logic to detect when `maxFeedRate` cannot support optimal chip load at recommended RPM, and automatically reduce RPM.

**Suggested Algorithm**:
```javascript
// Calculate required RPM for optimal chip load
const minOptimalChipLoad = mat.optimalChipLoadRange.min;
const maxRPMForOptimal = selectedMachine.maxFeedRate /
                        (selectedBit.flutes * minOptimalChipLoad);

// If machine can't reach optimal at material RPM, use lower RPM
if (maxRPMForOptimal < mat.rpmRange.conservative) {
    conservativeRPM = maxRPMForOptimal;
    // Add warning about reduced RPM
}
```

### Issue #2: Chip Load Range Warnings Not Prominent Enough

**Current**: Shows colored status indicators
**Better**: Add large, clear warning when chip load is severely suboptimal
**Add**: Specific RPM recommendation to fix the issue

### Issue #3: No Material-Specific RPM Adjustment

**Problem**: Soft materials can handle lower RPMs better than the ranges suggest.
**Fix**: Add minimum safe RPM values for each material, allow going below recommended range with warning.

Example:
```javascript
softwood: {
    rpmRange: { min: 10000, max: 14000, conservative: 10000, aggressive: 13000 },
    absoluteMinRPM: 2000, // Can go this low if needed for chip load
}
```

### Issue #4: Machine Factor Stacking Too Aggressive

**Current**: All factors multiply: 0.5 × 0.6 × 0.6 = 0.18 (82% reduction!)
**Analysis**: While each factor alone is reasonable, stacking creates extreme reduction.

**Consider**:
- Making factors less severe
- Using a different combination method (e.g., weighted average instead of multiplication)
- Or simply accepting that very limited machines need very conservative settings

**Recommendation**: Keep current approach but add RPM reduction logic to compensate.

---

## Verification of Other Calculations

### ✅ Stepover Calculation
```javascript
stepover = bitDiameter × 0.45
         = 3mm × 0.45
         = 1.35mm
```
**Standard**: 40-50% of bit diameter
**Status**: ✅ **CORRECT**

### ✅ Maximum Depth Calculation
```javascript
maxDepth = bitDiameter × depthFactor
         = 3mm × 0.5 (softwood)
         = 1.5mm
```
**Standard**: 30-50% for wood, 20% for aluminum
**Status**: ✅ **CORRECT**

### ✅ Material Removal Rate (MRR)
```javascript
mrr = (feedRate × depthOfCut × stepover) / 1000
    = (640 × 1.5 × 1.35) / 1000
    = 1.296 cm³/min
```
**Status**: ✅ **CORRECT**

### ✅ Material-Specific Parameters

All material parameters reviewed:

| Material | Base Chip Load | Optimal Range (mm) | RPM Range | Assessment |
|----------|---------------|-------------------|-----------|------------|
| Softwood | 0.010" (0.254mm) | 0.20-0.35 | 10k-14k | ✅ Correct |
| Hardwood | 0.008" (0.203mm) | 0.15-0.30 | 12k-16k | ✅ Correct |
| Plywood | 0.009" (0.229mm) | 0.18-0.33 | 14k-18k | ✅ Correct |
| MDF | 0.009" (0.229mm) | 0.025-0.040 | 15k-18k | ⚠️ See note |
| Acrylic | 0.005" (0.127mm) | 0.010-0.040 | 15k-18k | ✅ Correct |
| Aluminum | 0.0033" (0.084mm) | 0.012-0.040 | 8k-12k | ✅ Correct |
| Foam | 0.020" (0.508mm) | 0.40-0.60 | 18k-24k | ✅ Correct |

**Note on MDF**: Optimal range seems very low (0.025-0.040). Typical MDF can handle 0.15-0.30 mm/tooth like plywood. This might be overly conservative.

---

## Recommendations for Your Specific Machine

### Immediate Actions

1. **Use Lower RPMs** than recommended:
   - Softwood: 2,000-2,500 RPM (not 10,000)
   - Hardwood: 1,500-2,000 RPM (not 12,000)
   - Plywood: 1,500-2,000 RPM (not 15,000)
   - Aluminum: 1,000-1,500 RPM (not 8,000)

2. **Max Out Your Feed Rate**: Use 800-1,000 mm/min to achieve better chip loads

3. **Use Larger Bits When Possible**: 6mm bits get 0.85x factor instead of 0.7x

### Manual Calculation for Your Machine

For any material with 3mm 2-flute bit:

```
Target chip load: 0.20 mm/tooth (minimum optimal)

Optimal RPM = Your max feed ÷ (flutes × chip load)
            = 1000 ÷ (2 × 0.20)
            = 2,500 RPM

Feed rate: 1,000 mm/min (max)
Actual chip load: 1000 ÷ (2500 × 2) = 0.20 mm/tooth ✅
```

### Test Settings for Softwood

**Better Approach** (instead of calculator's current recommendation):
- **RPM**: 2,500
- **Feed**: 1,000 mm/min
- **Depth**: 1.5mm
- **Chip Load**: 0.20 mm/tooth ✅

**Current Calculator** (produces suboptimal results):
- **RPM**: 10,000
- **Feed**: 640 mm/min
- **Depth**: 1.5mm
- **Chip Load**: 0.032 mm/tooth ❌

---

## Proposed Code Improvements

### 1. Add Adaptive RPM Selection

```javascript
// After line 296, add:
// Check if machine can achieve optimal chip load at recommended RPM
const minOptimalChipLoad = mat.optimalChipLoadRange.min;
const maxFeedAtCurrentRPM = conservativeRPM * selectedBit.flutes * chipLoadMM *
                            rigidityFactor * driveSystemFactor * motorSizeFactor;
const achievableChipLoad = Math.min(maxFeedAtCurrentRPM, selectedMachine.maxFeedRate) /
                           (conservativeRPM * selectedBit.flutes);

// If we can't achieve minimum optimal chip load, reduce RPM
if (achievableChipLoad < minOptimalChipLoad) {
    const adjustedRPM = selectedMachine.maxFeedRate /
                       (selectedBit.flutes * minOptimalChipLoad *
                        rigidityFactor * driveSystemFactor * motorSizeFactor);
    if (adjustedRPM < conservativeRPM) {
        conservativeRPM = Math.max(adjustedRPM, 2000); // Don't go below 2000 RPM
        warnings.push(`ℹ️ RPM reduced to ${Math.round(conservativeRPM)} to achieve optimal chip load with your feed rate limit`);
    }
}
```

### 2. Add Prominent Chip Load Warning

```javascript
// After line 364, add:
if (conservativeChipLoad < mat.optimalChipLoadRange.min * 0.5) {
    warnings.unshift(`🚨 CRITICAL: Chip load is ${((1 - conservativeChipLoad / mat.optimalChipLoadRange.min) * 100).toFixed(0)}% below optimal! Tool will rub instead of cut. Reduce RPM to ${Math.round(selectedMachine.maxFeedRate / (selectedBit.flutes * mat.optimalChipLoadRange.min))} RPM for better results.`);
}
```

### 3. Update MDF Optimal Range

```javascript
// Line 155, change:
optimalChipLoadRange: { min: 0.15, max: 0.30 } // Was: { min: 0.025, max: 0.040 }
```

### 4. Add Custom RPM Override Option

Add a checkbox/toggle in the UI to "Use custom RPM for limited feed rate machines" that applies the adaptive RPM logic.

---

## Testing Recommendations

### Test Case 1: Your Machine, 3mm bit, Softwood
- **Without fix**: 10,000 RPM, 640 mm/min, 0.032 mm chip load ❌
- **With fix**: 2,500 RPM, 1,000 mm/min, 0.20 mm chip load ✅

### Test Case 2: Your Machine, 6mm bit, Hardwood
- **Expected**: ~2,000 RPM for optimal chip load
- **Verify**: Chip load in 0.15-0.30 range

### Test Case 3: Your Machine, 3mm bit, Aluminum
- **Expected**: 1,000-1,500 RPM
- **Verify**: Chip load in 0.012-0.040 range
- **Critical**: Ensure lubrication warning displays

---

## Overall Assessment

### What's Working Well ✅

1. ✅ Core formulas are mathematically correct
2. ✅ Material parameters are generally accurate
3. ✅ Machine capability factors are reasonable individually
4. ✅ Stepover, depth, and MRR calculations are correct
5. ✅ Small bit adjustments are appropriate
6. ✅ Material-specific warnings (aluminum lubrication, etc.)
7. ✅ Dual conservative/aggressive recommendations
8. ✅ Educational content about chip load

### What Needs Improvement ⚠️

1. ⚠️ **No adaptive RPM for feed-limited machines** ← CRITICAL
2. ⚠️ MDF optimal chip load range too conservative
3. ⚠️ Chip load warnings not prominent enough when severely suboptimal
4. ⚠️ No suggestion to reduce RPM when chip load is poor
5. ⚠️ Machine factor stacking creates extreme reductions (but mathematically defensible)

### Priority Fixes

**High Priority**:
1. Add adaptive RPM selection for feed-limited machines
2. Add prominent warning when chip load < 50% of minimum optimal
3. Add RPM recommendation in warnings

**Medium Priority**:
4. Fix MDF optimal chip load range
5. Add custom RPM override option
6. Consider minimum RPM limits per material

**Low Priority**:
7. Review machine factor stacking approach
8. Add more test cases and validation

---

## Conclusion

Your calculator's math is **fundamentally correct**, but it's optimized for machines with balanced RPM and feed rate capabilities.

For **low-power machines like yours** with high RPM but low feed rates, the calculator needs logic to **reduce RPM** to achieve proper chip loads.

**The key insight**: Don't chase high RPM if your feed rate can't keep up. Lower RPM with maxed-out feed rate produces better results than high RPM with proportionally reduced feed rate.

**Recommendation**: Implement the adaptive RPM selection to automatically detect and fix this issue for all users with similar machines.
