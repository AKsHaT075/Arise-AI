# Design Document: Arise AI (AI for Runtime Debugging)

## Overview

Arise AI is an offline-first mobile application that helps beginner programmers in Bharat learn debugging through visual, interactive experiences. The system architecture prioritizes on-device processing for core functionality (OCR, code parsing, AR visualization) while leveraging Amazon Bedrock for advanced AI-powered error analysis when connectivity is available.

The application follows a layered architecture with clear separation between:
- **Presentation Layer**: Camera UI, AR overlays, and user interactions
- **Processing Layer**: OCR, code parsing, and execution simulation
- **Intelligence Layer**: AI-powered error detection and explanation generation
- **Storage Layer**: Local caching of explanations and learning progress

Key design principles:
- **Offline-first**: Core features work without internet
- **Graceful degradation**: Smooth fallback when AI services unavailable
- **Resource-efficient**: Optimized for low-end Android devices
- **Privacy-focused**: Minimal data transmission, local storage priority

## Architecture

### High-Level System Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                     Mobile Application                       │
│  ┌────────────────────────────────────────────────────────┐ │
│  │              Presentation Layer                        │ │
│  │  ┌──────────┐  ┌──────────┐  ┌──────────────────────┐ │ │
│  │  │ Camera   │  │   AR     │  │  UI Components       │ │ │
│  │  │   UI     │  │ Overlay  │  │  (Error Display,     │ │ │
│  │  │          │  │          │  │   Progress, etc.)    │ │ │
│  │  └──────────┘  └──────────┘  └──────────────────────┘ │ │
│  └────────────────────────────────────────────────────────┘ │
│                            ↕                                 │
│  ┌────────────────────────────────────────────────────────┐ │
│  │              Processing Layer                          │ │
│  │  ┌──────────┐  ┌──────────┐  ┌──────────────────────┐ │ │
│  │  │   OCR    │  │   Code   │  │    Execution         │ │ │
│  │  │  Engine  │→ │  Parser  │→ │    Simulator         │ │ │
│  │  │(ML Kit)  │  │          │  │                      │ │ │
│  │  └──────────┘  └──────────┘  └──────────────────────┘ │ │
│  └────────────────────────────────────────────────────────┘ │
│                            ↕                                 │
│  ┌────────────────────────────────────────────────────────┐ │
│  │           Intelligence Layer (Hybrid)                  │ │
│  │  ┌──────────────────┐  ┌──────────────────────────┐   │ │
│  │  │  Local Error     │  │   AI Tutor               │   │ │
│  │  │  Detection       │  │   (Amazon Bedrock)       │   │ │
│  │  │  (Rule-based)    │  │   [Online Only]          │   │ │
│  │  └──────────────────┘  └──────────────────────────┘   │ │
│  └────────────────────────────────────────────────────────┘ │
│                            ↕                                 │
│  ┌────────────────────────────────────────────────────────┐ │
│  │              Storage Layer                             │ │
│  │  ┌──────────────┐  ┌──────────────┐  ┌─────────────┐ │ │
│  │  │   Cached     │  │   Learning   │  │   Booklet   │ │ │
│  │  │ Explanations │  │   Progress   │  │   Metadata  │ │ │
│  │  │  (SQLite)    │  │  (SQLite)    │  │  (SQLite)   │ │ │
│  │  └──────────────┘  └──────────────┘  └─────────────┘ │ │
│  └────────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────┘
                            ↕
                    [Internet Available]
                            ↕
              ┌──────────────────────────┐
              │    Amazon Bedrock API    │
              │  (Claude/Titan Models)   │
              └──────────────────────────┘
```

### Component Interaction Flow

**Offline Flow (No Internet):**
```
Camera → OCR → Parser → Local Error Detection → AR Visualizer → Display
                                ↓
                         Cached Explanations
```

**Online Flow (Internet Available):**
```
Camera → OCR → Parser → Amazon Bedrock → Enhanced Explanations → Cache → Display
                                ↓
                         AR Visualizer
```

### Technology Stack

- **Mobile Framework**: Unity 2022 LTS with C# (cross-platform support, strong AR capabilities)
- **OCR**: Google ML Kit Text Recognition v2 (on-device, supports multiple languages)
- **Code Parsing**: ANTLR4 runtime for mobile (Python, JavaScript, Java grammars)
- **AI Service**: Amazon Bedrock (Claude 3 Haiku for cost efficiency)
- **AR Framework**: AR Foundation 5.x (Unity's cross-platform AR solution)
- **Local Storage**: SQLite (lightweight, embedded database)
- **Voice**: Android TextToSpeech API (native, no additional dependencies)
- **Networking**: AWS SDK for .NET (Bedrock API integration)

## Components and Interfaces

### 1. Camera and Image Capture Component

**Responsibility**: Capture high-quality images of printed code snippets

**Interface**:
```csharp
interface ICameraController {
    // Start camera preview
    void StartCamera();
    
    // Capture image when user taps capture button
    Task<CapturedImage> CaptureImageAsync();
    
    // Provide real-time feedback on image quality
    ImageQualityFeedback GetImageQuality(CapturedImage image);
    
