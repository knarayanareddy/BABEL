BABEL — Comprehensive System Design Document
Version 1.0 | Build Reference for Autonomous Agent Implementation
TABLE OF CONTENTS

    Project Overview & Vision
    System Architecture
    Repository Structure
    Core Data Models
    Module: Core Daemon (core/)
    Module: Accessibility Layer (accessibility/)
    Module: OCR Fallback Layer (ocr/)
    Module: Translation Engine (translation/)
    Module: Overlay Renderer (overlay/)
    Module: Profile & Config System (profiles/ + config/)
    Module: Settings UI (ui/)
    Inter-Process Communication (IPC)
    Event System
    Caching Strategy
    Pipeline: End-to-End Data Flow
    Per-OS Implementation Guide
    Performance Targets & Budgets
    Error Handling & Resilience
    Security & Privacy Architecture
    Testing Strategy
    Build Phases & Milestones
    Dependencies & Toolchain
    Configuration Reference (babel.toml)
    Glossary

1. Project Overview & Vision
1.1 What BABEL Is

BABEL is an always-on, OS-level, local-first universal translation layer for the desktop. It reads all visible text from any application running on the user's computer, translates it into the user's preferred language using locally-running neural machine translation (NMT) models, and renders the translated text in-place via a transparent pixel-aligned overlay — making the entire desktop appear to speak the user's language, in real-time, with zero cloud dependency and zero per-app configuration.
1.2 What BABEL Is Not

    It is not a browser extension (it works at OS level, across all apps).
    It is not a cloud translation service (all inference is local).
    It is not a "select a region" translation lens (it is automatic and system-wide).
    It is not editing foreign apps' source code or UI elements (it draws over them).
    It is not a replacement for professional translation tools.

1.3 Core Design Principles
#	Principle	Description
P1	Local-first	No text ever leaves the device. All NMT inference runs on local CPU/GPU.
P2	In-place rendering	Translated text appears at the exact pixel location of the original.
P3	Zero configuration by default	Works on any app without per-app setup.
P4	Event-driven, not polling	Use OS accessibility events wherever available; OCR only as fallback.
P5	Latency over perfection	A fast "good enough" translation is better than a slow perfect one.
P6	Stability over completeness	A stable overlay beats perfect font matching. No flicker.
P7	Layered degradation	AX/UIA → AT-SPI → OCR, falling gracefully at each tier.
P8	User agency	Users can see originals on hover, flag translations, and exclude apps.
1.4 Supported Platforms (Priority Order)

    Windows 10/11 (primary MVP target — best UIA support)
    macOS 12+ (secondary — good AX support)
    Linux / X11 (tertiary — AT-SPI + compositing constraints)
    Linux / Wayland (future — security model complicates overlays)

1.5 Supported Language Pairs

    Source: Any language supported by fastText lid.176 (176 languages)
    Target: Any language supported by NLLB-200 (200 languages) or OPUS-MT (per direction)
    Default: Auto-detect source → User's system locale as target

2. System Architecture
2.1 High-Level Component Diagram

text

┌───────────────────────────────────────────────────────────────────────┐
│                          USER DESKTOP                                 │
│                                                                       │
│   ┌──────────────┐   ┌──────────────┐   ┌──────────────────────────┐ │
│   │  Chrome.exe  │   │   Mail.app   │   │    任何应用 (any app)     │ │
│   └──────┬───────┘   └──────┬───────┘   └───────────┬──────────────┘ │
│          │                  │                        │                │
│          └──────────────────┴────────────────────────┘                │
│                             │  OS Accessibility Events / Screen       │
│                             ▼                                         │
│   ┌─────────────────────────────────────────────────────────────────┐ │
│   │                    BABEL DAEMON (Rust)                          │ │
│   │                                                                 │ │
│   │  ┌───────────┐  ┌──────────────┐  ┌───────────────────────┐   │ │
│   │  │  Screen   │  │   Text Run   │  │   Change Detector /   │   │ │
│   │  │  Reader   │  │   Registry   │  │   Event Dispatcher    │   │ │
│   │  └─────┬─────┘  └──────┬───────┘  └───────────────────────┘   │ │
│   │        │               │                                        │ │
│   │  ┌─────▼───────────────▼──────────────────────────────────┐    │ │
│   │  │              Accessibility Router                       │    │ │
│   │  │   [macOS AX]   [Windows UIA]   [Linux AT-SPI]   [OCR]  │    │ │
│   │  └─────────────────────────┬──────────────────────────────┘    │ │
│   │                            │  TextRun[]                         │ │
│   │                            ▼                                    │ │
│   │  ┌─────────────────────────────────────────────────────────┐   │ │
│   │  │              Translation Pipeline (Python/Rust FFI)     │   │ │
│   │  │   LangDetect → Cache Lookup → Batch Queue → NMT Engine  │   │ │
│   │  └─────────────────────────┬───────────────────────────────┘   │ │
│   │                            │  TranslatedRun[]                   │ │
│   │                            ▼                                    │ │
│   │  ┌─────────────────────────────────────────────────────────┐   │ │
│   │  │              Overlay Renderer (Rust + GPU)              │   │ │
│   │  │   LayoutPreserver → FontMatcher → TransparentWindow     │   │ │
│   │  └─────────────────────────────────────────────────────────┘   │ │
│   └─────────────────────────────────────────────────────────────────┘ │
│                                                                       │
│   ┌─────────────────┐     ┌──────────────────────────────────────┐   │
│   │  Settings UI    │     │  Transparent Overlay Window(s)       │   │
│   │  (Tauri/WebView)│     │  (always-on-top, click-through)      │   │
│   └─────────────────┘     └──────────────────────────────────────┘   │
└───────────────────────────────────────────────────────────────────────┘

2.2 Process Architecture

BABEL runs as three cooperating processes:
Process	Language	Role
babel-daemon	Rust	Orchestration, accessibility reading, overlay rendering, IPC hub
babel-translate	Python	NMT model loading, inference, language detection, cache
babel-ui	Tauri (Rust + WebView)	Settings, tray icon, feedback, model management

Why split daemon and translator?

    Python is necessary for CTranslate2 / HuggingFace ecosystem.
    Rust daemon needs guaranteed low-latency response to OS events.
    Keeping them separate means a translation crash doesn't kill the daemon.
    The translator can be restarted independently without losing overlay state.

2.3 Communication Architecture

text

babel-daemon  ◄──── Unix Socket / Named Pipe (IPC) ────►  babel-translate
babel-daemon  ◄──── Tauri IPC / WebSocket             ────►  babel-ui
babel-daemon  ────► OS Accessibility APIs (read-only)
babel-daemon  ────► OS Overlay Window APIs (write)

3. Repository Structure

text

babel/
├── Cargo.toml                    # Rust workspace root
├── Cargo.lock
├── pyproject.toml                # Python project (babel-translate)
├── babel.toml                    # Default config file
├── README.md
├── BABEL.md                      # Original spec
│
├── crates/
│   ├── babel-daemon/             # Main Rust daemon crate
│   │   ├── src/
│   │   │   ├── main.rs           # Entry point, arg parsing, signal handling
│   │   │   ├── daemon.rs         # Main event loop, coordinator
│   │   │   ├── screen_reader.rs  # Unified text extraction facade
│   │   │   ├── text_detector.rs  # Text region normalization & dedup
│   │   │   ├── replacer.rs       # Drives overlay from translated runs
│   │   │   └── change_detector.rs# Dirty region / stale run tracking
│   │   └── Cargo.toml
│   │
│   ├── babel-accessibility/      # OS accessibility backends (Rust)
│   │   ├── src/
│   │   │   ├── lib.rs            # Trait: AccessibilityBackend
│   │   │   ├── macos/
│   │   │   │   ├── mod.rs
│   │   │   │   ├── ax_bridge.rs  # AXUIElement bindings
│   │   │   │   └── ax_walker.rs  # Tree walker + text extractor
│   │   │   ├── windows/
│   │   │   │   ├── mod.rs
│   │   │   │   ├── uia_bridge.rs # IUIAutomation COM bindings
│   │   │   │   └── uia_walker.rs # UIA tree walker + TextPattern
│   │   │   └── linux/
│   │   │       ├── mod.rs
│   │   │       ├── atspi_bridge.rs # AT-SPI2 D-Bus bindings
│   │   │       └── atspi_walker.rs
│   │   └── Cargo.toml
│   │
│   ├── babel-ocr/                # OCR fallback backend (Rust)
│   │   ├── src/
│   │   │   ├── lib.rs
│   │   │   ├── capture.rs        # Screen region capture
│   │   │   ├── tesseract.rs      # Tesseract FFI wrapper
│   │   │   ├── region_detector.rs# Heuristic text region finder
│   │   │   └── stabilizer.rs     # Jitter suppression / bbox smoothing
│   │   └── Cargo.toml
│   │
│   ├── babel-overlay/            # Overlay renderer (Rust)
│   │   ├── src/
│   │   │   ├── lib.rs
│   │   │   ├── renderer.rs       # Main render loop
│   │   │   ├── window.rs         # OS transparent window management
│   │   │   ├── layout_preservor.rs
│   │   │   ├── font_matcher.rs
│   │   │   ├── transparency.rs   # Click-through, hover detection
│   │   │   └── animation.rs      # Fade-in/out on text change
│   │   └── Cargo.toml
│   │
│   ├── babel-ipc/                # Shared IPC types (Rust)
│   │   ├── src/
│   │   │   ├── lib.rs
│   │   │   ├── messages.rs       # Request/Response enums
│   │   │   └── codec.rs          # MessagePack serialization
│   │   └── Cargo.toml
│   │
│   └── babel-config/             # Config loading/validation (Rust)
│       ├── src/
│       │   ├── lib.rs
│       │   ├── schema.rs         # Serde structs for babel.toml
│       │   └── profiles.rs       # App profile loading
│       └── Cargo.toml
│
├── babel-translate/              # Python translation worker
│   ├── __init__.py
│   ├── main.py                   # Entry point + IPC server
│   ├── engine.py                 # Translation orchestrator
│   ├── model_manager.py          # Download / load / unload models
│   ├── translator.py             # CTranslate2 wrapper
│   ├── batch_translator.py       # Batching logic
│   ├── language_detector.py      # fastText lid.176 wrapper
│   ├── cache.py                  # LRU translation cache
│   ├── ipc_server.py             # Unix socket / named pipe server
│   └── requirements.txt
│
├── babel-ui/                     # Tauri settings UI
│   ├── src-tauri/
│   │   ├── src/
│   │   │   ├── main.rs
│   │   │   ├── commands.rs       # Tauri commands (bridge to daemon)
│   │   │   └── tray.rs           # System tray management
│   │   └── Cargo.toml
│   ├── src/                      # Frontend (HTML/CSS/JS or Svelte)
│   │   ├── App.svelte
│   │   ├── pages/
│   │   │   ├── General.svelte
│   │   │   ├── Languages.svelte
│   │   │   ├── Models.svelte
│   │   │   ├── Apps.svelte       # Per-app profile config
│   │   │   └── Advanced.svelte
│   │   └── components/
│   └── tauri.conf.json
│
├── profiles/                     # Built-in app profiles
│   ├── browsers.toml
│   ├── terminals.toml
│   ├── games.toml
│   └── productivity.toml
│
├── models/                       # Model storage (downloaded at runtime)
│   └── .gitkeep
│
└── tests/
    ├── integration/
    ├── unit/
    └── fixtures/

4. Core Data Models

These types are the lingua franca of the entire system. Every module speaks in these terms.
4.1 TextRun — A unit of extracted text

Rust

/// A single contiguous run of text discovered on screen.
/// This is the fundamental unit passed from accessibility/OCR to the pipeline.
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct TextRun {
    /// Globally unique stable ID for this run across frames.
    /// Computed as: hash(app_pid + element_id + role + approximate_position)
    pub id: TextRunId,

    /// The raw text content as extracted from the OS or OCR.
    pub text: String,

    /// Bounding box in absolute screen coordinates (not window-relative).
    pub bbox: ScreenRect,

    /// Source of this run — determines rendering trust level.
    pub source: TextSource,

    /// Owning application PID.
    pub app_pid: u32,

    /// Owning window handle (HWND / CGWindowID / AT-SPI object path).
    pub window_id: WindowId,

    /// OS-level element identifier (for stable tracking).
    pub element_id: Option<String>,

    /// Accessibility role (e.g., "button", "label", "text area").
    pub role: Option<String>,

    /// Font info extracted from accessibility metadata (best-effort).
    pub font_hint: Option<FontHint>,

    /// Detected foreground text color (for overlay matching).
    pub color_hint: Option<Color>,

    /// Background color behind this text (for overlay plate).
    pub bg_color_hint: Option<Color>,

    /// Frame number this run was last seen in (for staleness detection).
    pub last_seen_frame: u64,

    /// Whether the user has manually excluded this run.
    pub excluded: bool,
}

