# PowerUI
## A declarative SwiftUI-inspired UI framework for Mac OS X PowerPC

**Depends on:** SwiftPPC  
**Primary renderer:** AppKit on Mac OS X 10.5 Leopard  
**Secondary renderer:** AppKit on Mac OS X 10.4 Tiger  
**Design goal:** SwiftUI-like source ergonomics without depending on Apple's SwiftUI, AttributeGraph or modern OS frameworks

**Document status:** implementation strategy; no framework code exists in this repository yet

---

# 0. Executive summary and feasibility gate

PowerUI should begin as a small, testable declarative core rather than as a catalogue of SwiftUI APIs. The first useful vertical slice is:

```text
ViewBuilder -> description tree -> reconciliation -> layout -> AppKit mount
       ^                                  |
       +------------- @State -------------+
```

Before committing to the public syntax shown in this document, create a compiler probe for the exact SwiftPPC toolchain and verify:

| Language/runtime feature | Required for proposed syntax | Fallback if unavailable |
|---|---:|---|
| associated types and generics | yes | no practical fallback; block the project |
| closures and Objective-C interop | yes | no practical fallback; block the AppKit renderer |
| property wrappers | for `@State`/`@Binding` | explicit `State`/`Binding` values |
| result builders | for stack closure syntax | explicit tuple/array child initializers |
| opaque result types (`some View`) | for SwiftUI-shaped `body` | type erasure at the public boundary |
| `@main` | for application syntax | generated or handwritten startup glue |

The probe must compile and run on the oldest supported target. Until it passes, examples using these features are design targets rather than confirmed source compatibility.

The architecture has three firm constraints:

1. `PowerUICore` must be testable without AppKit.
2. AppKit objects and mutations remain confined to the main thread.
3. Identity, ownership and invalidation semantics must be documented and tested before expanding the component API.

## Non-goals for the first release

- binary or behavioral compatibility with Apple SwiftUI;
- private Apple frameworks or a general-purpose dependency graph;
- multiple production renderers;
- implicit animation, advanced navigation or exhaustive styling;
- pixel-identical rendering across Tiger, Leopard and modern macOS.

---

# 1. Goal

PowerUI is a new, open-source declarative UI framework designed for SwiftPPC applications.

It should make code like this practical on Leopard:

```swift
import PowerUI

struct CounterView: View {
    @State private var count = 0

    var body: some View {
        VStack(spacing: 12) {
            Text("Count: \(count)")

            HStack {
                Button("-") { count -= 1 }
                Button("+") { count += 1 }
            }
        }
        .padding(20)
    }
}
```

while rendering ordinary native AppKit controls:

```text
PowerUI View tree
       |
PowerUI reconciler
       |
PowerUI layout / adapters
       |
NSView / NSControl / NSWindow
       |
Leopard AppKit
```

PowerUI is **not** a port of Apple's SwiftUI binary framework and should not depend on private Apple frameworks.

The desirable source compatibility target is "SwiftUI-shaped", not perfect implementation compatibility.

---

# 2. Why a fresh implementation is preferable

OpenSwiftUI is a valuable research reference and proves that significant portions of SwiftUI's public model can be independently implemented. However its current architecture and supported toolchains are modern, and much of its behavior relies on an AttributeGraph-compatible graph implementation and contemporary Swift features.

For Leopard PPC, directly porting the whole project would likely create a dependency tree larger than the framework we actually need.

Therefore PowerUI should:

- study OpenSwiftUI's public architecture;
- reuse open-source ideas/code only where licensing and dependency cost make sense;
- avoid Apple's private `AttributeGraph` entirely;
- implement a deliberately simpler state/reconciliation engine;
- render directly to public Leopard AppKit APIs;
- keep memory and CPU cost appropriate for G4/G5 hardware.

The result should feel familiar to a SwiftUI developer while remaining technically honest about the platform underneath.

---

# 3. API compatibility policy

Prefer SwiftUI-compatible names where semantics are close:

```text
View
Text
Image
Button
TextField
Toggle
VStack
HStack
ZStack
Spacer
ScrollView
List
ForEach
Group
Divider
@State
@Binding
@Environment
```

Do not claim compatibility for APIs whose behavior differs significantly.

PowerUI-specific functionality should have explicit names.

Example:

```swift
.powerWindowStyle(.leopardHUD)
.onLegacyEvent(...)
```

Avoid creating an `import SwiftUI` module. Applications should explicitly `import PowerUI`; modern projects can use conditional imports/type aliases if desired.

---

# 4. Architectural principle: retained native views, declarative descriptions

SwiftUI's internals are sophisticated. PowerUI does not need to reproduce them.

Use this simpler model:

```text
Swift value-tree descriptions
            |
            v
     Reconciliation tree
            |
            v
    Retained RenderNodes
            |
            v
       AppKit views
```

A `View` value is not necessarily an `NSView`. It describes desired UI.

Each mounted element has a persistent `RenderNode` containing:

- identity;
- current properties;
- child nodes;
- optional AppKit object;
- layout information;
- state storage keys;
- event handlers;
- environment snapshot/version.

On update, PowerUI evaluates the body again, compares the new description with the mounted tree and performs the minimum practical mutations.

Do **not** start with a general-purpose dependency graph like AttributeGraph. A predictable tree reconciler is vastly easier to port and debug.

---

# 5. Core protocols

A possible v1 model:

```swift
public protocol View {
    associatedtype Body: View

    @ViewBuilder
    var body: Body { get }
}
```

Primitive views can use an internal marker:

```swift
public protocol PrimitiveView: View {
    func _makeNode(context: BuildContext) -> RenderNode
    func _updateNode(_ node: RenderNode, context: BuildContext)
}
```

For primitives, `Body` may use a sentinel type similar to `NeverView`.

Avoid depending on hidden compiler intrinsics used by SwiftUI.

---

# 6. ViewBuilder

PowerUI needs result builders.

```swift
@resultBuilder
public enum ViewBuilder {
    public static func buildBlock() -> EmptyView
    public static func buildBlock<V: View>(_ v: V) -> V
    public static func buildBlock<A: View, B: View>(_ a: A, _ b: B) -> TupleView2<A, B>
    ...
}
```

Because enormous generic tuple types can punish compile time and code size on an old target, consider an internal type-erasure boundary after body evaluation:

```text
strongly typed user DSL
       -> lightweight AnyViewNode descriptors
       -> reconciler
```

This preserves source ergonomics while preventing the renderer from becoming a generic-metaprogramming museum.

---

# 7. State system without AttributeGraph

## 7.1 `@State`

A `View` is ephemeral, therefore state cannot live only inside the struct instance.

Use a persistent `StateStore` owned by the mounted tree.

Conceptual key:

```text
ViewIdentityPath + property slot
```

Example:

```swift
@propertyWrapper
public struct State<Value> {
    private let initialValue: Value
    private var location: StateLocation<Value>?

    public var wrappedValue: Value {
        get { ... }
        nonmutating set {
            location?.set(newValue)
        }
    }

    public var projectedValue: Binding<Value> { ... }
}
```

When state changes:

```text
StateLocation.set
   -> mark owning node dirty
   -> schedule update on main run loop
   -> reevaluate affected subtree
   -> reconcile
   -> layout if needed
```

No global graph is required initially.

## 7.2 Invalidations

Use hierarchical dirty flags:

```text
needsBodyUpdate
needsPropertiesUpdate
needsLayout
needsDisplay
```

Coalesce mutations until the next run-loop turn.

On Leopard:

```text
CFRunLoop / NSRunLoop
```

can schedule the flush.

## 7.3 `Binding`

Simple closure-backed implementation:

```swift
public struct Binding<Value> {
    private let getter: () -> Value
    private let setter: (Value) -> Void

    public var wrappedValue: Value {
        get { getter() }
        nonmutating set { setter(newValue) }
    }
}
```

Optimize allocations later if profiling proves necessary.

---

# 8. Identity and reconciliation

Identity errors are among the easiest ways to produce a UI framework that behaves as if possessed.

Use two identity modes:

## Structural identity

For static child positions:

```text
Root/VStack/child[0]/Text
Root/VStack/child[1]/Button
```

## Explicit identity

For collections:

```swift
ForEach(messages, id: \.id) { message in ... }
```

Reconciliation rules:

1. same primitive/container type + same identity -> update existing node;
2. different type or identity -> unmount old subtree and mount new one;
3. keyed collections -> reorder/reuse nodes by key;
4. state storage follows identity, not struct memory address.