    // Stop camera and release resources
    void StopCamera();
}

class CapturedImage {
    byte[] ImageData;
    int Width;
    int Height;
    DateTime CaptureTime;
    ImageQualityFeedback Quality;
}

enum ImageQualityFeedback {
    Good,           // Clear, well-lit, in focus
    TooBlurry,      // Motion blur or out of focus
    TooDark,        // Insufficient lighting
    TooSkewed       // Angle too extreme
}
```

**Implementation Notes**:
- Use Unity's WebCamTexture for camera access
- Apply real-time blur detection using Laplacian variance
- Provide visual guides (overlay frame) to help users align code
- Auto-focus on center region where code is expected

### 2. OCR Engine Component

**Responsibility**: Extract text from captured images using on-device ML

**Interface**:
```csharp
interface IOCREngine {
    // Initialize ML Kit text recognizer
    Task InitializeAsync();
    
    // Process image and extract text
    Task<OCRResult> RecognizeTextAsync(CapturedImage image);
    
    // Check if OCR engine is ready
    bool IsReady();
}

class OCRResult {
    string ExtractedText;
    List<TextBlock> TextBlocks;  // Structured text with positions
    float ConfidenceScore;       // Overall confidence (0-1)
    bool Success;
    string ErrorMessage;
}

class TextBlock {
    string Text;
    Rectangle BoundingBox;       // Position in image
    float Confidence;
    int LineNumber;              // Detected line number
}
```

**Implementation Notes**:
- Use Google ML Kit Text Recognition v2 via Unity plugin
- Preprocess image: grayscale conversion, contrast enhancement
- Post-process: remove common OCR artifacts (l→1, O→0 in code context)
- Maintain confidence threshold of 0.7 for acceptance

### 3. Code Parser Component

**Responsibility**: Parse extracted text into structured code representation

**Interface**:
```csharp
interface ICodeParser {
    // Detect programming language from code text
    ProgrammingLanguage DetectLanguage(string code);
    
    // Parse code into abstract syntax tree
    ParseResult Parse(string code, ProgrammingLanguage language);
    
    // Validate syntax and detect basic errors
    List<SyntaxError> ValidateSyntax(ParseResult parseResult);
}

class ParseResult {
    AbstractSyntaxTree AST;
    ProgrammingLanguage Language;
    bool ParseSuccessful;
    List<SyntaxError> Errors;
}

class AbstractSyntaxTree {
    ASTNode Root;
    Dictionary<int, ASTNode> LineToNodeMap;  // Map line numbers to AST nodes
}

class ASTNode {
    NodeType Type;              // Variable, Loop, Conditional, Function, etc.
    int StartLine;
    int EndLine;
    List<ASTNode> Children;
    Dictionary<string, object> Attributes;  // Language-specific attributes
}

enum NodeType {
    Program, Function, Variable, Assignment,
    IfStatement, WhileLoop, ForLoop,
    Expression, FunctionCall, Return
}

class SyntaxError {
    int LineNumber;
    int ColumnNumber;
    string ErrorMessage;
    ErrorSeverity Severity;
    string SuggestedFix;
}

enum ErrorSeverity {
    Error,      // Prevents execution
    Warning     // Suspicious but valid
}
```

**Implementation Notes**:
- Use ANTLR4 runtime with pre-built grammars for Python, JavaScript, Java
- Language detection: keyword frequency analysis (def/class→Python, function/const→JavaScript, public/class→Java)
- Build line-to-node mapping for efficient error highlighting
- Cache parsed ASTs to avoid re-parsing during visualization

### 4. Local Error Detection Component

**Responsibility**: Detect common errors using rule-based patterns (offline capability)

**Interface**:
```csharp
interface ILocalErrorDetector {
    // Detect errors using local rules
    List<DetectedError> DetectErrors(ParseResult parseResult);
    
    // Get cached explanation for known error pattern
    string GetCachedExplanation(DetectedError error);
}

class DetectedError {
    ErrorType Type;
    int LineNumber;
    string ErrorCode;           // e.g., "SYNTAX_001", "LOGIC_005"
    string ShortDescription;
    ErrorSeverity Severity;
    string CodeSnippet;         // The problematic code
}

enum ErrorType {
    SyntaxError,
    IndentationError,
    UndefinedVariable,
    TypeMismatch,
    InfiniteLoop,
    UnreachableCode,
    MissingReturn
}
```

**Implementation Notes**:
- Implement pattern matching for common beginner errors:
  - Missing colons in Python (if, while, def)
  - Undefined variables (use before declaration)
  - Indentation errors
  - Mismatched brackets/parentheses
- Store error patterns in local database with generic explanations
- Prioritize errors by severity and line number

### 5. AI Tutor Component (Amazon Bedrock Integration)

**Responsibility**: Generate detailed, personalized error explanations using AI

**Interface**:
```csharp
interface IAITutor {
    // Check if internet connectivity is available
    Task<bool> IsOnlineAsync();
    
    // Analyze code and generate explanations
    Task<AIAnalysisResult> AnalyzeCodeAsync(
        string code,
        List<DetectedError> localErrors,
        LearnerContext context
    );
    
