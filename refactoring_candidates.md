# Refactoring Candidates — Fowler's Principles

Analysis of the `barcodelib4j` main source code, identifying concrete refactoring opportunities based on Martin Fowler's catalog (*Refactoring: Improving the Design of Existing Code*).

---

## 1. `BarExporter.java` — *image* package (1 105 lines)

### 1.1 Duplicated Code — Path Iterator Logic

**Where:** [writePDFContentStream](file:///home/xande/Estudo/ES/Projeto/Real/src/main/java/de/vwsoft/barcodelib4j/image/BarExporter.java#L467-L501) and [writePureEPS](file:///home/xande/Estudo/ES/Projeto/Real/src/main/java/de/vwsoft/barcodelib4j/image/BarExporter.java#L641-L675)

**What:** The path iterator `switch` logic that walks over text shapes and converts `SEG_MOVETO`, `SEG_LINETO`, `SEG_QUADTO`, `SEG_CUBICTO`, and `SEG_CLOSE` into string commands is **nearly identical** in both methods (~35 lines each). The only difference is the `rectfill` command name (`"re"` vs `"r"`) and the final fill command (`"f"` vs `"fill"`).

**Fowler smell:** *Duplicated Code*
**Suggested refactoring:** **Extract Method** — create a private helper like `appendPathCommands(StringBuilder, AffineTransform, String rectCmd, String fillCmd)` and call it from both methods.

---

### 1.2 Duplicated Code — Bar Rectangles Rendering

**Where:** [writePDFContentStream:L460-L463](file:///home/xande/Estudo/ES/Projeto/Real/src/main/java/de/vwsoft/barcodelib4j/image/BarExporter.java#L460-L463) and [writePureEPS:L634-L637](file:///home/xande/Estudo/ES/Projeto/Real/src/main/java/de/vwsoft/barcodelib4j/image/BarExporter.java#L634-L637)

**What:** The loop that transforms bar rectangles and appends their coordinates is duplicated between PDF and EPS writing. Both do `at.createTransformedShape(r).getBounds2D()` then append x, y, width, height with a rect command.

**Fowler smell:** *Duplicated Code*
**Suggested refactoring:** Merge into the same **Extract Method** suggested in 1.1.

---

### 1.3 Duplicated Code — Image Writer Discovery (PNG / JPG)

**Where:** [toPNG:L873-L906](file:///home/xande/Estudo/ES/Projeto/Real/src/main/java/de/vwsoft/barcodelib4j/image/BarExporter.java#L873-L906) and [toJPG:L949-L985](file:///home/xande/Estudo/ES/Projeto/Real/src/main/java/de/vwsoft/barcodelib4j/image/BarExporter.java#L949-L985)

**What:** Both methods follow the exact same pattern:
1. Iterate `ImageIO.getImageWritersByFormatName(…)`
2. Check the `NativeMetadataFormatName`
3. Throw `IOException` if no suitable writer is found
4. Write with a `try/finally` disposing the writer

This pattern is repeated almost line-for-line.

**Fowler smell:** *Duplicated Code*
**Suggested refactoring:** **Extract Method** — create a private helper `findImageWriter(String formatName, String expectedMetadataFormat)` that returns both the writer and the metadata.

---

### 1.4 Long Method — `writePDF`

**Where:** [writePDF:L369-L441](file:///home/xande/Estudo/ES/Projeto/Real/src/main/java/de/vwsoft/barcodelib4j/image/BarExporter.java#L369-L441)

**What:** 72 lines that build PDF objects (Catalog, Pages, Page, Content Stream, Info Dictionary), generate a file ID via MD5, build the xref table, and write the final output. Each of these responsibilities is a separate concern.

**Fowler smell:** *Long Method*
**Suggested refactoring:** **Extract Method** — split into smaller helpers: `buildPDFInfoDictionary(…)`, `generatePDFFileId(…)`, `buildCrossReferenceTable(…)`.

---

### 1.5 Long Method — `writePureEPS`

**Where:** [writePureEPS:L607-L679](file:///home/xande/Estudo/ES/Projeto/Real/src/main/java/de/vwsoft/barcodelib4j/image/BarExporter.java#L607-L679)

**What:** 72 lines mixing header generation, PostScript shorthand definitions, background drawing, bar rendering, text shape rendering, and final output — all in one method.

**Fowler smell:** *Long Method*
**Suggested refactoring:** **Extract Method** — separate EPS header generation from content rendering.

---

### 1.6 Long If-Else Chain — `createTransform`

**Where:** [createTransform:L792-L819](file:///home/xande/Estudo/ES/Projeto/Real/src/main/java/de/vwsoft/barcodelib4j/image/BarExporter.java#L792-L819)

**What:** A chain of 8 sequential `if/else if` statements comparing `myTransform` to each `ImageTransform` enum constant. Each branch applies different rotation/scale/translate operations.

**Fowler smell:** *Long if-else chain (Replace Conditional with Polymorphism)*
**Suggested refactoring:** **Replace Conditional with Polymorphism** — move the transform logic into the `ImageTransform` enum itself, e.g., `myTransform.applyTo(at, mySize)`. Each enum constant would implement its own transformation. Alternatively, use a `switch` statement on the enum for slightly better readability.

---

### 1.7 Long If-Else Chain — `write` dispatcher

**Where:** [write:L292-L311](file:///home/xande/Estudo/ES/Projeto/Real/src/main/java/de/vwsoft/barcodelib4j/image/BarExporter.java#L292-L311)

**What:** A chain of `if/else if` dispatching on `ImageFormat` to call the appropriate write method.

**Fowler smell:** *Replace Conditional with Polymorphism*
**Suggested refactoring:** Either use a `switch` on the enum, or add the write logic as an abstract method/strategy in `ImageFormat` itself.

---

### 1.8 Large Class / God Class — `BarExporter`

**Where:** The entire [BarExporter](file:///home/xande/Estudo/ES/Projeto/Real/src/main/java/de/vwsoft/barcodelib4j/image/BarExporter.java) class (1 105 lines)

**What:** This single class is responsible for:
- PDF generation
- EPS generation (with optional TIFF preview embedding)
- SVG generation
- PNG, BMP, JPG raster image generation
- Transform handling
- Graphics2D partial stub implementation (`BarcodeGraphics2D`)

**Fowler smell:** *Large Class / Divergent Change*
**Suggested refactoring:** **Extract Class** — consider separating format-specific writers (e.g., `PdfWriter`, `EpsWriter`, `SvgWriter`, `RasterWriter`) that implement a common interface. The `BarcodeGraphics2D` inner class could also be extracted.

---

### 1.9 Inner Class Too Large — `BarcodeGraphics2D`

**Where:** [BarcodeGraphics2D:L1002-L1102](file:///home/xande/Estudo/ES/Projeto/Real/src/main/java/de/vwsoft/barcodelib4j/image/BarExporter.java#L1002-L1102)

**What:** ~100 lines of boilerplate `Graphics2D` stub methods throwing `UnsupportedOperationException`. While necessary, this clutters the host class.

**Fowler smell:** *Large Class*
**Suggested refactoring:** **Extract Class** — move `BarcodeGraphics2D` to its own top-level (package-private) file.

---

## 2. `Barcode.java` — *oned* package (861 lines)

### 2.1 Long Method — `draw(Graphics2D, …, barWidthCorrection)`

**Where:** [draw:L692-L740](file:///home/xande/Estudo/ES/Projeto/Real/src/main/java/de/vwsoft/barcodelib4j/oned/Barcode.java#L692-L740)

**What:** ~50 lines combining font size adjustment logic, text positioning, and bar rendering all in one method. The text-handling section alone (L698-L730) contains nested `if`/`do-while` constructs.

**Fowler smell:** *Long Method*
**Suggested refactoring:** **Extract Method** — split into `drawText(…)` and `drawBars(…)`.

---

### 2.2 Duplicated Code — Font Size Adjustment Pattern

**Where:** [Barcode.draw:L705-L712](file:///home/xande/Estudo/ES/Projeto/Real/src/main/java/de/vwsoft/barcodelib4j/oned/Barcode.java#L705-L712) and [UPCEANFamily.draw:L200-L208](file:///home/xande/Estudo/ES/Projeto/Real/src/main/java/de/vwsoft/barcodelib4j/oned/UPCEANFamily.java#L200-L208) and [UPCEANFamily.draw:L253-L258](file:///home/xande/Estudo/ES/Projeto/Real/src/main/java/de/vwsoft/barcodelib4j/oned/UPCEANFamily.java#L253-L258)

**What:** The same font-sizing `do { fontSize += INCREMENT; } while (textWidth < target); fontSize -= INCREMENT;` pattern is repeated **three times** across two files.

**Fowler smell:** *Duplicated Code*
**Suggested refactoring:** **Extract Method** — create a utility method like `adjustFontSize(Graphics2D, Font, String text, double targetWidth)` in `Barcode`.

---

### 2.3 Data Clumps — Draw Parameters

**Where:** All `draw(…)` method signatures in [Barcode](file:///home/xande/Estudo/ES/Projeto/Real/src/main/java/de/vwsoft/barcodelib4j/oned/Barcode.java#L631-L649) and [TwoDSymbol](file:///home/xande/Estudo/ES/Projeto/Real/src/main/java/de/vwsoft/barcodelib4j/twod/TwoDSymbol.java#L155-L175)

**What:** The parameter group `(double x, double y, double w, double h)` appears in every draw method overload (6+ occurrences across these two classes). Additional parameters like `dotSize`, `moduleSize`, `barWidthCorrection` are always used together.

**Fowler smell:** *Data Clumps*
**Suggested refactoring:** **Introduce Parameter Object** — create a `DrawingBounds` or `BarcodeRenderParams` class to encapsulate these frequently grouped parameters.

---

## 3. `GS1Validator.java` — *oned* package (489 lines)

### 3.1 Long Method — Constructor

**Where:** [GS1Validator(String, char):L273-L386](file:///home/xande/Estudo/ES/Projeto/Real/src/main/java/de/vwsoft/barcodelib4j/oned/GS1Validator.java#L273-L386)

**What:** A 113-line constructor that does **everything**: parses FNC1 characters, looks up Application Identifiers (AI), extracts AI data, validates fixed-length AIs, handles checksum placeholders, validates dates, builds both encoded content and human-readable text, and trims trailing FNC1s.

**Fowler smell:** *Long Method*
**Suggested refactoring:** **Extract Method** — break into `skipFnc1Characters(…)`, `extractApplicationIdentifier(…)`, `extractAIData(…)`, `validateAIData(…)`, and `stripTrailingFnc1(…)`.

---

### 3.2 Nested Conditionals — AI Validation Logic

**Where:** [GS1Validator:L340-L363](file:///home/xande/Estudo/ES/Projeto/Real/src/main/java/de/vwsoft/barcodelib4j/oned/GS1Validator.java#L340-L363)

**What:** The validation block (Step 3) has 3 levels of nesting:
```java
if (ai.delimiter > 0 && ...)  // fixed-length check
if (ai == APP_IDS[0] || ...)  // special handling for AI 00, 01, 02
    if (data.charAt(...) == CHECKSUM_PLACEHOLDER) // placeholder handling
    else ...
else if (!ai.matches(data))
    if (ai.isYYMMDD()) ...
    else ...
```

**Fowler smell:** *Deeply nested conditionals*
**Suggested refactoring:** **Decompose Conditional** / **Replace Nested Conditional with Guard Clauses** — extract each validation concern into its own method, e.g., `validateFixedLengthAI(…)`, `validateChecksumAI(…)`, `validatePatternAI(…)`.

---

### 3.3 Magic Numbers — AI Array Indexing

**Where:** [GS1Validator:L345](file:///home/xande/Estudo/ES/Projeto/Real/src/main/java/de/vwsoft/barcodelib4j/oned/GS1Validator.java#L345)

**What:** `if (ai == APP_IDS[0] || ai == APP_IDS[1] || ai == APP_IDS[2])` — Relies on positional indexing into the `APP_IDS` array to identify AI 00, 01, 02. If the array order ever changes, this silently breaks.

**Fowler smell:** *Magic Number*
**Suggested refactoring:** **Replace Magic Number with Symbolic Constant** — give these AIs named references, e.g., `AI_SSCC_00`, `AI_GTIN_01`, `AI_GTIN_02`, or add a boolean flag `requiresModulo10Check` to the `AI` class.

---

## 4. `TwoDCode.java` — *twod* package (586 lines)

### 4.1 Long Method / Switch Statement — `buildSymbol`

**Where:** [buildSymbol:L438-L502](file:///home/xande/Estudo/ES/Projeto/Real/src/main/java/de/vwsoft/barcodelib4j/twod/TwoDCode.java#L438-L502)

**What:** A 64-line method with a large `switch` statement on `myType` that:
1. Sets common hints
2. Creates the appropriate `Writer` and `BarcodeFormat` per type
3. Configures type-specific hints (QR version, DataMatrix shape, PDF417 dimensions, Aztec layers)
4. Encodes

Each `case` block is 8–12 lines of type-specific configuration.

**Fowler smell:** *Long Method / Switch Statements*
**Suggested refactoring:** **Replace Conditional with Polymorphism** — introduce a strategy or move the hint-building logic into the `TwoDType` enum itself, e.g., `myType.configureHints(hints, this)`. Each enum constant would know how to configure its own hints and instantiate its writer.

---

## 5. `UPCEANFamily.java` — *oned* package (400 lines)

### 5.1 Long Method — `draw` override

**Where:** [draw:L187-L297](file:///home/xande/Estudo/ES/Projeto/Real/src/main/java/de/vwsoft/barcodelib4j/oned/UPCEANFamily.java#L187-L297)

**What:** 110 lines handling font sizing, multi-part text drawing (`myNumberPart1` through `myNumberPart4`), add-on text, ISBN/ISSN text, guard bar heights, and the bar rendering loop. This is the longest method in the `oned` package.

**Fowler smell:** *Long Method*
**Suggested refactoring:** **Extract Method** — separate into `drawNumberParts(…)`, `drawAddOnText(…)`, `drawISxNTitle(…)`, and `drawBarsWithGuardPattern(…)`.

---

### 5.2 Data Clumps — Number Parts

**Where:** [UPCEANFamily:L51](file:///home/xande/Estudo/ES/Projeto/Real/src/main/java/de/vwsoft/barcodelib4j/oned/UPCEANFamily.java#L51)

**What:** `transient String myNumberPart1, myNumberPart2, myNumberPart3, myNumberPart4;` — four related string fields that are always used together when drawing the human-readable text.

**Fowler smell:** *Data Clumps*
**Suggested refactoring:** **Introduce Parameter Object** — group into a small class like `TextParts` or a `String[]` array.

---

## 6. `ImplCode128.java` — *oned* package (287 lines)

### 6.1 Long If-Else Chain — `encode` method

**Where:** [encode:L129-L176](file:///home/xande/Estudo/ES/Projeto/Real/src/main/java/de/vwsoft/barcodelib4j/oned/ImplCode128.java#L129-L176)

**What:** Nested `if/else if` chains handling:
- FNC1, FNC2, FNC3, FNC4 character detection (4 branches)
- Code set A vs B vs C encoding differences
- Code set switching via start codes

**Fowler smell:** *Complex Conditional / Switch Statements*
**Suggested refactoring:** **Decompose Conditional** — extract `getFncBarsIndex(char c, int codeSet)` and `getCharacterBarsIndex(char c, int codeSet)` helpers.

---

### 6.2 Complex Conditional — `chooseCodeSet`

**Where:** [chooseCodeSet:L187-L217](file:///home/xande/Estudo/ES/Projeto/Real/src/main/java/de/vwsoft/barcodelib4j/oned/ImplCode128.java#L187-L217)

**What:** 30 lines of deeply nested conditionals with multiple early returns based on `chooseSetCType` results. Logic like `if forward == SETC_FNC_1 → check more chars → decide` is hard to follow.

**Fowler smell:** *Complex Conditional*
**Suggested refactoring:** **Decompose Conditional** / add explanatory variables or small helper methods to clarify intent.

---

## 7. `BarcodeType.java` — *oned* package (225 lines)

### 7.1 Switch Statement — `newInstance`

**Where:** [newInstance:L70-L97](file:///home/xande/Estudo/ES/Projeto/Real/src/main/java/de/vwsoft/barcodelib4j/oned/BarcodeType.java#L70-L97)

**What:** A 22-case `switch` mapping each enum constant to `new ImplXxx()`. If a new barcode type is added, this switch must be manually updated.

**Fowler smell:** *Switch Statements*
**Suggested refactoring:** **Replace Constructor with Factory Method** — store a `Supplier<Barcode>` in each enum constant, e.g., `CODABAR("Codabar", 7, ImplCodabar::new)`. Then `newInstance()` simply calls `myFactory.get()`.

---

## 8. `TwoDSymbol.java` — *twod* package (229 lines)

### 8.1 Duplicated Code — Horizontal and Vertical Chunk Building

**Where:** [TwoDSymbol constructor:L68-L107](file:///home/xande/Estudo/ES/Projeto/Real/src/main/java/de/vwsoft/barcodelib4j/twod/TwoDSymbol.java#L68-L107)

**What:** The horizontal chunk-building loop (L68-L86) and vertical chunk-building loop (L89-L107) are structurally identical: they iterate over a matrix axis, track `barOrSpace` / `widthCounter` / `positionCounter`, and create `Rectangle` instances. The only differences are axis orientation and filter on 1×1 chunks.

**Fowler smell:** *Duplicated Code*
**Suggested refactoring:** **Extract Method** — create a helper `buildChunks(BitMatrix, boolean isHorizontal, int quietZoneSize, boolean skipSinglePixel)` to eliminate the duplication.

---

## 9. `CompoundColor.java` — *image* package (359 lines)

### 9.1 Constructor Overload Explosion

**Where:** [CompoundColor](file:///home/xande/Estudo/ES/Projeto/Real/src/main/java/de/vwsoft/barcodelib4j/image/CompoundColor.java#L39-L182)

**What:** 7 different constructors, including `(int r, int g, int b, int c, int m, int y, int k)`, `(int rgb, int cmyk)`, `(long rgbAndCmyk)`, `(int r, int g, int b)`, `(int c, int m, int y, int k)`, `(Color rgbColor)`, and `(int value, boolean isRGB)`. Several differ only in how they source RGB vs CMYK.

**Fowler smell:** *Long Parameter List / Telescoping Constructors*
**Suggested refactoring:** **Replace Constructor with Factory Method** — use named static factory methods like `CompoundColor.fromRGB(…)`, `CompoundColor.fromCMYK(…)`, `CompoundColor.fromRGBandCMYK(…)` for clarity.

---

## Summary Table

| # | File | Fowler Smell | Refactoring Technique | Severity |
|---|------|-------------|----------------------|----------|
| 1.1 | `BarExporter` | Duplicated Code | Extract Method (path iterator) | 🔴 High |
| 1.2 | `BarExporter` | Duplicated Code | Extract Method (bar rectangles) | 🔴 High |
| 1.3 | `BarExporter` | Duplicated Code | Extract Method (image writer discovery) | 🟡 Medium |
| 1.4 | `BarExporter` | Long Method | Extract Method (writePDF) | 🟡 Medium |
| 1.5 | `BarExporter` | Long Method | Extract Method (writePureEPS) | 🟡 Medium |
| 1.6 | `BarExporter` | Long if-else chain | Replace Conditional with Polymorphism | 🟡 Medium |
| 1.7 | `BarExporter` | Long if-else chain | Switch / Strategy pattern | 🟢 Low |
| 1.8 | `BarExporter` | Large Class | Extract Class (format writers) | 🔴 High |
| 1.9 | `BarExporter` | Large Class | Extract Class (inner G2D stub) | 🟢 Low |
| 2.1 | `Barcode` | Long Method | Extract Method (draw) | 🟡 Medium |
| 2.2 | `Barcode`/`UPCEANFamily` | Duplicated Code | Extract Method (font sizing) | 🟡 Medium |
| 2.3 | `Barcode`/`TwoDSymbol` | Data Clumps | Introduce Parameter Object | 🟢 Low |
| 3.1 | `GS1Validator` | Long Method | Extract Method (constructor) | 🔴 High |
| 3.2 | `GS1Validator` | Nested Conditionals | Decompose Conditional | 🟡 Medium |
| 3.3 | `GS1Validator` | Magic Number | Symbolic Constant / Flag | 🟡 Medium |
| 4.1 | `TwoDCode` | Switch Stmt / Long Method | Replace Conditional with Polymorphism | 🟡 Medium |
| 5.1 | `UPCEANFamily` | Long Method | Extract Method (draw) | 🔴 High |
| 5.2 | `UPCEANFamily` | Data Clumps | Introduce Parameter Object | 🟢 Low |
| 6.1 | `ImplCode128` | Complex Conditional | Decompose Conditional | 🟡 Medium |
| 6.2 | `ImplCode128` | Complex Conditional | Decompose Conditional | 🟡 Medium |
| 7.1 | `BarcodeType` | Switch Stmt | Replace with Supplier / Factory | 🟡 Medium |
| 8.1 | `TwoDSymbol` | Duplicated Code | Extract Method | 🟡 Medium |
| 9.1 | `CompoundColor` | Telescoping Constructors | Replace Constructor with Factory Method | 🟢 Low |