pub type TextRunId = u64;

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct ScreenRect {
    pub x: f32,       // Left edge, screen coordinates
    pub y: f32,       // Top edge, screen coordinates
    pub width: f32,
    pub height: f32,
    pub scale_factor: f32,  // HiDPI / Retina scale
}

#[derive(Debug, Clone, Serialize, Deserialize)]
pub enum TextSource {
    AccessibilityAPI,   // From OS accessibility tree (highest trust)
    OCRTesseract,       // From Tesseract OCR (lower trust, needs stabilization)
    OCRCustom,          // From alternative OCR engine
}

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct FontHint {
    pub family: Option<String>,
    pub size_pt: Option<f32>,
    pub weight: Option<FontWeight>,
    pub italic: bool,
}

#[derive(Debug, Clone, Serialize, Deserialize)]
pub enum FontWeight { Thin, Light, Regular, Medium, Bold, ExtraBold }

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct Color { pub r: u8, pub g: u8, pub b: u8, pub a: u8 }

4.2 TranslatedRun — A text run with its translation

Rust

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct TranslatedRun {
    /// Points back to the originating TextRun.
    pub run_id: TextRunId,

    /// Original text (preserved for hover-to-show-original).
    pub original_text: String,

    /// Translated output text.
    pub translated_text: String,

    /// Detected source language (BCP-47 tag, e.g. "zh", "ja", "fr").
    pub source_lang: String,

    /// Target language (BCP-47 tag).
    pub target_lang: String,

    /// Translation confidence [0.0, 1.0].
    pub confidence: f32,

    /// Which model produced this translation.
    pub model_id: String,

    /// Was this served from cache?
    pub from_cache: bool,

    /// Latency in milliseconds for this translation.
    pub latency_ms: f32,

    /// Bounding box (copied from TextRun, kept here for renderer convenience).
    pub bbox: ScreenRect,

    /// Font hint (copied from TextRun).
    pub font_hint: Option<FontHint>,

    /// User feedback on this translation (if any).
    pub feedback: Option<TranslationFeedback>,
}

#[derive(Debug, Clone, Serialize, Deserialize)]
pub enum TranslationFeedback { Good, Bad, Ignored }

4.3 TranslationRequest / TranslationResponse — IPC wire types

Rust

// Sent from Rust daemon → Python translator over IPC
#[derive(Debug, Serialize, Deserialize)]
pub struct TranslationRequest {
    pub request_id: Uuid,
    pub runs: Vec<TranslationItem>,
    pub target_lang: String,
    pub priority: TranslationPriority,
}

#[derive(Debug, Serialize, Deserialize)]
pub struct TranslationItem {
    pub run_id: TextRunId,
    pub text: String,
    pub hint_source_lang: Option<String>,  // skip lang detect if known
}

#[derive(Debug, Serialize, Deserialize)]
pub enum TranslationPriority {
    Immediate,   // Focused UI element (e.g., hovered menu item)
    Normal,      // Visible text
    Background,  // Off-screen / pre-warm
}

// Sent from Python translator → Rust daemon over IPC
#[derive(Debug, Serialize, Deserialize)]
pub struct TranslationResponse {
    pub request_id: Uuid,
    pub results: Vec<TranslatedRunResult>,
}

#[derive(Debug, Serialize, Deserialize)]
pub struct TranslatedRunResult {
    pub run_id: TextRunId,
    pub translated_text: String,
    pub source_lang: String,
    pub confidence: f32,
    pub model_id: String,
    pub from_cache: bool,
    pub latency_ms: f32,
    pub error: Option<String>,
}

4.4 OverlayCommand — Renderer instruction

Rust

#[derive(Debug, Clone, Serialize, Deserialize)]
pub enum OverlayCommand {
    /// Draw or update a translated text overlay at the given position.
    Upsert(OverlayEntry),
    /// Remove an overlay (run no longer visible).
    Remove(TextRunId),
    /// Remove all overlays for a specific window.
    ClearWindow(WindowId),
    /// Remove all overlays.
    ClearAll,
    /// Toggle visibility of all overlays.
    SetVisible(bool),
}

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct OverlayEntry {
    pub run_id: TextRunId,
    pub translated_text: String,
    pub original_text: String,
    pub bbox: ScreenRect,
    pub font_hint: Option<FontHint>,
    pub text_color: Color,
    pub bg_color: Color,
    pub opacity: f32,
    pub show_original_on_hover: bool,
}

4.5 WindowId — Cross-platform window handle

Rust

#[derive(Debug, Clone, PartialEq, Eq, Hash, Serialize, Deserialize)]
pub enum WindowId {
    Windows(isize),           // HWND as isize
    MacOS(u32),               // CGWindowID
    LinuxXCB(u32),            // XCB window id
    LinuxATSPI(String),       // AT-SPI object path
}

5. Module: Core Daemon (core/)
5.1 main.rs — Entry Point

Responsibilities:

    Parse CLI arguments (--config, --log-level, --no-overlay, --dry-run)
    Load babel.toml via babel-config
    Initialize logging (via tracing crate)
    Set up signal handlers (SIGTERM/SIGINT → graceful shutdown)
    Spawn the three top-level async tasks:
        Accessibility reader loop
        Translation coordinator loop
        Overlay render loop
    Start IPC server (for UI communication)
    Block on shutdown signal

Rust

// Pseudocode structure
#[tokio::main]
async fn main() {
    let args = Args::parse();
    let config = Config::load(&args.config_path)?;
    init_tracing(&config.log_level);

    let (text_tx, text_rx) = mpsc::channel::<Vec<TextRun>>(256);
    let (translated_tx, translated_rx) = mpsc::channel::<Vec<TranslatedRun>>(256);
    let (overlay_tx, overlay_rx) = mpsc::channel::<OverlayCommand>(512);
    let (shutdown_tx, shutdown_rx) = broadcast::channel(1);

    let daemon = Daemon::new(config, text_tx, translated_tx, overlay_tx);

    tokio::select! {
        _ = daemon.run_accessibility_loop(text_rx) => {}
        _ = daemon.run_translation_loop(text_rx, translated_tx) => {}
        _ = daemon.run_overlay_loop(translated_rx, overlay_tx) => {}
        _ = daemon.run_ipc_server() => {}
        _ = signal::ctrl_c() => { shutdown_tx.send(()).ok(); }
    }
}

5.2 daemon.rs — Main Coordinator

Responsibilities:

    Own the global TextRunRegistry (all currently tracked runs)
    Coordinate the pipeline: Read → Detect Change → Translate → Render
    Apply profile rules (exclusions, app-specific overrides)
    Manage translation request throttling and priority

Key data structures:

Rust

pub struct Daemon {
    config: Arc<Config>,
    run_registry: Arc<RwLock<TextRunRegistry>>,
    profiles: ProfileManager,
    translation_client: TranslationClient,  // IPC client to Python process
    overlay_controller: OverlayController,
    frame_counter: AtomicU64,
}

pub struct TextRunRegistry {
    runs: HashMap<TextRunId, TextRun>,
    window_to_runs: HashMap<WindowId, HashSet<TextRunId>>,
    pending_translation: HashSet<TextRunId>,
    translated: HashMap<TextRunId, TranslatedRun>,
    stale_threshold_frames: u64,
}

Main loop logic:

text

loop:
  1. Receive new_runs: Vec<TextRun> from accessibility reader
  2. For each run in new_runs:
     a. Compute run_id
     b. Check if already in registry AND text hasn't changed → skip
     c. Check profile rules → if excluded, skip
     d. Check if source lang == target lang → skip
     e. Add/update in registry, mark as pending_translation
  3. Garbage collect stale runs (not seen in N frames) → send OverlayCommand::Remove
  4. Batch pending_translation runs → send TranslationRequest
  5. Receive TranslationResponse → update registry, send OverlayCommand::Upsert

5.3 screen_reader.rs — Unified Text Extraction Facade

Responsibilities:

    Abstract over all backend types (AX, UIA, AT-SPI, OCR)
    Select the correct backend for each window/app
    Normalize outputs into Vec<TextRun>
    Manage per-window backend assignment

Rust

pub trait AccessibilityBackend: Send + Sync {
    fn name(&self) -> &'static str;
    fn is_available(&self) -> bool;
    fn supports_window(&self, window_id: &WindowId) -> bool;
    async fn extract_text_runs(&self, window_id: &WindowId) -> Result<Vec<TextRun>>;
    async fn subscribe_to_events(&self, window_id: &WindowId,
        tx: Sender<AccessibilityEvent>) -> Result<EventSubscription>;
}

pub struct ScreenReader {
    backends: Vec<Box<dyn AccessibilityBackend>>,
    ocr_backend: OcrBackend,
    window_backend_cache: HashMap<WindowId, BackendType>,
}

Backend selection algorithm:

text

fn select_backend(window_id) -> BackendType:
  1. Check window_backend_cache (fast path)
  2. Try AccessibilityAPI.supports_window(window_id)
     → if yes, cache + return AccessibilityAPI
  3. Try AT-SPI/UIA probe extraction (test with timeout 100ms)
     → if returns >0 runs with valid bbox, cache + return AccessibilityAPI
  4. Fall back to OCR
     → cache + return OCR
  5. On cache miss every 60s, re-probe (app may have changed)

5.4 text_detector.rs — Text Run Normalization

Responsibilities:

    Deduplicate runs from different sources (AX + OCR may overlap)
    Filter runs below minimum length (config: min_string_length, default: 3)
    Filter runs that are obviously not translatable (pure numbers, URLs, code tokens)
    Normalize Unicode (NFC normalization)
    Split long runs into sub-sentences for better translation quality
    Assign stable IDs

Rust

pub struct TextDetector {
    config: TextDetectorConfig,
    id_assigner: RunIdAssigner,
}

impl TextDetector {
    pub fn process(&self, raw_runs: Vec<RawTextRun>) -> Vec<TextRun> {
        raw_runs
            .into_iter()
            .filter(|r| self.is_translatable(&r.text))
            .map(|r| self.normalize(r))
            .flat_map(|r| self.split_if_long(r))
            .map(|r| self.assign_id(r))
            .collect()
    }

    fn is_translatable(&self, text: &str) -> bool {
        let trimmed = text.trim();
        if trimmed.len() < self.config.min_string_length { return false; }
        if is_pure_numeric(trimmed) { return false; }
        if is_url(trimmed) { return false; }
        if is_file_path(trimmed) { return false; }
        if is_code_token(trimmed) { return false; }
        if is_html_entity(trimmed) { return false; }
        true
    }
}

5.5 change_detector.rs — Staleness & Dirty Region Tracking

Responsibilities:

    Track which runs have changed since last frame
    Detect when a window has scrolled (bulk invalidation)
    Detect when a window is closed (full cleanup)
    Detect when app loses focus (optionally pause translation)
    Rate-limit OCR-path polling to avoid CPU thrash

Rust

pub struct ChangeDetector {
    last_seen: HashMap<TextRunId, RunSnapshot>,
    window_states: HashMap<WindowId, WindowSnapshot>,
    ocr_frame_rate: Duration,   // e.g., 100ms between OCR scans
}

pub struct RunSnapshot {
    text_hash: u64,
    bbox_hash: u64,
    frame: u64,
}

pub enum ChangeEvent {
    RunAdded(TextRunId),
    RunTextChanged(TextRunId),
    RunMoved(TextRunId),
    RunRemoved(TextRunId),
    WindowScrolled(WindowId),
    WindowClosed(WindowId),
}

5.6 replacer.rs — Overlay Orchestration

Responsibilities:

    Map TranslatedRun → OverlayCommand
    Handle layout policy (what to do when translated text is longer)
    Manage fade-in/out transitions
    Handle "hover to show original" state machine

Rust

pub struct Replacer {
    layout_policy: LayoutPolicy,
    transition_ms: u32,
    hover_state: HashMap<TextRunId, HoverState>,
}

pub enum LayoutPolicy {
    ShrinkFont,        // Reduce font size to fit in original bbox
    Wrap,              // Wrap text within original bbox width
    Ellipsize,         // Truncate with "…" if too long
    AllowOverflow,     // Let text overflow (useful for short UI)
    DrawPlate,         // Draw a bg plate that can grow
}

6. Module: Accessibility Layer (accessibility/)
6.1 Shared Trait: AccessibilityBackend

