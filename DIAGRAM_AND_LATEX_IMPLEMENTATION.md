# Diagram Support and LaTeX Fixes Implementation Guide

## Summary
This document outlines the complete implementation of diagram support using Excalidraw format and fixes for LaTeX syntax errors.

## Database Changes (COMPLETED)
- Added `diagram_json` (jsonb[]) - Array of diagrams for question statement
- Added `options_diagrams` (jsonb[]) - Array of diagrams for options
- Added `answer_diagram` (jsonb) - Single diagram for answer
- Added `solution_diagram` (jsonb) - Single diagram for solution
- Applied to: `questions`, `new_questions`, `questions_topic_wise` tables

## LaTeX Syntax Fixes Needed

### Problem Patterns to Fix
1. `ackslash` → `\` (backslash)
2. `rac` → `\frac` (fraction command)
3. `\backslashhat` → `\hat`
4. `\backslashbeta` → `\beta`
5. And all similar malformed LaTeX patterns

### Solution Approach
Create a `cleanLatexSyntax()` function that runs BEFORE sending to AI and AFTER receiving from AI:

```typescript
function cleanLatexSyntax(text: string): string {
  if (!text) return text;

  return text
    // Fix backslash references
    .replace(/ackslash/g, '\\')
    .replace(/\\backslash/g, '\\')

    // Fix frac command
    .replace(/([^\\])rac\{/g, '$1\\frac{')
    .replace(/^rac\{/g, '\\frac{')

    // Fix all backslash-prefixed commands
    .replace(/\\backslashhat/g, '\\hat')
    .replace(/\\backslashbeta/g, '\\beta')
    .replace(/\\backslashalpha/g, '\\alpha')
    .replace(/\\backslashgamma/g, '\\gamma')
    .replace(/\\backslashdelta/g, '\\delta')
    .replace(/\\backslashepsilon/g, '\\epsilon')
    .replace(/\\backslashtheta/g, '\\theta')
    .replace(/\\backslashlambda/g, '\\lambda')
    .replace(/\\backslashmu/g, '\\mu')
    .replace(/\\backslashsigma/g, '\\sigma')
    .replace(/\\backslashpi/g, '\\pi')
    .replace(/\\backslashomega/g, '\\omega')
    .replace(/\\backslashinfty/g, '\\infty')
    .replace(/\\backslashint/g, '\\int')
    .replace(/\\backslashsum/g, '\\sum')
    .replace(/\\backslashprod/g, '\\prod')
    .replace(/\\backslashlim/g, '\\lim')

    // Fix other malformed patterns
    .replace(/Δackslash/g, '\\')
    .replace(/⊗ackslash/g, '\\')
    .replace(/αackslash/g, '\\')
    .replace(/βackslash/g, '\\');
}
```

## Diagram Generation Instructions for AI

### Excalidraw Format Example
```json
{
  "diagram_json": [
    [
      {
        "type": "ellipse",
        "x": 100,
        "y": 100,
        "width": 100,
        "height": 100,
        "strokeColor": "#000000",
        "backgroundColor": "transparent",
        "strokeWidth": 2
      },
      {
        "type": "text",
        "x": 150,
        "y": 150,
        "text": "O",
        "fontSize": 16,
        "strokeColor": "#000000"
      }
    ]
  ]
}
```

### AI Prompt Addition for Diagram Generation
```
DIAGRAM GENERATION INSTRUCTIONS (When Applicable):
If the question requires a visual diagram (circuit, geometry, graph, etc.):
1. Generate Excalidraw-compatible JSON format
2. Use these element types:
   - "ellipse": circles/ovals
   - "rectangle": boxes
   - "line": straight lines
   - "arrow": arrows
   - "text": labels
   - "freedraw": hand-drawn curves
3. Place in appropriate field:
   - diagram_json: array of diagram element arrays for question statement
   - options_diagrams: array of diagrams for each option
   - answer_diagram: single diagram for answer
   - solution_diagram: single diagram for solution
4. Each diagram is an array of shape objects with x, y, width, height, type, etc.

EXAMPLE DIAGRAM - Circle with center O:
"diagram_json": [[
  {"type":"ellipse","x":100,"y":100,"width":100,"height":100,"strokeColor":"#000000","strokeWidth":2},
  {"type":"text","x":145,"y":145,"text":"O","fontSize":16,"strokeColor":"#000000"}
]]
```

## Files to Modify

### 1. src/lib/gemini.ts
- Add `cleanLatexSyntax()` function at top
- Update `ExtractedQuestion` interface to include diagram fields
- Apply `cleanLatexSyntax()` to all question text before parsing
- Update AI prompts to include diagram generation instructions
- Update JSON parsing to handle diagram fields

### 2. src/components/QuestionPreview.tsx
- Import `DiagramRenderer` component
- Render `diagram_json` if present (can be multiple diagrams)
- Render `options_diagrams` for each option if present
- Render `answer_diagram` if present
- Render `solution_diagram` if present
- Apply LaTeX cleaning before rendering with KaTeX

### 3. src/components/PDFScanner.tsx
- Update database insert to include diagram fields
- Handle diagram data from extracted questions

### 4. src/components/QuestionGenerator.tsx
- Update database insert to include diagram fields
- Pass diagram data through generation flow

## Implementation Priority

### HIGH PRIORITY (Must have)
1. LaTeX syntax cleaning function - CRITICAL
2. Update ExtractedQuestion interface
3. Apply LaTeX cleaning in QuestionPreview rendering
4. Basic diagram rendering in QuestionPreview

### MEDIUM PRIORITY (Should have)
1. AI prompt updates for diagram generation
2. Full diagram support in all components
3. Multiple diagram handling

### LOW PRIORITY (Nice to have)
1. Diagram editing capabilities
2. Diagram validation
3. Diagram templates

## Testing Checklist
- [ ] LaTeX with `ackslash` renders correctly as `\`
- [ ] LaTeX with `rac{a}{b}` renders as proper fraction
- [ ] Questions with diagrams display correctly
- [ ] Multiple diagrams in question statement work
- [ ] Option diagrams display correctly
- [ ] Answer and solution diagrams work
- [ ] Database saves diagram data correctly
- [ ] JSON parsing doesn't fail on diagram data

## Notes
- Not all questions need diagrams - diagram fields are nullable
- Diagrams are optional enhancement, system works without them
- LaTeX fixes are CRITICAL and must be implemented
- Focus on LaTeX fixes first, then gradually add diagram support