Start with O(n) keyed diffing using dictionaries. Exotic diff algorithms are unnecessary for typical G4-era list sizes.

---

# 9. Renderer abstraction

Define a renderer boundary even though AppKit is initially the only renderer.

```swift
protocol Renderer {
    func mount(_ node: RenderNode, parent: RenderNode?)
    func update(_ node: RenderNode, changes: ChangeSet)
    func unmount(_ node: RenderNode)
    func layout(_ root: RenderNode, in bounds: Rect)
}
```

Concrete implementation:

```text
PowerUIAppKitRenderer
```

This provides future options:

- modern macOS AppKit backend for testing;
- headless renderer for snapshots/tests;
- HTML/debug renderer;
- UIKit experiment;
- GNUstep experiment.

Do not build those now. Architecture is cheap; maintaining four renderers is not.

---

# 10. AppKit mapping

Recommended mappings:

| PowerUI | Leopard backend |
|---|---|
| `Text` | non-editable borderless `NSTextField` or custom text cell |
| `Image` | `NSImageView` |
| `Button` | `NSButton` |
| `TextField` | `NSTextField` |
| `SecureField` | `NSSecureTextField` |
| `Toggle` | checkbox-style `NSButton` |
| `Slider` | `NSSlider` |
| `ProgressView` | `NSProgressIndicator` |
| `ScrollView` | `NSScrollView` |
| `List` | `NSTableView` where possible |
| `Divider` | custom `NSView` / box |
| `Color` | `NSColor` wrapper |
| `WindowGroup` | `NSWindowController` + windows |
| `Menu` | `NSMenu` |
| `Alert` | `NSAlert` |

Do not custom draw native controls unless AppKit cannot provide required behavior. Native controls give accessibility, keyboard behavior and authentic system appearance essentially for free, a rare case where old software behaves generously.

---

# 11. Layout engine

AppKit 10.5 predates Auto Layout. PowerUI therefore needs its own deterministic layout engine.

## 11.1 Proposed layout protocol

```swift
protocol LayoutNode {
    func sizeThatFits(_ proposal: ProposedSize) -> Size
    func place(in bounds: Rect)
}
```

`ProposedSize` can contain finite or unspecified width/height.

## 11.2 Stack algorithm

For `VStack`:

1. subtract padding and fixed spacing;
2. measure fixed/intrinsic children;
3. calculate flexible children (`Spacer` etc.);
4. distribute remaining dimension;
5. place according to alignment.

Likewise for `HStack`.

## 11.3 Intrinsic AppKit measurement

Leaf controls ask AppKit for natural sizing:

```text
NSCell.cellSize
NSControl sizeToFit
NSString drawing APIs
NSImage.size
```

Wrap these behind renderer-specific measurement methods.

## 11.4 Layout phases

```text
body evaluation
 -> reconciliation
 -> measure
 -> place
 -> mutate NSView frames
 -> display
```

Cache measured sizes by:

```text
node id + proposal + style version + content version
```

Invalidate cache only when relevant inputs change.

---

# 12. Modifiers

Model modifiers as wrapper values or accumulated attributes.

Example:

```swift
Text("Hello")
    .font(.headline)
    .foregroundColor(.label)
    .padding(8)
    .frame(minWidth: 100)
```

Avoid generating a physical `NSView` for every modifier.

Classify modifiers:

### Property modifiers

Flatten into node configuration:

```text
font
foreground color
enabled
opacity
help text
```

### Layout modifiers

Represent in the layout tree:

```text
padding
frame
fixedSize
layoutPriority
```

### Structural modifiers

Create nodes:

```text
background
overlay
clip
```

This keeps AppKit hierarchy shallow.

---

# 13. Environment

Provide a small typed environment store:

```swift
public protocol EnvironmentKey {
    associatedtype Value
    static var defaultValue: Value { get }
}
```

Each subtree receives a lightweight environment snapshot or persistent parent-linked map.

Initial values:

```text
isEnabled
controlSize
font
locale
layoutDirection
accentColor
window
application
```

Avoid copying a large dictionary for every node. Use structural sharing or a parent chain with small override maps.

---

# 14. Events and control targets

Objective-C target/action does not naturally retain Swift closures in the form we want.

Use bridge objects:

```text
NSButton
  target -> ActionTrampoline : NSObject
                 |
                 -> Swift closure
```