Already defined above. Every platform implements this trait.
6.2 macOS Backend (macos/)
ax_bridge.rs — AXUIElement Bindings

Required macOS Accessibility permissions: App must request AXIsProcessTrusted() and prompt user to enable "BABEL" in System Settings → Privacy & Security → Accessibility.

Key AX attributes used:
AX Attribute	Purpose
kAXValueAttribute	Text content
kAXTitleAttribute	Button/control label
kAXPositionAttribute	Element screen position (AXValue → CGPoint)
kAXSizeAttribute	Element size (AXValue → CGSize)
kAXRoleAttribute	Element role (AXStaticText, AXButton, etc.)
kAXChildrenAttribute	Tree traversal
kAXFocusedUIElementChangedNotification	Focus change events
kAXValueChangedNotification	Value/text change events
kAXUIElementDestroyedNotification	Element destroyed

Rust

// ax_bridge.rs key functions
extern "C" {
    fn AXUIElementCreateSystemWide() -> AXUIElementRef;
    fn AXUIElementCopyAttributeValue(element: AXUIElementRef,
        attribute: CFStringRef, value: *mut CFTypeRef) -> AXError;
    fn AXUIElementCopyAttributeValues(element: AXUIElementRef,
        attribute: CFStringRef, index: CFIndex,
        maxValues: CFIndex, values: *mut CFArrayRef) -> AXError;
    fn AXObserverCreate(application: pid_t,
        callback: AXObserverCallback, outObserver: *mut AXObserverRef) -> AXError;
}

ax_walker.rs — Tree Walker

Rust

pub struct AXWalker {
    system_element: AXUIElementRef,
    visited: HashSet<AXUIElementRef>,
    max_depth: u32,          // Prevent runaway deep trees
    max_nodes: u32,          // Prevent memory explosion on complex UIs
}

impl AXWalker {
    pub fn walk_window(&mut self, window_pid: pid_t) -> Vec<TextRun> {
        // 1. Create app element from PID
        // 2. Get windows array
        // 3. For each visible window, walk tree recursively
        // 4. At each node, extract text + bbox if role is text-bearing
        // 5. Return flattened Vec<TextRun>
    }

    fn is_text_bearing_role(role: &str) -> bool {
        matches!(role,
            "AXStaticText" | "AXTextField" | "AXTextArea" |
            "AXButton" | "AXMenuItem" | "AXMenuBarItem" |
            "AXCell" | "AXColumn" | "AXRow" | "AXHeading" |
            "AXLink" | "AXTab" | "AXRadioButton" | "AXCheckBox"
        )
    }
}

6.3 Windows Backend (windows/)
uia_bridge.rs — IUIAutomation COM Bindings

COM interfaces used:
Interface	Purpose
IUIAutomation	Root factory / event subscription
IUIAutomationElement	Any UI element
IUIAutomationTextPattern	Rich text extraction
IUIAutomationTextPattern2	Caret position
IUIAutomationValuePattern	Simple value controls
IUIAutomationTreeWalker	Tree traversal
IUIAutomationEventHandler	Event callbacks
IUIAutomationFocusChangedEventHandler	Focus events
IUIAutomationStructureChangedEventHandler	UI structure changes

Rust

// uia_bridge.rs
use windows::Win32::UI::Accessibility::*;
use windows::Win32::Foundation::RECT;

pub struct UiaSession {
    automation: IUIAutomation,
    walker: IUIAutomationTreeWalker,
    true_condition: IUIAutomationCondition,
}

impl UiaSession {
    pub fn new() -> Result<Self> {
        let automation: IUIAutomation =
            unsafe { CoCreateInstance(&CUIAutomation, None, CLSCTX_INPROC_SERVER)? };
        let walker = unsafe { automation.CreateTreeWalker(
            &automation.CreateTrueCondition()?)? };
        Ok(Self { automation, walker, ... })
    }

    pub fn get_bounding_rect(element: &IUIAutomationElement) -> Result<ScreenRect> {
        let rect: RECT = unsafe { element.get_CurrentBoundingRectangle()? };
        Ok(ScreenRect {
            x: rect.left as f32,
            y: rect.top as f32,
            width: (rect.right - rect.left) as f32,
            height: (rect.bottom - rect.top) as f32,
            scale_factor: get_dpi_scale_for_hwnd(...),
        })
    }
}

uia_walker.rs — Tree Walker

Text extraction strategy for Windows:

    Try IUIAutomationTextPattern first (richest text info)
    Fall back to get_CurrentName() (label)
    Fall back to IUIAutomationValuePattern::get_CurrentValue() (input fields)
    Skip elements where get_CurrentIsOffscreen() == true
    Skip elements where get_CurrentIsEnabled() == false and role is not static text

Event subscription (Windows-specific):

Rust

// Register for these UIA events per monitored window:
automation.AddFocusChangedEventHandler(handler)?;
automation.AddStructureChangedEventHandler(
    root, TreeScope_Subtree, &cache_req, handler)?;
automation.AddPropertyChangedEventHandler(
    element, TreeScope_Element, &cache_req, handler,
    &[UIA_ValueValuePropertyId, UIA_NamePropertyId])?;

6.4 Linux Backend (linux/)
atspi_bridge.rs — AT-SPI2 D-Bus Bindings

AT-SPI2 D-Bus interfaces used:
Interface	Purpose
org.a11y.atspi.Accessible	Base element, children, role
org.a11y.atspi.Text	Text content, character extents
org.a11y.atspi.Component	Screen coordinates, bounding box
org.a11y.atspi.Value	Numeric/value controls
org.a11y.atspi.Event.Object	TextChanged, StateChanged events
org.a11y.atspi.Event.Focus	Focus events
org.a11y.Bus	Get accessibility bus address

Rust

// atspi_bridge.rs
use zbus::Connection;

pub struct AtSpiBridge {
    conn: Connection,
    registry: AtSpiRegistryProxy,
}

impl AtSpiBridge {
    pub async fn get_text_for_element(&self,
        obj_path: &str, iface: &str) -> Result<String> {
        let text_proxy = TextProxy::new(&self.conn, ATSPI_SERVICE, obj_path).await?;
        let char_count = text_proxy.character_count().await?;
        text_proxy.get_text(0, char_count).await.map_err(Into::into)
    }

    pub async fn get_component_extents(&self,
        obj_path: &str) -> Result<ScreenRect> {
        let comp_proxy = ComponentProxy::new(&self.conn, ATSPI_SERVICE, obj_path).await?;
        let (x, y, w, h) = comp_proxy.get_extents(CoordType::Screen).await?;
        Ok(ScreenRect { x: x as f32, y: y as f32,
                        width: w as f32, height: h as f32, scale_factor: 1.0 })
    }
}

7. Module: OCR Fallback Layer (ocr/)
7.1 capture.rs — Screen Region Capture

Platform-specific capture APIs:
Platform	API
Windows	BitBlt / PrintWindow / DXGI Desktop Duplication
macOS	CGWindowListCreateImage / CGDisplayCreateImage
Linux/X11	XGetImage / XCB SHM extension
Linux/Wayland	XDG desktop portal org.freedesktop.portal.Screenshot

Rust

pub struct Capturer {
    backend: Box<dyn CaptureBackend>,
    buffer: Arc<Mutex<CaptureBuffer>>,
}

pub struct CaptureBuffer {
    pub data: Vec<u8>,     // RGBA pixels
    pub width: u32,
    pub height: u32,
    pub stride: u32,
    pub timestamp: Instant,
    pub dirty_rects: Vec<ScreenRect>,
}

pub trait CaptureBackend: Send + Sync {
    async fn capture_window(&self, window_id: &WindowId) -> Result<CaptureBuffer>;
    async fn capture_rect(&self, rect: &ScreenRect) -> Result<CaptureBuffer>;
}

7.2 region_detector.rs — Text Region Detection

Algorithm (heuristic, fast):

    Convert to grayscale
    Apply Gaussian blur (σ=1.5) to suppress noise
    Compute horizontal Sobel gradient magnitude
    Threshold at mean + 1.5σ
    Dilate horizontally (kernel 30px wide, 3px tall) to merge text runs
    Find connected components (bounding boxes)
    Filter: aspect ratio 1:20 to 20:1, min area 100px², min height 8px

Rust

pub struct RegionDetector {
    min_region_height_px: u32,
    min_region_area_px2: u32,
    dilation_kernel_w: u32,
}

pub struct DetectedRegion {
    pub bbox: ScreenRect,
    pub confidence: f32,     // Heuristic confidence this is text
    pub pixel_data: Vec<u8>, // Cropped RGBA for OCR
}

7.3 tesseract.rs — Tesseract FFI Wrapper

Rust

use tesseract_sys::*;

pub struct TesseractEngine {
    api: *mut TessBaseAPI,
    lang_hint: Option<String>,
}

impl TesseractEngine {
    pub fn new(data_dir: &Path, lang: Option<&str>) -> Result<Self> {
        // Initialize with OEM_LSTM_ONLY (best accuracy)
        // Set PSM_SINGLE_BLOCK for region-cropped input
    }

    pub fn recognize_region(&self, image: &[u8],
        width: u32, height: u32) -> Result<Vec<OcrWord>> {
        // Returns words with confidence + individual word bboxes
    }
}

pub struct OcrWord {
    pub text: String,
    pub bbox: ScreenRect,
    pub confidence: f32,   // 0.0–100.0 (Tesseract scale)
    pub baseline: f32,
}

7.4 stabilizer.rs — Jitter Suppression

OCR bounding boxes jitter frame-to-frame due to subpixel rendering changes. The stabilizer:

    Maintains an exponential moving average (α=0.3) of bbox positions
    Only emits a "moved" event if centroid moves >4px from last stable position
    Holds a minimum 3-frame consensus before reporting a new text string
    Groups words into logical lines by Y-coordinate proximity (±0.5x line height)

8. Module: Translation Engine (translation/)
8.1 main.py — Python IPC Server Entry Point

Python

import asyncio
from ipc_server import IPCServer
from engine import TranslationEngine

async def main():
    config = load_config()
    engine = TranslationEngine(config)
    await engine.initialize()   # warm up models
    server = IPCServer(engine, config.ipc_socket_path)
    await server.serve_forever()

8.2 ipc_server.py — Unix Socket Server

Python

class IPCServer:
    """
    Listens on a Unix socket (Linux/macOS) or Named Pipe (Windows).
    Protocol: length-prefixed MessagePack frames.
    Frame format: [4 bytes LE uint32 length][N bytes msgpack payload]
    """

    async def handle_client(self, reader, writer):
        while True:
            length_bytes = await reader.readexactly(4)
            length = struct.unpack('<I', length_bytes)[0]
            payload = await reader.readexactly(length)
            request = msgpack.unpackb(payload, raw=False)
            response = await self.engine.handle_request(request)
            packed = msgpack.packb(response, use_bin_type=True)
            writer.write(struct.pack('<I', len(packed)) + packed)
            await writer.drain()

8.3 engine.py — Translation Orchestrator

Python

class TranslationEngine:
    def __init__(self, config):
        self.model_manager = ModelManager(config)
        self.translator = Translator(config)
        self.lang_detector = LanguageDetector(config)
        self.cache = TranslationCache(config.cache_size)
        self.batch_translator = BatchTranslator(self.translator, config)

    async def handle_request(self, request: dict) -> dict:
        items = request['runs']
        target_lang = request['target_lang']
        priority = request.get('priority', 'normal')

        # 1. Language detection (parallel)
        lang_tasks = [self.detect_lang(item) for item in items]
        detected_langs = await asyncio.gather(*lang_tasks)

        # 2. Filter items where source == target
        to_translate = [
            (item, lang) for item, lang in zip(items, detected_langs)
            if lang != target_lang and lang != 'unknown'
        ]

        # 3. Cache lookup
        results = {}
        cache_misses = []
        for item, src_lang in to_translate:
            cached = self.cache.get(item['text'], src_lang, target_lang)
            if cached:
                results[item['run_id']] = cached
            else:
                cache_misses.append((item, src_lang))

        # 4. Batch translate cache misses
        if cache_misses:
            translated = await self.batch_translator.translate(
                cache_misses, target_lang, priority)
            for run_id, result in translated.items():
                self.cache.put(result)
                results[run_id] = result

        return {'request_id': request['request_id'], 'results': list(results.values())}

8.4 model_manager.py — Model Lifecycle

Python

