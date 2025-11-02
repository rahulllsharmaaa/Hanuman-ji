# Complete Implementation Guide: Diagrams & LaTeX Fixes

## What Has Been Completed

### 1. Database Migration ✅
- Added diagram support columns to all question tables
- Columns added:
  - `diagram_json` (jsonb[]) - Multiple diagrams for question statement
  - `options_diagrams` (jsonb[]) - Diagrams for each option
  - `answer_diagram` (jsonb) - Single diagram for answer
  - `solution_diagram` (jsonb) - Single diagram for solution
- Applied to: `questions`, `new_questions`, `questions_topic_wise`

### 2. DiagramRenderer Component ✅
- Created `/src/components/DiagramRenderer.tsx`
- Renders Excalidraw-format diagrams on HTML canvas
- Supports all Excalidraw element types (ellipse, rectangle, line, arrow, text, etc.)
- Auto-scales diagrams to fit container
- Handles multiple diagrams in arrays

### 3. LaTeX Cleaner Utility ✅
- Created `/src/lib/latexCleaner.ts`
- Fixes ALL malformed LaTeX patterns:
  - `ackslash` → `\`
  - `rac{` → `\frac{`
  - `\backslashbeta` → `\beta`
  - And 50+ other patterns
- Provides `cleanLatexSyntax()` for strings
- Provides `cleanQuestionLatex()` for question objects

## What You Need to Do Next

### Step 1: Update gemini.ts (CRITICAL)

At the **top of the file**, add the import:
```typescript
import { cleanLatexSyntax, cleanQuestionLatex } from './latexCleaner';
```

Update the **ExtractedQuestion interface** (around line 296):
```typescript
export interface ExtractedQuestion {
  question_statement: string;
  question_type: 'MCQ' | 'MSQ' | 'NAT' | 'Subjective';
  options?: string[];
  answer?: string;
  solution?: string;
  question_number?: string;
  page_number?: number;
  has_image?: boolean;
  image_description?: string;
  is_continuation?: boolean;
  spans_multiple_pages?: boolean;
  uploaded_image?: string;
  topic_id?: string;
  is_wrong?: boolean;
  validation_reason?: string;

  // NEW DIAGRAM FIELDS
  diagram_json?: any[];  // Array of Excalidraw element arrays
  options_diagrams?: any[];  // Array of diagram arrays for each option
  answer_diagram?: any;  // Single Excalidraw element array
  solution_diagram?: any;  // Single Excalidraw element array
}
```

In **performExtraction()** function (around line 720), after parsing questions, add:
```typescript
// Clean LaTeX syntax in all extracted questions
questions = questions.map(q => cleanQuestionLatex(q));
```

In **generateQuestionsForTopic()** function (around line 1060), after parsing questions, add:
```typescript
// Clean LaTeX syntax in generated questions
questions = questions.map(q => cleanQuestionLatex(q));
```

In **generateSolutionsForPYQs()** function (around line 1196), after parsing solutions, add:
```typescript
// Clean LaTeX in solutions
solutions = solutions.map(sol => ({
  answer: cleanLatexSyntax(sol.answer),
  solution: cleanLatexSyntax(sol.solution)
}));
```

Add **DIAGRAM GENERATION INSTRUCTIONS** to the PDF extraction prompt (around line 648):
```typescript
const prompt = `You are an expert at extracting questions from academic exam papers...

[existing instructions]

DIAGRAM GENERATION (If visual elements present):
If the question contains geometric figures, circuits, graphs, or diagrams:
1. Generate Excalidraw-compatible JSON in "diagram_json" field
2. Use element types: ellipse, rectangle, line, arrow, text
3. Example format:
   "diagram_json": [[
     {"type":"ellipse","x":100,"y":100,"width":100,"height":100,"strokeColor":"#000000","strokeWidth":2},
     {"type":"text","x":145,"y":145,"text":"O","fontSize":16,"strokeColor":"#000000"}
   ]]
4. For dice example question: Create 3 dice cubes with visible numbers
5. Keep diagrams simple and clear

[rest of prompt]
`;
```

### Step 2: Update QuestionPreview.tsx (CRITICAL)

At the **top of the file**, add imports:
```typescript
import { cleanLatexSyntax } from '../lib/latexCleaner';
import { DiagramRenderer } from './DiagramRenderer';
```

In **renderMathContent()** function (around line 58), **FIRST LINE** should be:
```typescript
const renderMathContent = (content: string) => {
  if (!content) return null;

  // CRITICAL: Clean LaTeX syntax FIRST
  let cleanedContent = cleanLatexSyntax(content);

  // Then apply existing cleaning (keep all your existing code)
  cleanedContent = cleanedContent
    .replace(/\\backslashhat/g, '\\hat')
    // ... rest of your existing replacements ...
```

After the **Question Statement** section (around line 175), add diagram rendering:
```typescript
{/* Question Diagrams */}
{question.diagram_json && question.diagram_json.length > 0 && (
  <div className="mb-4">
    <h4 className="text-md font-semibold text-gray-800 mb-3">Question Diagrams:</h4>
    <div className="space-y-4">
      {question.diagram_json.map((diagram, idx) => (
        <DiagramRenderer
          key={idx}
          diagramData={diagram}
          maxWidth={600}
          maxHeight={400}
        />
      ))}
    </div>
  </div>
)}
```

In the **Options** section (around line 206), update to include option diagrams:
```typescript
{question.options && question.options.length > 0 && (
  <div>
    <h4 className="text-md font-semibold text-gray-800 mb-3">Options:</h4>
    <div className="grid grid-cols-1 md:grid-cols-2 gap-3">
      {question.options.map((option, optionIndex) => (
        <div key={optionIndex} className="bg-white p-3 rounded-lg border border-gray-100">
          <div className="flex items-start gap-3">
            <div className="bg-gradient-to-r from-purple-100 to-indigo-100 text-purple-700 w-6 h-6 rounded-full flex items-center justify-center text-sm font-semibold flex-shrink-0 mt-0.5">
              {String.fromCharCode(65 + optionIndex)}
            </div>
            <div className="flex-1">
              <div className="text-gray-700">
                {renderMathContent(option)}
              </div>
              {/* Option Diagram */}
              {question.options_diagrams && question.options_diagrams[optionIndex] && (
                <DiagramRenderer
                  diagramData={question.options_diagrams[optionIndex]}
                  maxWidth={300}
                  maxHeight={200}
                  className="mt-2"
                />
              )}
            </div>
          </div>
        </div>
      ))}
    </div>
  </div>
)}
```

In the **Answer and Solution** section (around line 230), add diagram rendering:
```typescript
{(question.answer || question.solution) && (
  <div className="mt-4 space-y-4">
    {question.answer && (
      <div>
        <h4 className="text-md font-semibold text-gray-800 mb-3">Answer:</h4>
        <div className="bg-green-50 p-3 rounded-lg border border-green-200">
          <span className="text-green-800 font-medium">{question.answer}</span>
        </div>
        {/* Answer Diagram */}
        {question.answer_diagram && (
          <DiagramRenderer
            diagramData={question.answer_diagram}
            maxWidth={400}
            maxHeight={300}
            className="mt-2"
          />
        )}
      </div>
    )}

    {question.solution && (
      <div>
        <h4 className="text-md font-semibold text-gray-800 mb-3">Solution:</h4>
        <div className="bg-blue-50 p-4 rounded-lg border border-blue-200 text-blue-800">
          {renderMathContent(question.solution)}
        </div>
        {/* Solution Diagram */}
        {question.solution_diagram && (
          <DiagramRenderer
            diagramData={question.solution_diagram}
            maxWidth={500}
            maxHeight={400}
            className="mt-2"
          />
        )}
      </div>
    )}
  </div>
)}
```

### Step 3: Update PDFScanner.tsx

In **savePdfQuestions()** function (around line 587), update the insert to include diagrams:
```typescript
const questionsToInsert = questions.map(q => {
  const config = typeConfigMap.get(q.question_type as any) || enabledQuestionTypes[0];

  return {
    question_type: q.question_type,
    question_statement: q.question_statement,
    options: q.options && q.options.length > 0 ? q.options : null,
    // NEW DIAGRAM FIELDS
    diagram_json: q.diagram_json || null,
    options_diagrams: q.options_diagrams || null,
    answer_diagram: q.answer_diagram || null,
    solution_diagram: q.solution_diagram || null,
    // ... rest of fields
    course_id: selectedCourse,
    year: parseInt(year),
    slot: slot.trim() || null,
    part: part.trim() || null,
    correct_marks: config.correct_marks,
    incorrect_marks: config.incorrect_marks,
    skipped_marks: config.skipped_marks,
    partial_marks: config.partial_marks,
    time_minutes: config.time_minutes,
    categorized: false,
  };
});
```

### Step 4: Update QuestionGenerator.tsx

In **generateNewQuestions()** function (around line 704), update the database insert:
```typescript
const questionToSave = {
  topic_id: topic.id,
  topic_name: topic.name,
  chapter_id: topic.chapter_id,
  question_statement: question.question_statement,
  question_type: questionType,
  options: question.options,
  answer: question.answer,
  solution: question.solution,
  // NEW DIAGRAM FIELDS
  diagram_json: question.diagram_json || null,
  options_diagrams: question.options_diagrams || null,
  answer_diagram: question.answer_diagram || null,
  solution_diagram: question.solution_diagram || null,
  // ... rest of fields
  slot: selectedSlot || null,
  part: selectedPart || null,
  correct_marks: questionConfig.correct_marks,
  incorrect_marks: questionConfig.incorrect_marks,
  skipped_marks: questionConfig.skipped_marks,
  partial_marks: questionConfig.partial_marks,
  time_minutes: questionConfig.time_minutes,
  difficulty_level: 'Medium',
  purpose: 'practice',
  is_wrong: isWrong || null
};
```

### Step 5: Update supabase.ts Types

In `/src/lib/supabase.ts`, update the Database types to include diagram fields in all three tables (around lines for each table definition):

```typescript
// In questions table Row type:
diagram_json: any[] | null;
options_diagrams: any[] | null;
answer_diagram: any | null;
solution_diagram: any | null;

// In new_questions table Row type:
diagram_json: any[] | null;
options_diagrams: any[] | null;
answer_diagram: any | null;
solution_diagram: any | null;

// In questions_topic_wise table Row type:
diagram_json: any[] | null;
options_diagrams: any[] | null;
answer_diagram: any | null;
solution_diagram: any | null;
```

## Testing Checklist

### LaTeX Fixes
- [ ] Open a question with `ackslash` → Should render as proper backslash
- [ ] Test `rac{a}{b}` → Should render as fraction
- [ ] Test all Greek letters → Should render properly
- [ ] No LaTeX errors in console

### Diagram Support
- [ ] Generate/extract questions with diagrams
- [ ] Diagrams display correctly in QuestionPreview
- [ ] Multiple diagrams in question work
- [ ] Option diagrams display
- [ ] Answer/solution diagrams work
- [ ] Database stores diagram JSON correctly

### Integration
- [ ] PDF scanning extracts questions with diagrams
- [ ] Question generation creates questions (with/without diagrams)
- [ ] PYQ solutions generation works
- [ ] No build errors
- [ ] No runtime errors

## Build and Test

```bash
npm run build
# Should complete without errors

# Test in development
npm run dev
```

## Key Points

1. **LaTeX cleaning is CRITICAL** - Must be applied everywhere LaTeX is used
2. **Diagrams are OPTIONAL** - Not all questions need them
3. **Database supports diagrams** - But questions without diagrams work fine
4. **Backward compatible** - Existing questions without diagrams continue to work
5. **Clean code first, then display** - Always clean LaTeX before rendering

## Example: Dice Question with Diagrams

```json
{
  "question_statement": "Three different views of a dice are shown. The piece of paper that can be folded to make this dice is:",
  "question_type": "MCQ",
  "diagram_json": [[
    {"type":"rectangle","x":50,"y":50,"width":60,"height":60,"strokeColor":"#1e40af","strokeWidth":2},
    {"type":"text","x":75,"y":65,"text":"5","fontSize":24,"strokeColor":"#1e40af"},
    {"type":"text","x":75,"y":85,"text":"4","fontSize":20,"strokeColor":"#1e40af"},
    {"type":"text","x":70,"y":95,"text":"1","fontSize":16,"strokeColor":"#1e40af"},

    {"type":"rectangle","x":150,"y":50,"width":60,"height":60,"strokeColor":"#1e40af","strokeWidth":2},
    {"type":"text","x":175,"y":65,"text":"4","fontSize":24,"strokeColor":"#1e40af"},
    {"type":"text","x":170,"y":85,"text":"6","fontSize":20,"strokeColor":"#1e40af"},
    {"type":"text","x":175,"y":95,"text":"3","fontSize":16,"strokeColor":"#1e40af"},

    {"type":"rectangle","x":250,"y":50,"width":60,"height":60,"strokeColor":"#1e40af","strokeWidth":2},
    {"type":"text","x":275,"y":65,"text":"2","fontSize":24,"strokeColor":"#1e40af"},
    {"type":"text","x":270,"y":85,"text":"6","fontSize":20,"strokeColor":"#1e40af"},
    {"type":"text","x":275,"y":95,"text":"5","fontSize":16,"strokeColor":"#1e40af"}
  ]],
  "options": ["Option A", "Option B", "Option C", "Option D"],
  "options_diagrams": [
    [/* Diagram for option A - net of dice */],
    [/* Diagram for option B */],
    [/* Diagram for option C */],
    [/* Diagram for option D */]
  ],
  "answer": "A"
}
```

## Support

If you encounter issues:
1. Check browser console for errors
2. Verify database migration applied successfully
3. Ensure all imports are correct
4. Test LaTeX cleaning in isolation
5. Verify diagram JSON format is valid

All implementation files are ready. Follow steps 1-5 above to complete the integration.