    // Generate explanation for specific error
    Task<string> ExplainErrorAsync(
        DetectedError error,
        string codeContext,
        LearnerContext context
    );
}

class AIAnalysisResult {
    List<EnhancedError> Errors;
    string OverallFeedback;
    List<string> LearningTips;
    bool FromCache;             // True if served from cache
}

class EnhancedError {
    DetectedError BaseError;
    string DetailedExplanation;  // Natural language explanation
    string WhyItMatters;         // Educational context
    string HowToFix;             // Step-by-step fix
    List<string> RelatedConcepts; // Links to learning resources
    string PersonalizedNote;     // Based on learner history
}

class LearnerContext {
    string LearnerId;
    List<string> PreviousErrors;  // Error codes from history
    int ExperienceLevel;          // 1-5 scale
    ProgrammingLanguage PreferredLanguage;
}
```

**Implementation Notes**:
- Use Amazon Bedrock with Claude 3 Haiku model (cost-effective, fast)
- Prompt engineering:
  ```
  You are a patient programming tutor for beginners in India.
  Explain this error in simple terms:
  
  Code: {code_snippet}
  Error: {error_description}
  Learner Level: {experience_level}
  Previous Mistakes: {error_history}
  
  Provide:
  1. What went wrong (simple language)
  2. Why it matters
  3. How to fix it (step-by-step)
  4. One tip to avoid this in future
  ```
- Cache responses using error code + code hash as key
- Implement exponential backoff for API failures
- Batch multiple errors into single API call when possible

### 6. Execution Simulator Component

**Responsibility**: Simulate code execution step-by-step for visualization

**Interface**:
```csharp
interface IExecutionSimulator {
    // Simulate code execution
    SimulationResult Simulate(ParseResult parseResult);
    
    // Get execution state at specific step
    ExecutionState GetStateAtStep(SimulationResult result, int stepIndex);
}

class SimulationResult {
    List<ExecutionStep> Steps;
    bool CompletedSuccessfully;
    ExecutionError RuntimeError;  // If execution failed
}

class ExecutionStep {
    int StepNumber;
    int CurrentLine;
    Dictionary<string, VariableState> Variables;  // Variable name → state
    string ControlFlowInfo;      // e.g., "Entering loop iteration 2"
    StackFrame CurrentFrame;     // For function calls
}

class VariableState {
    string Name;
    object Value;
    string Type;
    bool JustChanged;            // Highlight in AR if true
}

class StackFrame {
    string FunctionName;
    Dictionary<string, VariableState> LocalVariables;
    int ReturnLine;
}
```

**Implementation Notes**:
- Implement lightweight interpreter for supported languages
- Limit execution to 1000 steps (prevent infinite loops)
- Track variable changes at each step for AR visualization
- Handle common runtime errors gracefully (division by zero, index out of bounds)

### 7. AR Visualizer Component

**Responsibility**: Overlay execution flow and variable states on scanned code

**Interface**:
```csharp
interface IARVisualizer {
    // Initialize AR session
    Task InitializeARAsync();
    
    // Start AR visualization for code
    void StartVisualization(
        CapturedImage codeImage,
        SimulationResult simulation,
        List<DetectedError> errors
    );
    
    // Control playback
    void PlaySimulation();
    void PauseSimulation();
    void StepForward();
    void StepBackward();
    void SetPlaybackSpeed(float speed);  // 0.5x to 2.0x
    
    // Highlight specific error
    void HighlightError(DetectedError error);
    
    // Stop AR and release resources
    void StopVisualization();
}

class AROverlay {
    // Visual elements to render
    List<LineHighlight> ErrorHighlights;
    List<VariableDisplay> VariableDisplays;
    List<FlowArrow> ControlFlowArrows;
    CurrentLineIndicator ExecutionPointer;
}

class LineHighlight {
    int LineNumber;
    Color HighlightColor;       // Red for errors, yellow for warnings
    float Opacity;
}

class VariableDisplay {
    string VariableName;
    string CurrentValue;
    Vector2 Position;           // Position relative to code
    bool IsAnimating;           // Animate when value changes
}

class FlowArrow {
    Vector2 StartPosition;
    Vector2 EndPosition;
    ArrowType Type;             // Loop, Conditional, Function call
}

enum ArrowType {
    SequentialFlow,
    ConditionalTrue,
    ConditionalFalse,
    LoopIteration,
    FunctionCall,
    FunctionReturn
}
```

**Implementation Notes**:
- Use AR Foundation's image tracking to anchor overlays to printed code
- Render overlays using Unity UI Canvas in world space
- Implement smooth animations for variable changes (0.3s duration)
- Provide fallback 2D mode if AR not supported (overlay on static image)
- Use color coding: Red (errors), Yellow (warnings), Green (current line), Blue (variables)

### 8. Cache Manager Component

**Responsibility**: Manage local storage of AI explanations and learning data

**Interface**:
```csharp
interface ICacheManager {
    // Cache AI-generated explanation
    Task CacheExplanationAsync(string errorCode, string codeHash, string explanation);
    