SUPPORTED_MODELS = {
    'nllb-200-distilled-600M': {
        'type': 'nllb',
        'hf_repo': 'facebook/nllb-200-distilled-600M',
        'ct2_repo': 'Helsinki-NLP/opus-mt-...',  # CTranslate2 converted
        'size_gb': 2.4,
        'languages': 200,
        'speed': 'medium',
        'quality': 'high',
    },
    'nllb-200-distilled-1.3B': {
        'type': 'nllb',
        'size_gb': 5.2,
        'languages': 200,
        'speed': 'slow',
        'quality': 'best',
    },
    'opus-mt-{src}-{tgt}': {
        'type': 'opus',
        'size_gb': 0.3,  # per direction pair
        'speed': 'fast',
        'quality': 'good',
    },
}

class ModelManager:
    def download_model(self, model_id: str, progress_cb) -> Path:
        """Download CTranslate2-converted model from HuggingFace Hub."""

    def load_model(self, model_id: str) -> ctranslate2.Translator:
        """Load model into RAM/VRAM with configured quantization."""
        return ctranslate2.Translator(
            model_path=str(self.model_dir / model_id),
            device=self.config.device,             # 'cuda' | 'cpu' | 'auto'
            inter_threads=self.config.inter_threads,
            intra_threads=self.config.intra_threads,
            compute_type=self.config.compute_type,  # 'int8' | 'float16' | 'auto'
        )

    def unload_model(self, model_id: str):
        """Free model from VRAM/RAM (e.g., when switching models)."""

8.5 translator.py — CTranslate2 Wrapper

Python

class Translator:
    def __init__(self, model_manager: ModelManager, config):
        self.model_manager = model_manager
        self.tokenizer = None   # SentencePiece tokenizer
        self.model = None

    def translate_batch(self,
                        texts: List[str],
                        src_lang: str,
                        tgt_lang: str) -> List[str]:
        """
        Tokenize → translate → detokenize.
        For NLLB: set forced_bos_token_id to target language token.
        For OPUS-MT: use appropriate src/tgt model pair.
        """
        # Tokenize
        tokenized = [self.tokenizer.encode(t, out_type=str) for t in texts]

        # Translate
        results = self.model.translate_batch(
            tokenized,
            target_prefix=[[self._lang_token(tgt_lang)]] * len(tokenized),
            max_decoding_length=256,
            beam_size=self.config.beam_size,     # default 2 for speed
            max_batch_size=self.config.max_batch_size,
            asynchronous=False,
        )

        # Detokenize
        return [self.tokenizer.decode(r.hypotheses[0][1:]) for r in results]

8.6 language_detector.py — fastText Language Identification

Python

import fasttext

class LanguageDetector:
    def __init__(self, model_path: str):
        self.model = fasttext.load_model(model_path)  # lid.176.bin

    def detect(self, text: str) -> Tuple[str, float]:
        """
        Returns (bcp47_lang_code, confidence).
        Strips newlines (fastText requirement).
        Falls back to 'unknown' if confidence < threshold.
        """
        text_clean = text.replace('\n', ' ').strip()
        if len(text_clean) < 3:
            return 'unknown', 0.0

        predictions = self.model.predict(text_clean, k=1)
        label = predictions[0][0].replace('__label__', '')
        confidence = float(predictions[1][0])

        if confidence < self.config.min_confidence:
            return 'unknown', confidence

        return self._to_bcp47(label), confidence

    def _to_bcp47(self, fasttext_label: str) -> str:
        """Map fastText ISO 639-1/3 labels to BCP-47 tags."""
        # e.g., 'zh' → 'zh', 'zht' → 'zh-Hant', 'srp' → 'sr'
        return FASTTEXT_TO_BCP47.get(fasttext_label, fasttext_label)

8.7 batch_translator.py — Adaptive Batching

Python

class BatchTranslator:
    """
    Collects translation items and dispatches in optimal batches.
    Priority items (Immediate) bypass the batch queue.
    Normal items are batched by:
      - Batch size limit (config: max_batch_size, default 16)
      - Time limit (config: batch_timeout_ms, default 20ms)
      - Same source language (tokenizer is per-lang for OPUS-MT)
    """

    async def translate(self, items, target_lang, priority):
        if priority == 'Immediate':
            return await self._translate_now(items, target_lang)
        else:
            return await self._enqueue_and_wait(items, target_lang)

    async def _dispatch_batch(self, batch: List[TranslationItem]):
        # Group by source language
        by_lang = defaultdict(list)
        for item in batch:
            by_lang[item.src_lang].append(item)

        results = {}
        for src_lang, lang_items in by_lang.items():
            texts = [i.text for i in lang_items]
            translated = self.translator.translate_batch(
                texts, src_lang, self.target_lang)
            for item, translation in zip(lang_items, translated):
                results[item.run_id] = TranslatedRunResult(
                    run_id=item.run_id,
                    translated_text=translation,
                    source_lang=src_lang,
                    ...)
        return results

8.8 cache.py — LRU Translation Cache

Python

from functools import lru_cache
from collections import OrderedDict

class TranslationCache:
    """
    Two-level cache:
    Level 1: Exact match (text + src_lang + tgt_lang) → translation
    Level 2: Normalized match (lowercase + whitespace-normalized) → translation
    """

    def __init__(self, max_size: int = 50_000):
        self.exact: OrderedDict = OrderedDict()
        self.normalized: OrderedDict = OrderedDict()
        self.max_size = max_size
        self.hits = 0
        self.misses = 0

    def _key(self, text: str, src: str, tgt: str) -> str:
        return f"{src}|{tgt}|{text}"

    def _norm_key(self, text: str, src: str, tgt: str) -> str:
        normalized = ' '.join(text.lower().split())
        return f"{src}|{tgt}|{normalized}"

    def get(self, text: str, src: str, tgt: str) -> Optional[str]:
        key = self._key(text, src, tgt)
        if key in self.exact:
            self.exact.move_to_end(key)
            self.hits += 1
            return self.exact[key]

        norm_key = self._norm_key(text, src, tgt)
        if norm_key in self.normalized:
            self.normalized.move_to_end(norm_key)
            self.hits += 1
            return self.normalized[norm_key]

        self.misses += 1
        return None

    def put(self, text: str, src: str, tgt: str, translation: str):
        # Evict LRU if at capacity
        if len(self.exact) >= self.max_size:
            self.exact.popitem(last=False)
        self.exact[self._key(text, src, tgt)] = translation

        norm_key = self._norm_key(text, src, tgt)
        if len(self.normalized) >= self.max_size:
            self.normalized.popitem(last=False)
        self.normalized[norm_key] = translation

9. Module: Overlay Renderer (overlay/)
9.1 window.rs — Transparent Overlay Window Management

Each monitor gets one full-screen transparent overlay window. This is more efficient than per-run windows and avoids OS limits on window counts.
Windows Implementation

Rust

use windows::Win32::UI::WindowsAndMessaging::*;
use windows::Win32::Graphics::Gdi::*;

pub struct Win32OverlayWindow {
    hwnd: HWND,
    dc: HDC,
    width: u32,
    height: u32,
}

impl Win32OverlayWindow {
    pub fn create(monitor: &Monitor) -> Result<Self> {
        // Class registration
        let wc = WNDCLASSEXW {
            style: CS_HREDRAW | CS_VREDRAW,
            lpfnWndProc: Some(wnd_proc),
            hInstance: get_instance(),
            lpszClassName: w!("BABELOverlay"),
            ..Default::default()
        };
        RegisterClassExW(&wc);

        // Create layered, transparent, always-on-top window
        let hwnd = CreateWindowExW(
            WS_EX_LAYERED | WS_EX_TRANSPARENT | WS_EX_TOPMOST | WS_EX_NOACTIVATE,
            w!("BABELOverlay"),
            w!("BABEL Overlay"),
            WS_POPUP,
            monitor.x, monitor.y, monitor.width, monitor.height,
            None, None, get_instance(), None,
        );

        // Set layered window color key + alpha
        SetLayeredWindowAttributes(hwnd, 0x00FF00FF, 0, LWA_COLORKEY);

        ShowWindow(hwnd, SW_SHOWNOACTIVATE);
        Ok(Self { hwnd, ... })
    }
}

macOS Implementation

Rust

use cocoa::appkit::*;
use cocoa::foundation::*;

pub struct MacOSOverlayWindow {
    ns_window: id,
    ns_view: id,
}

impl MacOSOverlayWindow {
    pub fn create(monitor: &Monitor) -> Result<Self> {
        unsafe {
            let ns_window: id = NSWindow::alloc(nil).initWithContentRect_styleMask_backing_defer_(
                NSRect::new(
                    NSPoint::new(monitor.x as f64, monitor.y as f64),
                    NSSize::new(monitor.width as f64, monitor.height as f64),
                ),
                NSWindowStyleMask::NSBorderlessWindowMask,
                NSBackingStoreType::NSBackingStoreBuffered,
                NO,
            );

            ns_window.setLevel_(NSScreenSaverWindowLevel + 1);
            ns_window.setIgnoresMouseEvents_(YES);           // Click-through
            ns_window.setOpaque_(NO);
            ns_window.setBackgroundColor_(NSColor::clearColor(nil));
            ns_window.setCollectionBehavior_(
                NSWindowCollectionBehavior::NSWindowCollectionBehaviorCanJoinAllSpaces |
                NSWindowCollectionBehavior::NSWindowCollectionBehaviorStationary
            );
            ns_window.makeKeyAndOrderFront_(nil);

            Ok(Self { ns_window, ... })
        }
    }
}

Linux/X11 Implementation

Rust

use x11rb::protocol::xproto::*;
use x11rb::protocol::shape::*;

pub fn create_x11_overlay(conn: &Connection, screen: &Screen,
    monitor: &Monitor) -> Result<Window> {
    let win = conn.generate_id()?;

    conn.create_window(
        COPY_DEPTH_FROM_PARENT,
        win,
        screen.root,
        monitor.x as i16, monitor.y as i16,
        monitor.width as u16, monitor.height as u16,
        0,
        WindowClass::INPUT_OUTPUT,
        screen.root_visual,
        &CreateWindowAux::new()
            .background_pixel(0)
            .override_redirect(1u32)   // Don't go through WM
            .event_mask(EventMask::EXPOSURE | EventMask::STRUCTURE_NOTIFY),
    )?;

    // Make window click-through using XShape extension
    let region = conn.generate_id()?;
    conn.create_region(region, &[])?;
    conn.x_fixes_set_window_shape_region(win, SK::INPUT, 0, 0, region)?;

    // Set _NET_WM_WINDOW_TYPE_DOCK to hint WM to keep on top
    let atom = get_atom(conn, b"_NET_WM_WINDOW_TYPE_DOCK")?;
    conn.change_property32(PropMode::REPLACE, win,
        AtomEnum::WM_WINDOW_TYPE, AtomEnum::ATOM, &[atom])?;

    conn.map_window(win)?;
    Ok(win)
}

9.2 renderer.rs — Main Render Loop

Rust

pub struct Renderer {
    overlay_windows: HashMap<MonitorId, Box<dyn OverlayWindow>>,
    entries: HashMap<TextRunId, OverlayEntry>,
    render_backend: Box<dyn RenderBackend>,  // Skia or tiny-skia
    dirty: bool,
}

impl Renderer {
    pub fn apply_command(&mut self, cmd: OverlayCommand) {
        match cmd {
            OverlayCommand::Upsert(entry) => {
                self.entries.insert(entry.run_id, entry);
                self.dirty = true;
            }
            OverlayCommand::Remove(id) => {
                self.entries.remove(&id);
                self.dirty = true;
            }
            OverlayCommand::ClearAll => {
                self.entries.clear();
                self.dirty = true;
            }
            _ => { ... }
        }
    }

    pub fn render_frame(&mut self) {
        if !self.dirty { return; }

        // Group entries by monitor
        let by_monitor = self.group_by_monitor();

        for (monitor_id, entries) in by_monitor {
            let surface = self.overlay_windows[&monitor_id].get_surface();
            surface.clear();

            for entry in entries {
                self.draw_entry(surface, &entry);
            }

            self.overlay_windows[&monitor_id].flush(surface);
        }

        self.dirty = false;
    }

    fn draw_entry(&self, surface: &mut Surface, entry: &OverlayEntry) {
        let layout = self.layout_preservor.compute(entry);

        // Draw background plate to cover original text
        surface.fill_rect(
            entry.bbox,
            entry.bg_color.with_alpha(entry.opacity),
        );

        // Draw translated text
        surface.draw_text(
            &layout.text,
            layout.computed_bbox,
            layout.font,
            entry.text_color,
        );
    }
}

9.3 layout_preservor.rs — Layout Policy Engine

Rust

pub struct LayoutPreservor {
    policy: LayoutPolicy,
}