The RenderNode retains the trampoline for the lifetime of the control.

Use equivalent delegates/trampolines for:

- `NSTextFieldDelegate` behavior;
- table view data/delegate;
- window delegates;
- menu actions.

Avoid associated-object tricks unless confirmed on Leopard's Objective-C runtime.

---

# 15. Text

Text is deceptively expensive.

Phase 1:

```swift
Text("plain text")
```

support:

- system font;
- bold/italic via font descriptors where available;
- foreground color;
- line wrapping;
- alignment;
- selectable flag optionally.

Phase 2:

- attributed runs;
- links;
- inline images;
- markdown-lite for PowerGPT.

Potential backend:

```text
NSString / NSAttributedString
AppKit text measurement/drawing
```

Avoid building a custom font shaping engine.

---

# 16. List and virtualization

A chat client cannot instantiate ten thousand full message view trees casually on a G4.

`List` should have two implementations:

## Native table mode

For row-shaped children:

```text
NSTableView
```

with reusable row/cell host containers.

## Generic lazy stack mode

For arbitrary heterogeneous content:

```text
NSScrollView
 + viewport-aware child mounting
```

Only create/render nodes near the visible viewport plus a buffer.

Provide:

```swift
LazyVStack
```

before attempting a perfect SwiftUI `List` clone.

PowerGPT should use virtualized message rows from the beginning.

---

# 17. Images

`Image` sources:

```swift
Image(named: "logo")
Image(nsImage: image)
Image(data: data)
```

Backend:

```text
NSImage / NSImageView
ImageIO where useful
```

Add async-like loading later in PowerFoundation integration.

Cache decoded images with a bounded LRU cache. RAM on real PowerPC systems is not an abstract concept.

---

# 18. Navigation and presentation

Desktop UI should not blindly mimic iPhone navigation.

PowerUI should provide SwiftUI-shaped concepts but adapt them to Mac idioms.

```swift
NavigationSplitView {
    Sidebar()
} detail: {
    Detail()
}
```

On Leopard this may map to:

```text
NSSplitView + sidebar NSTableView + detail host
```

Presentation:

```text
.sheet -> NSWindow sheet
.alert -> NSAlert
.popover -> custom/panel initially; NSPopover unavailable on Leopard
```

Where Leopard lacks a native control, use a compatibility implementation without pretending the missing API exists.

---

# 19. Scenes and application lifecycle

Provide:

```swift
public protocol App {
    associatedtype Body: Scene
    @SceneBuilder var body: Body { get }
}
```

Example:

```swift
@main
struct PowerGPTApp: App {
    var body: some Scene {
        WindowGroup("PowerGPT") {
            ChatView()
        }
        .commands {
            CommandMenu("Conversation") {
                Button("New Chat") { ... }
            }
        }
    }
}
```

Internally:

```text
PowerUI ApplicationMain
      |
NSApplication.sharedApplication
      |
PowerUIApplicationDelegate
      |
Scene manager
      |
NSWindowController(s)
```

If `@main` proves problematic on the selected Swift compiler baseline, provide generated startup glue while preserving the API to user code.

---

# 20. Styling and classic Mac identity

Do not imitate current macOS visually on Leopard. Use system controls and let Leopard look like Leopard.

Add optional higher-level themes for applications like PowerGPT:

```text
.system
.leopard
.tiger
.aquaUnified
.chatBubble
```

But prefer semantic style primitives:

```swift
.buttonStyle(.bordered)
.listStyle(.sidebar)
.textFieldStyle(.rounded)
```

with backend mapping appropriate to the OS version.

For the iOS 6/iMessage-inspired PowerGPT design, build reusable PowerUI components rather than baking chat appearance into the framework core.

---

# 21. Animation

Phase 1: no implicit graph-driven animation.

Provide explicit basic animations:

```swift
withAnimation(.easeInOut(duration: 0.2)) {
    expanded.toggle()
}
```

Implementation options:

- `NSAnimation`;
- timer/run-loop interpolation;
- Core Animation only where available/reliable.

Animate a small set of scalar properties:

```text
frame origin
frame size
opacity
```

Do not make animation a prerequisite for reconciliation.

---

# 22. Concurrency model

PowerUI v1 is **main-thread-bound**.

Rules:

- UI tree evaluation occurs on main thread;
- state invalidation is delivered to main thread;
- AppKit objects are touched only on main thread;
- background operations communicate through callbacks/events.

Later, if SwiftPPC gains async/await, PowerUI can provide helpers without rewriting its core.

---

# 23. PowerFoundation integration

PowerUI should not own networking or persistence.

Example architecture:

```swift
final class ChatModel {
    let client: HTTPClient

    func send(_ text: String, completion: @escaping (Result<Message, Error>) -> Void) {
        ...
    }
}
```

PowerUI state observes model callbacks:

```swift
client.send(request) { result in
    MainQueue.async {
        self.messages.append(...)
    }
}
```

If libdispatch is not suitable on the target, PowerFoundation exposes its own `MainQueue.async` abstraction backed by `CFRunLoopPerformBlock` where available or a custom run-loop source.

---

# 24. Observability model

Do not begin with contemporary `@Observable` macro support.

Start with:

```text
@State
@Binding
ObservableObject-like protocol (optional)
manual invalidation
```

Potential lightweight model:

```swift
protocol ObservableModel: AnyObject {
    var objectWillChange: Signal<Void> { get }
}
```

`Signal` can be a tiny observer collection with weak subscriptions.

Later syntax sugar can be added without changing renderer internals.

---

# 25. Accessibility

Native AppKit controls automatically provide a useful baseline.

PowerUI should expose semantic modifiers:

```swift
.accessibilityLabel("Send")
.accessibilityHelp("Send the current message")
```

Map to Leopard accessibility APIs where available.

Custom-drawn controls must explicitly implement accessibility before being considered production-ready.

---

# 26. Resource loading

PowerUI apps need predictable bundle resources:

```swift
Image(resource: "AvatarPlaceholder")
String(localized: "send_button")
```

PowerUI should wrap `NSBundle` rather than assume modern Swift `Bundle.module` behavior.

Build tooling generates resource manifests and copies assets into `.app/Contents/Resources`.

---

# 27. Localization

Phase 1:

- `.strings` files;
- `NSLocalizedString` bridge;
- UTF-8/UTF-16 correctness;
- locale/environment propagation.

Do not replicate modern String Catalog infrastructure.

---

# 28. Testing PowerUI without hardware for every test

Create a headless representation:

```text
View source
 -> reconciler
 -> TestRenderer
 -> tree snapshot
```

Example snapshot:

```text
VStack spacing=8
  Text value="Hello"
  Button title="Continue" enabled=true
```

Test:

- builder semantics;
- state retention;
- identity;
- keyed collections;
- diffing;
- layout calculations;
- environment inheritance.

Then maintain smaller hardware integration tests for actual AppKit rendering.

Optional screenshot tests can run on a dedicated Leopard machine, but pixel equality across GPUs/fonts should not be the primary correctness mechanism.

---

# 29. Performance budget

PowerUI should be designed for machines where a 1 GHz G4 is plausible.

Initial targets for ordinary UI updates:

- avoid whole-app reevaluation where subtree invalidation suffices;
- no permanent polling timers for state;
- reuse AppKit controls;
- virtualize long lists;
- coalesce state updates per run-loop turn;
- keep view hierarchy shallow;
- avoid reflection in hot paths;
- minimize heap allocation caused by modifiers/builders;
- cache text/image measurement.

Provide debug counters:

```text
body evaluations
nodes mounted
nodes updated
nodes removed
layout passes
AppKit views allocated
state invalidations
```

Debug overlay concept:

```swift
PowerUIDebug.enablePerformanceHUD()
```

On Leopard this can be a tiny floating `NSPanel`.

---

# 30. Memory management

PowerUI must be hostile to retain cycles by design.

Common risk:

```text
RenderNode
 -> control
 -> trampoline
 -> closure
 -> model/view owner
 -> RenderNode
```

Define ownership rules explicitly.

Recommended:

- RenderNode strongly owns AppKit view + trampolines;
- parent strongly owns children;
- child weakly references parent;
- callbacks capture models according to user semantics;
- internal scheduling closures capture nodes weakly when possible;
- unmount clears actions/delegates aggressively.

Add leak-oriented stress tests that mount/unmount a subtree thousands of times.

---

# 31. Public component roadmap

## PowerUI 0.1 - Core

