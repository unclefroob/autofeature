---
adapted-from: source/review-design-checklist.md
changes: |
  - Removed gstack-diff-scope bin reference — detect frontend files directly via git diff
  - Kept all 5 categories and confidence tiers unchanged
  - Added React Native specific checks (StyleSheet vs inline styles, platform-specific)
  - Added Swift/SwiftUI specific design checks
  - Kept output format and classification unchanged
  - Kept suppressions unchanged
status: ADAPTED
---

# Design Review Checklist

Only runs when the diff touches frontend files (React, React Native, Swift UI, Kotlin XML layouts, web views).

**Detect frontend scope:**
```bash
git diff origin/main --name-only | grep -E '\.(tsx|jsx|ts|js|css|scss|sass|less|html|swift|xml)$' | grep -v -E '(test|spec|__tests__|\.test\.|\.spec\.)' | head -5
```

If no frontend files changed: skip silently.

**DESIGN.md calibration:** If `DESIGN.md` or `design-system.md` exists, read it first. Patterns blessed there are NOT flagged.

---

## Confidence Tiers

- **[HIGH]** — Pattern-matchable. Definitive.
- **[MEDIUM]** — Heuristic. Some noise expected.
- **[LOW]** — Requires visual judgment. Present as "Possible — verify visually."

---

## Classification

**AUTO-FIX** (mechanical, HIGH confidence, no design judgment):
- `outline: none` → add `:focus-visible { outline: 2px solid currentColor; }`
- `!important` in new CSS → remove, fix specificity
- Body text `font-size` < 16px → bump to 16px
- React Native: inline style objects in list items → extract to `StyleSheet.create()`

**ASK** (design judgment required):
- All AI slop findings, typography, spacing, interaction states, DESIGN.md violations

**LOW confidence** → "Possible: [description]. Verify visually."

---

## Output

```
Design Review: N issues (X auto-fixed, Y need input, Z possible)

AUTO-FIXED:
- [file:line] Problem → fix applied

NEEDS INPUT:
- [file:line] Problem
  Fix: recommended fix

POSSIBLE (verify visually):
- [file:line] Possible issue
```

If clean: `Design Review: No issues found.`

---

## Category 1: AI Slop Detection — highest priority

Signs of AI-generated UI that a real designer wouldn't ship.

- **[MEDIUM]** Purple/violet/indigo gradients or blue-to-purple color schemes
  (`linear-gradient` with values `#6366f1`–`#8b5cf6` or similar)

- **[LOW]** 3-column feature grid: icon-in-colored-circle + bold title + 2-line description × 3

- **[LOW]** Icons in colored circles as decoration (`border-radius: 50%` + background color as icon container)

- **[HIGH]** Centered everything: >60% of text containers use `text-align: center`

- **[MEDIUM]** Uniform bubbly border-radius: >80% of elements use the same `border-radius ≥ 16px`

- **[MEDIUM]** Generic hero copy:
  - "Welcome to [X]"
  - "Unlock the power of..."
  - "Your all-in-one solution for..."
  - "Revolutionize your..."
  - "Streamline your workflow"

---

## Category 2: Typography

- **[HIGH]** Body text `font-size` < 16px (CSS) or < 14sp (Android) or < 14pt (iOS)

- **[HIGH]** More than 3 font families in the diff

- **[HIGH]** Heading hierarchy skipping levels (h1 → h3 without h2)

- **[HIGH]** Blacklisted fonts: Papyrus, Comic Sans, Lobster, Impact, Jokerman

---

## Category 3: Spacing & Layout

- **[MEDIUM]** Arbitrary spacing not on a 4px/8px scale (when DESIGN.md defines a scale)

- **[MEDIUM]** Fixed `width: NNNpx` on containers without `max-width` or responsive breakpoints

- **[MEDIUM]** Text containers without `max-width` → lines > 75 characters

- **[HIGH]** `!important` in new CSS rules

**React Native specific:**
- **[MEDIUM]** Inline style objects `style={{ margin: 8 }}` inside `FlatList` `renderItem` → creates new object on every render, impacts performance. Extract to `StyleSheet.create()`.
- **[MEDIUM]** `flex: 1` on a child without parent having defined height → invisible on some Android versions

**Swift/SwiftUI specific:**
- **[MEDIUM]** Hard-coded pixel values instead of Dynamic Type / scaled sizes
- **[MEDIUM]** Layout that doesn't account for safe areas (`safeAreaInsets`, `ignoresSafeArea` without justification)

---

## Category 4: Interaction States

- **[MEDIUM]** Interactive elements missing hover/focus states (buttons, links, inputs)

- **[HIGH]** `outline: none` or `outline: 0` without replacement focus indicator (accessibility violation)

- **[LOW]** Touch targets < 44px (iOS HIG) or < 48dp (Android Material) on interactive elements

**React Native specific:**
- **[MEDIUM]** `TouchableOpacity` / `Pressable` without `accessibilityLabel`
- **[HIGH]** `onPress` handler with no visual feedback (no `activeOpacity`, no press state)

**Swift/SwiftUI specific:**
- **[MEDIUM]** Interactive elements without `.accessibilityLabel()` modifier
- **[MEDIUM]** Missing `.disabled()` state styling when element can be disabled

---

## Category 5: DESIGN.md Violations (conditional)

Only applies when `DESIGN.md` or `design-system.md` exists.

- **[MEDIUM]** Colors not in the stated palette
- **[MEDIUM]** Fonts not in the stated typography list
- **[MEDIUM]** Spacing values outside the stated scale

---

## Category 6: Missing UI States

For any new component or screen, check that all states are handled:

- **[MEDIUM]** Loading state missing (shows nothing while fetching)
- **[MEDIUM]** Empty state missing (shows nothing when data is empty — "No results" is better than blank)
- **[MEDIUM]** Error state missing (silent failure on network error)
- **[LOW]** Partial/degraded state (some data loaded, some failed)

---

## Suppressions

Do NOT flag:
- Patterns documented in `DESIGN.md` as intentional
- Third-party component libraries (node_modules)
- CSS resets, normalize.css
- Test fixture files
- Generated or minified CSS
- Storybook/test stories
