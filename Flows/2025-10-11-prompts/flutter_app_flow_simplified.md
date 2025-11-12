# Flutter Application Flow: User-AI Interaction Tree (Simplified)

**Project**: TestAI (fe_kommentall)  
**Framework**: Flutter/Dart  
**Purpose**: Multi-branch AI conversation interface with sub-prompt capabilities

---

## Table of Contents

1. [Application Entry Point](#1-application-entry-point)
2. [Home Screen Initialization](#2-home-screen-initialization)
3. [User Input Flow](#3-user-input-flow)
4. [Prompt Processing Flow](#4-prompt-processing-flow)
5. [Home Screen Manager Processing](#5-home-screen-manager-processing)
6. [Thread Map Update](#6-thread-map-update-core-logic)
7. [Tree View Update](#7-tree-view-update)
8. [Tree Rendering Flow](#8-tree-rendering-flow)
9. [Tree Structure Building](#9-tree-structure-building)
10. [Tree Item Rendering](#10-tree-item-rendering)
11. [Individual Tree Item Display](#11-individual-tree-item-display)
12. [Sub-Prompt Preparation](#12-sub-prompt-preparation)
13. [Node Selection](#13-node-selection)
14. [Data Models](#14-data-models)
15. [Complete Flow Summary](#15-complete-flow-summary)
16. [Key Architectural Patterns](#key-architectural-patterns)

---

## 1. Application Entry Point

### Flow
```
main.dart
  └─ main()
     └─ runApp(MyApp())
```

---

## 2. Home Screen Initialization

### `HomeScreen` (StatefulWidget)
**Location**: `lib/screens/home_screen.dart`

### Flow
```
HomeScreen
  └─ HomeScreenState.initState()
     ├─ Initialize PromptArgs (empty state)
     ├─ Create BackendService()
     ├─ Create PointManager(backendService)
     ├─ Create TreeViewManager()
     ├─ Create ShardManager(pointManager, backendService)
     ├─ Create ThreadManager(pointManager, shardManager, backendService, threadMap, onMapUpdated)
     ├─ Create ContextManager(threadManager, apiService, threadMap)
     └─ Create HomeScreenManager(all managers, callbacks)
```

### State Variables
```
- threadMap: Map<String, Point> (cache of all Points)
- treeViewList: List<Point> (Points to display)
- promptArgs: PromptArgs (current prompt context)
- loadingExchanges: Set<String> (track loading states)
- isLoading: bool (global loading state)
- errorMessage: String? (error display)
```

---

## 3. User Input Flow

### Flow
```
PromptInputField (StatefulWidget)
  └─ User types in TextField
     └─ User presses Send button
        └─ PromptInputField._handleSendRequest()
           ├─ Validate text (not empty)
           └─ widget.onPrompt(text)
              └─ HomeScreen.onPrompt(text)
```

---

## 4. Prompt Processing Flow

### Flow
```
HomeScreen.onPrompt(promptText)
  ├─ Clear errors: setState()
  ├─ Determine prompt type (regular vs sub-prompt)
  ├─ Create/update PromptArgs
  ├─ Call HomeScreenManager.handlePrompt(...)
  ├─ Clear promptArgs: setState()
  └─ Handle errors: setState()
```

---

## 5. Home Screen Manager Processing

### Flow
```
HomeScreenManager.handlePrompt(...)
  ├─ Validate prompt text
  ├─ onLoadingStateChanged(true)
  ├─ ThreadManager.updateThreadMap(promptText, promptArgs)
  │  └─ Returns: Point newPoint
  ├─ Create AiModelProperties (model config)
  ├─ ContextManager.createPromptContext(newPoint)
  │  └─ Returns: List<RequestMessage> promptContextMessages
  ├─ Track loading state:
  │  ├─ Extract exchangeId from newPoint
  │  └─ updateLoadingExchanges(add exchangeId)
  ├─ ApiService.sendPromptToAi(pointId, promptContextMessages, aiModelProperties)
  │  └─ Returns: Point updatedNewPoint (with response)
  ├─ updateLoadingExchanges(remove exchangeId)
  ├─ Update threadMap[updatedNewPoint.id]
  ├─ onMapUpdated()
  ├─ populateTreeViewList(treeViewList, updateTreeViewList)
  ├─ onClearPromptInput()
  └─ onLoadingStateChanged(false)
```

---

## 6. Thread Map Update (Core Logic)

### Flow - Decision Branch
```
ThreadManager.updateThreadMap(promptText, promptArgs)
  ├─ PointManager.createNewPoint(promptArgs, promptText)
  │  └─ Returns: Point newPoint
  │
  ├─ IF newPoint.id == newPoint.parentPointId:
  │  └─ ROOT POINT: Add to threadMap and return
  │
  ├─ Get Parent Point:
  │  ├─ IF NOT in threadMap:
  │  │  ├─ PointManager.getParentPoint(fetchArgs, newPoint.id)
  │  │  └─ Cache in threadMap
  │  └─ ELSE: Retrieve from threadMap
  │
  └─ DECISION BRANCH:
     │
     ├─ BRANCH A: SHARDING FLOW (isShardChild=true, selectedText exists)
     │  ├─ ShardManager.addShardToParentPoint(newPoint.id, parentPoint, promptArgs)
     │  │  └─ Returns: Point updatedParentPoint (with new Shard)
     │  ├─ getParentShardId(updatedParentPoint, newPoint.id)
     │  ├─ newPoint.copyWith(parentShardId: parentShardId)
     │  ├─ Update threadMap with both points
     │  └─ onMapUpdated()
     │
     └─ BRANCH B: REGULAR FLOW (continue conversation)
        ├─ PointManager.updateParentPoint(newPoint.id, parentPoint)
        │  └─ Returns: Point updatedParentPoint
        ├─ Update threadMap with both points
        └─ onMapUpdated()
```

---

## 7. Tree View Update

### Flow
```
HomeScreenManager.populateTreeViewList(treeViewList, updateTreeViewList)
  └─ TreeViewManager.updateTreeViewList(threadMap, treeViewList)
     └─ Returns: List<Point> newTreeViewList
        └─ updateTreeViewList(newTreeViewList)
           └─ HomeScreen.setState()
```

---

## 8. Tree Rendering Flow

### Flow
```
TreeSliverThread (StatefulWidget)
  │
  ├─ _TreeSliverThreadState.initState()
  │  ├─ Create TreeSliverManager()
  │  └─ _updateTreeList()
  │
  ├─ didUpdateWidget(oldWidget)
  │  └─ IF treeViewList or threadMap changed:
  │     └─ _updateTreeList()
  │
  └─ _updateTreeList()
     └─ TreeSliverManager.buildFlatTreeList(threadMap, treeViewList)
        └─ Returns: List<TreeNodeModel> _flatTreeList
           └─ setState()
              └─ Triggers build()
```

### Build Method
```
TreeSliverThread.build()
  └─ SliverList.builder(itemCount: _flatTreeList.length)
     └─ For each TreeNodeModel:
        ├─ Check if isLoading (exchangeId in loadingExchanges)
        └─ Return TreeSliverItem(node, isLoading, onTap, prepareSubPromptInput)
```

### Node Tap Handler
```
_handleNodeTap(node)
  ├─ IF node.hasChildren:
  │  ├─ TreeSliverManager.toggleExpansion(node.id)
  │  └─ _updateTreeList()
  └─ widget.onNodeSelected(node.pointId)
```

---

## 9. Tree Structure Building

### Main Algorithm
```
TreeSliverManager.buildFlatTreeList(threadMap, treeViewList)
  ├─ _buildTreeStructure(treeViewList, threadMap)
  │  └─ Returns: List<TreeNodeModel> treeNodes
  │
  └─ For each node in treeNodes:
     └─ _flattenNode(node, flatList) [Recursive]
        └─ Returns: List<TreeNodeModel> flatList
```

### Build Tree Structure (Three-Pass Algorithm)
```
TreeSliverManager._buildTreeStructure(treeViewList, threadMap)
  │
  ├─ PASS 1: Create all nodes
  │  └─ For each Point in treeViewList:
  │     ├─ Check if Point has valid shards
  │     │
  │     ├─ IF has valid shards:
  │     │  └─ _createShardPointNode(point, exchange, nodeMap, threadMap)
  │     │     ├─ Creates main prompt node (no response)
  │     │     ├─ Creates shard segment nodes (response parts)
  │     │     ├─ Adds child points under shard segments
  │     │     └─ Creates "after" segment if text remains
  │     │
  │     └─ ELSE:
  │        └─ Create TreeNodeModel (regular, full response)
  │
  ├─ PASS 2: Build hierarchy
  │  └─ For each Point:
  │     ├─ IF root (parentPointId == id):
  │     │  ├─ Add to rootNodes
  │     │  └─ Add shard segments as siblings (if any)
  │     │
  │     └─ ELSE (child):
  │        ├─ Find parent (could be shard segment)
  │        ├─ Add as child to parent
  │        ├─ Update level (parent.level + 1)
  │        └─ Add child's shard segments as siblings (if any)
  │
  └─ PASS 3: Update flags
     └─ For each node:
        └─ IF node.children.isNotEmpty:
           └─ node.copyWith(hasChildren: true)
```

### Create Shard Point Node
```
TreeSliverManager._createShardPointNode(point, exchange, nodeMap, threadMap)
  ├─ Create main prompt node (responseContent: null)
  ├─ Filter valid shards (validate anchor positions)
  ├─ Sort shards by start position
  │
  └─ For each shard:
     ├─ Extract response segment (currentPosition to shard.endPosition)
     ├─ Create TreeNodeModel for segment (level 0, sibling of prompt)
     ├─ Store in nodeMap (NOT child of main node)
     │
     ├─ For each child in shard.shardChildren:
     │  ├─ Get childPoint from threadMap
     │  ├─ _buildChildNode(childPoint, shardSegment.id, shard.shardId, level=1, ...)
     │  ├─ Add child to shard segment's children
     │  └─ IF child has shards:
     │     └─ Add child's shard segments as siblings
     │
     └─ Create "after" segment if text remains after last shard
```

### Build Child Node (Recursive)
```
TreeSliverManager._buildChildNode(childPoint, parentId, parentShardId, level, threadMap, nodeMap)
  ├─ Check if childPoint has valid shards
  │
  ├─ IF has valid shards:
  │  └─ _createShardPointNodeRecursive(childPoint, exchange, level, parentId, parentShardId, ...)
  │     └─ [Same logic as _createShardPointNode but with baseLevel parameter]
  │
  └─ ELSE:
     └─ Create TreeNodeModel (regular child node)
```

### Flatten Node (Recursive)
```
TreeSliverManager._flattenNode(node, flatList)
  ├─ flatList.add(node)
  │
  └─ IF node.isExpanded AND node.children.isNotEmpty:
     └─ For each child:
        └─ _flattenNode(child, flatList) [Recursive]
```

**Visual Representation**:
```
Regular Point (no shards):
├─ Prompt: "What's Earth?"
└─ Response: "Third planet from the Sun"

Shard Point (with shards):
├─ Prompt: "What's Earth?"
├─ Response Segment 1: "Third planet from the "
├─ Response Segment 2: "Sun"
│  └─ Child Point (sub-prompt)
│     ├─ Prompt: "Sun - what's this?"
│     └─ Response: "The star..."
└─ Response Segment 3: " [remaining text]"
```

---

## 10. Tree Item Rendering

### Flow
```
TreeSliverThread.build()
  └─ SliverList.builder()
     └─ For each index:
        ├─ Get TreeNodeModel from _flatTreeList[index]
        ├─ Check isLoading (node.exchangeId in loadingExchanges)
        └─ Return TreeSliverItem(node, isLoading, onTap, prepareSubPromptInput)
```

---

## 11. Individual Tree Item Display

### Flow
```
TreeSliverItem.build()
  └─ Container(margin: left = node.level * _levelIndent)
     └─ Card
        └─ InkWell(onTap: onTap)
           └─ Column
              ├─ IF node.hasChildren:
              │  └─ _buildExpansionHeader()
              │
              ├─ IF node.promptContent != null:
              │  └─ _buildPromptSection()
              │     └─ _buildContentSection(promptStyle)
              │
              └─ Response rendering (varies by nodeType):
                 │
                 ├─ IF nodeType == exchange AND responseContent != null:
                 │  └─ isLoading ? _buildStatusSection() : _buildResponseSection()
                 │
                 ├─ IF nodeType == shard AND promptContent != null:
                 │  └─ isLoading ? _buildStatusSection() 
                 │     : (responseContent ? _buildResponseSection() : _buildStatusSection())
                 │
                 └─ IF nodeType == shardResponse:
                    └─ _buildResponseSection()
```

### Content Section (with Sub-Prompt Menu)
```
TreeSliverItem._buildContentSection(config)
  └─ _buildSelectableText(text, textStyle)
     └─ SelectableText(contextMenuBuilder: ...)
        └─ IF selectedText.isNotEmpty:
           └─ AdaptiveTextSelectionToolbar with "Sub-prompt" option
              └─ onPressed: _handleSubPrompt(selectedText, selection)
```

### Handle Sub-Prompt
```
TreeSliverItem._handleSubPrompt(selectedText, selection)
  ├─ Clipboard.setData(ClipboardData(text: selectedText))
  ├─ Create PromptArgs with shard information:
  │  ├─ currentPointId: node.pointId
  │  ├─ parentPointId: node.pointId
  │  ├─ parentShardId: node.shardId
  │  ├─ isShardChild: true
  │  ├─ selectedText, startPosition, endPosition
  ├─ prepareSubPromptInput(promptArgs)
  └─ ContextMenuController.removeAny()
```

---

## 12. Sub-Prompt Preparation

### Flow
```
HomeScreen.prepareSubPromptInput(promptArgs)
  ├─ Validate selectedText
  ├─ setState():
  │  ├─ Clear errorMessage
  │  └─ Update this.promptArgs (mark isShardChild = true)
  ├─ Show SnackBar: "Text copied to input..."
  └─ WidgetsBinding.instance.addPostFrameCallback():
     ├─ PromptInputField.setText(selectedText)
     └─ PromptInputField.focusInput()
```

---

## 13. Node Selection

### Flow
```
HomeScreen.onNodeSelected(currentPointId)
  ├─ setState():
  │  └─ Update promptArgs (mark isShardChild = false)
  └─ PromptInputField.focusInput()
```

---

## 14. Data Models

**Location**: `lib/models/`

### Point Model
```
Point
  ├─ id: String
  ├─ parentPointId: String
  ├─ pointChildren: List<String>
  ├─ parentShardId: String?
  ├─ shardsList: List<Shard>
  ├─ exchangesList: List<Exchange>
  └─ metadata: Metadata?
```

### Shard Model
```
Shard
  ├─ shardId: String
  ├─ shardChildren: List<String>
  └─ anchor: Anchor
      ├─ startPosition: int
      ├─ endPosition: int
      └─ selectedText: String
```

### Exchange Model
```
Exchange
  ├─ exchangeId: String
  ├─ exchangeTitle: String?
  ├─ prompt: Prompt
  │   ├─ model: String
  │   ├─ promptMessage: PromptMessage (role, content)
  │   └─ maxTokens: int
  └─ response: Response?
      ├─ id, model, created, etc.
      ├─ choices: List<Choice>
      │   └─ message: Message (role, content)
      └─ usage: Usage
```

### TreeNodeModel
```
TreeNodeModel (UI representation)
  ├─ id: String
  ├─ pointId: String
  ├─ level: int (indentation)
  ├─ isExpanded: bool
  ├─ hasChildren: bool
  ├─ promptContent: String?
  ├─ responseContent: String?
  ├─ nodeType: NodeType (exchange, shard, shardResponse)
  ├─ children: List<TreeNodeModel>
  └─ ... (other fields)
```

### PromptArgs
```
PromptArgs (context for prompt)
  ├─ currentPointId: String
  ├─ parentPointId: String?
  ├─ parentShardId: String?
  ├─ isShardChild: bool
  ├─ selectedText: String?
  ├─ startPosition, endPosition: int?
  └─ ... (other fields)
```

---

## 15. Complete Flow Summary

### Example: "What's Earth?" Conversation

#### Step 1: Initial Prompt

```
User types: "What's Earth?"
  ↓
PromptInputField._handleSendRequest()
  ↓
HomeScreen.onPrompt("What's Earth?")
  ↓
HomeScreenManager.handlePrompt(...)
  ↓
ThreadManager.updateThreadMap()
  ├─ PointManager.createNewPoint() → Point T001
  └─ (T001.id == T001.parentPointId) → Root point
  ↓
ContextManager.createPromptContext(T001)
  ↓
OpenAIService.sendPromptToAi(...) → Returns T001 with response
  ↓
TreeViewManager.updateTreeViewList() → treeViewList = [T001]
  ↓
TreeSliverManager.buildFlatTreeList()
  ├─ _buildTreeStructure() → Creates TreeNodeModel for T001
  └─ _flattenNode() → [T001]
  ↓
TreeSliverThread.build() → TreeSliverItem renders:
  ├─ _buildPromptSection() → Blue "You" box
  └─ _buildResponseSection() → Green "AI Assistant" box
```

**UI Result**:
```
┌──────────────────────────────┐
│ 👤 You                       │
│ What's Earth?                │
└──────────────────────────────┘
┌──────────────────────────────┐
│ 🤖 AI Assistant              │
│ Third planet from the Sun    │
└──────────────────────────────┘
```

---

#### Step 2: First Sub-Prompt (Selected: "Sun")

```
User selects "Sun" → Right-click → "Sub-prompt"
  ↓
TreeSliverItem._handleSubPrompt("Sun", selection)
  ├─ Clipboard.setData()
  └─ Creates PromptArgs (isShardChild=true, selectedText="Sun", positions)
  ↓
HomeScreen.prepareSubPromptInput(promptArgs)
  ├─ setState() updates promptArgs
  └─ PromptInputField.setText("Sun")
  ↓
User modifies: "Sun - what's this?" and sends
  ↓
HomeScreen.onPrompt("Sun - what's this?")
  ↓
HomeScreenManager.handlePrompt(...)
  ↓
ThreadManager.updateThreadMap() [isShardChild=true]
  ├─ ShardManager.addShardToParentPoint(T001, promptArgs)
  │  └─ Creates Shard S01 in T001 (anchor: start=24, end=27)
  ├─ PointManager.createNewPoint() → Point T002 (parentShardId: S01)
  └─ Updates T001.shardsList[0].shardChildren = ["T002"]
  ↓
OpenAIService.sendPromptToAi(T002, ...) → Returns T002 with response
  ↓
TreeViewManager.updateTreeViewList() → treeViewList = [T001, T002]
  ↓
TreeSliverManager.buildFlatTreeList()
  ├─ _buildTreeStructure()
  │  └─ Detects T001 has valid shards
  │     └─ _createShardPointNode(T001, ...)
  │        ├─ Creates main prompt node (T001)
  │        ├─ Creates shard segment (T001_shard_S01) with response text
  │        └─ Adds T002 as child of shard segment (level 1)
  └─ _flattenNode() → [T001_prompt, T001_shard_S01, T002]
  ↓
UI renders nested structure:
  ├─ T001 prompt
  ├─ T001_shard_S01 (response segment)
  └─── T002 (indented, level 1)
```

**UI Result**:
```
┌──────────────────────────────┐
│ 👤 You                       │
│ What's Earth?                │
└──────────────────────────────┘
┌──────────────────────────────┐
│ ✂️ Shard Segment             │
│ Third planet from the Sun    │
│   ┌──────────────────────────┤
│   │ 👤 Shard                 │
│   │ Sun - what's this?       │
│   └──────────────────────────┤
│   ┌──────────────────────────┐
│   │ 🤖 AI Assistant          │
│   │ The star                 │
│   └──────────────────────────┘
└──────────────────────────────┘
```

---

#### Step 3: Second Sub-Prompt (Selected: "star")

```
User selects "star" → "star - what's this?"
  ↓
Similar flow creates:
  ├─ Shard S02 in Point T002
  ├─ Point T003 (child of T002, shard S02)
  └─ TreeSliverManager recursively splits T002's response
  ↓
UI renders with 2 levels of nesting:
  T001 → T001_shard_S01 → T002 → T002_shard_S02 → T003
```

---

#### Step 4: Third Sub-Prompt (Selected: "sphere")

```
User selects "sphere" → "sphere - what's this?"
  ↓
Creates:
  ├─ Shard S03 in Point T003
  ├─ Point T004 (child of T003, shard S03)
  └─ TreeSliverManager recursively splits at 3 levels
  ↓
Final nested structure (as seen in screenshot):
  T001
    └─ T001_shard_S01
       └─ T002
          └─ T002_shard_S02
             └─ T003
                └─ T003_shard_S03
                   └─ T004
```

**Final UI** (matching screenshot):
```
┌──────────────────────────────┐
│ 👤 You                       │
│ What's Earth?                │
└──────────────────────────────┘
┌──────────────────────────────┐
│ ✂️ Shard Segment             │
│ Third planet from the Sun    │
│   ┌──────────────────────────┤
│   │ 👤 Shard                 │
│   │ Sun - what's this?       │
│   └──────────────────────────┤
│   ┌──────────────────────────┐
│   │ ✂️ Shard Segment         │
│   │ The star                 │
│   │   ┌──────────────────────┤
│   │   │ 👤 Shard             │
│   │   │ star - what's this?  │
│   │   └──────────────────────┤
│   │   ┌──────────────────────┐
│   │   │ ✂️ Shard Segment     │
│   │   │ A luminous sphere    │
│   │   │   ┌──────────────────┤
│   │   │   │ 👤 Shard         │
│   │   │   │ sphere - what's? │
│   │   │   └──────────────────┤
│   │   │   ┌──────────────────┐
│   │   │   │ 🤖 AI Assistant  │
│   │   │   │ 3D geometric...  │
│   │   │   └──────────────────┘
│   │   └──────────────────────┘
│   │   ┌──────────────────────┐
│   │   │ ✂️ Response Segment  │
│   │   │ of plasma...         │
│   │   └──────────────────────┘
│   │   [... more segments ...]
│   └──────────────────────────┘
└──────────────────────────────┘
```

---

### Key Call Sequence for Complete Flow

```
User Action (type/select/send)
  ↓
PromptInputField._handleSendRequest()
  ↓
HomeScreen.onPrompt()
  ↓
HomeScreenManager.handlePrompt()
  ├─ ThreadManager.updateThreadMap()
  │  ├─ PointManager.createNewPoint()
  │  │  └─ BackendService.createPoint()
  │  ├─ IF isShardChild:
  │  │  └─ ShardManager.addShardToParentPoint()
  │  │     └─ BackendService.updatePoint()
  │  └─ PointManager.updateParentPoint()
  │     └─ BackendService.updatePoint()
  ├─ ContextManager.createPromptContext()
  ├─ OpenAIService.sendPromptToAi()
  │  └─ BackendService.updatePoint()
  └─ HomeScreenManager.populateTreeViewList()
     └─ TreeViewManager.updateTreeViewList()
        └─ TreeSliverManager.buildFlatTreeList()
           ├─ TreeSliverManager._buildTreeStructure()
           │  └─ TreeSliverManager._createShardPointNode() [if has shards]
           │     └─ TreeSliverManager._buildChildNode() [recursive]
           └─ TreeSliverManager._flattenNode() [recursive]
  ↓
HomeScreen.setState()
  ↓
TreeSliverThread.didUpdateWidget()
  ↓
TreeSliverThread._updateTreeList()
  ↓
TreeSliverThread.setState()
  ↓
TreeSliverThread.build()
  ├─ SliverList.builder()
  │  └─ For each TreeNodeModel:
  │     └─ TreeSliverItem.build()
  │        ├─ _buildPromptSection()
  │        ├─ _buildResponseSection()
  │        └─ _buildSelectableText()
  │           └─ (custom context menu with "Sub-prompt")
  └─ User interaction continues...
```

---

## Key Architectural Patterns

### 1. Manager Pattern
- **PointManager**: Point CRUD operations
- **ShardManager**: Shard creation/management
- **ThreadManager**: Conversation orchestration
- **ContextManager**: AI context building
- **TreeViewManager**: Tree data transformation
- **TreeSliverManager**: Tree rendering logic
- **HomeScreenManager**: Overall orchestration

### 2. State Management via Callbacks
- Parent (HomeScreen) manages state
- Children receive callbacks
- Callbacks trigger setState() in parent
- UI rebuilds reactively

### 3. Immutable Data Structures
- Never mutate, always copy
- `point.copyWith(field: newValue)`
- Predictable state changes

### 4. Tree Flattening
- Hierarchical → Flat list
- Based on expansion states
- Efficient rendering (SliverList.builder)

### 5. Error Boundaries
- Try-catch at every level
- Graceful degradation
- User feedback on errors
- App never crashes

### 6. Lazy Loading / Caching
- Fetch only when needed
- Cache in threadMap
- Reduce network calls

### 7. Separation of Data/Display
- Point (backend) → TreeNodeModel (display) → Widget (UI)
- Backend changes don't break UI

### 8. Composite Pattern
- TreeNodeModel contains children of same type
- Recursive algorithms naturally

### 9. Factory Pattern
- Specialized node creation methods
- `_createShardPointNode()`
- `_buildChildNode()`

### 10. Observer Pattern (via Callbacks)
- `onMapUpdated.call()`
- Notifies all listeners
- Triggers UI updates

### 11. Production Logging
- `developer.log()` instead of `print()`
- Structured, filterable logs

### 12. Context Preservation
- PromptArgs carries full context
- Easy to pass between methods

### 13. Defensive Programming
- Validate everything
- Check `mounted` before setState
- Null-safe access (`.firstOrNull`)

### 14. Key-based Widget Identity
- GlobalKey for programmatic control
- Preserve state across rebuilds

### 15. Three-Pass Tree Building
- Pass 1: Create nodes
- Pass 2: Build hierarchy
- Pass 3: Update flags

---

## Conclusion

### Core Features
1. Multi-level conversation threading
2. Sub-prompt creation from text
3. Response segmentation
4. Dynamic expand/collapse
5. Granular loading states

### Technical Excellence
1. Production-ready error handling
2. Immutable data structures
3. Manager pattern separation
4. Efficient rendering
5. Type-safe models
6. Comprehensive logging

### User Experience
1. Responsive UI
2. Clear visual hierarchy
3. Color-coded types
4. Contextual actions
5. Per-exchange loading
6. Error feedback

### Scalability
1. Lazy loading
2. Efficient rendering
3. Recursive algorithms
4. Modular architecture
5. Clean separation

---

**End of Document**