    // Retrieve cached explanation
    Task<string> GetCachedExplanationAsync(string errorCode, string codeHash);
    
    // Check if explanation exists in cache
    Task<bool> HasCachedExplanationAsync(string errorCode, string codeHash);
    
    // Store learner progress
    Task SaveProgressAsync(string learnerId, LearningProgress progress);
    
    // Retrieve learner progress
    Task<LearningProgress> GetProgressAsync(string learnerId);
    
    // Clear old cache entries (LRU eviction)
    Task CleanupCacheAsync(int maxEntries);
}

class LearningProgress {
    string LearnerId;
    List<CompletedLesson> CompletedLessons;
    Dictionary<string, int> ErrorHistory;  // Error code → occurrence count
    int TotalErrorsFixed;
    DateTime LastActive;
}

class CompletedLesson {
    string LessonId;
    DateTime CompletedDate;
    int AttemptsRequired;
    List<string> ErrorsEncountered;
}
```

**Implementation Notes**:
- Use SQLite for local storage (lightweight, no server required)
- Schema:
  ```sql
  CREATE TABLE cached_explanations (
      error_code TEXT,
      code_hash TEXT,
      explanation TEXT,
      created_at INTEGER,
      access_count INTEGER,
      PRIMARY KEY (error_code, code_hash)
  );
  
  CREATE TABLE learning_progress (
      learner_id TEXT PRIMARY KEY,
      progress_json TEXT,
      last_updated INTEGER
  );
  
  CREATE TABLE error_history (
      learner_id TEXT,
      error_code TEXT,
      occurrence_count INTEGER,
      last_seen INTEGER,
      PRIMARY KEY (learner_id, error_code)
  );
  ```
- Implement LRU cache eviction (keep 500 most recent explanations)
- Hash code using SHA-256 (first 16 chars) for cache key

### 9. Voice Assistant Component

**Responsibility**: Provide text-to-speech for explanations

**Interface**:
```csharp
interface IVoiceAssistant {
    // Initialize TTS engine
    Task InitializeAsync();
    
    // Speak explanation text
    Task SpeakAsync(string text);
    
    // Control playback
    void Pause();
    void Resume();
    void Stop();
    
    // Set speech parameters
    void SetSpeechRate(float rate);  // 0.5 to 2.0
    void SetLanguage(string languageCode);  // "en-IN", "hi-IN"
    
    // Check if TTS is available
    bool IsAvailable();
}
```

**Implementation Notes**:
- Use Android TextToSpeech API via Unity plugin
- Default to Indian English ("en-IN") for accent familiarity
- Support Hindi ("hi-IN") for regional language learners
- Break long explanations into sentences for better pacing
- Highlight corresponding text while speaking

### 10. Connectivity Manager Component

**Responsibility**: Monitor network status and manage online/offline transitions

**Interface**:
```csharp
interface IConnectivityManager {
    // Check current connectivity status
    bool IsOnline();
    
    // Register callback for connectivity changes
    void RegisterConnectivityCallback(Action<bool> onConnectivityChanged);
    
    // Unregister callback
    void UnregisterConnectivityCallback(Action<bool> callback);
    
    // Get connection quality
    ConnectionQuality GetConnectionQuality();
}

enum ConnectionQuality {
    Offline,
    Poor,      // High latency, low bandwidth
    Fair,      // Moderate latency
    Good       // Low latency, high bandwidth
}
```

**Implementation Notes**:
- Use Unity's Application.internetReachability
- Implement ping test to AWS endpoint for quality assessment
- Notify all dependent components when connectivity changes
- Queue AI requests when offline, process when online

## Data Models

### Code Representation

```csharp
class CodeDocument {
    string DocumentId;           // UUID
    string SourceText;           // Original extracted text
    ProgrammingLanguage Language;
    ParseResult ParsedCode;
    List<DetectedError> Errors;
    SimulationResult Simulation;
    DateTime CreatedAt;
    string BookletLessonId;      // Reference to booklet lesson
}
```

### Error Representation

```csharp
class ErrorRecord {
    string ErrorId;              // UUID
    string ErrorCode;            // Standardized code (e.g., "PY_SYNTAX_001")
    ErrorType Type;
    int LineNumber;
    string CodeSnippet;
    string ShortDescription;
    string DetailedExplanation;
    string HowToFix;
    List<string> RelatedConcepts;
    bool FromAI;                 // True if from Bedrock, false if local
    DateTime DetectedAt;
}
```

### Learner Profile

```csharp
class LearnerProfile {
    string LearnerId;            // UUID
    string DisplayName;
    int ExperienceLevel;         // 1 (beginner) to 5 (advanced)
    ProgrammingLanguage PreferredLanguage;
    List<string> CompletedLessonIds;
    Dictionary<string, ErrorStats> ErrorStatistics;
    UserPreferences Preferences;
    DateTime CreatedAt;
    DateTime LastActiveAt;
}

class ErrorStats {
    string ErrorCode;
    int TotalOccurrences;
    int SuccessfulFixes;
    DateTime FirstSeen;
    DateTime LastSeen;
    bool Mastered;               // True if not seen in last 10 sessions
}