pub struct ComputedLayout {
    pub text: String,              // Possibly truncated
    pub font: ResolvedFont,
    pub computed_bbox: ScreenRect,
    pub overflow: bool,
    pub scale_factor: f32,
}

impl LayoutPreservor {
    pub fn compute(&self, entry: &OverlayEntry) -> ComputedLayout {
        let bbox = &entry.bbox;
        let mut font = self.resolve_font(entry);
        let mut text = entry.translated_text.clone();

        match self.policy {
            LayoutPolicy::ShrinkFont => {
                // Binary search for max font size that fits bbox
                let mut lo = font.size_pt * 0.5;
                let mut hi = font.size_pt;
                for _ in 0..8 {
                    let mid = (lo + hi) / 2.0;
                    font.size_pt = mid;
                    if self.measure_text(&text, &font).width <= bbox.width {
                        lo = mid;
                    } else {
                        hi = mid;
                    }
                }
                font.size_pt = lo;
            }
            LayoutPolicy::Ellipsize => {
                while self.measure_text(&text, &font).width > bbox.width && text.len() > 1 {
                    text = truncate_unicode(&text);
                }
                if text != entry.translated_text {
                    text.push('…');
                }
            }
            LayoutPolicy::Wrap => {
                // Let renderer wrap at bbox.width, grow bbox height if needed
            }
            LayoutPolicy::DrawPlate => {
                // Allow bbox to grow; plate background follows text
            }
            _ => {}
        }

        ComputedLayout { text, font, computed_bbox: *bbox, ... }
    }
}

9.4 font_matcher.rs — Font Resolution

Rust

pub struct FontMatcher {
    system_fonts: SystemFontDb,
}

impl FontMatcher {
    pub fn resolve(&self, hint: &Option<FontHint>, bbox: &ScreenRect) -> ResolvedFont {
        if let Some(hint) = hint {
            // Try to find exact font family
            if let Some(family) = &hint.family {
                if let Some(font) = self.system_fonts.find(family) {
                    return ResolvedFont {
                        family: family.clone(),
                        size_pt: hint.size_pt.unwrap_or(
                            self.bbox_to_pt(bbox.height)),
                        weight: hint.weight.clone().unwrap_or(FontWeight::Regular),
                        italic: hint.italic,
                    };
                }
            }
        }

        // Fallback: use default UI font for platform
        ResolvedFont {
            family: self.platform_default_font(),
            size_pt: self.bbox_to_pt(bbox.height),
            weight: FontWeight::Regular,
            italic: false,
        }
    }

    fn bbox_to_pt(&self, height_px: f32) -> f32 {
        // Heuristic: text height is ~75% of bbox height
        height_px * 0.75 * (72.0 / 96.0)   // px to pt
    }

    fn platform_default_font(&self) -> String {
        #[cfg(target_os = "windows")] { "Segoe UI".into() }
        #[cfg(target_os = "macos")]   { "SF Pro Text".into() }
        #[cfg(target_os = "linux")]   { "Ubuntu".into() }
    }
}

9.5 transparency.rs — Hover & Interaction State

Rust

pub struct TransparencyManager {
    hover_runs: HashMap<TextRunId, HoverState>,
    interaction_mode: InteractionMode,
}

pub enum InteractionMode {
    Passthrough,     // Default: overlay is invisible to mouse
    Interactive,     // Held modifier key: overlay responds to hover
}

pub enum HoverState {
    None,
    Hovering { since: Instant },
    ShowingOriginal { original: String },
}

impl TransparencyManager {
    pub fn on_mouse_move(&mut self, pos: (f32, f32),
        entries: &HashMap<TextRunId, OverlayEntry>) {
        if self.interaction_mode != InteractionMode::Interactive {
            return;
        }

        for (run_id, entry) in entries {
            if entry.bbox.contains(pos) {
                match self.hover_runs.entry(*run_id) {
                    Entry::Vacant(e) => {
                        e.insert(HoverState::Hovering { since: Instant::now() });
                    }
                    Entry::Occupied(mut e) => {
                        if let HoverState::Hovering { since } = e.get() {
                            if since.elapsed() > Duration::from_millis(300) {
                                e.insert(HoverState::ShowingOriginal {
                                    original: entry.original_text.clone(),
                                });
                            }
                        }
                    }
                }
            }
        }
    }

    pub fn should_show_original(&self, run_id: TextRunId) -> Option<&str> {
        match self.hover_runs.get(&run_id)? {
            HoverState::ShowingOriginal { original } => Some(original),
            _ => None,
        }
    }
}

10. Module: Profile & Config System (profiles/ + config/)
10.1 schema.rs — Config Struct (Full)

Rust

#[derive(Debug, Deserialize, Serialize)]
pub struct Config {
    pub general: GeneralConfig,
    pub translation: TranslationConfig,
    pub overlay: OverlayConfig,
    pub ocr: OcrConfig,
    pub ipc: IpcConfig,
    pub logging: LoggingConfig,
    pub profiles: ProfilesConfig,
}

#[derive(Debug, Deserialize, Serialize)]
pub struct GeneralConfig {
    pub target_language: String,          // BCP-47, e.g. "en"
    pub auto_detect_source: bool,         // default: true
    pub enabled_on_startup: bool,         // default: true
    pub translate_system_ui: bool,        // default: true
    pub translate_notifications: bool,    // default: true
    pub translate_browser: bool,          // default: true
    pub translate_games: bool,            // default: false (OCR heavy)
    pub translate_terminal: bool,         // default: false
    pub translate_code_editors: bool,     // default: false
    pub pause_on_fullscreen_game: bool,   // default: true
    pub hotkey_toggle: String,            // e.g. "Ctrl+Shift+T"
    pub hotkey_show_original: String,     // e.g. "Alt" (hold)
}

#[derive(Debug, Deserialize, Serialize)]
pub struct TranslationConfig {
    pub model: String,                    // "nllb-200-distilled-600M" | "opus-mt" | "auto"
    pub device: String,                   // "cuda" | "cpu" | "mps" | "auto"
    pub compute_type: String,             // "int8" | "float16" | "float32" | "auto"
    pub beam_size: u32,                   // default: 2 (speed/quality tradeoff)
    pub inter_threads: u32,               // CTranslate2 inter-thread parallelism
    pub intra_threads: u32,               // CTranslate2 intra-thread parallelism
    pub max_batch_size: u32,              // default: 16
    pub batch_timeout_ms: u64,            // default: 20
    pub cache_size: usize,                // default: 50_000 entries
    pub min_string_length: usize,         // default: 3 chars
    pub min_lang_confidence: f32,         // default: 0.6
    pub models_dir: PathBuf,
    pub ipc_socket: String,
}

#[derive(Debug, Deserialize, Serialize)]
pub struct OverlayConfig {
    pub enabled: bool,
    pub opacity: f32,                     // 0.0–1.0, default: 0.95
    pub layout_policy: String,            // "shrink_font" | "ellipsize" | "wrap" | "plate"
    pub show_original_on_hover: bool,     // default: true
    pub hover_delay_ms: u64,              // default: 300
    pub animation_duration_ms: u64,       // default: 100 (fade in/out)
    pub font_override: Option<String>,    // Force a specific font
    pub text_color_override: Option<String>, // Force "#RRGGBB"
    pub bg_color_override: Option<String>,
    pub game_mode: bool,                  // Extra jitter suppression
}

#[derive(Debug, Deserialize, Serialize)]
pub struct OcrConfig {
    pub enabled: bool,                    // Whether to use OCR fallback
    pub engine: String,                   // "tesseract"
    pub tesseract_data_dir: PathBuf,
    pub scan_interval_ms: u64,            // default: 100ms
    pub confidence_threshold: f32,        // Min Tesseract confidence (0–100)
    pub stabilizer_frames: u32,           // Consensus frames before emit (default: 3)
    pub stabilizer_move_threshold_px: f32,// Pixel movement to count as "moved" (default: 4)
}

10.2 profiles.rs — Per-App Profile System

toml

# Example: profiles/browsers.toml
[[profile]]
name = "Google Chrome"
match_executable = ["chrome", "chrome.exe"]
match_window_class = ["Chrome_WidgetWin_1"]
accessibility_backend = "uia"   # Force UIA even if OCR available
translate_address_bar = false
translate_devtools = false
min_string_length = 5

[[profile]]
name = "Firefox"
match_executable = ["firefox", "firefox.exe"]
accessibility_backend = "uia"
translate_address_bar = false

# Example: profiles/terminals.toml
[[profile]]
name = "Windows Terminal"
match_executable = ["WindowsTerminal.exe"]
enabled = false   # Disable translation entirely

[[profile]]
name = "iTerm2"
match_executable = ["iTerm2"]
enabled = false

Rust

#[derive(Debug, Deserialize)]
pub struct AppProfile {
    pub name: String,
    pub match_executable: Vec<String>,
    pub match_window_class: Option<Vec<String>>,
    pub match_bundle_id: Option<String>,       // macOS
    pub enabled: Option<bool>,
    pub accessibility_backend: Option<String>, // Override auto-detection
    pub layout_policy_override: Option<String>,
    pub min_string_length: Option<usize>,
    pub exclude_roles: Option<Vec<String>>,    // Don't translate these AX roles
    pub custom_rules: Vec<CustomRule>,
}

pub struct ProfileManager {
    profiles: Vec<AppProfile>,
    pid_cache: HashMap<u32, AppProfile>,
}

impl ProfileManager {
    pub fn get_profile(&self, pid: u32, exe: &str) -> Option<&AppProfile> {
        // 1. Check PID cache (fast path)
        // 2. Match by executable name (case-insensitive, stem only)
        // 3. Match by window class (Windows-specific)
        // 4. Match by bundle ID (macOS-specific)
        // 5. Return None → use global defaults
    }
}

11. Module: Settings UI (ui/)
11.1 Tauri Architecture

The UI process (babel-ui) is a separate process running via Tauri. It communicates with the daemon via Tauri's IPC (sidecar pattern or named pipe) for settings reads/writes and live status.
11.2 Tray Icon (tray.rs)

Rust

pub fn setup_tray(app: &AppHandle) -> Result<()> {
    let tray_menu = SystemTrayMenu::new()
        .add_item(CustomMenuItem::new("toggle", "Pause BABEL"))
        .add_item(CustomMenuItem::new("target_lang", "Target Language: English"))
        .add_native_item(SystemTrayMenuItem::Separator)
        .add_item(CustomMenuItem::new("settings", "Settings..."))
        .add_item(CustomMenuItem::new("models", "Manage Models..."))
        .add_native_item(SystemTrayMenuItem::Separator)
        .add_item(CustomMenuItem::new("quit", "Quit BABEL"));

    SystemTray::new()
        .with_menu(tray_menu)
        .with_icon(app.default_window_icon().unwrap().clone())
        .on_event(|event| {
            if let SystemTrayEvent::MenuItemClick { id, .. } = event {
                match id.as_str() {
                    "toggle" => daemon_client::toggle_enabled(),
                    "settings" => open_settings_window(),
                    "quit" => std::process::exit(0),
                    _ => {}
                }
            }
        })
        .build(app)?;

    Ok(())
}

11.3 Settings Pages

General Page: Enable/disable toggles per context (browser, games, terminal, notifications), target language picker, hotkey config.

Languages Page: Source language detection settings, per-language-pair overrides (e.g., "always use OPUS-MT for French→English").

Models Page:

    List of available models with size, quality, speed rating
    Download progress bars (chunked download with progress via Tauri events)
    Currently loaded model indicator
    VRAM/RAM usage estimate per model

Apps Page:

    List of running apps with toggle + backend selection
    Add custom profile (executable picker + settings form)
    Profile import/export

Advanced Page:

    CTranslate2 compute type selector
    Beam size slider (1–5)
    Batch size / timeout sliders
    Cache size setting
    OCR scan interval
    Log level

11.4 Tauri Commands

Rust