```text
View
ViewBuilder
AnyView
EmptyView
Group
VStack
HStack
ZStack
Spacer
Text
Button
@State
@Binding
padding
frame
background
foregroundColor
font
```

## PowerUI 0.2 - Forms

```text
TextField
SecureField
Toggle
Slider
ProgressView
Form-ish containers
focus basics
keyboard actions
```

## PowerUI 0.3 - Scrolling/data

```text
ScrollView
ForEach
LazyVStack
List
Divider
selection
```

## PowerUI 0.4 - App shell

```text
App
Scene
WindowGroup
commands/menus
Alert
Sheet
Settings window
```

## PowerUI 0.5 - Rich app support

```text
NavigationSplitView-like API
Toolbar
context menu
drag/drop subset
pasteboard helpers
custom drawing
basic animations
```

## PowerUI 1.0

Criteria:

- sufficient to implement PowerGPT without direct AppKit code for normal UI;
- stable state identity semantics;
- virtualized chat/list performance;
- Leopard G4/G5 hardware test suite;
- documented escape hatch to AppKit.

---

# 32. AppKit escape hatch

PowerUI must never trap developers behind an incomplete abstraction.

Provide:

```swift
AppKitView(
    make: { MyLegacyView(frame: .zero) },
    update: { view, context in
        ...
    }
)
```

and controller equivalent:

```swift
AppKitViewController(...)
```

This is essential for WebKit, advanced text views, media controls and odd Leopard-only components.

---

# 33. Example: PowerGPT architecture

```text
PowerGPTApp
 |
 +-- WindowGroup
      |
      +-- MainSplitView
          |
          +-- ConversationSidebar
          |    +-- List / LazyVStack
          |
          +-- ChatView
               +-- MessageScrollView
               |    +-- LazyVStack
               |         +-- MessageBubble
               |
               +-- ComposerView
                    +-- TextEditor/AppKit bridge
                    +-- Send Button

PowerFoundation
 |
 +-- HTTPClient
 +-- JSON
 +-- TLS/libcurl
 +-- Keychain adapter
```

Message streaming can be represented by incremental state updates. Coalesce token updates to, for example, a display-friendly cadence rather than rebuilding the text view for every network byte.

---

# 34. Example: source compatibility with modern SwiftUI

Application code may use a tiny compatibility prelude:

```swift
#if SWIFTPPC
import PowerUI
#else
import SwiftUI
#endif
```

Then write common components against the intersection of APIs.

Do **not** contort PowerUI to support every SwiftUI edge case. Maintain a documented compatibility profile:

```text
PowerUI SwiftUI Compatibility Level: Core-1
```

A linter could eventually flag APIs outside the profile.

---

# 35. Suggested source tree

```text
PowerUI/
├── Sources/
│   ├── PowerUICore/
│   │   ├── View.swift
│   │   ├── ViewBuilder.swift
│   │   ├── AnyView.swift
│   │   ├── Identity.swift
│   │   ├── Environment.swift
│   │   ├── State.swift
│   │   ├── Binding.swift
│   │   ├── Transaction.swift
│   │   ├── Reconciler/
│   │   ├── Layout/
│   │   └── Modifiers/
│   ├── PowerUIControls/
│   │   ├── Text.swift
│   │   ├── Button.swift
│   │   ├── TextField.swift
│   │   ├── Image.swift
│   │   ├── ScrollView.swift
│   │   └── List.swift
│   ├── PowerUIAppKit/
│   │   ├── AppKitRenderer.swift
│   │   ├── NativeViewNode.swift
│   │   ├── ControlAdapters/
│   │   ├── Trampolines/
│   │   ├── Measurement/
│   │   └── Windowing/
│   └── PowerUITestRenderer/
├── Tests/
│   ├── CoreTests/
│   ├── StateTests/
│   ├── ReconciliationTests/
│   ├── LayoutTests/
│   └── AppKitHardwareTests/
└── Examples/
    ├── Counter/
    ├── ControlsGallery/
    ├── TodoList/
    └── ChatPrototype/
```

---

# 36. Implementation sequence for Codex

Do this in order:

0. pin the SwiftPPC compiler revision and run the feature probe from section 0;
1. define geometry primitives (`Size`, `Rect`, `ProposedSize`);
2. define `View`, primitive views and `ViewBuilder`, using only syntax proven by the probe;
3. implement type-erased description tree;
4. implement RenderNode and structural identity;
5. implement reconciler using a `TestRenderer`;
6. add `@State` with persistent location storage;
7. add `Binding`;
8. add dirty-node scheduling abstraction;
9. implement `VStack`, `HStack`, `Spacer` layout;
10. test layout headlessly;
11. create AppKit renderer;
12. implement `Text`;
13. implement `Button` + target/action trampoline;
14. implement modifiers and padding/frame;
15. mount a Counter application on Leopard;
16. add `TextField`, `Toggle`, `Image`;
17. add `ScrollView`;
18. add keyed `ForEach`;
19. add lazy/virtualized vertical stack;
20. add `List`;
21. add scenes/windows/application lifecycle;
22. add environment;
23. add sheets/alerts/menus;
24. dogfood with a PowerGPT chat prototype;
25. profile G4 and fix allocations/layout invalidations before expanding API.

## First vertical-slice acceptance criteria

The Counter milestone is complete only when all of the following hold:

- the compiler probe result and exact toolchain revision are recorded;
- headless tests prove mount, no-op update, property update, replacement and unmount;
- state survives reevaluation when identity is stable and resets when identity changes;
- multiple state writes in one run-loop turn produce one reconciliation flush;
- `VStack`, `HStack`, `Spacer`, padding and frame have deterministic layout tests;
- a Leopard application renders native `Text` and `Button` controls and updates the count;
- 10,000 repeated updates do not grow the mounted node or AppKit view counts;
- mount/unmount stress testing shows no framework-owned retain cycle;
- a debug build reports body evaluations, reconciliation operations and layout passes.

This milestone deliberately excludes `List`, networking, navigation and animation.

Do not implement animation, navigation or fancy styling until the Counter + Todo + Chat prototypes all survive repeated updates correctly.

---

# 37. Technical rules

- AppKit is the source of truth for native control behavior.
- Never depend on private Apple frameworks.
- Never require AttributeGraph.
- Keep core independent of AppKit where practical.
- UI changes happen on main thread.
- State follows view identity.
- Long collections must support virtualization.
- Modifiers should not create unnecessary `NSView`s.
- Provide escape hatches to raw AppKit.
- Prefer deterministic algorithms over clever metaprogramming.
- Keep PPC32 memory/code-size impact visible in profiling.
- Add Tiger-specific behavior via platform adapters, not scattered version checks.

---

# 38. Research references

1. OpenSwiftUI repository and architecture: https://github.com/OpenSwiftUIProject/OpenSwiftUI
2. OpenSwiftUI `State`: https://github.com/OpenSwiftUIProject/OpenSwiftUI/blob/main/Sources/OpenSwiftUICore/Data/State/State.swift
3. OpenSwiftUI graph host: https://github.com/OpenSwiftUIProject/OpenSwiftUI/blob/main/Sources/OpenSwiftUICore/Graph/GraphHost.swift
4. OpenAttributeGraph: https://github.com/OpenSwiftUIProject/OpenAttributeGraph
5. Swift compiler architecture: https://www.swift.org/documentation/swift-compiler/

OpenSwiftUI/OpenAttributeGraph should be treated as research references, not assumed dependencies. Their current implementations target modern Swift/toolchains and OpenAttributeGraph remains incomplete, reinforcing the case for a smaller purpose-built reconciler for PowerUI.

---

# 39. Definition of success

PowerUI 1.0 succeeds when this class of application can be written predominantly declaratively:

```swift
struct ChatView: View {
    @State private var text = ""
    @State private var messages: [Message] = []

    var body: some View {
        VStack(spacing: 0) {
            LazyVStack(messages, id: \.id) { message in
                MessageBubble(message: message)
            }

            Divider()

            HStack {
                TextField("Message", text: $text)
                Button("Send", action: send)
            }
            .padding(8)
        }
    }
}
```

and the resulting application:

- launches on real Leopard PowerPC hardware;
- uses native AppKit controls;
- retains state correctly across body reevaluation;
- handles hundreds/thousands of list items with virtualization;
- does not leak substantially during navigation/repeated updates;
- remains responsive on a G4-class system;
- requires raw AppKit only for genuinely specialized controls.

That is enough to make SwiftPPC a productive application platform rather than merely a language port.