class UserPreferences {
    bool VoiceEnabled;
    float VoiceSpeed;
    string VoiceLanguage;
    bool AREnabled;
    float SimulationSpeed;
    bool ShowHintsImmediately;
}
```

### Booklet Metadata

```csharp
class BookletLesson {
    string LessonId;
    string Title;
    int ChapterNumber;
    int LessonNumber;
    ProgrammingLanguage Language;
    DifficultyLevel Difficulty;
    List<string> LearningObjectives;
    string ExpectedCode;         // The correct code for this lesson
    List<string> CommonMistakes; // Error codes commonly seen
    string HintText;             // Offline hint if AI unavailable
}

enum DifficultyLevel {
    Beginner,
    Intermediate,
    Advanced
}
```

## Correctness Properties

*A property is a characteristic or behavior that should hold true across all valid executions of a system—essentially, a formal statement about what the system should do. Properties serve as the bridge between human-readable specifications and machine-verifiable correctness guarantees.*


### Property Reflection

After analyzing all acceptance criteria, I've identified several areas where properties can be consolidated:

**Offline Operation**: Requirements 1.5, 2.5, 3.5, 6.6, 8.5 all test offline operation of different components. These can be consolidated into comprehensive offline operation properties.

**Performance Requirements**: Requirements 2.2, 4.4, 12.2, 12.3 all test performance bounds. These should remain separate as they test different aspects (OCR speed, API response, memory usage).

**Error Highlighting**: Requirements 5.1, 5.2, 5.3, 5.4 all relate to error display. Requirements 5.1 and 5.4 can be combined into one property about persistent highlighting.

**Cache Behavior**: Requirements 14.1, 14.2, 14.3 all test caching. These can be consolidated into properties about cache-first behavior.

**Progress Tracking**: Requirements 18.1, 18.2, 18.3, 18.4 all relate to progress. Requirements 18.1 and 18.2 can be combined into a general tracking property.

### Correctness Properties

Property 1: Image Capture Produces Valid Output
*For any* code snippet positioned in front of the camera, when the capture button is pressed, the system should produce a CapturedImage object with non-null ImageData and valid dimensions.
**Validates: Requirements 1.2, 1.3**

Property 2: Image Quality Assessment Accuracy
*For any* captured image, if the image is blurry, dark, or skewed beyond acceptable thresholds, the system should classify it as insufficient quality and prompt for recapture.
**Validates: Requirements 1.4**

Property 3: OCR Extraction Completeness
*For any* captured image containing printed code, the OCR engine should extract text that preserves the line structure and indentation of the original code.
**Validates: Requirements 2.1, 2.3**

Property 4: OCR Performance Bound
*For any* captured image, OCR processing should complete within 5 seconds on devices meeting minimum specifications (2GB RAM).
**Validates: Requirements 2.2, 12.2**

Property 5: Code Parsing Produces Valid AST
*For any* syntactically valid code text, the parser should produce an AbstractSyntaxTree with a non-null Root node and a complete LineToNodeMap covering all code lines.
**Validates: Requirements 3.1, 3.4**

Property 6: Language Construct Identification
*For any* code containing variables, loops, conditionals, or functions, the parser should create corresponding ASTNode objects with the correct NodeType for each construct.
**Validates: Requirements 3.2**

Property 7: Syntax Error Detection
*For any* code containing syntax errors (missing colons, unmatched brackets, invalid indentation), the parser should flag each error with the correct line number and error type.
**Validates: Requirements 3.3**

Property 8: Error Classification Completeness
*For any* detected error, the system should assign both an ErrorType and an ErrorSeverity classification.
**Validates: Requirements 4.2, 4.3**

Property 9: AI Analysis Performance Bound
*For any* code submitted for AI analysis when online, the system should receive a response from Amazon Bedrock within 5 seconds or timeout gracefully.
**Validates: Requirements 4.4**

Property 10: Error Highlighting Persistence
*For any* code with detected errors, all faulty lines should remain highlighted with appropriate colors until the code is rescanned or the session ends.
**Validates: Requirements 5.1, 5.4**

Property 11: Error Type Visual Distinction
*For any* code with multiple errors of different types, each error type should have a visually distinct indicator (different colors or patterns).
**Validates: Requirements 5.2**

Property 12: Error Detail Interaction
*For any* highlighted error line, tapping on it should display detailed error information including the error message and suggested fix.
**Validates: Requirements 5.3**

Property 13: Rescan Updates Highlighting
*For any* code that is scanned, modified, and rescanned, the error highlighting should reflect only the errors present in the rescanned version (round-trip property).
**Validates: Requirements 5.5**

Property 14: Execution Step Sequencing
*For any* simulated code execution, the execution steps should be ordered by increasing step number, and each step should reference a valid line number from the code.
**Validates: Requirements 6.2**

Property 15: Variable State Tracking
*For any* code execution simulation, at each execution step where a variable changes value, the AR visualizer should display the updated value.
**Validates: Requirements 6.3**

Property 16: Loop Iteration Visualization
*For any* code containing loops, the AR visualizer should display iteration count that increments by 1 for each loop iteration and resets when the loop exits.
**Validates: Requirements 6.4**

Property 17: Conditional Branch Highlighting
*For any* code containing conditional statements, the AR visualizer should highlight exactly one branch (true or false) based on the condition evaluation.
**Validates: Requirements 6.5**

Property 18: AI Explanation Structure
*For any* error analyzed by the AI tutor, the generated explanation should contain at least three sections: what went wrong, why it matters, and how to fix it.
**Validates: Requirements 7.2**

Property 19: Voice Playback Completeness
*For any* explanation text, when voice mode is activated, the TTS engine should vocalize the complete text without truncation.
**Validates: Requirements 8.2**

Property 20: Voice Speed Adjustment
*For any* speech rate setting between 0.5x and 2.0x, the TTS engine should adjust playback speed proportionally to the setting.
**Validates: Requirements 8.4**

Property 21: Error Recording Completeness
*For any* detected error, the Learning Tracker should create a record containing the error code, timestamp, and learner ID.
**Validates: Requirements 9.1**

Property 22: Repeated Error Pattern Detection
*For any* error code that occurs 3 or more times for the same learner, the system should identify it as a learning gap.
**Validates: Requirements 9.2**

Property 23: Learning Gap Enhanced Explanations
*For any* identified learning gap, the AI tutor should provide explanations that include additional conceptual context beyond the standard error explanation.
**Validates: Requirements 9.3**

Property 24: Progress Marking on Success
*For any* error that was previously marked as a learning gap, when the learner successfully fixes it, the system should update the error statistics to reflect the successful fix.
**Validates: Requirements 9.4**

Property 25: Offline Core Functionality
*For any* code scanning session with network connectivity disabled, the system should successfully complete OCR, parsing, basic error detection, and AR visualization without errors or crashes.
**Validates: Requirements 10.1, 10.2, 1.5, 2.5, 3.5, 6.6, 8.5**

Property 26: Offline Cache Retrieval
*For any* error pattern that has been previously cached, when the same error is encountered offline, the system should display the cached explanation without attempting network access.
**Validates: Requirements 10.3**

Property 27: Graceful API Failure Handling
*For any* API call failure (timeout, error response, network loss), the system should display an appropriate message and fall back to cached content or local capabilities without crashing.
**Validates: Requirements 11.1, 11.2, 11.5**

Property 28: Component Failure Isolation
*For any* single component failure (OCR, parser, AR, voice), the remaining functional components should continue operating normally.
**Validates: Requirements 11.4**

Property 29: Memory Usage Bound
*For any* normal operation session (scan, analyze, visualize), the application should maintain memory usage below 200MB.
**Validates: Requirements 12.3**

Property 30: API Payload Minimization
*For any* code sent to Amazon Bedrock, the payload should contain only the code text and error context, excluding learner history and cached data.
**Validates: Requirements 13.3**

Property 31: Cache-First API Strategy
*For any* error with a matching cache entry (same error code and code hash), the system should return the cached explanation without making an API call.
**Validates: Requirements 14.1, 14.2**

Property 32: API Request Batching
*For any* code with multiple errors detected simultaneously, the system should batch all errors into a single API request rather than making separate requests per error.
**Validates: Requirements 14.3**

Property 33: Booklet Lesson Identification
*For any* scanned booklet page containing a lesson identifier (QR code or marker), the system should correctly extract the lesson ID and retrieve corresponding metadata.
**Validates: Requirements 15.1, 15.2**

Property 34: Context-Appropriate Hints
*For any* identified lesson, the hints provided should reference concepts and terminology specific to that lesson's learning objectives.
**Validates: Requirements 15.3**

Property 35: Difficulty Level Recognition
*For any* booklet lesson, the system should assign a difficulty level (Beginner, Intermediate, Advanced) that matches the lesson's chapter progression.
**Validates: Requirements 15.5**

Property 36: Language Detection Accuracy
*For any* code text containing language-specific keywords, the system should correctly identify the programming language (Python, JavaScript, or Java).
**Validates: Requirements 17.3**

Property 37: Language-Specific Error Rules
*For any* detected programming language, the error detection should apply syntax rules specific to that language (e.g., indentation for Python, semicolons for JavaScript).
**Validates: Requirements 17.4**

Property 38: Language-Appropriate Explanations
*For any* AI-generated explanation, the text should reference syntax and concepts specific to the detected programming language.
**Validates: Requirements 17.5**

Property 39: Error Count Tracking
*For any* learner session, the system should maintain an accurate count of errors debugged that increments by 1 for each successfully fixed error.
**Validates: Requirements 18.1**

Property 40: Lesson Completion Tracking
*For any* completed booklet lesson, the system should mark it as complete in the learner's progress data and include it in the completed lessons list.
**Validates: Requirements 18.2**

Property 41: Progress Dashboard Accuracy
*For any* learner, the progress dashboard should display data that matches the stored progress records (completed lessons, error counts, skill improvements).
**Validates: Requirements 18.3**

Property 42: Milestone Feedback Triggering
*For any* learner achievement milestone (10 errors fixed, 5 lessons completed, etc.), the system should display encouraging feedback within the same session.
**Validates: Requirements 18.4**

Property 43: Critical Error Prioritization
*For any* code with multiple errors, when displaying errors, syntax errors should appear before warnings, and errors on earlier lines should appear before errors on later lines.
**Validates: Requirements 16.4**

## Error Handling

### Error Categories

The system handles four categories of errors:

1. **User Errors**: Errors in scanned code (syntax, logic)
   - Detection: Parser + AI analysis
   - Response: Highlight, explain, guide to fix
   - Recovery: User rescans corrected code

2. **System Errors**: Internal failures (OCR failure, parsing crash)
   - Detection: Try-catch blocks, validation checks
   - Response: Log error, show user-friendly message
   - Recovery: Graceful degradation, retry option

3. **Network Errors**: Connectivity issues, API failures
   - Detection: HTTP status codes, timeouts
   - Response: Fall back to offline mode, use cache
   - Recovery: Auto-retry when connectivity restored

4. **Resource Errors**: Low memory, insufficient storage
   - Detection: Memory monitoring, storage checks
   - Response: Reduce quality (2D instead of AR), clear cache
   - Recovery: Prompt user to free resources

### Error Handling Patterns

**OCR Failures**:
```csharp
try {
    OCRResult result = await ocrEngine.RecognizeTextAsync(image);
    if (result.ConfidenceScore < 0.7) {
        ShowMessage("Image quality too low. Please recapture.");
        return;
    }
    ProcessExtractedText(result.ExtractedText);
} catch (OCRException ex) {
    LogError("OCR failed", ex);
    ShowMessage("Unable to read code. Please try again.");
    OfferManualInput();
}
```

**API Failures with Retry**:
```csharp
async Task<AIAnalysisResult> AnalyzeWithRetry(string code) {
    int maxRetries = 1;
    for (int attempt = 0; attempt <= maxRetries; attempt++) {
        try {
            return await bedrockClient.AnalyzeAsync(code);
        } catch (NetworkException ex) when (attempt < maxRetries) {
            await Task.Delay(1000);  // Wait before retry
            continue;
        } catch (Exception ex) {
            LogError("AI analysis failed", ex);
            return FallbackToCache(code);
        }
    }
    return FallbackToCache(code);
}
```

**Graceful Degradation**:
```csharp
void StartVisualization(CodeDocument code) {
    if (ARFoundation.IsSupported() && deviceHasGPU) {
        StartARVisualization(code);
    } else if (deviceHasGPU) {
        Start2DVisualization(code);
    } else {
        StartTextOnlyVisualization(code);
    }
}
```

**Memory Management**:
```csharp
void OnLowMemoryWarning() {
    // Clear non-essential caches
    imageCache.Clear();
    
    // Reduce AR quality
    if (arVisualizer.IsActive) {
        arVisualizer.SetQualityLevel(QualityLevel.Low);
    }
    
    // Force garbage collection
    System.GC.Collect();
    
    // Warn user if still critical
    if (GetMemoryUsage() > 180MB) {
        ShowMessage("Low memory. Some features may be limited.");
    }
}
```

### Offline Mode Behavior

**Feature Availability Matrix**:

| Feature | Online | Offline |
|---------|--------|---------|
| Code Scanning | ✓ | ✓ |
| OCR | ✓ | ✓ |
| Code Parsing | ✓ | ✓ |
| Basic Error Detection | ✓ | ✓ |
| AR Visualization | ✓ | ✓ |
| Voice Explanations | ✓ | ✓ |
| AI Error Analysis | ✓ | ✗ (cached only) |
| Detailed Explanations | ✓ | ✗ (cached only) |
| Personalized Feedback | ✓ | ✗ (cached only) |

**Connectivity State Management**:
```csharp
class ConnectivityStateMachine {
    State currentState = State.Unknown;
    
