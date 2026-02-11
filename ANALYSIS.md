# Exsurge Codebase Analysis

## Executive Summary

Exsurge is a ~13,500-line JavaScript library for rendering Gregorian Chant in square note notation from GABC to SVG. It is architecturally functional but has accumulated significant structural debt that makes extending, configuring, and scaling the system difficult. This document catalogs all identified weaknesses, architectural issues, and improvement barriers, with particular attention to drop cap image customization and uniform chant size scaling.

---

## Table of Contents

1. [Architecture Overview](#1-architecture-overview)
2. [Critical: Dual Scaling Systems Are Disconnected](#2-critical-dual-scaling-systems-are-disconnected)
3. [Critical: Drop Cap System Is Text-Only and Rigid](#3-critical-drop-cap-system-is-text-only-and-rigid)
4. [Critical: God Objects and File Bloat](#4-critical-god-objects-and-file-bloat)
5. [High: Mutable Shared State Throughout Layout Pipeline](#5-high-mutable-shared-state-throughout-layout-pipeline)
6. [High: Hard-Coded Magic Numbers Everywhere](#6-high-hard-coded-magic-numbers-everywhere)
7. [High: Missing Type Safety](#7-high-missing-type-safety)
8. [High: Tight Coupling via Context Object](#8-high-tight-coupling-via-context-object)
9. [High: No Image Rendering Path Exists](#9-high-no-image-rendering-path-exists)
10. [Medium: Minimal Test Coverage](#10-medium-minimal-test-coverage)
11. [Medium: Outdated Build Toolchain](#11-medium-outdated-build-toolchain)
12. [Medium: DOM Dependency Without Abstraction](#12-medium-dom-dependency-without-abstraction)
13. [Medium: Code Duplication](#13-medium-code-duplication)
14. [Medium: Missing Error Handling](#14-medium-missing-error-handling)
15. [Medium: Performance Concerns](#15-medium-performance-concerns)
16. [Low: Incomplete Documentation and API Surface](#16-low-incomplete-documentation-and-api-surface)
17. [Improvement Roadmap](#17-improvement-roadmap)

---

## 1. Architecture Overview

### Module Structure

| File | Lines | Role |
|------|-------|------|
| `Exsurge.Drawing.js` | 3,292 | SVG rendering, context, all visualizers, text elements |
| `Exsurge.Chant.ChantLine.js` | 2,204 | Line layout, justification, spacing |
| `Exsurge.Gabc.js` | 2,031 | GABC parsing, notation/mapping creation |
| `Exsurge.Chant.Neumes.js` | 1,447 | Neume types and neume layout builder |
| `Exsurge.Chant.js` | 1,054 | Score, notes, clefs, mappings |
| `Exsurge.Glyphs.js` | 925 | Static glyph definitions (SVG paths) |
| `Exsurge.Text.js` | 666 | Syllabification, language support |
| `Exsurge.Chant.Markings.js` | 423 | Episemata, morae, ictus, braces |
| `Exsurge.Core.js` | 340 | Point, Rect, Margins, Size, Pitch |
| `Exsurge.Chant.Signs.js` | 343 | Custos, dividers, accidentals |
| `Exsurge.Titles.js` | 230 | Title/supertitle rendering |

### Rendering Pipeline

```
GABC String
  -> Gabc.createMappingsFromSource()   [parse]
  -> ChantScore(ctxt, mappings)        [model creation]
  -> score.performLayout(ctxt)         [layout: bounds, positions]
  -> score.layoutChantLines(ctxt, w)   [line breaking, justification]
  -> score.createSvgNode(ctxt)         [render to SVG DOM]
     score.createSvgTree(ctxt, zoom)   [render to React-compatible tree]
     score.createSvgFragment(ctxt)     [render to SVG string]
     score.draw(ctxt)                  [render to Canvas]
```

### Class Hierarchy

```
ChantLayoutElement (abstract base)
  +-- GlyphVisualizer         (renders font glyphs)
  +-- DividerLineVisualizer   (bar lines)
  +-- NeumeLineVisualizer     (connecting lines)
  +-- VirgaLineVisualizer     (vertical note lines)
  +-- CurlyBraceVisualizer    (braces)
  +-- RoundBraceVisualizer    (braces)
  +-- TextElement             (text rendering base)
  |     +-- Lyric
  |     +-- TranslationText
  |     +-- AboveLinesText
  |     +-- DropCap           <-- text-only, no image support
  |     +-- TitleTextElement
  |           +-- Supertitle, Title, Subtitle, TextLeftRight
  +-- ChantNotationElement    (musical notation base)
        +-- Clef -> DoClef, FaClef, TrebleClef
        +-- Neume -> Punctum, Virga, Podatus, Clivis, ... (20+ types)
        +-- Custos, Dividers, TextOnly, Accidental, Virgula
```

---

## 2. Critical: Dual Scaling Systems Are Disconnected

**This is the root cause of the scaling problem described in the requirements.**

The codebase has two completely independent scaling systems that do not communicate:

### System A: Glyph/Staff Scaling

Controlled by `setGlyphScaling(glyphScaling)` at `Exsurge.Drawing.js:762`:

```javascript
setGlyphScaling(glyphScaling) {
  this.glyphScaling = glyphScaling;
  this.staffInterval = this.glyphPunctumWidth * this.glyphScaling;
  this.staffLineWeight = Math.ceil((5 * this.staffInterval) / 8) / 5;
  this.neumeLineWeight = this.staffLineWeight;
  this.intraNeumeSpacing = this.staffInterval / 2.0;
  // ... rebuilds glyph defs
}
```

Every glyph dimension is multiplied by `glyphScaling` (`Exsurge.Drawing.js:1208-1214`):

```javascript
this.origin.x = this.glyph.origin.x * ctxt.glyphScaling;
this.bounds.width = this.glyph.bounds.width * ctxt.glyphScaling;
this.bounds.height = this.glyph.bounds.height * ctxt.glyphScaling;
```

### System B: Text Sizing

Controlled by `setFont(font, size)` at `Exsurge.Drawing.js:683`:

```javascript
setFont(font, size = 16, baseStyle = {}, fontDictionary) {
  for (let [key, textType] of Object.entries(TextTypes)) {
    let textStyle = (this.textStyles[key] = this.textStyles[key] || {});
    textStyle.size = textType.defaultSize
      ? textType.defaultSize(size, this)  // ignores glyphScaling entirely
      : textType.size(this);
  }
}
```

Text sizes are defined as multipliers of a base `size` parameter that has no relationship to `glyphScaling`:

```javascript
// Exsurge.Drawing.js:71-166
lyric:       { defaultSize: (size) => size * 0.9 },
annotation:  { defaultSize: (size) => (size * 2) / 3 },
dropCap:     { defaultSize: (size) => size * 4 },
translation: { defaultSize: (size) => size * 0.75 },
```

### Why This Is Hard to Fix

1. **No shared scale factor**: `setGlyphScaling()` and `setFont()` are independent methods with no awareness of each other. The `fixme` comment at line 592-594 acknowledges this:

   ```javascript
   // fixme: for now, we just set these using the glyph scales...
   // Really what we should do is scale the punctum size based
   // on the text metrics, right? 1 punctum ~ x height size?
   ```

2. **Text metrics are measured absolutely**: Text width/height comes from browser measurement APIs (`measureText()`, SVG `getBBox()`) using pixel font-size values. Scaling text after measurement invalidates all layout calculations.

3. **Layout coupling**: Line heights are calculated once from text sizes (`Exsurge.Chant.ChantLine.js:111`):
   ```javascript
   this.lyricLineHeight = ctxt.textStyles.lyric.size * (ctxt.textStyles.lyric.lineHeight || 1.1);
   ```
   This never recalculates when glyphScaling changes.

4. **Spacing uses mixed units**: Some spacing is in `staffInterval` multiples (scales with glyphs), some is in absolute pixels (does not scale). There is no consistent unit system.

### What Needs to Change

- Introduce a single "chant scale" factor that governs both glyph and text sizing proportionally
- Text sizes should be expressed relative to `staffInterval` rather than as absolute pixel values
- All spacing parameters need to use consistent units
- `setGlyphScaling()` must trigger text size recalculation, or both must derive from a common source

---

## 3. Critical: Drop Cap System Is Text-Only and Rigid

**This directly blocks the drop cap image customization goal.**

### Current Implementation

The `DropCap` class (`Exsurge.Drawing.js:2834-2852`) extends `TextElement`:

```javascript
export class DropCap extends TextElement {
  constructor(ctxt, text, sourceIndex) {
    super(ctxt, text,
      (ctxt) => ctxt.textStyles.dropCap.font,
      (ctxt) => ctxt.textStyles.dropCap.size,
      "middle", sourceIndex, text);
    this.padding = ctxt.staffInterval * ctxt.textStyles.dropCap.padding;
  }
}
```

### Specific Limitations

1. **Text-only rendering**: `TextElement` renders via `<text>` SVG elements or Canvas `fillText()`. There is no code path for `<image>`, `<svg>`, or any raster/vector image embedding anywhere in the library.

2. **Fixed aspect ratio**: The drop cap is a single character. Its width and height are determined by font metrics. There is no mechanism to control aspect ratio.

3. **No line-span control**: The drop cap is positioned on the first chant line only (`Exsurge.Chant.ChantLine.js:255`). It occupies horizontal space on one line via `staffLeft` padding (`Exsurge.Chant.ChantLine.js:732-734`):
   ```javascript
   padding = this.score.dropCap.bounds.width + this.score.dropCap.padding * 2;
   this.staffLeft += padding;
   ```
   There is no mechanism for a drop cap to span multiple chant lines vertically.

4. **Auto-generated from lyrics**: The drop cap is always extracted from the first lyric character (`Exsurge.Drawing.js:2695-2724`). Users cannot specify an arbitrary image or override the content.

5. **Limited configuration**: Only `padding`, `font`, `size`, and `color` are configurable via `textStyles.dropCap`. No options for:
   - Image source (URL, data URI, SVG inline)
   - Width/height independently
   - Vertical span (number of chant lines)
   - Margin/spacing per-side
   - Alignment within the reserved space
   - Fallback behavior (image fails to load -> text drop cap)

6. **Single padding value**: `padding` at `Exsurge.Drawing.js:541` is a single number applied equally to both sides. No per-side margin control.

### What Needs to Change

- Create an `ImageElement` base class (or extend `ChantLayoutElement`) that can render `<image>` in SVG, `drawImage()` on Canvas, and `<image>` in SVG strings
- Create `DropCapImage` that extends this, supporting configurable width, height, aspect ratio, line span
- Modify `ChantLine.buildFromChantNotationIndex()` to reserve vertical space across multiple lines when a multi-line drop cap is specified
- Create a "drop cap package" configuration concept: a named collection of images per letter that users can reference
- Add per-side margin/padding properties (top, right, bottom, left)
- Support async image loading in the layout pipeline

---

## 4. Critical: God Objects and File Bloat

Several files have grown far beyond a single responsibility:

### `Exsurge.Drawing.js` (3,292 lines)

This single file contains:
- `ChantContext` class (configuration, state, fonts, measurement, SVG defs)
- `TextMeasuringStrategy` enum
- `TextTypes` dictionary (8 text type definitions)
- `GlyphCode` enum (50+ glyph codes)
- `QuickSvg` utility object (20+ SVG DOM methods)
- `ChantLayoutElement` base class
- `GlyphVisualizer` class
- `ChantNotationElement` class
- `HorizontalEpisemaVisualizer` class
- `RoundBraceVisualizer` and `CurlyBraceVisualizer`
- `NeumeLineVisualizer`, `VirgaLineVisualizer`, `LineaVisualizer`
- `DividerLineVisualizer`
- `TextSpan` class
- `TextElement` base class (400+ lines of text measurement and rendering)
- `Lyric`, `TranslationText`, `AboveLinesText`, `DropCap` classes
- `TitleTextElement`, `Supertitle`, `Title`, `Subtitle`, `TextLeftRight`
- `Annotations` class

**Impact**: Any change to text rendering risks breaking glyph rendering. Adding image support means modifying this 3,292-line file. Testing any individual component requires loading the entire module.

### `Exsurge.Chant.ChantLine.js` (2,204 lines)

Contains all of:
- Line building (deciding which notations fit on a line)
- Notation positioning
- Lyric positioning and centering
- Translation positioning
- Drop cap positioning
- Annotation positioning
- Justification and condensing
- Ledger line creation
- Brace attachment
- Staff line rendering
- Four parallel rendering methods (draw, svgNode, svgFragment, svgTree)

### `Exsurge.Gabc.js` (2,031 lines)

Contains:
- GABC header parsing
- GABC notation parsing
- Note creation
- Neume creation
- Mapping management
- Clef parsing
- All special notation handling (braces, markings, signs)

---

## 5. High: Mutable Shared State Throughout Layout Pipeline

The `ChantContext` object is passed through every method call and is mutated extensively during layout:

### State Mutations During Layout

| Property | Mutated At | Purpose |
|----------|-----------|---------|
| `ctxt.activeClef` | `ChantLine:783`, `ChantLine:899` | Tracks current clef during line building |
| `ctxt.currNotationIndex` | `Drawing:614`, various | Tracks position in notation array |
| `ctxt.activeNotations` | `Drawing:613` | Reference to current notation array |
| `ctxt.lastStartBrace` | `ChantLine:812` | Tracks brace state across notations |

The comment at `Exsurge.Drawing.js:612` acknowledges the fragility:

```javascript
// these are only gauranteed to be valid during the performLayout phase!
```

### Object Mutation

- `Note.bounds` and `Note.origin` are mutated in-place during layout
- `Lyric.bounds.y` is set to `Number.MAX_SAFE_INTEGER` to "hide" lyrics (`ChantLine:767`)
- Properties are added and `delete`-ed from objects to track transient state (`Gabc:418-430`)
- Lyric arrays are cloned then individually mutated (`ChantLine:755-769`)

**Impact**: Layout is not re-entrant. Running layout twice with the same context may produce different results. State pollution between runs is possible. Parallel or incremental layout is impossible.

---

## 6. High: Hard-Coded Magic Numbers Everywhere

Configuration values that should be externalized are embedded as literals throughout the code:

### In `Exsurge.Drawing.js` (ChantContext constructor)

| Line | Value | Purpose |
|------|-------|---------|
| 499 | `"'Palatino Linotype', 'Book Antiqua', Palatino, serif"` | Default font family |
| 499 | `16` | Default font size |
| 501 | `"#d00"` | Rubric color |
| 541 | `1` | Drop cap padding (staffIntervals) |
| 543 | `1` | Annotation padding |
| 545 | `2` | Min ledger separation |
| 546 | `2` | Min space above staff |
| 547 | `1` | Min space below staff |
| 548 | `1.5` | Space between systems |
| 555 | `0.5` | Max extra justification space |
| 595 | `1.0 / 16.0` | Default glyph scaling |
| 598 | `2.5` | Inter-syllabic spacing multiplier |
| 601 | `2` | Accidental space multiplier |
| 604 | `1` | Inter-verbal spacing multiplier |
| 631 | `0.3` | Condensing tolerance |

### In `Exsurge.Chant.Markings.js`

| Line | Value | Purpose |
|------|-------|---------|
| 80 | `0.25` | Minimum episema distance (staffInterval multiplier) |
| 107 | `0.5` | Episema offset (staffInterval fraction) |
| Various | `3/4`, `1.5`, `2/3`, `1/3` | Positioning fractions with no explanation |

### In `Exsurge.Chant.ChantLine.js`

| Line | Value | Purpose |
|------|-------|---------|
| 74 | Comment says "fixme: make these configurable values from the score" | Acknowledged but not fixed |
| 1482 | `0.5` | Justification threshold |

**Impact**: Users cannot tune the rendering without forking the code. Different use cases (screen vs. print, small vs. large scores) require different values.

---

## 7. High: Missing Type Safety

### TypeScript Definitions Are Stubs

From `src/index.d.ts:3-11`:

```typescript
type ChantNotation = unknown; // TODO: Add types for these
type ChantLine = unknown;
type Note = unknown;
type Clef = unknown;
type DropCap = unknown;
type Rect = unknown;
```

The entire public API surface is untyped. The `ChantContext` interface in the `.d.ts` file is the most complete type definition, but it only covers property names, not method signatures for internal classes.

### Dynamic Property Access

- `Exsurge.Gabc.js:126`: `this[match[1]] = match[2]` -- arbitrary property assignment from parsed GABC headers
- `Exsurge.Drawing.js:398`: `obj[camelCase] = obj[key]` -- dynamic property renaming
- `Exsurge.Drawing.js:685`: `this.textStyles[key] = this.textStyles[key] || {}` -- dynamic style dictionary

### Duck Typing Instead of Interfaces

Boolean flags like `notation.isNeume`, `notation.isClef`, `notation.isDivider` are used for type discrimination instead of `instanceof` checks or proper polymorphism. This makes it easy to forget setting a flag on a new class.

---

## 8. High: Tight Coupling via Context Object

`ChantContext` is a god object that every component depends on. It holds:

- Font configuration
- Style configuration
- Scaling parameters
- Line weight calculations
- Mutable layout state (activeClef, currNotationIndex)
- SVG defs management
- Text measurement infrastructure (canvas, SVG measurer)
- DOM manipulation methods

Every class and every method receives `ctxt` as its first parameter. Components cannot function without a fully initialized context. This makes:

- Unit testing individual components impossible without a full context
- Reusing components in different contexts (e.g., thumbnail rendering) difficult
- Understanding data flow opaque -- any method might read or write any context property

---

## 9. High: No Image Rendering Path Exists

The rendering pipeline has four output modes:

1. Canvas (`draw()` -- uses `ctx.fillText()`, `ctx.fillRect()`, `ctx.stroke()`)
2. SVG DOM (`createSvgNode()` -- creates `<text>`, `<path>`, `<line>`, `<use>` elements)
3. SVG String (`createSvgFragment()` -- concatenates SVG markup strings)
4. SVG Tree (`createSvgTree()` -- builds JSON objects for React)

None of these paths support:
- `<image>` SVG elements
- `ctx.drawImage()` Canvas calls
- External resource loading
- Asynchronous element rendering

Adding image support requires modifying all four rendering paths and adding image loading/caching infrastructure.

---

## 10. Medium: Minimal Test Coverage

### Current State

- **7 test cases** in `test/index.js`
- Tests cover only: `Point`, `Rect`, `Margins`, `Size`, and Latin syllabification
- No tests for: rendering, layout, GABC parsing, neume creation, line breaking, drop caps, scaling, text measurement, or any chant-specific functionality
- README says "Under construction" for the Tests section

### Impact on Improvement

Without tests, any refactoring or feature addition risks breaking existing behavior with no safety net. The complex layout algorithms in `ChantLine` (2,204 lines) have zero test coverage.

---

## 11. Medium: Outdated Build Toolchain

| Dependency | Current Version | Latest Major |
|-----------|----------------|--------------|
| Webpack | 1.x | 5.x |
| Babel | 6.x | 7.x |
| ESLint | 1.x | 9.x |
| Mocha | 2.x | 10.x |
| Chai | 3.x | 5.x |

The outdated toolchain means:
- No tree-shaking (bundle is 217 KB minified, all-or-nothing)
- No modern ES module output
- No code splitting
- Security vulnerabilities in old dependencies
- Limited developer tooling integration

---

## 12. Medium: DOM Dependency Without Abstraction

The library conditionally accesses the DOM:

```javascript
// Exsurge.Drawing.js:48
const canAccessDOM = typeof document !== "undefined";
```

But DOM access is scattered throughout `ChantContext`, `QuickSvg`, and rendering methods without a proper abstraction layer:
- `document.body.insertBefore()` at line 578 for text measurement
- `document.createElement()` at line 821 for canvas
- `document.getElementById()` at line 791 for font insertion
- `document.createElementNS()` throughout `QuickSvg`

**Impact**: Cannot run in Node.js for server-side rendering. Cannot unit test rendering logic in a headless environment without a full DOM polyfill.

---

## 13. Medium: Code Duplication

### Clef Implementations

`DoClef`, `FaClef`, and `TrebleClef` (`Exsurge.Chant.js:225-352`) have nearly identical `pitchToStaffPosition()` and `staffPositionToPitch()` methods with only the base step constant differing.

### Episema Positioning

`Exsurge.Chant.Markings.js:100-150` has mirrored above/below positioning logic that is copy-pasted with inverted signs.

### Rendering Methods

Every class that renders must implement four nearly identical methods: `draw()`, `createSvgNode()`, `createSvgFragment()`, and `createSvgTree()`. These share logic but duplicate traversal and coordinate calculations.

---

## 14. Medium: Missing Error Handling

### String Throws

```javascript
// Exsurge.Drawing.js:808
throw "findNextNeume() called without a valid currNotationIndex set";

// Exsurge.Gabc.js:1580
throw "Invalid note data: " + data;
```

These are plain string throws, not `Error` objects, so they lack stack traces and cannot be caught by type.

### Silent Failures

```javascript
// Exsurge.Gabc.js:176
} catch (e) {
  console.warn(e);  // swallowed, layout continues with potentially corrupt state
}
```

### No Input Validation

- No validation that GABC input is well-formed before parsing
- No validation that font metrics are available before layout
- No validation that context is fully initialized before rendering

---

## 15. Medium: Performance Concerns

### Unnecessary Allocations in Tight Loops

- `ChantLine:756-763`: Creates new `Lyric` objects inside `.map()` for every lyric on every layout pass
- `Neumes:57`: `.slice(-1)[0]` creates an intermediate array to get the last element
- `Gabc:1623-1624`: Same `.slice(-1)[0]` pattern repeated

### Multiple Collection Passes

- GABC parsing iterates over notations multiple times
- Line layout does multiple passes to position elements, calculate bounds, and justify
- No caching of intermediate results

### Full Glyph Dictionary Loaded

`Exsurge.Glyphs.js` (925 lines) loads all glyph definitions as a single static dictionary. No lazy loading or tree-shaking -- every glyph is in memory even if unused.

---

## 16. Low: Incomplete Documentation and API Surface

- README marked "Under construction"
- API Reference section is empty
- JSDoc comments are sparse and inconsistent
- Many `TextTypes` entries have inconsistent property sets (`size` vs `defaultSize`)
- `fixme` and `TODO` comments scattered through codebase (lines 74, 339, 370, 592, 1002 in various files)

---

## 17. Improvement Roadmap

Based on the analysis above, here is a prioritized roadmap for getting the codebase to a state where the desired features can be built cleanly.

### Phase 1: Foundation (Prerequisites for Everything Else)

1. **Unify the scaling system**: Create a single scale factor that derives both glyph dimensions and text sizes proportionally. All spacing should use units relative to `staffInterval`.

2. **Break apart `Exsurge.Drawing.js`**: Extract into separate modules:
   - `ChantContext.js` -- configuration only
   - `QuickSvg.js` -- SVG DOM utilities
   - `GlyphVisualizer.js` -- glyph rendering
   - `TextElement.js` -- text measurement and rendering
   - `Visualizers.js` -- line, brace, and other visualizers

3. **Externalize configuration**: Move all magic numbers into a defaults object that users can override. Create a `ChantConfig` or similar that feeds into `ChantContext`.

4. **Add basic test infrastructure**: Set up a modern test framework with snapshot tests for known GABC inputs. This provides a regression safety net for all subsequent changes.

### Phase 2: Enable Image Support

5. **Create `ImageElement` base class**: Implement `draw()`, `createSvgNode()`, `createSvgFragment()`, `createSvgTree()` for image rendering across all four output modes.

6. **Create `DropCapImage` class**: Support configurable width, height, aspect ratio, and image source (URL, data URI, inline SVG).

7. **Add multi-line drop cap support**: Modify `ChantLine` to reserve vertical space across multiple lines. The drop cap space reservation currently only affects `staffLeft` on the first line -- extend it to affect the first N lines.

8. **Create drop cap package system**: A configuration object that maps characters to image sources, with fallback to text rendering.

9. **Add per-side margin control**: Replace the single `padding` value with `{ top, right, bottom, left }` margins.

### Phase 3: Scaling and Uniformity

10. **Make text sizes relative to staff**: Express `defaultSize` functions in terms of `staffInterval` or `glyphScaling` rather than an absolute `size` parameter.

11. **Couple `setGlyphScaling` to text recalculation**: When glyph scaling changes, automatically recalculate text sizes, line heights, and spacing.

12. **Create a `setScale(scale)` unified API**: A single method that proportionally adjusts everything -- glyphs, text, spacing, line weights.

### Phase 4: Architecture Cleanup

13. **Eliminate mutable context state**: Use immutable data structures for layout input. Pass layout state as return values rather than mutating `ctxt`.

14. **Add proper TypeScript types**: Convert source files to TypeScript or complete the `.d.ts` definitions for all public and internal classes.

15. **Modernize build toolchain**: Upgrade to current Webpack/Vite, Babel 7+, enable tree-shaking, add ESM output.

16. **Abstract DOM access**: Create a rendering backend interface so the library works in Node.js and can be tested headlessly.

17. **Expand test coverage**: Target coverage for GABC parsing, line breaking, justification, drop cap positioning, and scaling behavior.

---

## Summary Table

| Issue | Severity | Blocks |
|-------|----------|--------|
| Dual scaling systems disconnected | CRITICAL | Uniform chant scaling |
| Drop cap is text-only | CRITICAL | Drop cap image customization |
| God objects (Drawing.js 3,292 lines) | CRITICAL | All extensibility |
| Mutable shared state in layout | HIGH | Reliable rendering, testing |
| Magic numbers everywhere | HIGH | User configuration |
| Missing type safety | HIGH | Safe refactoring |
| Tight coupling via context | HIGH | Component reuse, testing |
| No image rendering path | HIGH | Drop cap images |
| Minimal test coverage (7 tests) | MEDIUM | Safe refactoring |
| Outdated build toolchain | MEDIUM | Modern deployment |
| DOM dependency without abstraction | MEDIUM | Server-side rendering |
| Code duplication | MEDIUM | Maintenance burden |
| Missing error handling | MEDIUM | Debugging, reliability |
| Performance concerns | MEDIUM | Large score rendering |
| Incomplete docs/API | LOW | Developer onboarding |
