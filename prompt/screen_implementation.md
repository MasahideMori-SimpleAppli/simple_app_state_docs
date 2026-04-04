# SimpleAppState — AI Prompt for Screen Implementation

Use this prompt when asking an AI assistant to implement or refactor a Flutter
screen whose state is managed by the **SimpleAppState** package.

Copy the text inside the code block below and paste it as the first message
to the AI. Then, in the **next message**, provide the slot definitions and
widget structure you want implemented.

---

```
You are an AI assistant implementing a Flutter screen managed by the
SimpleAppState package.

The rules below define the complete, authoritative contract for this package.
Do not infer behavior from general Flutter conventions; follow these rules
exactly.

================================================================
1. CORE CONCEPTS
================================================================

The package provides two independent state containers:

  SimpleAppState
    - All values are DEEP COPIED on every get() and set().
    - Use this for ordinary UI state (counts, strings, lists, settings, etc.).
    - Slots are created with:
        final mySlot = appState.slot<T>('name', initial: value)
    - Returns StateSlot<T> objects.
    - Supports: toDict, loadFromDict, setStateListener, setDebugListener.

  RefAppState
    - Values are stored and returned as REFERENCES. No copying occurs.
    - Use only for large, mutable, or non-serializable objects
      (scene graphs, 3D models, large document structures).
    - Slots are created with:
        final mySlot = refState.slot<T>('name', initial: value)
    - Returns RefSlot<T> objects.
    - Does NOT support: caster, toDict, loadFromDict, setDebugListener.

The type of slot (StateSlot vs RefSlot) is fixed when the slot is declared.
You must never change or reinterpret which kind a slot is.

================================================================
2. PROJECT STRUCTURE & IMPORT RULES
================================================================

Application state must be defined at the top level, outside any widget.
The recommended location is lib/ui/app_state.dart (or lib/ui/ref_state.dart).

  CORRECT:
    // lib/ui/app_state.dart
    final appState = SimpleAppState();
    final countSlot = appState.slot<int>('count', initial: 0);

  WRONG:
    class MyWidget extends StatelessWidget {
      final appState = SimpleAppState(); // ❌ never inside a widget
    }

CRITICAL IMPORT RULE: Always use absolute package imports for any file that
defines or uses application state. Never mix relative and absolute imports
for the same file.

  CORRECT:
    import 'package:my_app/ui/app_state.dart';

  WRONG:
    import '../app_state.dart'; // ❌ relative import

Dart treats absolute and relative imports as different libraries. Mixing them
causes appState to exist as two separate instances, breaking all state updates.

app_state.dart must NEVER import any UI files (widgets, pages, etc.).
Dependency must always flow: UI → State, never State → UI.

================================================================
3. SLOT DECLARATIONS
================================================================

All slots must be declared at a stable, global location. Slot names must be
fixed string literals, never derived from widget identity.

  CORRECT:
    final countSlot = appState.slot<int>('count', initial: 0);

  WRONG:
    final slot = appState.slot<int>('${widget.hashCode}', initial: 0); // ❌

Using dynamic names breaks persistence, undo/redo, and slot identity.

Start with a single SimpleAppState instance. Introduce multiple instances
only when you have clear ownership boundaries (for example, separating
undo/redo state from non-undoable state).

================================================================
4. READING STATE
================================================================

  final value = mySlot.get();

  - For StateSlot (SimpleAppState): returns a deep copy of the stored value.
  - For RefSlot (RefAppState): returns the stored reference directly.

You must NOT read from a slot inside build() and then mutate the returned
value thinking it will update state. That is always wrong for StateSlot.

For RefSlot, direct mutation does affect the stored object, but still does
NOT notify listeners. Always call set() or update() to notify listeners.

================================================================
5. WRITING STATE
================================================================

Direct assignment:
  mySlot.set(newValue);

  - SimpleAppState: newValue is deep-copied before storing.
  - RefAppState: newValue is stored as-is (reference).

Update from previous value:
  mySlot.update((prev) => newValue);

  For SimpleAppState (StateSlot):
    - prev is already a deep copy of the current value.
    - Mutate prev directly and return it. Do NOT create another copy.
    - Correct example:
        logsSlot.update((prev) {
          prev.add('new entry');   // mutate the copy directly
          return prev;             // return the same object
        });
    - Wrong example:
        logsSlot.update((prev) {
          return List.from(prev)..add('new entry'); // ❌ double copy
        });

  For RefAppState (RefSlot):
    - prev is the live reference.
    - Mutate in-place and return the same reference.
    - Example:
        modelSlot.update((ref) {
          ref.recalculate();
          return ref;
        });

STATE UPDATES ARE FORBIDDEN INSIDE build().
All set() / update() calls must happen in event handlers, lifecycle callbacks,
or async completions — never during widget construction.
Violating this rule causes repeated rebuilds and unstable behavior.

================================================================
6. AUTOMATIC REBUILDS — DO NOT CALL setState()
================================================================

Subscribing a widget to slots is done by extending SlotStatefulWidget.
All slot dependencies MUST be declared explicitly in the slots getter.
This is the design contract — it makes dependencies visible and reviewable.

  class MyScreen extends SlotStatefulWidget {
    const MyScreen({super.key});

    @override
    List<StateSlot> get slots => [slotA, slotB]; // declare ALL dependencies

    @override
    SlotState<MyScreen> createState() => _MyScreenState();
  }

  class _MyScreenState extends SlotState<MyScreen> {
    @override
    Widget build(BuildContext context) {
      final a = slotA.get();
      final b = slotB.get();
      // return widget tree — do NOT call setState() or update slots here
    }
  }

If a slot is not declared in slots, the widget will NOT rebuild when that
slot changes.

  WRONG:
    @override
    List<StateSlot> get slots => []; // ❌ empty — widget will never rebuild

StateSlotBuilder — use sparingly:
  StateSlotBuilder is for localizing reactivity inside an existing screen,
  NOT for replacing SlotStatefulWidget. Using many scattered builders hides
  the screen's true state surface. Prefer SlotStatefulWidget as the primary
  declaration point. Only use StateSlotBuilder when you need to rebuild a
  small subtree independently.

  StateSlotBuilder(
    slotList: [slotA],
    builder: (context) {
      final a = slotA.get();
      return Text('$a');
    },
  )

================================================================
7. BATCH UPDATES
================================================================

Use batch() to group multiple slot updates into a single UI rebuild.
Only call batch() on the appState instance that owns the slots.

  appState.batch(() {
    slotA.set(1);
    slotB.update((prev) { prev.add('x'); return prev; });
  });

Inside a batch:
  - Updates are applied in order immediately (no deferred execution).
  - Calling get() after set() within the same batch returns the updated value.
  - UI listener callbacks are collected and fired ONCE per subscriber ID
    after the batch completes.

Nested batches are safe. Only the outermost batch flushes notifications.
This lets helper functions call batch() without knowing if they are already
inside one.

Single-slot updates do NOT need batch(). Wrapping every update in batch()
reduces clarity and is unnecessary.

To coordinate batch updates across multiple SimpleAppState / RefAppState
instances, use AppStateGroup:

  final group = AppStateGroup([simpleState, refState]);

  // Any member's batch() now batches the whole group:
  simpleState.batch(() {
    simpleSlot.set(1);
    refSlot.set(someObject);
  });

================================================================
8. CASTER (SimpleAppState ONLY)
================================================================

When T is a typed collection (List<X> or Map<K, V>), you MUST supply a caster.
After internal deep-copy, generic collections lose their type parameter.
The caster restores the correct type by casting only — no copying.

  final logsSlot = appState.slot<List<String>>(
    'logs',
    initial: [],
    caster: (raw) => (raw as List).cast<String>(),
  );

  final configSlot = appState.slot<Map<String, int>>(
    'config',
    initial: {},
    caster: (raw) => (raw as Map).cast<String, int>(),
  );

  // Nested example:
  final nestedSlot = appState.slot<Map<String, List<String>>>(
    'nested',
    initial: {},
    caster: (raw) => (raw as Map<String, dynamic>).map(
      (k, v) => MapEntry(k, (v as List).cast<String>()),
    ),
  );

Caster rules:
  - Cast types only. Do not copy or create new objects inside the caster.
  - Primitives (int, double, bool, String, null) do NOT need a caster.
  - Custom classes extending CloneableFile do NOT need a caster.
  - RefAppState.slot does NOT support the caster parameter.

================================================================
9. STORABLE VALUE TYPES
================================================================

A value stored in a SimpleAppState slot must be one of:

  (a) A primitive: int, double, bool, String, or null
  (b) A List or Map composed recursively of (a) or (c)
  (c) A custom class that extends CloneableFile
      (from the file_state_manager package)

Any other type MUST be wrapped in a CloneableFile subclass before storing.

Common types that REQUIRE a CloneableFile wrapper:
  - DateTime, Duration
  - Color, Offset, Rect, Size
  - Any Flutter framework class
  - Any domain object with identity or behavior

Types that do NOT require a wrapper:
  - int, double, bool, String, null
  - List<String>, Map<String, int>, etc. (all-primitive collections)

NEVER store secrets (passwords, API keys, tokens) in slots. Slot values
participate in persistence, undo/redo, debug listeners, and crash reports.
For authentication status, store only derived non-sensitive state:
  final isLoggedInSlot = appState.slot<bool>('is_logged_in', initial: false);

================================================================
10. CloneableFile REQUIREMENTS
================================================================

Every custom class stored in a SimpleAppState slot MUST extend CloneableFile
and implement ALL of the following three members:

1. clone()
   - Returns a new instance that is a full deep copy.
   - All properties must be copied.
   - Nested CloneableFile objects must also be cloned via their own clone().

2. toDict()
   - Returns Map<String, dynamic>.
   - The Map may contain ONLY:
       • primitive values (int, double, bool, String, null)
       • List or Map composed recursively of the above
       • Other CloneableFile instances converted via their own toDict()
   - No raw Flutter types, no non-serializable objects.

3. factory fromDict(Map<String, dynamic> src)
   - Fully reconstructs the object from the Map produced by toDict().
   - Must be deterministic and complete.

Also:
   - Override == and hashCode for correct equality and rebuild behavior.
   - The className static constant must be unique across all CloneableFile
     subclasses used in the same fromDictMap.

Example:

  class AppTimestamp extends CloneableFile {
    static const String className = 'AppTimestamp';
    final int epochMillis;

    AppTimestamp(this.epochMillis);

    factory AppTimestamp.fromDateTime(DateTime dt) =>
        AppTimestamp(dt.millisecondsSinceEpoch);

    DateTime toDateTime() =>
        DateTime.fromMillisecondsSinceEpoch(epochMillis);

    @override
    AppTimestamp clone() => AppTimestamp(epochMillis);

    @override
    Map<String, dynamic> toDict() => {
          'className': className,
          'epochMillis': epochMillis,
        };

    factory AppTimestamp.fromDict(Map<String, dynamic> src) =>
        AppTimestamp(src['epochMillis'] as int);

    @override
    bool operator ==(Object other) =>
        other is AppTimestamp && other.epochMillis == epochMillis;

    @override
    int get hashCode => epochMillis.hashCode;
  }

================================================================
11. OBJECTS THAT MUST NEVER BE STORED IN SLOTS
================================================================

The following must NEVER be stored in any slot, even inside a CloneableFile:

  - FocusNode, TextEditingController, ScrollController
  - Stream, StreamController, Future
  - Callbacks / closures
  - BuildContext
  - AnimationController, TickerProvider
  - Any object tied to the Flutter widget lifecycle

These objects represent runtime behavior, not application state.
Create, own, and dispose them within widgets or State objects.

================================================================
12. UTILITY FUNCTIONS — DO NOT ACCESS SLOTS DIRECTLY
================================================================

Utility / helper functions must not access StateSlots or SimpleAppState
directly. Doing so hides state mutations and breaks explicit state flow.

  WRONG:
    // util.dart
    static void addLog(String message) {
      logsSlot.update((prev) { prev.add(message); return prev; }); // ❌
    }

  CORRECT:
    // util.dart
    static List<String> addLog(List<String> prev, String message) {
      prev.add(message);
      return prev;
    }

    // call site
    logsSlot.update((prev) => MyUtil.addLog(prev, message));

Utilities transform values. StateSlots own state. Keep these roles separate.

================================================================
13. PERSISTENCE (SimpleAppState ONLY)
================================================================

Serialization:
  final dict = appState.toDict();  // returns Map<String, dynamic>

Restoration at startup (before slots are declared — rare):
  final appState = SimpleAppState.fromDict(src, fromDictMap);

Late loading after slots are declared (typical, e.g. from cloud storage):
  appState.loadFromDict(src, fromDictMap);
  // All slots must be declared via slot<T>() before calling this.
  // Notifies all listeners in a single batch after loading.
  // Pass notifyListeners: false to suppress notifications until ready.

fromDictMap maps each CloneableFile's className to its fromDict factory:
  final fromDictMap = {
    AppTimestamp.className: (m) => AppTimestamp.fromDict(m),
  };

================================================================
14. UNDO / REDO PATTERN
================================================================

Undo/redo uses FileStateManager from the file_state_manager package.

Setup:
  final fsm = FileStateManager(appState, stackSize: 20);
  appState.setStateListener((state) {
    fsm.push(state);  // called once per logical commit (not per set())
  });

Undo:
  final previous = fsm.undo();
  if (previous != null) {
    fsm.skipNextPush();              // prevent restoring from re-entering history
    appState.replaceDataFrom(previous as SimpleAppState);
  }

Redo:
  final next = fsm.redo();
  if (next != null) {
    fsm.skipNextPush();
    appState.replaceDataFrom(next as SimpleAppState);
  }

replaceDataFrom() replaces only internal data while keeping all slot
references and widget subscriptions intact. It triggers a full rebuild.

================================================================
15. FORBIDDEN PATTERNS (SUMMARY)
================================================================

NEVER do these:

  ✗ Call slot.set() or slot.update() inside build()
  ✗ Call setState() — rebuilds are triggered automatically
  ✗ Mutate the value returned by StateSlot.get()
      (it is a copy; mutations are silently discarded)
  ✗ Store FocusNode, Stream, Future, BuildContext, or callbacks in a slot
  ✗ Create an extra copy inside update() for SimpleAppState
      (prev is already a deep copy; double-copying wastes memory)
  ✗ Use a caster on RefAppState.slot
  ✗ Store a type that requires CloneableFile without extending it
  ✗ Implement toDict() that embeds non-serializable types
  ✗ Define SimpleAppState or slots inside a widget class
  ✗ Use relative imports for files that define or use application state
  ✗ Derive slot names from widget identity (hashCode, runtimeType, etc.)
  ✗ Store secrets, passwords, or API keys in slots
  ✗ Access slots directly from utility or helper functions
  ✗ Scatter StateSlotBuilder across the widget tree as a substitute
    for declaring slot dependencies in SlotStatefulWidget.slots
  ✗ Wrap every single-slot update in batch() unnecessarily

================================================================
READY
================================================================

I understand all the rules above. Please provide:
  1. The slot definitions (lib/ui/app_state.dart contents)
  2. The screen skeleton or mock UI to implement
  3. Any interactions that should update state
```