    enum State {
        Unknown,
        Online,
        Offline,
        Transitioning
    }
    
    void OnConnectivityChanged(bool isOnline) {
        State previousState = currentState;
        currentState = isOnline ? State.Online : State.Offline;
        
        if (previousState == State.Offline && currentState == State.Online) {
            // Transitioned to online
            EnableAIFeatures();
            ProcessQueuedRequests();
            ShowNotification("AI features now available");
        } else if (previousState == State.Online && currentState == State.Offline) {
            // Transitioned to offline
            DisableAIFeatures();
            ShowNotification("Working offline - basic features available");
        }
    }
}
```

## Testing Strategy

### Dual Testing Approach

The testing strategy combines unit testing and property-based testing to ensure comprehensive coverage:

**Unit Tests**: Focus on specific examples, edge cases, and integration points
- Specific code snippets with known errors
- Edge cases (empty code, very long code, special characters)
- Component integration (OCR → Parser → Error Detector)
- UI interactions (button clicks, gestures)
- Error conditions (API failures, low memory)

**Property-Based Tests**: Verify universal properties across randomized inputs
- Generate random code snippets with various error patterns
- Test with random image quality variations
- Verify behavior across random network conditions
- Test with random learner histories and preferences
- Minimum 100 iterations per property test

### Property-Based Testing Configuration

**Framework**: Use **fast-check** (JavaScript/TypeScript) or **FsCheck** (C#/.NET) for property-based testing in Unity

**Test Configuration**:
```csharp
[Property(Iterations = 100)]
public Property OCRExtractsAllLines(CodeSnippet snippet) {
    // Feature: arise-ai-runtime-debugging, Property 3: OCR Extraction Completeness
    var image = RenderCodeToImage(snippet);
    var result = ocrEngine.RecognizeTextAsync(image).Result;
    
    return (result.TextBlocks.Count == snippet.LineCount)
        .Label($"Expected {snippet.LineCount} lines, got {result.TextBlocks.Count}");
}
```

**Generators**:
```csharp
// Generate random Python code snippets
Gen<CodeSnippet> PythonCodeGen = 
    from lines in Gen.Int[1, 20]
    from statements in Gen.ArrayOf(lines, StatementGen)
    select new CodeSnippet(statements, Language.Python);

