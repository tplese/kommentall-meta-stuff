# Flutter Application Flow: User-AI Interaction Tree

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
6. [Thread Map Update (Core Logic)](#6-thread-map-update-core-logic)
7. [Tree View Update](#7-tree-view-update)
8. [Tree Rendering Flow](#8-tree-rendering-flow)
9. [Tree Structure Building](#9-tree-structure-building)
10. [Tree Item Rendering](#10-tree-item-rendering)
11. [Individual Tree Item Display](#11-individual-tree-item-display)
12. [Sub-Prompt Preparation](#12-sub-prompt-preparation)
13. [Node Selection](#13-node-selection)
14. [Data Models](#14-data-models)
15. [Complete Flow Summary](#15-complete-flow-summary-for-screenshot-example)
16. [Key Architectural Patterns](#key-architectural-patterns)

---

## 1. Application Entry Point

### `main.dart`

```dart
main() → runApp(MyApp())
```

**Purpose**: Entry point that initializes the Flutter application.

---

## 2. Home Screen Initialization

### `HomeScreen` (StatefulWidget)

**Location**: `lib/screens/home_screen.dart`

#### State Initialization (`HomeScreenState.initState()`)

```dart
@override
void initState() {
  super.initState();

  // 1. Initialize PromptArgs with empty state
  promptArgs = PromptArgs(
    currentPointId: '',
    isShardChild: false,
  );

  // 2. Create BackendService instance for API communication
  backendService = BackendService();

  // 3. Create PointManager for Point CRUD operations
  pointManager = PointManager(
    backendService: backendService,
  );

  // 4. Create TreeViewManager for tree visualization logic
  treeViewManager = TreeViewManager();

  // 5. Create ShardManager for handling sub-prompt sharding
  shardManager = ShardManager(
    pointManager: pointManager,
    backendService: backendService,
  );

  // 6. Create ThreadManager for managing conversation threads
  threadManager = ThreadManager(
    pointManager: pointManager,
    shardManager: shardManager,
    backendService: backendService,
    threadMap: threadMap,
    onMapUpdated: onMapUpdated,
  );

  // 7. Create ContextManager for building AI prompt context
  contextManager = ContextManager(
    threadManager: threadManager,
    apiService: currentApiService,
    threadMap: threadMap,
  );

  // 8. Create HomeScreenManager as orchestrator
  homeScreenManager = HomeScreenManager(
    pointManager: pointManager,
    shardManager: shardManager,
    threadManager: threadManager,
    treeViewManager: treeViewManager,
    contextManager: contextManager,
    threadMap: threadMap,
    onMapUpdated: onMapUpdated,
    onLoadingStateChanged: (isLoading) {
      if (mounted) {
        setState(() {
          this.isLoading = isLoading;
        });
      }
    },
    onErrorOccurred: showErrorSnackBar,
    onClearPromptInput: () {
      promptInputFieldKey.currentState?.clearText();
    },
  );
}
```

#### UI State Variables

```dart
final Map<String, Point> threadMap = {};           // Cache of all Points
late List<Point> treeViewList = [];                // Points to display in tree
late PromptArgs promptArgs;                        // Current prompt context
final Set<String> loadingExchanges = {};           // Track loading states
final GlobalKey<PromptInputFieldState> promptInputFieldKey = GlobalKey();
bool isLoading = false;                            // Global loading state
String? errorMessage;                              // Error display
```

**Key Points**:
- All managers are initialized with dependency injection
- Callbacks are passed to managers for UI updates
- State is managed centrally in HomeScreen

---

## 3. User Input Flow

### `PromptInputField` (StatefulWidget)

**Location**: `lib/ui/widgets/prompt_input_field.dart`

#### User Interaction Sequence

```dart
// User types in TextField
TextField(
  controller: _textController,
  focusNode: _focusNode,
  enabled: !widget.isLoading,
  onSubmitted: widget.isLoading ? null : (_) => _handleSendRequest(),
)

// User presses Send button or Enter key
void _handleSendRequest() {
  final text = _textController.text.trim();
  
  if (text.isEmpty) {
    // Show validation feedback
    ScaffoldMessenger.of(context).showSnackBar(
      const SnackBar(
        content: Text('Please enter a message'),
        duration: Duration(seconds: 1),
        behavior: SnackBarBehavior.floating,
      ),
    );
    return;
  }

  try {
    // Call parent callback
    widget.onPrompt(text);  // → HomeScreen.onPrompt()
  } catch (e) {
    debugPrint('Error sending request: $e');
    // Show error feedback
  }
}
```

**Key Points**:
- TextEditingController manages input state
- FocusNode enables programmatic focus control
- Validation prevents empty submissions
- Callbacks propagate to parent (HomeScreen)

---

## 4. Prompt Processing Flow

### `HomeScreen.onPrompt(String promptText)`

**Location**: `lib/screens/home_screen.dart`

```dart
Future<void> onPrompt(String promptText) async {
  // 1. Clear previous errors
  if (mounted) {
    setState(() {
      errorMessage = null;
    });
  }

  try {
    // 2. Determine prompt type (regular vs sub-prompt)
    final bool isSubPrompt = promptArgs.isShardChild;
    
    // 3. Create/update PromptArgs
    PromptArgs currentPromptArgs = isSubPrompt
        ? promptArgs  // Use existing args with shard info
        : PromptArgs(
            currentPointId: treeViewList.isNotEmpty ? treeViewList.last.id : '',
            parentPointId: treeViewList.isNotEmpty ? treeViewList.last.id : null,
            isShardChild: false,
          );

    // 4. Call HomeScreenManager to handle the prompt
    await homeScreenManager.handlePrompt(
      promptText: promptText,
      promptArgs: currentPromptArgs,
      updatePromptArgs: (newArgs) {
        if (mounted) {
          setState(() {
            promptArgs = newArgs;
          });
        }
      },
      treeViewList: treeViewList,
      updateTreeViewList: (newTreeViewList) {
        if (mounted) {
          setState(() {
            treeViewList = newTreeViewList;
          });
        }
      },
      loadingExchanges: loadingExchanges,
      updateLoadingExchanges: (newLoadingExchanges) {
        if (mounted) {
          setState(() {
            loadingExchanges.clear();
            loadingExchanges.addAll(newLoadingExchanges);
          });
        }
      },
    );

    // 5. Clear prompt args after successful handling
    if (mounted) {
      setState(() {
        promptArgs = promptArgs.clear();
      });
    }
  } catch (e) {
    debugPrintStack(label: 'Error in onPrompt: $e');
    
    // 6. Display error to user
    if (mounted) {
      setState(() {
        errorMessage = 'Failed to send request: ${e.toString()}';
      });
    }
  } finally {
    // 7. Ensure loading state is cleared
    if (mounted) {
      setState(() {
        isLoading = false;
      });
    }
  }
}
```

**Key Points**:
- Differentiates between regular prompts and sub-prompts
- Uses callbacks to update state in parent component
- Comprehensive error handling prevents app crashes
- Always clears loading state in finally block

---

## 5. Home Screen Manager Processing

### `HomeScreenManager.handlePrompt(...)`

**Location**: `lib/managers/home_screen_manager.dart`

```dart
Future<void> handlePrompt({
  required String promptText,
  required PromptArgs promptArgs,
  required Function(PromptArgs newArgs) updatePromptArgs,
  required List<Point> treeViewList,
  required Function(List<Point> newTreeViewList) updateTreeViewList,
  required Set<String> loadingExchanges,
  required Function(Set<String>) updateLoadingExchanges,
}) async {
  // 1. Validate prompt text
  if (promptText.trim().isEmpty) {
    onErrorOccurred('Please enter a valid prompt');
    return;
  }

  // 2. Set loading state
  onLoadingStateChanged(true);

  try {
    // 3. Update ThreadMap - creates Point with prompt (no response yet)
    Point newPoint = await threadManager.updateThreadMap(
      promptText: promptText,
      promptArgs: promptArgs,
    );
    developer.log('ThreadMap updated with new point: ${newPoint.id}');

    // 4. Create AI model configuration
    AiModelProperties aiModelProperties = AiModelProperties(
      modelProvider: 'openai',
      model: 'gpt-3.5-turbo',
      max_tokens: 50,
    );

    // 5. Build prompt context for AI (includes conversation history)
    List<RequestMessage> promptContextMessages = 
        contextManager.createPromptContext(
          newPoint: newPoint,
        );

    // 6. Track loading state for this specific exchange
    String? exchangeId;
    if (newPoint.exchangesList.isNotEmpty) {
      exchangeId = newPoint.exchangesList.last.exchangeId;
      Set<String> updatedLoading = Set.from(loadingExchanges);
      updatedLoading.add(exchangeId);
      updateLoadingExchanges(updatedLoading);
      developer.log('Added exchange to loading state: $exchangeId');
    }

    // 7. Send Request to AI API
    Point updatedNewPoint = await apiService.sendPromptToAi(
      pointId: newPoint.id,
      promptContextMessages: promptContextMessages,
      aiModelProperties: aiModelProperties,
    );
    developer.log('Received response from AI for point: ${updatedNewPoint.id}');

    // 8. Remove from loading state
    if (exchangeId != null) {
      Set<String> updatedLoading = Set.from(loadingExchanges);
      updatedLoading.remove(exchangeId);
      updateLoadingExchanges(updatedLoading);
      developer.log('Removed exchange from loading state: $exchangeId');
    }

    // 9. Update ThreadMap with returned Point (now has response)
    threadMap[updatedNewPoint.id] = updatedNewPoint;
    onMapUpdated.call();

    // 10. Update tree view for display
    populateTreeViewList(treeViewList, updateTreeViewList);

    // 11. Clear input field
    onClearPromptInput();
    
    // 12. Clear loading state
    onLoadingStateChanged(false);
    developer.log('Prompt handled successfully');
    
  } catch (e, stackTrace) {
    onLoadingStateChanged(false);
    developer.log(
      'Failed to process prompt',
      error: e,
      stackTrace: stackTrace,
    );
    onErrorOccurred('Failed to process prompt: ${e.toString()}');
  }
}
```

**Key Points**:
- Orchestrates entire prompt processing pipeline
- Uses developer.log for debugging (production-ready logging)
- Granular loading state per exchange (not just global)
- Comprehensive error handling with user feedback
- Callbacks enable reactive UI updates

---

## 6. Thread Map Update (Core Logic)

### `ThreadManager.updateThreadMap(...)`

**Location**: `lib/managers/thread_manager.dart`

This is the core logic that determines whether to create a regular continuation or a sub-prompt (shard).

```dart
Future<Point> updateThreadMap({
  required String promptText,
  required PromptArgs promptArgs,
}) async {
  try {
    // STEP 1: Create new Point with prompt (no response yet)
    final Point newPoint = await pointManager.createNewPoint(
        promptArgs: promptArgs, 
        promptText: promptText
    );
    developer.log('Created new point: ${newPoint.id}');

    // STEP 2: Handle Root Point (first message in conversation)
    if (newPoint.id == newPoint.parentPointId) {
      threadMap[newPoint.id] = newPoint;
      developer.log('Added root point to threadMap: ${newPoint.id}');
      return newPoint;
    }

    // STEP 3: Get Parent Point
    Point parentPoint;
    String parentPointId = newPoint.parentPointId;

    if (!threadMap.containsKey(parentPointId)) {
      // Parent not in cache - fetch from backend
      developer.log('Fetching parent point from backend: $parentPointId');
      
      PromptArgs fetchArgs = PromptArgs(
        currentPointId: '',
        parentPointId: parentPointId,
        parentShardId: null,
        isShardChild: false,
        exchangeId: null,
        choiceIndex: null,
        selectedText: null,
        startPosition: null,
        endPosition: null,
        error: null,
      );
      
      parentPoint = await pointManager.getParentPoint(fetchArgs, newPoint.id);
      
      if (parentPoint.id.isNotEmpty) {
        threadMap[parentPoint.id] = parentPoint;
        developer.log('Cached parent point from backend: $parentPointId');
      } else {
        developer.log('Parent point not found: $parentPointId', level: 900);
        throw Exception('Parent point not found: $parentPointId');
      }
    } else {
      // Parent already in cache
      parentPoint = threadMap[parentPointId]!;
      developer.log('Retrieved parent point from threadMap: $parentPointId');
    }

    // STEP 4: Decision Branch - Sharding vs Regular Flow
    
    // ════════════════════════════════════════════════════════════
    // BRANCH A: SHARDING FLOW (Sub-prompt from selected text)
    // ════════════════════════════════════════════════════════════
    if (promptArgs.isShardChild && promptArgs.selectedText != null) {
      try {
        developer.log('Starting sharding flow for point: ${newPoint.id}');

        // Add shard to parent point
        final Point updatedParentPoint = 
            await shardManager.addShardToParentPoint(
              newPointId: newPoint.id,
              parentPoint: parentPoint,
              promptArgs: promptArgs,
            );

        // Extract parent shard ID
        String parentShardId = getParentShardId(updatedParentPoint, newPoint.id);

        // Update new point with shard reference
        final Point updatedNewPoint = newPoint.copyWith(
          parentShardId: parentShardId,
        );

        // Update threadMap with both points
        threadMap[updatedNewPoint.id] = updatedNewPoint;
        threadMap[updatedParentPoint.id] = updatedParentPoint;

        // Notify UI
        onMapUpdated();

        developer.log('Sharding flow completed for point: ${newPoint.id}');
        return updatedNewPoint;
        
      } catch (e, stackTrace) {
        developer.log(
          'Error in sharding flow',
          error: e,
          stackTrace: stackTrace,
        );
        debugPrintStack(label: 'Error in sharding flow: $e');
        rethrow;
      }
    } 
    
    // ════════════════════════════════════════════════════════════
    // BRANCH B: REGULAR FLOW (Continue conversation)
    // ════════════════════════════════════════════════════════════
    else {
      developer.log('Starting regular prompt flow for point: ${newPoint.id}');
      
      // Update parent point to include new child
      final Point updatedParentPoint = await pointManager.updateParentPoint(
        newPointId: newPoint.id,
        parentPoint: parentPoint,
      );

      // Update threadMap with both points
      threadMap[newPoint.id] = newPoint;
      threadMap[updatedParentPoint.id] = updatedParentPoint;

      // Notify UI
      onMapUpdated();

      developer.log('Regular prompt flow completed for point: ${newPoint.id}');
      return newPoint;
    }
  } catch (e, stackTrace) {
    developer.log(
      'Failed to update thread map',
      error: e,
      stackTrace: stackTrace,
    );
    rethrow;
  }
}
```

#### Helper Method: Get Parent Shard ID

```dart
String getParentShardId(Point updatedParentPoint, String newPointId) {
  try {
    final Shard parentShardDTO = updatedParentPoint.shardsList.firstWhere(
      (shard) => shard.shardChildren.contains(newPointId),
      orElse: () => Shard.empty(),
    );

    if (parentShardDTO.shardId.isEmpty) {
      developer.log('Parent shard not found for point: $newPointId', level: 900);
      return '';
    }

    final String parentShardId = parentShardDTO.shardId;
    developer.log('Found parent shard: $parentShardId for point: $newPointId');

    return parentShardId;
  } catch (e, stackTrace) {
    developer.log(
      'Failed to get parent shard ID for point: $newPointId',
      error: e,
      stackTrace: stackTrace,
    );
    return '';
  }
}
```

**Key Points**:
- Central decision point: regular continuation vs sub-prompt
- Parent point fetching with caching strategy
- Immutable updates: Points are copied, not modified
- Comprehensive logging for debugging
- Error handling with rethrow for proper propagation

---

## 7. Tree View Update

### `HomeScreenManager.populateTreeViewList(...)`

**Location**: `lib/managers/home_screen_manager.dart`

```dart
void populateTreeViewList(
    List<Point> treeViewList, 
    Function(List<Point>) updateTreeViewList
) {
  try {
    final List<Point> newTreeViewList = 
        treeViewManager.updateTreeViewList(threadMap, treeViewList);
    
    updateTreeViewList(newTreeViewList);
    developer.log('TreeView list populated successfully');
    
  } catch (e, stackTrace) {
    developer.log(
      'TreeView population failed',
      error: e,
      stackTrace: stackTrace,
    );
    debugPrintStack(label: 'Warning: TreeView population failed: $e');
    // Continue without crashing the app
  }
}
```

**Key Points**:
- TreeViewManager converts threadMap to displayable list
- Failures don't crash the app (logged and continue)
- Callback pattern updates HomeScreen state

---

## 8. Tree Rendering Flow

### `TreeSliverThread` (StatefulWidget)

**Location**: `lib/ui/widgets/tree_sliver_thread.dart`

#### Lifecycle Methods

```dart
class _TreeSliverThreadState extends State<TreeSliverThread> {
  late TreeSliverManager _treeSliverManager;
  List<TreeNodeModel> _flatTreeList = [];

  @override
  void initState() {
    super.initState();
    _treeSliverManager = TreeSliverManager();
    _updateTreeList();
  }

  @override
  void didUpdateWidget(TreeSliverThread oldWidget) {
    super.didUpdateWidget(oldWidget);
    
    // Rebuild tree if data changed
    if (widget.treeViewList != oldWidget.treeViewList ||
        widget.threadMap != oldWidget.threadMap) {
      _updateTreeList();
    }
  }

  void _updateTreeList() {
    try {
      setState(() {
        _flatTreeList = _treeSliverManager.buildFlatTreeList(
          widget.threadMap,
          widget.treeViewList,
        );
      });
    } catch (e) {
      debugPrint('Error updating tree list: $e');
    }
  }
}
```

#### Build Method

```dart
@override
Widget build(BuildContext context) {
  if (_flatTreeList.isEmpty) {
    return const SliverFillRemaining(
      child: Center(child: Text('Start a conversation...')),
    );
  }

  return SliverList.builder(
    itemCount: _flatTreeList.length,
    itemBuilder: (context, index) {
      try {
        TreeNodeModel node = _flatTreeList[index];
        
        // Check if this exchange is loading
        bool isLoading = node.exchangeId != null &&
            widget.loadingExchanges.contains(node.exchangeId);

        return TreeSliverItem(
          node: node,
          isLoading: isLoading,
          onTap: () => _handleNodeTap(node),
          prepareSubPromptInput: widget.prepareSubPromptInput,
        );
      } catch (e) {
        debugPrint('Error building tree item at index $index: $e');
        return const SizedBox.shrink();  // Graceful failure
      }
    },
  );
}
```

#### Node Tap Handler

```dart
void _handleNodeTap(TreeNodeModel node) {
  try {
    // Collapse/expand if node has children
    if (node.hasChildren) {
      _treeSliverManager.toggleExpansion(node.id);
      _updateTreeList();
    }

    // Select node for context (next prompt will continue from here)
    widget.onNodeSelected(node.pointId);
    
  } catch (e) {
    debugPrint('Error handling node tap: $e');
  }
}
```

**Key Points**:
- Reactive to prop changes (didUpdateWidget)
- SliverList.builder for efficient rendering
- Per-item error handling prevents full list failure
- Collapse/expand state managed in TreeSliverManager

---

## 9. Tree Structure Building

### `TreeSliverManager.buildFlatTreeList(...)`

**Location**: `lib/managers/tree_sliver_manager.dart`

This is the **most complex algorithm** in the application. It transforms the Point data structure into a renderable tree.

#### Main Method

```dart
List<TreeNodeModel> buildFlatTreeList(
  Map<String, Point> threadMap,
  List<Point> treeViewList,
) {
  try {
    if (threadMap.isEmpty || treeViewList.isEmpty) {
      return [];
    }

    List<TreeNodeModel> flatList = [];

    // Step 1: Build hierarchical tree structure
    List<TreeNodeModel> treeNodes = _buildTreeStructure(treeViewList, threadMap);

    // Step 2: Flatten the tree based on expansion states
    for (TreeNodeModel node in treeNodes) {
      _flattenNode(node, flatList);
    }

    return flatList;
  } catch (e) {
    debugPrint('Error building flat tree list: $e');
    return [];
  }
}
```

#### Build Tree Structure (Core Algorithm)

```dart
List<TreeNodeModel> _buildTreeStructure(
    List<Point> treeViewList, 
    Map<String, Point> threadMap
) {
  Map<String, TreeNodeModel> nodeMap = {};
  List<TreeNodeModel> rootNodes = [];

  try {
    // ════════════════════════════════════════════════════════════
    // FIRST PASS: Create all nodes
    // ════════════════════════════════════════════════════════════
    for (int i = 0; i < treeViewList.length; i++) {
      Point point = treeViewList[i];

      if (point.exchangesList.isNotEmpty) {
        Exchange exchange = point.exchangesList.first;
        String? responseContent = 
            exchange.response?.choices.firstOrNull?.message.content;

        final contentLength = responseContent?.length ?? 0;
        
        // Check if this point has valid shards
        final hasValidShards = point.shardsList.any((shard) {
          if (shard.anchor.startPosition < 0 || 
              shard.anchor.endPosition < 0) return false;
          if (shard.anchor.startPosition > shard.anchor.endPosition) return false;
          if (shard.anchor.endPosition > contentLength) return false;
          return true;
        });

        if (hasValidShards && responseContent != null) {
          // ┌─────────────────────────────────────────────┐
          // │ CREATE SHARD POINT NODE (Split Response)   │
          // └─────────────────────────────────────────────┘
          debugPrint('Creating SHARD point node');
          TreeNodeModel node = _createShardPointNode(
              point, exchange, nodeMap, threadMap);
          nodeMap[point.id] = node;
          
        } else {
          // ┌─────────────────────────────────────────────┐
          // │ CREATE REGULAR POINT NODE (Full Response)  │
          // └─────────────────────────────────────────────┘
          debugPrint('Creating REGULAR point node');
          TreeNodeModel node = TreeNodeModel(
            id: point.id,
            pointId: point.id,
            parentShardId: point.parentShardId,
            exchangeId: exchange.exchangeId,
            level: 0,  // Will be calculated later
            isExpanded: _expansionState[point.id] ?? true,
            promptContent: exchange.prompt.promptMessage.content,
            responseContent: responseContent,
            promptRole: exchange.prompt.promptMessage.role,
            responseRole: exchange.response?.choices.firstOrNull?.message.role,
            nodeType: point.parentShardId != null 
                ? NodeType.shard 
                : NodeType.exchange,
            children: [],
          );

          nodeMap[point.id] = node;
        }
      }
    }

    // ════════════════════════════════════════════════════════════
    // SECOND PASS: Build hierarchy and add shard segments
    // ════════════════════════════════════════════════════════════
    for (Point point in treeViewList) {
      TreeNodeModel? node = nodeMap[point.id];
      if (node == null) continue;

      if (point.parentPointId == point.id) {
        // ┌─────────────────────────────────────────────┐
        // │ ROOT NODE                                   │
        // └─────────────────────────────────────────────┘
        rootNodes.add(node);
        
        // If this point has shards, add the shard segments as siblings
        if (point.shardsList.isNotEmpty) {
          for (Shard shard in point.shardsList) {
            String shardSegmentKey = '${point.id}_shard_${shard.shardId}';
            TreeNodeModel? shardSegment = nodeMap[shardSegmentKey];
            if (shardSegment != null) {
              rootNodes.add(shardSegment);
            }
          }
          
          // Add "after" segment if it exists
          String afterKey = '${point.id}_after';
          TreeNodeModel? afterSegment = nodeMap[afterKey];
          if (afterSegment != null) {
            rootNodes.add(afterSegment);
          }
        }
      } else {
        // ┌─────────────────────────────────────────────┐
        // │ CHILD NODE                                  │
        // └─────────────────────────────────────────────┘
        // Find correct parent (could be shard segment)
        String parentKey = point.parentShardId != null
            ? '${point.parentPointId}_shard_${point.parentShardId}'
            : point.parentPointId;

        TreeNodeModel? parent = nodeMap[parentKey] ?? nodeMap[point.parentPointId];
        
        if (parent != null) {
          parent.children.add(node);
          node = node.copyWith(
            parentId: parent.id,
            level: parent.level + 1,
          );
          nodeMap[node.id] = node;
          
          // If this child point has shards, add shard segments as siblings
          if (point.shardsList.isNotEmpty) {
            for (Shard shard in point.shardsList) {
              String shardSegmentKey = '${point.id}_shard_${shard.shardId}';
              TreeNodeModel? shardSegment = nodeMap[shardSegmentKey];
              if (shardSegment != null) {
                shardSegment = shardSegment.copyWith(
                  parentId: parent.id,
                  level: parent.level + 1,
                );
                parent.children.add(shardSegment);
                nodeMap[shardSegmentKey] = shardSegment;
              }
            }
            
            // Add "after" segment if it exists
            String afterKey = '${point.id}_after';
            TreeNodeModel? afterSegment = nodeMap[afterKey];
            if (afterSegment != null) {
              afterSegment = afterSegment.copyWith(
                parentId: parent.id,
                level: parent.level + 1,
              );
              parent.children.add(afterSegment);
              nodeMap[afterKey] = afterSegment;
            }
          }
        }
      }
    }

    // ════════════════════════════════════════════════════════════
    // THIRD PASS: Update hasChildren flags
    // ════════════════════════════════════════════════════════════
    for (TreeNodeModel node in nodeMap.values) {
      if (node.children.isNotEmpty) {
        nodeMap[node.id] = node.copyWith(hasChildren: true);
      }
    }

    return rootNodes;
  } catch (e) {
    debugPrint('Error building tree structure: $e');
    return [];
  }
}
```

#### Create Shard Point Node (Response Splitting)

```dart
TreeNodeModel _createShardPointNode(
    Point point, 
    Exchange exchange,
    Map<String, TreeNodeModel> nodeMap, 
    Map<String, Point> threadMap
) {
  String responseContent = exchange.response!.choices.first.message.content;
  int contentLength = responseContent.length;

  // ┌─────────────────────────────────────────────────────────────┐
  // │ Create Main Prompt Node (NO response content)              │
  // └─────────────────────────────────────────────────────────────┘
  TreeNodeModel mainNode = TreeNodeModel(
    id: point.id,
    pointId: point.id,
    parentShardId: point.parentShardId,
    exchangeId: exchange.exchangeId,
    level: 0,
    isExpanded: _expansionState[point.id] ?? true,
    promptContent: exchange.prompt.promptMessage.content,
    responseContent: null,  // ← No response in main node
    promptRole: exchange.prompt.promptMessage.role,
    responseRole: exchange.response?.choices.firstOrNull?.message.role,
    nodeType: point.parentShardId != null 
        ? NodeType.shard 
        : NodeType.exchange,
    children: [],
  );

  // ┌─────────────────────────────────────────────────────────────┐
  // │ Filter out shards with invalid anchors                     │
  // └─────────────────────────────────────────────────────────────┘
  List<Shard> validShards = point.shardsList.where((shard) {
    if (shard.anchor.startPosition < 0) return false;
    if (shard.anchor.endPosition < 0) return false;
    if (shard.anchor.startPosition > contentLength) return false;
    if (shard.anchor.endPosition > contentLength) return false;
    if (shard.anchor.endPosition < shard.anchor.startPosition) return false;
    return true;
  }).toList();

  // If no valid shards, add full response as child
  if (validShards.isEmpty && point.shardsList.isNotEmpty) {
    debugPrint('Warning: Point ${point.id} has invalid shards');
    TreeNodeModel fullResponse = TreeNodeModel(
      id: '${point.id}_full_response',
      pointId: point.id,
      level: 1,
      isExpanded: true,
      responseContent: responseContent,
      responseRole: exchange.response?.choices.firstOrNull?.message.role,
      nodeType: NodeType.shardResponse,
      children: [],
    );
    mainNode.children.add(fullResponse);
    nodeMap['${point.id}_full_response'] = fullResponse;
    return mainNode;
  }

  // ┌─────────────────────────────────────────────────────────────┐
  // │ Sort shards by start position                              │
  // └─────────────────────────────────────────────────────────────┘
  List<Shard> sortedShards = List.from(validShards)
    ..sort((a, b) => a.anchor.startPosition.compareTo(b.anchor.startPosition));

  int currentPosition = 0;

  // ┌─────────────────────────────────────────────────────────────┐
  // │ Create shard segments                                      │
  // └─────────────────────────────────────────────────────────────┘
  for (int i = 0; i < sortedShards.length; i++) {
    Shard shard = sortedShards[i];
    int shardStart = shard.anchor.startPosition;
    int shardEnd = shard.anchor.endPosition;

    // Calculate segment text (from currentPosition to shardEnd)
    String segmentText;
    try {
      if (currentPosition < shardEnd && currentPosition < contentLength) {
        int safeEnd = shardEnd < contentLength ? shardEnd : contentLength;
        segmentText = responseContent.substring(currentPosition, safeEnd);
      } else {
        continue;  // Skip invalid segment
      }
      
      // Create the shard segment at level 0 (sibling of prompt)
      TreeNodeModel shardSegment = TreeNodeModel(
        id: '${point.id}_shard_${shard.shardId}',
        pointId: point.id,
        level: 0,  // ← Same level as prompt node (sibling)
        isExpanded: _expansionState['${point.id}_shard_${shard.shardId}'] ?? true,
        hasChildren: shard.shardChildren.isNotEmpty,
        responseContent: segmentText,
        responseRole: exchange.response?.choices.firstOrNull?.message.role,
        nodeType: NodeType.shardResponse,
        children: [],
        shardId: shard.shardId,
        shardStartPosition: currentPosition,
        shardEndPosition: shardEnd,
      );

      // Store in nodeMap (NOT added as child of mainNode)
      nodeMap['${point.id}_shard_${shard.shardId}'] = shardSegment;

      // ┌───────────────────────────────────────────────────────────┐
      // │ Add all child points under this shard segment            │
      // └───────────────────────────────────────────────────────────┘
      for (String childId in shard.shardChildren) {
        try {
          // CRITICAL: Get from threadMap, not treeViewList
          Point? childPoint = threadMap[childId];

          if (childPoint != null && childPoint.exchangesList.isNotEmpty) {
            debugPrint('Adding shard child $childId under shard ${shard.shardId}');
            
            // Build child node at level 1 (indented under shard segment)
            TreeNodeModel childNode = _buildChildNode(
              childPoint,
              shardSegment.id,
              shard.shardId,
              1,  // ← Level 1 (indented)
              threadMap,
              nodeMap,
            );

            shardSegment.children.add(childNode);
            nodeMap[childId] = childNode;
            
            // If this child has shards, add its shard segments as siblings
            if (childPoint.shardsList.isNotEmpty) {
              debugPrint('Child $childId has shards, adding segments as siblings');
              for (Shard childShard in childPoint.shardsList) {
                String childShardSegmentKey = 
                    '${childPoint.id}_shard_${childShard.shardId}';
                TreeNodeModel? childShardSegment = nodeMap[childShardSegmentKey];
                if (childShardSegment != null) {
                  childShardSegment = childShardSegment.copyWith(
                    parentId: shardSegment.id,
                    level: 1,  // Same level as the child node
                  );
                  shardSegment.children.add(childShardSegment);
                  nodeMap[childShardSegmentKey] = childShardSegment;
                }
              }
              
              // Add "after" segment if it exists
              String childAfterKey = '${childPoint.id}_after';
              TreeNodeModel? childAfterSegment = nodeMap[childAfterKey];
              if (childAfterSegment != null) {
                childAfterSegment = childAfterSegment.copyWith(
                  parentId: shardSegment.id,
                  level: 1,
                );
                shardSegment.children.add(childAfterSegment);
                nodeMap[childAfterKey] = childAfterSegment;
              }
            }
          } else {
            debugPrint('Warning - shard child $childId not found in threadMap');
          }
        } catch (e) {
          debugPrint('Error adding shard child $childId: $e');
        }
      }

      currentPosition = shardEnd;
    } catch (e) {
      debugPrint('Error creating shard segment: $e');
    }
  }

  // ┌─────────────────────────────────────────────────────────────┐
  // │ Create "after" segment if there's text after the last shard│
  // └─────────────────────────────────────────────────────────────┘
  if (currentPosition < contentLength) {
    try {
      String afterText = responseContent.substring(currentPosition);
      if (afterText.isNotEmpty) {
        TreeNodeModel afterSegment = TreeNodeModel(
          id: '${point.id}_after',
          pointId: point.id,
          level: 0,  // Same level as prompt node
          isExpanded: true,
          responseContent: afterText,
          responseRole: exchange.response?.choices.firstOrNull?.message.role,
          nodeType: NodeType.shardResponse,
          children: [],
        );
        // Store in nodeMap (NOT added as child of mainNode)
        nodeMap['${point.id}_after'] = afterSegment;
      }
    } catch (e) {
      debugPrint('Error creating after segment: $e');
    }
  }

  return mainNode;
}
```

#### Flatten Tree (Based on Expansion State)

```dart
void _flattenNode(TreeNodeModel node, List<TreeNodeModel> flatList) {
  try {
    // Always add the node itself
    flatList.add(node);

    // Add children if node is expanded
    if (node.isExpanded && node.children.isNotEmpty) {
      for (TreeNodeModel child in node.children) {
        _flattenNode(child, flatList);  // Recursive
      }
    }
  } catch (e) {
    debugPrint('Error flattening node: $e');
  }
}
```

**Key Points**:
- Three-pass algorithm: create nodes → build hierarchy → update flags
- Response splitting: segments are siblings, not children of prompt
- Recursive handling of nested shards
- Expansion state persisted in manager
- Comprehensive validation of shard anchors
- Graceful handling of invalid data

**Visual Representation**:

```
Regular Point (no shards):
├─ Prompt: "What's Earth?"
└─ Response: "Third planet from the Sun"

Shard Point (with shards):
├─ Prompt: "What's Earth?"
├─ Response Segment 1: "Third planet from the "
├─ Response Segment 2: "Sun"
│  └─ Child Point (sub-prompt about "Sun")
│     ├─ Prompt: "Sun - what's this?"
│     └─ Response: "The star..."
└─ Response Segment 3: " [remaining text]"
```

---

## 10. Tree Item Rendering

### `TreeSliverThread.build(...)`

**Location**: `lib/ui/widgets/tree_sliver_thread.dart`

```dart
@override
Widget build(BuildContext context) {
  if (_flatTreeList.isEmpty) {
    return const SliverFillRemaining(
      child: Center(child: Text('Start a conversation...')),
    );
  }

  return SliverList.builder(
    itemCount: _flatTreeList.length,
    itemBuilder: (context, index) {
      try {
        TreeNodeModel node = _flatTreeList[index];
        
        // Check if this exchange is currently loading
        bool isLoading = node.exchangeId != null &&
            widget.loadingExchanges.contains(node.exchangeId);

        return TreeSliverItem(
          node: node,
          isLoading: isLoading,
          onTap: () => _handleNodeTap(node),
          prepareSubPromptInput: widget.prepareSubPromptInput,
        );
      } catch (e) {
        debugPrint('Error building tree item at index $index: $e');
        return const SizedBox.shrink();  // Graceful failure
      }
    },
  );
}
```

**Key Points**:
- SliverList.builder for lazy loading (efficient memory usage)
- Per-item loading state (not just global)
- Error handling per item (one failure doesn't break list)
- Empty state for better UX

---

## 11. Individual Tree Item Display

### `TreeSliverItem` (StatelessWidget)

**Location**: `lib/ui/widgets/tree_sliver_item.dart`

#### Constants

```dart
static const double _basePadding = 8.0;
static const double _cardPadding = 12.0;
static const double _levelIndent = 24.0;      // Indentation per level
static const double _iconSize = 16.0;
static const double _expansionIconSize = 20.0;
static const double _fontSize = 14.0;
static const double _labelFontSize = 12.0;
static const double _spinnerSize = 16.0;
static const double _spinnerStrokeWidth = 2.0;
static const double _borderRadius = 8.0;
```

#### Build Method

```dart
@override
Widget build(BuildContext context) {
  return Container(
    margin: EdgeInsets.only(
      left: node.level * _levelIndent,  // ← Dynamic indentation
      top: 4.0,
      right: _basePadding,
      bottom: 4.0,
    ),
    child: Card(
      elevation: node.level == 0 ? 2 : 1,  // Root has more elevation
      child: InkWell(
        onTap: onTap,
        borderRadius: BorderRadius.circular(_borderRadius),
        child: Padding(
          padding: const EdgeInsets.all(_cardPadding),
          child: Column(
            crossAxisAlignment: CrossAxisAlignment.start,
            children: [
              // ┌─────────────────────────────────────────────┐
              // │ Expansion Header (if has children)         │
              // └─────────────────────────────────────────────┘
              if (node.hasChildren) _buildExpansionHeader(),
              
              // ┌─────────────────────────────────────────────┐
              // │ Prompt Section                             │
              // └─────────────────────────────────────────────┘
              if (node.promptContent != null) ...[
                _buildPromptSection(),
              ],
              
              // ┌─────────────────────────────────────────────┐
              // │ Response Section (varies by node type)     │
              // └─────────────────────────────────────────────┘
              if (node.nodeType == NodeType.exchange && 
                  node.responseContent != null) ...[
                const SizedBox(height: _basePadding),
                if (isLoading)
                  _buildStatusSection(StatusSectionConfig.loading())
                else
                  _buildResponseSection(),
                  
              ] else if (node.nodeType == NodeType.shard && 
                         node.promptContent != null) ...[
                // Shard with prompt (sub-prompt node)
                if (isLoading)
                  _buildStatusSection(StatusSectionConfig.loading())
                else if (node.responseContent != null) ...[
                  const SizedBox(height: _basePadding),
                  _buildResponseSection(),
                ] else
                  _buildStatusSection(StatusSectionConfig.waiting()),
                  
              ] else if (node.nodeType == NodeType.shardResponse) ...[
                // Response segment (no prompt, just response text)
                _buildResponseSection(),
              ],
            ],
          ),
        ),
      ),
    ),
  );
}
```

#### Build Expansion Header

```dart
Widget _buildExpansionHeader() {
  return Container(
    margin: const EdgeInsets.only(bottom: _basePadding),
    child: Row(
      children: [
        Icon(
          node.isExpanded ? Icons.expand_less : Icons.expand_more,
          size: _expansionIconSize,
          color: Colors.grey[600],
        ),
        const SizedBox(width: 4),
        Text(
          node.isExpanded ? 'Collapse' : 'Expand',
          style: TextStyle(
            fontSize: _labelFontSize,
            color: Colors.grey[600],
            fontWeight: FontWeight.w500,
          ),
        ),
      ],
    ),
  );
}
```

#### Build Prompt Section

```dart
Widget _buildPromptSection() {
  final promptStyle = SectionStyle.forPrompt(node.nodeType);
  return _buildContentSection(
    config: ContentSectionConfig(
      backgroundColor: promptStyle.backgroundColor,  // Blue
      borderColor: promptStyle.borderColor,
      icon: promptStyle.icon,                        // Person icon
      iconColor: promptStyle.iconColor,
      label: promptStyle.label,                      // "You" or "Shard"
      content: node.promptContent!,
    ),
  );
}
```

#### Build Response Section

```dart
Widget _buildResponseSection() {
  final responseStyle = SectionStyle.forResponse(node.nodeType, node.shardId);
  return _buildContentSection(
    config: ContentSectionConfig(
      backgroundColor: responseStyle.backgroundColor,  // Green or Yellow
      borderColor: responseStyle.borderColor,
      icon: responseStyle.icon,                        // Different per type
      iconColor: responseStyle.iconColor,
      label: responseStyle.label,                      // Different per type
      content: node.responseContent!,
    ),
  );
}
```

#### Build Content Section (Generic)

```dart
Widget _buildContentSection({required ContentSectionConfig config}) {
  return Container(
    padding: const EdgeInsets.all(_basePadding),
    decoration: BoxDecoration(
      color: config.backgroundColor,
      borderRadius: BorderRadius.circular(_borderRadius),
      border: Border.all(color: config.borderColor),
    ),
    child: Column(
      crossAxisAlignment: CrossAxisAlignment.start,
      children: [
        // ┌─────────────────────────────────────────────┐
        // │ Header (icon + label)                      │
        // └─────────────────────────────────────────────┘
        Row(
          children: [
            Icon(
              config.icon,
              size: _iconSize,
              color: config.iconColor,
            ),
            const SizedBox(width: 4),
            Text(
              config.label,
              style: TextStyle(
                fontWeight: FontWeight.bold,
                color: config.iconColor,
                fontSize: _labelFontSize,
              ),
            ),
          ],
        ),
        const SizedBox(height: 4),
        
        // ┌─────────────────────────────────────────────┐
        // │ Content (with custom context menu)         │
        // └─────────────────────────────────────────────┘
        _buildSelectableText(
          text: config.content,
          textStyle: const TextStyle(
            fontSize: _fontSize,
            color: Colors.black87,
          ),
        ),
      ],
    ),
  );
}
```

#### Build Status Section (Loading/Waiting)

```dart
Widget _buildStatusSection(StatusSectionConfig config) {
  return Container(
    padding: const EdgeInsets.all(_cardPadding),
    decoration: BoxDecoration(
      color: config.backgroundColor,        // Yellow
      borderRadius: BorderRadius.circular(_borderRadius),
      border: Border.all(color: config.borderColor),
    ),
    child: Row(
      children: [
        if (config.showSpinner)
          SizedBox(
            width: _spinnerSize,
            height: _spinnerSize,
            child: CircularProgressIndicator(
              strokeWidth: _spinnerStrokeWidth,
              valueColor: AlwaysStoppedAnimation<Color>(config.iconColor),
            ),
          )
        else
          Icon(
            config.icon,
            size: config.iconSize,
            color: config.iconColor,
          ),
        const SizedBox(width: _basePadding),
        Text(
          config.message,                   // "Waiting for response..."
          style: TextStyle(
            fontSize: _labelFontSize,
            color: config.iconColor,
            fontStyle: FontStyle.italic,
          ),
        ),
      ],
    ),
  );
}
```

#### Build Selectable Text (with Sub-Prompt Menu)

```dart
Widget _buildSelectableText({
  required String text,
  required TextStyle textStyle,
}) {
  return SelectableText(
    text,
    style: textStyle,
    contextMenuBuilder: (context, editableTextState) {
      final TextSelection selection = 
          editableTextState.currentTextEditingValue.selection;
      final String selectedText = selection.textInside(text);

      if (selectedText.isEmpty) {
        // No selection - show default menu
        return AdaptiveTextSelectionToolbar.buttonItems(
          anchors: editableTextState.contextMenuAnchors,
          buttonItems: editableTextState.contextMenuButtonItems,
        );
      }

      // ┌───────────────────────────────────────────────────────────┐
      // │ Custom menu with "Sub-prompt" option                     │
      // └───────────────────────────────────────────────────────────┘
      return AdaptiveTextSelectionToolbar.buttonItems(
        anchors: editableTextState.contextMenuAnchors,
        buttonItems: [
          ...editableTextState.contextMenuButtonItems,  // Copy, etc.
          ContextMenuButtonItem(
            label: 'Sub-prompt',
            onPressed: () => _handleSubPrompt(selectedText, selection),
          ),
        ],
      );
    },
  );
}
```

#### Handle Sub-Prompt Action

```dart
void _handleSubPrompt(String selectedText, TextSelection selection) {
  try {
    // 1. Copy to clipboard
    Clipboard.setData(ClipboardData(text: selectedText));

    // 2. Create PromptArgs with shard information
    final PromptArgs promptArgs = PromptArgs(
      currentPointId: node.pointId,
      parentPointId: node.pointId,
      parentShardId: node.shardId,
      isShardChild: true,
      exchangeId: node.exchangeId,
      choiceIndex: node.choiceIndex ?? 0,
      selectedText: selectedText,
      startPosition: selection.start,
      endPosition: selection.end,
      error: null,
    );

    // 3. Call parent callback to prepare input
    prepareSubPromptInput(promptArgs);
    
    // 4. Close context menu
    ContextMenuController.removeAny();
    
  } catch (e) {
    debugPrint('Error in sub-prompt: $e');
  }
}
```

**Key Points**:
- Dynamic indentation based on tree level
- Conditional rendering based on node type
- Custom context menu for sub-prompt creation
- Color-coded sections (blue = prompt, green = AI response, yellow = shard segment)
- Loading states per exchange
- Production-ready error handling

---

## 12. Sub-Prompt Preparation

### `HomeScreen.prepareSubPromptInput(PromptArgs promptArgs)`

**Location**: `lib/screens/home_screen.dart`

```dart
void prepareSubPromptInput(PromptArgs promptArgs) {
  // 1. Validate selected text
  if (promptArgs.selectedText != null && 
      promptArgs.selectedText!.isNotEmpty) {
    String selectedText = promptArgs.selectedText!;

    if (selectedText.trim().isEmpty) {
      showErrorSnackBar('No text selected.');
      return;
    }
  }

  // 2. Update state with shard information
  if (mounted) {
    setState(() {
      errorMessage = null;
      this.promptArgs = promptArgs.copyWith(
        currentPointId: promptArgs.currentPointId,
        isShardChild: true,  // ← Mark as sub-prompt
      );
    });
  }

  // 3. Show feedback to user
  ScaffoldMessenger.of(context).showSnackBar(
    const SnackBar(
      content: Text('Text copied to input. Modify if needed and send.'),
      duration: Duration(seconds: 2),
      behavior: SnackBarBehavior.floating,
    ),
  );

  // 4. Set text in input field (post frame callback)
  WidgetsBinding.instance.addPostFrameCallback((_) {
    try {
      if (mounted && promptInputFieldKey.currentState != null) {
        if (promptArgs.selectedText != null && 
            promptArgs.selectedText!.isNotEmpty) {
          // Set text in input field
          promptInputFieldKey.currentState!.setText(promptArgs.selectedText!);
          // Focus input field
          promptInputFieldKey.currentState!.focusInput();
        } else {
          promptInputFieldKey.currentState!.clearText();
        }
      }
    } catch (e) {
      debugPrint('Error setting text in input field: $e');
      showErrorSnackBar('Failed to set text in input field: ${e.toString()}');
    }
  });
}
```

**Key Points**:
- Validates selected text before proceeding
- Updates PromptArgs with shard context
- Uses post-frame callback for safe UI updates
- Provides user feedback via SnackBar
- Focuses input field for immediate editing

---

## 13. Node Selection

### `HomeScreen.onNodeSelected(String currentPointId)`

**Location**: `lib/screens/home_screen.dart`

```dart
void onNodeSelected(String currentPointId) {
  try {
    // Update promptArgs to continue from this node
    setState(() {
      promptArgs = PromptArgs(
        currentPointId: currentPointId,
        parentPointId: currentPointId,
        isShardChild: false,  // ← NOT a sub-prompt
      );
    });

    // Focus input field for user to type next prompt
    promptInputFieldKey.currentState?.focusInput();
    
  } catch (e) {
    debugPrint('Error in node selection: $e');
  }
}
```

**Key Points**:
- Sets selected node as parent for next prompt
- Resets isShardChild flag (regular continuation)
- Focuses input for immediate user interaction
- Silent error handling (non-critical operation)

---

## 14. Data Models

### Point Model

**Location**: `lib/models/point.dart`

```dart
@JsonSerializable(explicitToJson: true)
class Point {
  final String id;                      // e.g., "T001"
  final String parentPointId;           // Self-reference for root
  final List<String> pointChildren;     // Child Point IDs
  final String? parentShardId;          // If this is a shard child
  final List<Shard> shardsList;         // Shards in this Point
  final List<Exchange> exchangesList;   // Prompt/Response pairs
  final Metadata? metadata;             // Timestamps, state, etc.

  Point({
    required this.id,
    required this.parentPointId,
    List<String>? pointChildren,
    this.parentShardId,
    List<Shard>? shardsList,
    List<Exchange>? exchangesList,
    this.metadata,
  }) : pointChildren = pointChildren ?? [],
       shardsList = shardsList ?? [],
       exchangesList = exchangesList ?? [];

  // Factory methods, copyWith, JSON serialization...
}
```

### Shard Model

```dart
@JsonSerializable(explicitToJson: true)
class Shard {
  final String shardId;                 // e.g., "S01"
  final List<String> shardChildren;     // Child Point IDs under this shard
  final Anchor anchor;                  // Text position in parent response

  Shard({
    required this.shardId,
    required this.shardChildren,
    required this.anchor,
  });

  // Factory methods, JSON serialization...
}
```

### Anchor Model

```dart
@JsonSerializable(explicitToJson: true)
class Anchor {
  final int startPosition;              // Start index in response text
  final int endPosition;                // End index in response text
  final String selectedText;            // The actual selected text

  Anchor({
    required this.startPosition,
    required this.endPosition,
    required this.selectedText
  });

  // Factory methods, JSON serialization...
}
```

### Exchange Model

```dart
@JsonSerializable(explicitToJson: true)
class Exchange {
  final String exchangeId;              // Unique ID
  final String? exchangeTitle;          // Optional title
  final Prompt prompt;                  // User's prompt
  final Response? response;             // AI's response (nullable if loading)

  Exchange({
    required this.exchangeId,
    this.exchangeTitle,
    required this.prompt,
    this.response
  });

  // Factory methods, JSON serialization...
}
```

### Prompt Model

```dart
@JsonSerializable(explicitToJson: true)
class Prompt {
  final String model;                   // e.g., "gpt-3.5-turbo"
  final PromptMessage promptMessage;    // Role + Content
  final int maxTokens;                  // Token limit

  Prompt({
    required this.model,
    required this.promptMessage,
    required this.maxTokens
  });
}

@JsonSerializable()
class PromptMessage {
  final String role;                    // e.g., "user"
  final String content;                 // The prompt text

  PromptMessage({
    required this.role,
    required this.content
  });
}
```

### Response Model

```dart
@JsonSerializable(explicitToJson: true)
class Response {
  final String id;                      // OpenAI response ID
  final String object;                  // e.g., "chat.completion"
  final int created;                    // Unix timestamp
  final String model;                   // Model that generated response
  final List<Choice> choices;           // Response choices (usually 1)
  final Usage usage;                    // Token usage stats
  final String serviceTier;             // Service level
  final String? systemFingerprint;      // Model version

  Response({
    required this.id,
    required this.object,
    required this.created,
    required this.model,
    required this.choices,
    required this.usage,
    required this.serviceTier,
    this.systemFingerprint,
  });
}

@JsonSerializable(explicitToJson: true)
class Choice {
  final int index;                      // Choice index (0 for first)
  final Message message;                // The actual response message
  final dynamic logprobs;               // Log probabilities (if requested)
  final String finishReason;            // Why generation stopped

  Choice({
    required this.index,
    required this.message,
    this.logprobs,
    required this.finishReason
  });
}

@JsonSerializable(explicitToJson: true)
class Message {
  final String role;                    // e.g., "assistant"
  final String content;                 // The response text
  final String? refusal;                // Refusal reason (if any)

  Message({
    required this.role,
    required this.content,
    this.refusal
  });
}
```

### TreeNodeModel

**Location**: `lib/models/tree_node_model.dart`

```dart
class TreeNodeModel {
  final String id;                      // Unique ID for rendering
  final String pointId;                 // Actual Point.id
  final String? parentId;               // Parent TreeNode ID
  final String? parentShardId;          // Parent Shard ID (if shard child)
  final String? exchangeId;             // Exchange ID
  final String? shardId;                // Shard ID (if shard segment)
  
  final int level;                      // Indentation level (0 = root)
  final bool isExpanded;                // Collapsed or expanded
  final bool hasChildren;               // Has child nodes
  
  final String? promptContent;          // Prompt text
  final String? responseContent;        // Response text
  final String? promptRole;             // e.g., "user"
  final String? responseRole;           // e.g., "assistant"
  
  final NodeType nodeType;              // exchange, shard, shardResponse
  final List<TreeNodeModel> children;   // Child nodes
  
  final int? shardStartPosition;        // For shard segments
  final int? shardEndPosition;          // For shard segments
  final int? choiceIndex;               // Which response choice

  TreeNodeModel({
    required this.id,
    required this.pointId,
    this.parentId,
    this.parentShardId,
    this.exchangeId,
    this.shardId,
    required this.level,
    required this.isExpanded,
    this.hasChildren = false,
    this.promptContent,
    this.responseContent,
    this.promptRole,
    this.responseRole,
    required this.nodeType,
    required this.children,
    this.shardStartPosition,
    this.shardEndPosition,
    this.choiceIndex,
  });

  // copyWith method for immutable updates...
}

enum NodeType {
  exchange,        // Regular prompt-response
  shard,          // Sub-prompt (shard child)
  shardResponse,  // Response segment
}
```

### PromptArgs

**Location**: `lib/models/prompt_args.dart`

```dart
class PromptArgs {
  final String currentPointId;          // Current Point ID
  final String? parentPointId;          // Parent Point ID
  final String? parentShardId;          // Parent Shard ID (if shard child)
  final bool isShardChild;              // Is this a sub-prompt?
  final String? exchangeId;             // Exchange ID
  final int? choiceIndex;               // Response choice index
  final String? selectedText;           // Selected text for sub-prompt
  final int? startPosition;             // Selection start
  final int? endPosition;               // Selection end
  final String? error;                  // Error message (if any)

  PromptArgs({
    required this.currentPointId,
    this.parentPointId,
    this.parentShardId,
    required this.isShardChild,
    this.exchangeId,
    this.choiceIndex,
    this.selectedText,
    this.startPosition,
    this.endPosition,
    this.error,
  });

  // copyWith, clear methods...
}
```

**Key Points**:
- JSON serialization with json_annotation
- Immutable data structures (copyWith pattern)
- Nullable fields for optional data
- Comprehensive metadata for debugging and features
- Type-safe enums for node types

---

## 15. Complete Flow Summary for Screenshot Example

### Scenario: "What's Earth?" Conversation

Based on the uploaded screenshot, here's the complete flow:

#### Step 1: Initial Prompt

```
User types: "What's Earth?"
  ↓
HomeScreen.onPrompt("What's Earth?")
  ↓
HomeScreenManager.handlePrompt(...)
  ↓
ThreadManager.updateThreadMap(promptArgs)
  → Creates Point T001 (root point)
  → Point T001 has empty exchangesList initially
  ↓
PointManager.createNewPoint(...)
  → Sends to backend, gets Point T001 with Exchange E001
  → Exchange has prompt "What's Earth?" but no response yet
  ↓
ContextManager.createPromptContext(T001)
  → Builds conversation history: [{ role: "user", content: "What's Earth?" }]
  ↓
ApiService.sendPromptToAi(...)
  → Sends to OpenAI API
  → Receives response: "Third planet from the Sun"
  → Returns Point T001 with updated Exchange E001 (now has response)
  ↓
threadMap[T001] = updated Point T001
  ↓
TreeViewManager.updateTreeViewList(...)
  → treeViewList = [T001]
  ↓
TreeSliverManager.buildFlatTreeList(...)
  → Creates TreeNodeModel for T001:
    - promptContent: "What's Earth?"
    - responseContent: "Third planet from the Sun"
    - nodeType: NodeType.exchange
    - level: 0
  ↓
UI renders:
┌────────────────────────────────────────┐
│ 👤 You                                 │
│ What's Earth?                          │
└────────────────────────────────────────┘
┌────────────────────────────────────────┐
│ 🤖 AI Assistant                        │
│ Third planet from the Sun              │
└────────────────────────────────────────┘
```

#### Step 2: First Sub-Prompt (Selected Text: "Sun")

```
User selects "Sun" text in response
  ↓
Right-click → "Sub-prompt"
  ↓
TreeSliverItem._handleSubPrompt(...)
  → Clipboard.setData("Sun")
  → Creates PromptArgs:
    - currentPointId: "T001"
    - parentPointId: "T001"
    - isShardChild: true
    - selectedText: "Sun"
    - startPosition: 24
    - endPosition: 27
  ↓
HomeScreen.prepareSubPromptInput(promptArgs)
  → setState: this.promptArgs = promptArgs
  → PromptInputFieldKey.currentState.setText("Sun")
  → Shows SnackBar: "Text copied to input..."
  ↓
User modifies text: "Sun - what's this?"
User presses Send
  ↓
HomeScreen.onPrompt("Sun - what's this?")
  ↓
HomeScreenManager.handlePrompt(...)
  ↓
ThreadManager.updateThreadMap(promptArgs with isShardChild=true)
  ↓
ShardManager.addShardToParentPoint(...)
  → Creates Shard S01 in Point T001:
    - shardId: "S01"
    - anchor: { start: 24, end: 27, text: "Sun" }
    - shardChildren: []  (will be updated)
  → Sends updated T001 to backend
  ↓
PointManager.createNewPoint(...)
  → Creates Point T002:
    - id: "T002"
    - parentPointId: "T001"
    - parentShardId: "S01"
    - exchangesList: [{ prompt: "Sun - what's this?", response: null }]
  ↓
ShardManager updates:
  → T001.shardsList[0].shardChildren = ["T002"]
  → Backend updates T001
  ↓
ApiService.sendPromptToAi(T002, ...)
  → Gets response: "The star"
  → Updates T002 with response
  ↓
threadMap[T001] = updated T001 (with Shard S01)
threadMap[T002] = updated T002 (with response)
  ↓
TreeViewManager.updateTreeViewList(...)
  → treeViewList = [T001, T002]
  ↓
TreeSliverManager.buildFlatTreeList(...)
  → Detects T001 has valid shards
  → Calls _createShardPointNode(T001, ...)
    Creates:
    1. Main node (prompt only):
       - id: "T001"
       - promptContent: "What's Earth?"
       - responseContent: null
       - level: 0
    
    2. Shard segment (response part with "Sun"):
       - id: "T001_shard_S01"
       - responseContent: "Third planet from the Sun"
       - nodeType: NodeType.shardResponse
       - level: 0 (sibling of prompt)
       - children: [T002]
    
    3. Child node T002:
       - id: "T002"
       - promptContent: "Sun - what's this?"
       - responseContent: "The star"
       - parentId: "T001_shard_S01"
       - level: 1 (indented)
  ↓
UI renders:
┌────────────────────────────────────────┐
│ 👤 You                                 │
│ What's Earth?                          │
└────────────────────────────────────────┘
┌────────────────────────────────────────┐
│ ✂️ Shard Segment                       │
│ Third planet from the Sun              │
│   ┌────────────────────────────────────┤
│   │ 👤 Shard                           │
│   │ Sun - what's this?                 │
│   └────────────────────────────────────┤
│   ┌────────────────────────────────────┤
│   │ 🤖 AI Assistant                    │
│   │ The star                           │
│   └────────────────────────────────────┘
└────────────────────────────────────────┘
```

#### Step 3: Second Sub-Prompt (Selected Text: "star")

```
User selects "star" text in T002's response
  ↓
Right-click → "Sub-prompt"
  ↓
Creates PromptArgs:
  - currentPointId: "T002"
  - parentPointId: "T002"
  - parentShardId: "S01"  (inherited)
  - isShardChild: true
  - selectedText: "star"
  - startPosition: 4
  - endPosition: 8
  ↓
User types: "star - what's this?"
User presses Send
  ↓
ThreadManager.updateThreadMap(...)
  → Creates Shard S02 in Point T002
  → Creates Point T003 (child of T002, shard S02)
  ↓
ApiService.sendPromptToAi(T003, ...)
  → Gets response with multiple parts
  ↓
threadMap updates:
  - T001 (unchanged, has S01)
  - T002 (now has S02)
  - T003 (new, with response)
  ↓
TreeViewManager.updateTreeViewList(...)
  → treeViewList = [T001, T002, T003]
  ↓
TreeSliverManager.buildFlatTreeList(...)
  → Now T002 also has shards!
  → Recursive shard splitting:
    - T001 splits into segments (has S01)
    - T002 splits into segments (has S02)
    - T003 is leaf node
  ↓
UI renders:
┌────────────────────────────────────────┐
│ 👤 You                                 │
│ What's Earth?                          │
└────────────────────────────────────────┘
┌────────────────────────────────────────┐
│ ✂️ Shard Segment                       │
│ Third planet from the Sun              │
│   ┌────────────────────────────────────┤
│   │ 👤 Shard                           │
│   │ Sun - what's this?                 │
│   └────────────────────────────────────┤
│   ┌────────────────────────────────────┐
│   │ ✂️ Shard Segment                   │
│   │ The star                           │
│   │   ┌────────────────────────────────┤
│   │   │ 👤 Shard                       │
│   │   │ star - what's this?            │
│   │   └────────────────────────────────┤
│   │   ┌────────────────────────────────┐
│   │   │ 🤖 AI Assistant                │
│   │   │ A luminous sphere              │
│   │   └────────────────────────────────┘
│   └────────────────────────────────────┘
└────────────────────────────────────────┘
```

#### Step 4: Third Sub-Prompt (Selected Text: "sphere")

```
User selects "sphere" text
  ↓
Creates sub-prompt: "sphere - what's this?"
  ↓
Similar flow as above...
  → Creates Shard S03 in Point T003
  → Creates Point T004 (child of T003, shard S03)
  → Gets AI response with multiple choice segments
  ↓
UI renders final nested tree structure (as seen in screenshot):

┌────────────────────────────────────────┐
│ 👤 You                                 │
│ What's Earth?                          │
└────────────────────────────────────────┘
┌────────────────────────────────────────┐
│ ✂️ Shard Segment                       │
│ Third planet from the Sun              │
│   ┌────────────────────────────────────┤
│   │ 👤 Shard                           │
│   │ Sun - what's this?                 │
│   └────────────────────────────────────┤
│   ┌────────────────────────────────────┐
│   │ ✂️ Shard Segment                   │
│   │ The star                           │
│   │   ┌────────────────────────────────┤
│   │   │ 👤 Shard                       │
│   │   │ star - what's this?            │
│   │   └────────────────────────────────┤
│   │   ┌────────────────────────────────┐
│   │   │ ✂️ Shard Segment               │
│   │   │ A luminous sphere              │
│   │   │   ┌────────────────────────────┤
│   │   │   │ 👤 Shard                   │
│   │   │   │ sphere - what's this?      │
│   │   │   └────────────────────────────┤
│   │   │   ┌────────────────────────────┐
│   │   │   │ 🤖 AI Assistant            │
│   │   │   │ A three-dimensional...     │
│   │   │   └────────────────────────────┘
│   │   └────────────────────────────────┘
│   │   ┌────────────────────────────────┐
│   │   │ ✂️ Response Segment            │
│   │   │ of plasma held together...     │
│   │   └────────────────────────────────┘
│   │   ┌────────────────────────────────┐
│   │   │ ✂️ Response Segment            │
│   │   │ at the center of our...        │
│   │   └────────────────────────────────┘
│   │   ┌────────────────────────────────┐
│   │   │ ✂️ Response Segment            │
│   │   │ our home in the solar...       │
│   │   └────────────────────────────────┘
│   └────────────────────────────────────┘
└────────────────────────────────────────┘
```

**Key Points**:
- Each sub-prompt creates a shard in the parent Point
- Shards split responses into segments
- Sub-prompts are displayed as indented children
- Tree structure grows dynamically with each interaction
- Recursive algorithm handles arbitrary nesting depth
- All state maintained in threadMap for consistency

---

## Key Architectural Patterns

### 1. **Manager Pattern**

**Purpose**: Separate concerns into specialized managers

- **PointManager**: Point CRUD operations
- **ShardManager**: Shard creation and management
- **ThreadManager**: Thread/conversation orchestration
- **ContextManager**: AI context building
- **TreeViewManager**: Tree data transformation
- **TreeSliverManager**: Tree rendering logic
- **HomeScreenManager**: Overall orchestration

**Benefits**:
- Single Responsibility Principle
- Testable units
- Reusable components
- Clear dependencies

### 2. **State Management via Callbacks**

**Pattern**: Parent manages state, children receive callbacks

```dart
// Parent (HomeScreen)
setState(() { isLoading = true; });

// Manager uses callback
onLoadingStateChanged(true);  // → calls setState in parent

// Child widget receives updated prop
TreeSliverThread(isLoading: isLoading)
```

**Benefits**:
- Centralized state in HomeScreen
- Reactive UI updates
- Type-safe callbacks
- No global state management needed

### 3. **Immutable Data Structures**

**Pattern**: Never mutate, always copy

```dart
// Bad ❌
point.parentShardId = "S01";

// Good ✅
final updatedPoint = point.copyWith(parentShardId: "S01");
```

**Benefits**:
- Predictable state changes
- Easy debugging (old state preserved)
- No side effects
- Thread-safe (if needed later)

### 4. **Tree Flattening for Rendering**

**Pattern**: Convert hierarchical data to flat list

```dart
Hierarchical:
A
├─ B
│  └─ C
└─ D

Flat (based on expansion):
[A, B, C, D]  // All expanded

[A, B, D]     // A expanded, B collapsed
```

**Benefits**:
- Efficient rendering with SliverList.builder
- Simple scroll handling
- Memory efficient (only visible items rendered)
- Easy pagination if needed

### 5. **Error Boundaries**

**Pattern**: Try-catch at every level with graceful degradation

```dart
try {
  // Risky operation
} catch (e) {
  debugPrint('Error: $e');
  // Show user feedback
  // Continue execution (don't crash)
}
```

**Benefits**:
- Application never crashes
- User always gets feedback
- Debugging information preserved
- Production-ready resilience

### 6. **Lazy Loading / On-Demand Fetching**

**Pattern**: Fetch data only when needed

```dart
if (!threadMap.containsKey(parentPointId)) {
  // Fetch from backend
  parentPoint = await pointManager.getParentPoint(...);
  threadMap[parentPointId] = parentPoint;  // Cache
} else {
  // Use cached data
  parentPoint = threadMap[parentPointId]!;
}
```

**Benefits**:
- Reduced network calls
- Faster user experience
- Memory efficient
- Offline-friendly (cached data)

### 7. **Separation of Display Logic and Data Logic**

**Pattern**: Data models separate from view models

- **Point**: Backend data structure (JSON serializable)
- **TreeNodeModel**: Display structure (tree rendering)

**Transformation**:
```
Point (backend) → TreeNodeModel (display) → Widget (UI)
```

**Benefits**:
- Backend changes don't break UI
- Multiple views of same data
- Easy to add new display modes
- Testable transformations

### 8. **Composite Pattern for Tree Structure**

**Pattern**: TreeNodeModel contains children of same type

```dart
class TreeNodeModel {
  final List<TreeNodeModel> children;
  // ...
}
```

**Benefits**:
- Recursive algorithms naturally
- Arbitrary nesting depth
- Uniform handling of nodes
- Easy to add new node types

### 9. **Factory Pattern for Node Creation**

**Pattern**: Specialized creation methods

```dart
_createShardPointNode(...)      // For points with shards
_buildChildNode(...)            // For child nodes
TreeNodeModel(...)              // For regular nodes
```

**Benefits**:
- Complex creation logic encapsulated
- Consistent node structure
- Easy to maintain
- Reusable builders

### 10. **Observer Pattern (Implicit via Callbacks)**

**Pattern**: Notify observers of state changes

```dart
onMapUpdated.call();  // Notifies all listeners
```

**Benefits**:
- Decoupled components
- Multiple listeners possible
- Reactive updates
- Extensible

### 11. **Production-Ready Logging**

**Pattern**: developer.log instead of print

```dart
import 'dart:developer' as developer;

developer.log(
  'Message',
  name: 'ClassName.methodName',
  error: e,
  stackTrace: stackTrace,
  level: 900,  // Error level
);
```

**Benefits**:
- Filterable by name/level
- Structured logging
- Performance profiling
- Production debugging

### 12. **Context Preservation**

**Pattern**: PromptArgs carries full context

```dart
PromptArgs(
  currentPointId: "T002",
  parentPointId: "T001",
  parentShardId: "S01",
  isShardChild: true,
  selectedText: "Sun",
  startPosition: 24,
  endPosition: 27,
)
```

**Benefits**:
- All information in one place
- Easy to pass between methods
- Clear what's needed for operation
- Type-safe

### 13. **Optimistic UI Updates**

**Pattern**: Update UI before backend confirmation

```dart
// Add to loading state immediately
updateLoadingExchanges(updatedLoading);

// Send to backend
await apiService.sendPromptToAi(...);

// Remove from loading state after response
updateLoadingExchanges(updatedLoading);
```

**Benefits**:
- Responsive UI
- Better perceived performance
- User can continue interacting
- Rollback on error (if implemented)

### 14. **Defensive Programming**

**Pattern**: Validate everything, assume nothing

```dart
// Check mounted before setState
if (mounted) {
  setState(() { ... });
}

// Validate array access
final content = exchange.response?.choices.firstOrNull?.message.content;

// Validate shard anchors
if (shard.anchor.startPosition < 0) return false;
if (shard.anchor.endPosition > contentLength) return false;
```

**Benefits**:
- Prevents crashes
- Handles edge cases
- Works with incomplete data
- Production-ready robustness

### 15. **Key-based Widget Identity**

**Pattern**: GlobalKey for programmatic control

```dart
final GlobalKey<PromptInputFieldState> promptInputFieldKey = GlobalKey();

// Later...
promptInputFieldKey.currentState?.clearText();
promptInputFieldKey.currentState?.focusInput();
```

**Benefits**:
- Preserve state across rebuilds
- Programmatic control of widgets
- Access child state from parent
- No need for complex state management

---

## Conclusion

This Flutter application demonstrates a sophisticated implementation of a branching conversation tree with the following characteristics:

### **Core Features**:
1. Multi-level conversation threading
2. Sub-prompt creation from selected text (sharding)
3. Response segmentation and display
4. Dynamic tree expansion/collapse
5. Granular loading states
6. Comprehensive error handling

### **Technical Excellence**:
1. Production-ready error handling at every level
2. Immutable data structures with copyWith pattern
3. Separation of concerns via manager pattern
4. Efficient rendering with tree flattening
5. Type-safe models with JSON serialization
6. Comprehensive logging for debugging
7. Graceful degradation on failures

### **User Experience**:
1. Responsive UI with optimistic updates
2. Clear visual hierarchy with indentation
3. Color-coded message types
4. Contextual actions (sub-prompt menu)
5. Loading indicators per exchange
6. Error feedback via SnackBars

### **Scalability**:
1. Lazy loading with caching
2. Efficient list rendering (SliverList)
3. Recursive algorithms for arbitrary depth
4. Modular architecture for easy extension
5. Clean separation of display and data logic

This architecture serves as an excellent foundation for a production application handling complex conversational AI interactions with support for branching, context preservation, and user-friendly visualization.

---

**End of Document**