#[tauri::command]
async fn get_config(state: State<'_, DaemonClient>) -> Result<Config, String> { ... }

#[tauri::command]
async fn update_config(config: Config, state: State<'_, DaemonClient>)
    -> Result<(), String> { ... }

#[tauri::command]
async fn get_translation_stats(state: State<'_, DaemonClient>)
    -> Result<TranslationStats, String> { ... }

#[tauri::command]
async fn download_model(model_id: String, app: AppHandle,
    state: State<'_, DaemonClient>) -> Result<(), String> {
    // Stream download progress as Tauri events to frontend
}

#[tauri::command]
async fn get_running_apps(state: State<'_, DaemonClient>)
    -> Result<Vec<RunningApp>, String> { ... }

#[tauri::command]
async fn set_app_profile(pid: u32, profile: AppProfile,
    state: State<'_, DaemonClient>) -> Result<(), String> { ... }

#[tauri::command]
async fn submit_feedback(run_id: u64, feedback: String,
    state: State<'_, DaemonClient>) -> Result<(), String> { ... }

12. Inter-Process Communication (IPC)
12.1 Daemon ↔ Translator IPC

Transport:

    Linux/macOS: Unix domain socket at $XDG_RUNTIME_DIR/babel-translate.sock or /tmp/babel-translate.sock
    Windows: Named pipe \\.\pipe\babel-translate

Protocol: Length-prefixed MessagePack frames

text

Frame:
┌─────────────────────┬────────────────────────────────────┐
│ Length (4 bytes LE) │ MessagePack Payload (Length bytes) │
└─────────────────────┴────────────────────────────────────┘

Message types:

text

Daemon → Translator:
  { type: "translate", request_id: UUID, runs: [...], target_lang: "en", priority: "normal" }
  { type: "ping" }
  { type: "set_model", model_id: "nllb-200-distilled-600M" }
  { type: "get_stats" }
  { type: "shutdown" }

Translator → Daemon:
  { type: "translate_response", request_id: UUID, results: [...] }
  { type: "pong" }
  { type: "model_loaded", model_id: "..." }
  { type: "stats", cache_hit_rate: 0.87, ... }
  { type: "error", message: "..." }

12.2 Daemon ↔ UI IPC

text

Daemon → UI (events, streamed via Tauri events or WebSocket):
  { type: "stats_update", translations_per_sec: 45.2, cache_hit_rate: 0.89 }
  { type: "model_download_progress", model_id: "...", pct: 0.45 }
  { type: "enabled_changed", enabled: false }

UI → Daemon (commands, via Tauri commands):
  { type: "get_config" }
  { type: "set_config", config: {...} }
  { type: "toggle_enabled" }
  { type: "reload_profiles" }

12.3 Translator Process Lifecycle

text

Startup sequence:
1. babel-daemon starts
2. daemon spawns babel-translate subprocess
3. babel-translate starts IPC server, loads fastText model
4. daemon pings babel-translate, retries up to 10x with 500ms backoff
5. babel-translate loads NMT model (can take 2–15s depending on model/GPU)
6. babel-translate sends model_loaded event
7. daemon marks translator as "ready", begins accepting text runs

Crash recovery:
- daemon watches subprocess PID
- If subprocess exits unexpectedly → daemon logs error, re-spawns after 1s
- If restart fails 3x in 60s → daemon enters "degraded" mode (no translation)
- daemon sends notification to UI about degraded mode

13. Event System
13.1 Internal Async Event Bus

Rust

// All internal events flow through a typed async event bus
pub enum BabelEvent {
    // From accessibility backends
    TextRunsDiscovered { window_id: WindowId, runs: Vec<TextRun> },
    TextRunChanged { run_id: TextRunId, new_text: String },
    TextRunRemoved { run_id: TextRunId },
    WindowClosed { window_id: WindowId },
    WindowFocused { window_id: WindowId },

    // From translation engine
    TranslationReady { response: TranslationResponse },
    TranslationFailed { request_id: Uuid, error: String },

    // From config/UI
    ConfigChanged { new_config: Config },
    ProfileChanged { pid: u32, profile: AppProfile },
    ToggleEnabled { enabled: bool },

    // From overlay
    HoverStarted { run_id: TextRunId },
    HoverEnded { run_id: TextRunId },
    FeedbackSubmitted { run_id: TextRunId, feedback: TranslationFeedback },

    // System
    MonitorLayoutChanged,
    Shutdown,
}

13.2 Accessibility Event Subscription (per OS)
Windows (UIA Events)

Rust

// Register per-window structural change + value change handlers
unsafe {
    uia.AddStructureChangedEventHandler(
        window_element,
        TreeScope_Subtree,
        &cache_request,
        &structure_handler,
    )?;
    uia.AddPropertyChangedEventHandler(
        window_element,
        TreeScope_Subtree,
        &cache_request,
        &prop_handler,
        &[UIA_NamePropertyId, UIA_ValueValuePropertyId,
          UIA_TextTextPropertyId],
    )?;
}

macOS (AX Notifications)

Rust

// Per-app AX observer
let notifications = [
    kAXValueChangedNotification,
    kAXTitleChangedNotification,
    kAXUIElementDestroyedNotification,
    kAXFocusedUIElementChangedNotification,
    kAXWindowCreatedNotification,
    kAXWindowMiniaturizedNotification,
];
for notif in &notifications {
    AXObserverAddNotification(observer, app_element, notif, callback_ptr);
}
CFRunLoopAddSource(run_loop, AXObserverGetRunLoopSource(observer),
    kCFRunLoopDefaultMode);

Linux (AT-SPI Events)

Python

# Using pyatspi2 or zbus AT-SPI
registry.registerEventListener(
    on_text_change,
    "object:text-changed:insert",
    "object:text-changed:delete",
    "object:state-changed:showing",
    "window:activate",
    "window:deactivate",
)

14. Caching Strategy
14.1 Three-Tier Caching

text

Tier 1: In-process memory (Python process)
  - LRU dictionary (exact match + normalized match)
  - Size: 50,000 entries (configurable)
  - TTL: No TTL (evict LRU)
  - Hit rate target: >85% for typical UI workloads

Tier 2: Run registry (Rust daemon)
  - HashMap<TextRunId, TranslatedRun>
  - Prevents re-requesting translation for unchanged runs
  - Invalidated when: text changes, window closes, config changes

Tier 3: Persistent disk cache (SQLite)
  - Optional: enabled by config
  - Stores translations across daemon restarts
  - Table: (text_hash, src_lang, tgt_lang, model_id, translation, timestamp)
  - Max size: 100MB (configurable), evict oldest on overflow
  - Useful for: app menus, common UI strings that are the same every session

14.2 Cache Invalidation Rules

text

Invalidate a cached translation when:
  1. Target language changes (flush entire cache)
  2. Model changes (flush entire cache)
  3. User reports "bad translation" for a specific run (invalidate that key)
  4. Source text changes (already handled by run_id based lookup)
  5. Disk cache entry is older than 30 days (configurable TTL)

15. Pipeline: End-to-End Data Flow
15.1 Complete Pipeline Sequence Diagram

text

┌─────────┐  ┌──────────────┐  ┌────────────┐  ┌─────────────────┐  ┌──────────┐
│  OS App  │  │  AX/OCR      │  │  Daemon    │  │  Translator     │  │ Overlay  │
│         │  │  Backend     │  │            │  │  (Python)       │  │ Renderer │
└────┬────┘  └──────┬───────┘  └─────┬──────┘  └────────┬────────┘  └────┬─────┘
     │              │                │                   │                │
     │ UI Change    │                │                   │                │
     │─────────────►│                │                   │                │
     │              │ AX Event fired │                   │                │
     │              │───────────────►│                   │                │
     │              │                │                   │                │
     │              │                │ 1. ChangeDetector  │                │
     │              │                │    filters unchanged runs          │
     │              │                │                   │                │
     │              │                │ 2. ProfileManager │                │
     │              │                │    excludes if    │                │
     │              │                │    app is blocked │                │
     │              │                │                   │                │
     │              │                │ 3. LangDetect hint│                │
     │              │                │    (source lang   │                │
     │              │                │    already known?)│                │
     │              │                │                   │                │
     │              │                │ 4. Cache lookup   │                │
     │              │                │    (hit? → skip   │                │
     │              │                │     translation)  │                │
     │              │                │                   │                │
     │              │                │──TranslationReq──►│                │
     │              │                │                   │ 5. Lang detect │
     │              │                │                   │    (fastText)  │
     │              │                │                   │                │
     │              │                │                   │ 6. Cache lookup│
     │              │                │                   │                │
     │              │                │                   │ 7. Batch + NMT │
     │              │                │                   │    inference   │
     │              │                │◄─TranslationResp──│                │
     │              │                │                   │                │
     │              │                │ 8. Update run     │                │
     │              │                │    registry       │                │
     │              │                │                   │                │
     │              │                │──OverlayCommand──────────────────►│
     │              │                │                   │                │
     │              │                │                   │ 9. Layout      │
     │              │                │                   │    policy      │
     │              │                │                   │                │
     │              │                │                   │ 10. Draw on    │
     │◄─ Translated overlay drawn ─────────────────────────────────────────
     │   on top of original UI       │                   │                │

15.2 Latency Budget Per Step

text

Step 1: AX event received + tree walk       →  0–10ms (event-driven)
Step 2: ChangeDetector filter               →  <1ms
Step 3: ProfileManager check                →  <1ms
Step 4: Daemon-side cache lookup            →  <1ms
Step 5: IPC serialization + send            →  1–3ms
Step 6: fastText language detect            →  <1ms (per string)
Step 7: Python-side cache lookup            →  <1ms
Step 8: NMT inference (GPU, batch)          →  5–30ms (GPU), 50–200ms (CPU)
Step 9: IPC response + deserialization      →  1–3ms
Step 10: Layout computation                 →  <1ms
Step 11: Overlay render                     →  1–5ms (GPU-accelerated)

────────────────────────────────────────────────
Total (GPU, cache miss):     ~10–55ms    ✓ Target
Total (GPU, cache hit):      ~5–15ms     ✓ Target
Total (CPU, cache miss):     ~60–220ms   ✓ Acceptable
Total (CPU, cache hit):      ~5–15ms     ✓ Target

16. Per-OS Implementation Guide
16.1 Windows — Implementation Details
Required Windows APIs

text

- UI Automation (UIAutomationCore.dll)
- SetLayeredWindowAttributes (user32.dll)
- GetDpiForWindow (user32.dll — per-monitor DPI)
- RegisterHotKey (user32.dll — global hotkeys)
- SetWinEventHook (user32.dll — backup event hook)
- Direct2D / DirectWrite — for overlay rendering
- DXGI — for fast screen capture (game mode)

DPI Handling

All bounding rectangles from UIA must be adjusted for per-monitor DPI:

Rust

let dpi = unsafe { GetDpiForWindow(hwnd) };
let scale = dpi as f32 / 96.0;
// UIA returns physical pixels on Win 8.1+
// Overlay must also be created in physical pixels

Windows-specific Gotchas

    UIA may require CoInitializeEx(NULL, COINIT_MULTITHREADED) per thread.
    UIA events come on a COM STA thread — must marshal to tokio thread.
    Some apps (UWP, WinUI3) expose text via IUIAutomationTextPattern2 — handle both v1 and v2.
    Chrome/Electron expose text via IAccessible (MSAA), not UIA — may need MSAA bridge.
    Game fullscreen exclusive mode makes DXGI Desktop Duplication necessary.
    Overlay window WS_EX_NOACTIVATE is critical — never steal focus.

Windows Installer Requirements

text

- Manifest: requireAdministrator = false (user-level install)
- DLL: UIAutomationCore.dll (bundled with Windows, no extra install)
- Python: bundled as embedded distribution
- Models: downloaded post-install via model manager
- Auto-start: HKCU\Software\Microsoft\Windows\CurrentVersion\Run

16.2 macOS — Implementation Details
Required Entitlements / Permissions

XML

<key>NSAccessibilityUsageDescription</key>
<string>BABEL needs Accessibility access to read text from applications for translation.</string>

<key>com.apple.security.automation.apple-events</key>
<true/>

Must call AXIsProcessTrustedWithOptions on first run and guide user to System Settings → Privacy & Security → Accessibility.
macOS-specific Gotchas

    AX API is synchronous — must run on a dedicated thread, not the main Cocoa thread.
    NSWindow.Level.screenSaver + 1 is needed to float above most windows including full-screen apps.
    NSScreen.screens for multi-monitor must be polled for NSApplicationDidChangeScreenParametersNotification.
    CGWindowListCreateImage returns a CGImage — must convert to RGBA bytes for Tesseract.
    Some Electron/CEF apps (like VS Code) may not expose AX tree by default — check electron_expose_accessibility.
    macOS Sequoia / Sonoma may add new privacy restrictions around screen capture — handle CGRequestScreenCaptureAccess().

macOS App Bundle Structure

text

BABEL.app/
├── Contents/
│   ├── Info.plist
│   ├── MacOS/
│   │   └── babel-daemon          # Rust binary
│   ├── Frameworks/
│   │   └── Python.framework/     # Embedded Python
│   ├── Resources/
│   │   ├── babel-translate/      # Python package
│   │   ├── models/               # NMT models
│   │   ├── babel-ui.app/         # Tauri app (nested)
│   │   └── tessdata/             # Tesseract data
│   └── LaunchAgents/
│       └── com.babel.daemon.plist # LaunchAgent for auto-start

16.3 Linux / X11 — Implementation Details
Required Libraries

text

libatspi-2.0          AT-SPI2 accessibility
libxtst               X Test Extension (event injection, not needed but common)
libx11                Core X11
libxcb                XCB (for overlay window)
libxfixes             XFixes extension (input regions, click-through)
libxcomposite         Compositing extension (alpha windows)
libxrender            RENDER extension (alpha blending)
libtesseract          Tesseract OCR

AT-SPI Setup

Bash

# AT-SPI2 bus must be running. Usually started by GNOME/KDE session.
# Check: ps aux | grep at-spi
# If not running, start: /usr/lib/at-spi2-core/at-spi-bus-launcher

Linux-specific Gotchas

    AT-SPI is D-Bus based — requires DBUS_SESSION_BUS_ADDRESS to be set.
    Some apps (Firefox, Chrome) need --force-renderer-accessibility flag to expose AT-SPI.
    X11 overlay with override_redirect = true bypasses WM — correct for overlay, but be careful not to cover taskbars.
    XFixes SetWindowShapeRegion with empty region = fully click-through.
    Multiple displays with different DPIs require querying Xrandr output scales.
    Wayland: wlr-layer-shell protocol (sway/wlroots compositors) is the right overlay mechanism, but GNOME Mutter doesn't support it as of writing → fall back to X11 XWayland.

17. Performance Targets & Budgets
17.1 Latency Targets
Scenario	Target P50	Target P95	Maximum Acceptable
UI button label (GPU, cached)	<5ms	<15ms	30ms
UI button label (GPU, uncached)	<30ms	<60ms	100ms
Paragraph text (GPU, uncached)	<50ms	<100ms	200ms
Any text (CPU, cached)	<10ms	<20ms	50ms
Any text (CPU, uncached)	<150ms	<250ms	500ms
OCR scan cycle	<100ms	<200ms	400ms
Overlay render frame	<5ms	<10ms	16ms
17.2 Resource Budgets
Resource	Target	Maximum
Daemon CPU (idle, AX backend)	<0.5%	2%
Daemon CPU (active translation)	<5%	15%
Python translator CPU (idle)	<0.5%	2%
Python translator CPU (active)	<20%	40%
Daemon RAM	<100MB	200MB
Translator RAM (NLLB-600M, int8)	~2.5GB	4GB
Translator RAM (OPUS-MT)	~400MB	1GB
Translator VRAM (NLLB-600M, int8)	~1.8GB	3GB
Disk (models)	2–6GB	20GB
Disk (disk cache)	100MB default	2GB max
17.3 Throughput Targets
Scenario	Target
Strings translated per second (GPU, batch 16)	>200/s
Strings translated per second (CPU, batch 8)	>20/s
Cache hit rate (typical desktop UI)	>85%
Language detection per second	>10,000/s
18. Error Handling & Resilience
18.1 Error Taxonomy

Rust

#[derive(thiserror::Error, Debug)]
pub enum BabelError {
    // Accessibility
    #[error("Accessibility permission denied: {0}")]
    AccessibilityPermissionDenied(String),

    #[error("Accessibility backend not available on this OS: {0}")]
    AccessibilityBackendUnavailable(String),

    #[error("Failed to walk accessibility tree: {0}")]
    AccessibilityTreeError(String),

    // Translation
    #[error("Translation worker not ready (still loading model)")]
    TranslatorNotReady,

    #[error("Translation worker crashed: {0}")]
    TranslatorCrashed(String),

    #[error("Translation timeout after {0}ms")]
    TranslationTimeout(u64),

    #[error("Model not found: {0}")]
    ModelNotFound(String),

    #[error("Model download failed: {0}")]
    ModelDownloadFailed(String),

    #[error("Insufficient VRAM for model {model}: need {needed}MB, have {available}MB")]
    InsufficientVram { model: String, needed: u64, available: u64 },

    // OCR
    #[error("Screen capture failed: {0}")]
    CaptureFailed(String),

    #[error("OCR engine failed: {0}")]
    OcrFailed(String),

    // Overlay
    #[error("Failed to create overlay window: {0}")]
    OverlayCreationFailed(String),

    // Config
    #[error("Config parse error: {0}")]
    ConfigParseError(String),

    #[error("Profile not found for app: {0}")]
    ProfileNotFound(String),
}

18.2 Recovery Strategies
Error	Recovery Strategy
AccessibilityPermissionDenied	Show permission grant UI, retry on permission change notification
AccessibilityTreeError	Retry 3x, then fall back to OCR for this window
TranslatorNotReady	Queue requests, drain when ready (max queue: 500 items)
TranslatorCrashed	Restart subprocess, re-queue pending requests, alert UI
TranslationTimeout	Skip this batch, log, continue
InsufficientVram	Auto-downgrade to smaller model or CPU compute
CaptureFailed	Skip frame, try next OCR cycle
OcrFailed	Log, skip run, try next frame
OverlayCreationFailed	Retry with fallback window style, log to UI
18.3 Health Check System

Rust

pub struct HealthChecker {
    last_translator_ping: Instant,
    translator_ping_interval: Duration,   // 10s
    translator_timeout: Duration,         // 5s
}

// Background task: ping translator every 10s
// If no pong in 5s → declare crashed → restart
async fn health_check_loop(mut checker: HealthChecker) {
    loop {
        tokio::time::sleep(checker.translator_ping_interval).await;
        match timeout(checker.translator_timeout,
                      checker.ping_translator()).await {
            Ok(Ok(_)) => { checker.last_translator_ping = Instant::now(); }
            _ => {
                tracing::error!("Translator health check failed, restarting");
                checker.restart_translator().await;
            }
        }
    }
}

19. Security & Privacy Architecture
19.1 Privacy Guarantees

    No network calls during normal operation. All translation is local. The only network calls are model downloads (explicit user action, from HuggingFace Hub over HTTPS).
    No telemetry by default. Opt-in only, and only sends: model used, avg latency, crash reports (no text content).
    No text logging to disk by default. Debug logging is disabled unless explicitly enabled, and even then does NOT log translated text content.
    IPC is local-only. Unix sockets / Named Pipes are not exposed over network.

19.2 Threat Model
Threat	Mitigation
Malicious app reading IPC socket	Named pipe ACL / Unix socket permissions (owner-only)
Injecting fake translations via IPC	IPC socket only readable by daemon process owner
Overlay obscuring security dialogs	System-level UAC/SIP dialogs are on separate desktop/layer; overlay cannot cover them
Disk cache leaking sensitive text	Disk cache is opt-in; when enabled, stored in user home with 600 permissions
Model integrity	SHA256 checksum verification on model download
19.3 Accessibility Permission Model

text

Windows:
  - UIA works without special permissions for standard user-level apps
  - Screen capture requires no special permission (unlike macOS)
  - Admin required: none (except installing as service)

macOS:
  - Accessibility: requires explicit user grant via System Settings
  - Screen Recording: required for OCR fallback (CGWindowListCreateImage)
  - BABEL requests minimal permissions: Accessibility only; screen recording only
    if OCR mode is enabled

Linux:
  - AT-SPI: no special permissions needed if session bus is running
  - X11 screen capture: no special permissions for own-display X server
  - Wayland: portal requires D-Bus permission grant (one-time)

20. Testing Strategy
20.1 Unit Tests

text

babel-accessibility:
  - test_uia_bbox_extraction: Mock UIA COM interface, verify ScreenRect conversion
  - test_ax_tree_walk: Mock AXUIElement tree, verify TextRun extraction
  - test_atspi_text_extraction: Mock D-Bus responses, verify text + bbox

babel-daemon:
  - test_change_detector_stable_run: Same text same bbox → no event
  - test_change_detector_text_change: Changed text → TextRunChanged event
  - test_text_detector_filter: URLs, numbers, short strings → filtered
  - test_run_id_stability: Same run position → same ID across frames
  - test_profile_exclusion: Terminal app → runs excluded

babel-translate (Python):
  - test_language_detector_zh: Chinese text → "zh", confidence > 0.9
  - test_language_detector_short: "OK" → "unknown" (below threshold)
  - test_cache_lru_eviction: Fill cache past max_size → oldest evicted
  - test_cache_normalized_match: "Hello  World" matches "Hello World" key
  - test_batch_grouping: Mixed src langs → grouped correctly
  - test_translator_nllb: "Bonjour monde" → "Hello world" (approx)
  - test_translator_opus: Spot-check common language pairs

babel-overlay:
  - test_layout_shrink_font: Text too wide → font size reduced
  - test_layout_ellipsize: Very long text → ellipsized
  - test_font_matcher_fallback: No hint → platform default font
  - test_renderer_upsert: Add entry → drawn on next frame
  - test_renderer_remove: Remove entry → cleared from surface

20.2 Integration Tests

text

test_e2e_windows_notepad:
  1. Launch Notepad with Japanese text fixture
  2. Start BABEL daemon with target=en
  3. Wait for overlay (timeout: 2s)
  4. Assert overlay window exists and covers Notepad window area
  5. Assert translated text is English (basic NLP check)

test_e2e_cache_behavior:
  1. Launch app with repeating text
  2. Measure translation latency: first occurrence vs. second occurrence
  3. Assert second occurrence latency < 5ms (cache hit)

test_e2e_overlay_stability:
  1. Scroll a foreign-language webpage
  2. Record overlay positions over 5 seconds at 60fps
  3. Assert no run persists > 1 second after scrolling away
  4. Assert no jitter >4px in stable runs

test_e2e_crash_recovery:
  1. Start daemon + translator
  2. Kill translator subprocess
  3. Assert daemon restarts translator within 5s
  4. Assert translations resume (send test request, assert response within 10s)

test_e2e_ocr_fallback:
  1. Use an app known to not expose AX tree (e.g., custom game with fixed text)
  2. Assert BABEL falls back to OCR mode for this window
  3. Assert translated overlay appears within 3 OCR cycles

20.3 Performance Benchmarks

Rust

// benches/translation_bench.rs
use criterion::{criterion_group, criterion_main, Criterion};

fn bench_lang_detect(c: &mut Criterion) {
    let detector = LanguageDetector::new();
    let texts = load_fixture("bench_texts_mixed.txt");
    c.bench_function("lang_detect_100_strings", |b| {
        b.iter(|| texts.iter().map(|t| detector.detect(t)).collect::<Vec<_>>())
    });
}

fn bench_translation_cached(c: &mut Criterion) { ... }
fn bench_translation_uncached_gpu(c: &mut Criterion) { ... }
fn bench_ax_tree_walk_1000_elements(c: &mut Criterion) { ... }
fn bench_overlay_render_100_runs(c: &mut Criterion) { ... }

20.4 Test Fixtures

text

tests/fixtures/
├── japanese_ui_screenshot.png      # For OCR tests
├── chinese_menu_screenshot.png
├── french_article_screenshot.png
├── bench_texts_mixed.txt           # 1000 mixed-language strings
├── sample_ax_tree.json             # Serialized AX tree for unit tests
├── sample_uia_tree.json
└── translations_expected.json      # Expected translations for regression

21. Build Phases & Milestones
Phase 0: Foundation (Week 1)

Goal: Repo structure, build system, shared types, basic IPC.

Deliverables:

    Cargo workspace with all crates stubbed
    babel-ipc crate with all message types and MessagePack codec
    babel-config crate with full schema, TOML parsing, validation
    Python package skeleton with IPC server stub
    babel.toml default config file
    CI pipeline (GitHub Actions): build + unit tests on Windows + macOS + Linux
    All shared data models (TextRun, TranslatedRun, OverlayCommand, etc.)

Exit criteria: cargo build --workspace succeeds on all platforms. IPC roundtrip test passes.
Phase 1: Translation Core (Weeks 2–3)

Goal: Working local translation pipeline, independently testable.

Deliverables:

    language_detector.py: fastText lid.176 integration + BCP-47 mapping
    model_manager.py: download + load + verify NLLB-200 and OPUS-MT
    translator.py: CTranslate2 wrapper, NLLB + OPUS paths
    batch_translator.py: adaptive batching, priority queue
    cache.py: two-level LRU cache (exact + normalized)
    engine.py: full orchestration
    ipc_server.py: length-prefixed MessagePack server
    babel-ipc/ipc_client.rs: Rust client for translation IPC
    Unit tests: all Python modules
    CLI smoke test: echo "Bonjour" | babel-translate --to en → "Hello"

Exit criteria: 100 French strings translated to English in <2s on CPU. Cache hit rate test passes. IPC roundtrip with Rust client works.
Phase 2: OS Text Extraction (Weeks 4–6)

Goal: Reliable text + bounding box extraction on each supported OS.

Deliverables:

    babel-accessibility crate: AccessibilityBackend trait
    Windows UIA backend: tree walk + bounding rect + event subscription
    macOS AX backend: tree walk + bounding rect + AX Observer
    Linux AT-SPI backend: D-Bus proxy + text + component extents
    babel-ocr crate: screen capture + Tesseract integration
    ocr/stabilizer.rs: bbox stabilization
    core/screen_reader.rs: unified facade + backend selection
    core/text_detector.rs: normalization, filtering, ID assignment
    core/change_detector.rs: dirty tracking + staleness
    Unit tests: mock AX/UIA trees → correct TextRun output
    Integration test: Notepad/TextEdit → readable TextRuns

Exit criteria: Open any native app with foreign text, see TextRun structs logged to console with correct text and screen-accurate bounding boxes.
Phase 3: Overlay Renderer (Weeks 7–8)

Goal: Translated text rendered in-place on all platforms.

Deliverables:

    babel-overlay crate: OverlayWindow trait
    Windows layered window overlay
    macOS floating NSWindow overlay
    Linux/X11 override-redirect overlay with XFixes click-through
    renderer.rs: render loop, entry management, dirty-region tracking
    layout_preservor.rs: ShrinkFont + Ellipsize + Wrap policies
    font_matcher.rs: platform font resolution
    transparency.rs: hover state machine
    animation.rs: fade-in/out transitions
    Multi-monitor support
    HiDPI / Retina support
    Integration test: overlay window visible, click-through confirmed

Exit criteria: Given a Vec<OverlayEntry>, translated text appears at exact screen positions, click-through works, hover shows original, no flicker.
Phase 4: Daemon Integration (Weeks 9–10)

Goal: Full end-to-end pipeline: screen → extract → translate → overlay.

Deliverables:

    babel-daemon crate: main coordinator, event loop
    core/daemon.rs: full pipeline orchestration
    core/replacer.rs: TranslatedRun → OverlayCommand mapping
    Profile system: built-in profiles for browsers, terminals, editors
    Subprocess lifecycle management (spawn + health check + restart)
    Graceful shutdown (SIGTERM → drain queues → cleanup overlays)
    Global hotkey support (toggle, show-original)
    End-to-end integration tests
    Performance profiling + bottleneck identification

Exit criteria: Open a foreign-language app, BABEL auto-translates it, overlay appears within target latency, app remains fully usable.
Phase 5: Settings UI & Tray (Week 11)

Goal: User-facing settings, model management, tray icon.

Deliverables:

    Tauri app setup
    System tray (all platforms)
    Settings pages: General, Languages, Models, Apps, Advanced
    Model download UI with progress
    Per-app exclusion management
    Live stats display (translations/sec, cache hit rate, latency P50/P95)
    Translation feedback UI (flag bad translation)

Exit criteria: User can configure all settings, download models, exclude apps, and toggle BABEL without touching the config file.
Phase 6: Polish, Performance & Packaging (Week 12)

Goal: Production-quality, installable, stable.

Deliverables:

    Auto-start at login (all platforms)
    Installer: NSIS (Windows), .dmg (macOS), .deb/.rpm (Linux)
    First-run wizard: permission grants + model download
    Crash reporting (opt-in, no text content)
    Performance regression test suite
    All integration tests passing on all platforms
    README with screenshots + install guide
    License file (Apache 2.0 or MIT)

22. Dependencies & Toolchain
22.1 Rust Dependencies (Cargo.toml)

toml

[workspace.dependencies]

# Async runtime
tokio = { version = "1", features = ["full"] }

# Serialization
serde = { version = "1", features = ["derive"] }
serde_json = "1"
rmp-serde = "1"          # MessagePack

# Error handling
thiserror = "1"
anyhow = "1"

# Logging / tracing
tracing = "0.1"
tracing-subscriber = { version = "0.3", features = ["env-filter"] }

# Config
toml = "0.8"

# Caching (Rust-side)
moka = { version = "0.12", features = ["future"] }

# UUID
uuid = { version = "1", features = ["v4"] }

# Hashing (for run IDs)
xxhash-rust = { version = "0.8", features = ["xxh64"] }

# Cross-platform hotkeys
global-hotkey = "0.5"

# System tray (via Tauri)
tauri = { version = "2", features = ["system-tray"] }

# OS-specific (Windows)
[target.'cfg(target_os = "windows")'.dependencies]
windows = { version = "0.58", features = [
    "Win32_UI_Accessibility",
    "Win32_UI_WindowsAndMessaging",
    "Win32_Graphics_Gdi",
    "Win32_Graphics_Direct2D",
    "Win32_Graphics_DirectWrite",
    "Win32_Foundation",
    "Win32_System_Com",
] }

# OS-specific (macOS)
[target.'cfg(target_os = "macos")'.dependencies]
cocoa = "0.25"
objc = "0.2"
core-foundation = "0.9"
core-graphics = "0.23"

# OS-specific (Linux)
[target.'cfg(target_os = "linux")'.dependencies]
zbus = { version = "4", features = ["tokio"] }
x11rb = { version = "0.13", features = ["all-extensions"] }

# Rendering
tiny-skia = "0.11"    # Pure-Rust 2D renderer (fallback)
# OR: skia-safe = "0.75"  (full Skia, better quality, harder to build)

# Font handling
fontdb = "0.16"
rustybuzz = "0.13"    # HarfBuzz text shaping
ttf-parser = "0.21"

# Image (for OCR)
image = "0.25"

# Tesseract FFI
tesseract-sys = "0.6"
leptonica-sys = "0.4"

22.2 Python Dependencies (requirements.txt)

text

ctranslate2>=4.0.0           # Fast Transformer inference
sentencepiece>=0.1.99        # Tokenizer for NLLB/OPUS
fasttext-langdetect>=1.0.5   # Language identification
msgpack>=1.0.7               # IPC serialization
numpy>=1.24.0                # Required by ctranslate2
huggingface_hub>=0.20.0      # Model downloading
tqdm>=4.66.0                 # Progress bars for download
torch>=2.0.0                 # Optional: needed if using PyTorch path
transformers>=4.37.0         # Optional: tokenizer loading via HF

22.3 Build Requirements

text

Rust:        1.75+ (stable)
Python:      3.10–3.12
CUDA:        11.8+ or 12.x (optional, for GPU acceleration)
Tesseract:   5.x (system library or bundled)
Node.js:     18+ (for Tauri/frontend build)
LLVM:        16+ (for some sys crate compilation)

Windows SDK: 10.0.19041.0+
macOS SDK:   13.0+
Linux:       libatspi-2.0-dev, libx11-dev, libxfixes-dev, libxcb*-dev

23. Configuration Reference (babel.toml)

toml

# ============================================================
# BABEL Configuration File
# Default location: ~/.config/babel/babel.toml (Linux/macOS)
#                   %APPDATA%\babel\babel.toml (Windows)
# ============================================================

[general]
target_language = "en"                # BCP-47 target language
auto_detect_source = true             # Auto-detect source language
enabled_on_startup = true             # Start translating on launch

# What to translate
translate_system_ui = true
translate_notifications = true
translate_browser = true
translate_games = false               # Disabled: heavy OCR usage
translate_terminal = false            # Disabled: code/output
translate_code_editors = false        # Disabled: code

pause_on_fullscreen_game = true       # Pause when exclusive fullscreen detected
hotkey_toggle = "Ctrl+Shift+T"        # Toggle all overlays
hotkey_show_original = "Alt"          # Hold to show all originals

[translation]
model = "nllb-200-distilled-600M"     # Model ID (see model_manager)
device = "auto"                       # "cuda" | "cpu" | "mps" | "auto"
compute_type = "auto"                 # "int8" | "float16" | "float32" | "auto"
beam_size = 2                         # Higher = better quality, slower
inter_threads = 4                     # CTranslate2 inter-thread count
intra_threads = 2                     # CTranslate2 intra-thread count
max_batch_size = 16                   # Max strings per inference call
batch_timeout_ms = 20                 # Wait up to 20ms to form a batch
cache_size = 50000                    # Max LRU cache entries (memory)
disk_cache_enabled = false            # Persistent disk cache
disk_cache_max_mb = 100
min_string_length = 3                 # Skip strings shorter than 3 chars
min_lang_confidence = 0.60            # Min fastText confidence to translate
models_dir = "~/.local/share/babel/models"
ipc_socket = "/tmp/babel-translate.sock"

[overlay]
enabled = true
opacity = 0.95                        # Overlay background opacity
layout_policy = "shrink_font"         # "shrink_font" | "ellipsize" | "wrap" | "plate"
show_original_on_hover = true
hover_delay_ms = 300                  # Delay before showing original on hover
animation_duration_ms = 100          # Fade in/out duration
font_override = ""                    # "" = auto-match; or "Arial" etc.
text_color_override = ""             # "" = auto; or "#FFFFFF"
bg_color_override = ""               # "" = auto; or "#000000"
game_mode = false                    # Extra stabilization for game UIs

[ocr]
enabled = true                        # Enable OCR fallback
engine = "tesseract"
tesseract_data_dir = ""               # "" = auto-detect system tessdata
scan_interval_ms = 100                # OCR polling interval
confidence_threshold = 60.0          # Min Tesseract confidence (0–100)
stabilizer_frames = 3                 # Frames before reporting new text
stabilizer_move_threshold_px = 4.0   # Pixel move to count as "moved"

[logging]
level = "warn"                        # "error" | "warn" | "info" | "debug" | "trace"
log_file = ""                         # "" = stderr only
log_translations = false              # NEVER log translated text (privacy)

[ipc]
daemon_socket = "/tmp/babel-daemon.sock"
ui_port = 9876                        # Local-only WebSocket for UI

# ---- Per-app profile overrides ----
# (These override global settings for specific apps)

[[app_profile]]
name = "Terminal Override"
match_executable = ["wt", "WindowsTerminal", "iTerm2", "Alacritty", "kitty"]
enabled = false

[[app_profile]]
name = "VS Code Override"
match_executable = ["code", "code-oss", "codium"]
enabled = false

[[app_profile]]
name = "Game Mode"
match_executable = ["game.exe"]        # Replace with actual game exe
overlay.layout_policy = "plate"
overlay.game_mode = true
translation.beam_size = 1             # Fastest possible

24. Glossary
Term	Definition
AX / AXUIElement	macOS Accessibility API. Provides structured access to UI elements.
AT-SPI	Assistive Technology Service Provider Interface. Linux accessibility framework over D-Bus.
BCP-47	Language tag standard. E.g., "en", "zh-Hant", "fr-CA".
bbox / Bounding Box	Pixel-coordinate rectangle (x, y, width, height) enclosing a UI element on screen.
CTranslate2	C++/Python library for fast inference of Transformer models (including quantization).
Daemon	Long-running background process (babel-daemon) coordinating the entire system.
fastText lid.176	Facebook's language identification model supporting 176 languages.
FontHint	Best-effort font metadata (family, size, weight) extracted from accessibility attributes.
HiDPI / Retina	High-resolution displays. Screen coordinates must be multiplied by scale_factor.
IPC	Inter-Process Communication. BABEL uses Unix sockets / Named Pipes with MessagePack encoding.
Layout Policy	How to handle translated text that is longer/shorter than the original.
NLLB-200	Meta's "No Language Left Behind" multilingual NMT model supporting 200 languages.
NMT	Neural Machine Translation.
OCR	Optical Character Recognition. Used as fallback when accessibility APIs don't expose text.
Overlay	Transparent always-on-top window drawn by BABEL to display translated text in-place.
OPUS-MT	Open-source NMT models from the OPUS project. Per-direction model pairs.
Profile	Per-application configuration overriding global BABEL settings.
TextRun	A single contiguous piece of text discovered on screen, with its bounding box.
TextRunId	Stable hash-based identifier for a TextRun across frames.
TranslatedRun	A TextRun that has been translated, with translation metadata attached.
UIA / UI Automation	Windows UI Automation. COM-based accessibility API for reading UI elements.
XFixes	X11 extension providing SetWindowShapeRegion for click-through overlay windows.

    Document Version: 1.0 Target: Autonomous agent implementation Status: Complete — ready for build Language targets: Rust (daemon, accessibility, overlay), Python (translation), TypeScript/Svelte + Rust (UI) Primary build target: Windows 10/11 → macOS 12+ → Linux X11