// Generate random syntax errors
Gen<SyntaxError> SyntaxErrorGen =
    Gen.OneOf(
        Gen.Constant(new SyntaxError("Missing colon", ErrorType.SyntaxError)),
        Gen.Constant(new SyntaxError("Unmatched bracket", ErrorType.SyntaxError)),
        Gen.Constant(new SyntaxError("Invalid indentation", ErrorType.IndentationError))
    );
```

### Test Organization

**Unit Tests** (Examples and Edge Cases):
```
tests/
  unit/
    camera/
      - test_camera_activation.cs
      - test_image_capture.cs
      - test_quality_feedback.cs
    ocr/
      - test_ocr_basic.cs
      - test_ocr_edge_cases.cs (empty, special chars)
      - test_ocr_performance.cs
    parser/
      - test_python_parser.cs
      - test_javascript_parser.cs
      - test_error_detection.cs
    ai/
      - test_bedrock_integration.cs
      - test_cache_behavior.cs
      - test_offline_fallback.cs
    ar/
      - test_visualization.cs
      - test_execution_simulation.cs
    storage/
      - test_cache_manager.cs
      - test_progress_tracking.cs
```

**Property Tests**:
```
tests/
  properties/
    - test_ocr_properties.cs (Properties 3, 4)
    - test_parser_properties.cs (Properties 5, 6, 7)
    - test_error_detection_properties.cs (Properties 8, 10, 11, 12)
    - test_visualization_properties.cs (Properties 14, 15, 16, 17)
    - test_ai_properties.cs (Properties 18, 31, 32, 38)
    - test_offline_properties.cs (Properties 25, 26, 27)
    - test_tracking_properties.cs (Properties 21, 22, 23, 24, 39, 40, 41, 42)
    - test_performance_properties.cs (Properties 4, 9, 29)
```

### Integration Testing

**End-to-End Scenarios**:
1. **Happy Path**: Scan → OCR → Parse → Detect Error → Show Explanation → Visualize
2. **Offline Path**: Scan → OCR → Parse → Local Detection → Cached Explanation
3. **Error Recovery**: Scan → OCR Fails → Retry → Success
4. **Network Transition**: Start Online → Lose Connection → Continue Offline → Reconnect

**Test Devices**:
- Low-end: 2GB RAM, basic GPU (minimum spec)
- Mid-range: 4GB RAM, moderate GPU (typical user)
- High-end: 8GB RAM, advanced GPU (optimal experience)

### Performance Testing

**Benchmarks**:
- OCR processing: < 5 seconds (low-end device)
- Code parsing: < 1 second (any device)
- AI analysis: < 5 seconds (with good connectivity)
- AR rendering: 30 FPS minimum (mid-range device)
- Memory usage: < 200MB (normal operation)
- App startup: < 3 seconds (cold start)

**Load Testing**:
- Cache with 500 explanations: retrieval < 100ms
- Learning history with 1000 errors: query < 200ms
- Simulate 100 concurrent API requests: success rate > 95%

### Manual Testing

**Usability Testing**:
- Test with actual beginner programmers
- Observe scanning behavior (alignment, lighting)
- Evaluate explanation clarity
- Assess AR visualization intuitiveness

**Accessibility Testing**:
- Test voice features with eyes-free usage
- Verify text size and contrast
- Test with screen readers (TalkBack)
- Verify touch target sizes (minimum 48dp)

### Continuous Integration

**CI Pipeline**:
1. Run unit tests (fast feedback)
2. Run property tests (comprehensive coverage)
3. Build for Android (APK generation)
4. Run integration tests on emulator
5. Performance benchmarks
6. Generate test coverage report (target: 80%+)

**Test Tags**:
- Each property test tagged with: `Feature: arise-ai-runtime-debugging, Property N: [property text]`
- Unit tests tagged with: `Requirements: X.Y`
- Integration tests tagged with: `Scenario: [scenario name]`

This testing strategy ensures both concrete correctness (unit tests) and general correctness (property tests), providing confidence that the system behaves correctly across all inputs while catching specific edge cases and integration issues.
