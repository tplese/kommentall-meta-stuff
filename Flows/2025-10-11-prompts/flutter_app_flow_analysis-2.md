1. Application Entry Point
  main.dart
    main() → runApp(MyApp())
2. Home Screen Initialization
  HomeScreen (StatefulWidget)
  Location: lib/screens/home_screen.dart
  State Initialization (HomeScreenState.initState())
    1. Initialize PromptArgs (empty state)
    2. Create BackendService instance
    3. Create PointManager(backendService)
    4. Create TreeViewManager
    5. Create ShardManager(pointManager, backendService)
    6. Create ThreadManager(pointManager, shardManager, backendService, threadMap, onMapUpdated)
    7. Create ContextManager(threadManager, apiService, threadMap)
    8. Create HomeScreenManager(managers..., callbacks...)
    9. Initialize UI state variables:
      - threadMap: Map<String, Point>
      - treeViewList: List<Point>
      - loadingExchanges: Set<String>
      - isLoading: bool
      - errorMessage: String?
3. User Input Flow
  PromptInputField (StatefulWidget)
  Location: lib/ui/widgets/prompt_input_field.dart
  User Types Prompt
    TextField(controller: _textController)
      ↓
    User presses Send button or Enter
      ↓
    _handleSendRequest()
      ↓
    widget.onPrompt(text) → calls HomeScreen.onPrompt()
4. Prompt Processing Flow
  HomeScreen.onPrompt(String promptText)
  Async method that orchestrates the entire prompt handling
    1. Clear error messages (setState)
    2. Determine if this is a sub-prompt or regular prompt
    3. Create/update PromptArgs:
      - For regular prompt: use last Point in treeViewList as parent
      - For sub-prompt: use existing promptArgs with shard info
    4. Call HomeScreenManager.handlePrompt(...)
    5. Clear promptArgs (setState)
    6. Handle errors with setState and error display
5. Home Screen Manager Processing
  HomeScreenManager.handlePrompt(...)
  Location: lib/managers/home_screen_manager.dart
    1. Validate prompt text (not empty)
    2. onLoadingStateChanged(true) → updates UI loading state
    3. ThreadManager.updateThreadMap(promptText, promptArgs)
      → Returns: Point newPoint
    4. Create AiModelProperties (model config)
    5. ContextManager.createPromptContext(newPoint)
      → Returns: List<RequestMessage> promptContextMessages
    6. Track loading state for this exchange:
      - Extract exchangeId from newPoint.exchangesList.last
      - Add to loadingExchanges set
      - updateLoadingExchanges callback → updates HomeScreen state
    7. ApiService.sendPromptToAi(pointId, promptContextMessages, aiModelProperties)
      → Returns: Point updatedNewPoint (with AI response)
    8. Remove exchangeId from loadingExchanges
    9. Update threadMap[updatedNewPoint.id] = updatedNewPoint
    10. onMapUpdated() callback
    11. populateTreeViewList(treeViewList, updateTreeViewList)
    12. onClearPromptInput() → clears input field
    13. onLoadingStateChanged(false)
6. Thread Map Update (Core Logic)
  ThreadManager.updateThreadMap(...)
  Location: lib/managers/thread_manager.dart
    1. PointManager.createNewPoint(promptArgs, promptText)
      → Returns: Point newPoint (with prompt, no response yet)

    2. IF newPoint.id == newPoint.parentPointId:
      → Root point: Add to threadMap and return

    3. ELSE: Get parent point
      - IF NOT in threadMap:
        → PointManager.getParentPoint(fetchArgs, newPoint.id)
        → Cache parent in threadMap
      - ELSE:
        → Retrieve parent from threadMap

    4. Decision Branch:
      
      A. IF promptArgs.isShardChild AND selectedText exists:
          **SHARDING FLOW** (Sub-prompt)
          ↓
          ShardManager.addShardToParentPoint(newPoint.id, parentPoint, promptArgs)
            → Returns: Point updatedParentPoint (with new Shard added)
          ↓
          Extract parentShardId from updatedParentPoint.shardsList
          ↓
          newPoint.copyWith(parentShardId: parentShardId)
          ↓
          Update threadMap with both points
          ↓
          onMapUpdated() callback
      
      B. ELSE:
          **REGULAR FLOW** (Continue conversation)
          ↓
          PointManager.updateParentPoint(newPoint.id, parentPoint)
            → Returns: Point updatedParentPoint (with newPoint.id in pointChildren)
          ↓
          Update threadMap with both points
          ↓
          onMapUpdated() callback

    5. Return newPoint (or updatedNewPoint)
7. Tree View Update
  HomeScreenManager.populateTreeViewList(...)
    TreeViewManager.updateTreeViewList(threadMap, treeViewList)
      → Returns: List<Point> newTreeViewList
      ↓
    updateTreeViewList callback → updates HomeScreen.treeViewList
      ↓
    setState() triggers UI rebuild
8. Tree Rendering Flow
  TreeSliverThread (StatefulWidget)
  Location: lib/ui/widgets/tree_sliver_thread.dart
  State Update (_TreeSliverThreadState)
    didUpdateWidget(TreeSliverThread oldWidget)
      ↓
    IF treeViewList or threadMap changed:
      ↓
      _updateTreeList()
        ↓
        TreeSliverManager.buildFlatTreeList(threadMap, treeViewList)
          → Returns: List<TreeNodeModel> _flatTreeList
        ↓
        setState() → triggers rebuild
9. Tree Structure Building
  TreeSliverManager.buildFlatTreeList(...)
  Location: lib/managers/tree_sliver_manager.dart
  # This is the core algorithm that creates the tree structure shown in the UI
    1. _buildTreeStructure(treeViewList, threadMap)
      → Returns: List<TreeNodeModel> treeNodes
      
      Step A: Create all nodes (First Pass)
      ─────────────────────────────────────
      FOR EACH Point in treeViewList:
        - Get first Exchange
        - Check if Point has valid shards:
          
          IF has valid shards:
            → _createShardPointNode(point, exchange, nodeMap, threadMap)
              Creates:
              ├─ Main prompt node (no response)
              ├─ Shard segment nodes (response parts) [stored in nodeMap]
              └─ "After" segment if text remains
          
          ELSE:
            → Create regular TreeNodeModel
              Contains both prompt and full response
      
      Step B: Build hierarchy (Second Pass)
      ──────────────────────────────────────
      FOR EACH Point in treeViewList:
        IF Point.parentPointId == Point.id:
          → Add as root node
          → IF has shards: Add shard segments as siblings
        
        ELSE:
          → Find parent (could be a shard segment)
          → Add as child to parent
          → Update level (parent.level + 1)
          → IF this child has shards: Add child's shard segments as siblings
      
      Step C: Update hasChildren flags
      ──────────────────────────────────

    2. _flattenNode(node, flatList) [Recursive]
      → Flattens tree based on expansion states
      → Returns: List<TreeNodeModel> flatList
  # Shard Point Node Creation (_createShardPointNode)
    1. Create main prompt node (no response content)
    2. Filter valid shards (validate anchor positions)
    3. Sort shards by start position
    4. FOR EACH shard:
      - Extract response segment (currentPosition to shard.endPosition)
      - Create TreeNodeModel for segment (NodeType.shardResponse)
      - Store in nodeMap (NOT added as child of main node)
      - FOR EACH child in shard.shardChildren:
        → _buildChildNode(childPoint, shardSegment.id, shard.shardId, level=1, ...)
        → Add child as child of shard segment
        → IF child has shards: Add child's shard segments as siblings
      - Update currentPosition
    5. Create "after" segment if text remains
    6. Return main prompt node
10. Tree Item Rendering
  TreeSliverThread.build(...)
    SliverList.builder(
      itemCount: _flatTreeList.length
      itemBuilder: (context, index)
        ↓
        Get TreeNodeModel node from _flatTreeList[index]
        ↓
        Check if node.exchangeId is in loadingExchanges
        ↓
        Return TreeSliverItem(node, isLoading, callbacks...)
    )
11. Individual Tree Item Display
  TreeSliverItem (StatelessWidget)
  Location: lib/ui/widgets/tree_sliver_item.dart
  # Build Method Structure
    Container (with left margin based on node.level)
      ↓
      Card
        ↓
        InkWell (onTap: collapses/expands, selects node)
          ↓
          Padding
            ↓
            Column:
              ├─ IF hasChildren: _buildExpansionHeader()
              │    → Shows "Collapse" / "Expand" with icon
              │
              ├─ IF promptContent != null: _buildPromptSection()
              │    → Shows prompt with blue background
              │    → Icon: person icon
              │    → Label: "You" or "Shard"
              │
              └─ IF responseContent != null:
                  IF isLoading:
                    → _buildStatusSection(StatusSectionConfig.loading())
                      Shows "Waiting for response..." with spinner
                  ELSE:
                    → _buildResponseSection()
                      Shows response with color based on type:
                      - Green for AI Assistant
                      - Yellow for Shard Segments
  # Content Section Rendering (_buildContentSection)
    Container (colored background)
      ↓
      Column:
        ├─ Row (icon + label)
        └─ _buildSelectableText(content)
            ↓
            SelectableText with custom context menu
              ↓
              IF text selected:
                → Custom menu includes "Sub-prompt" option
                → _handleSubPrompt(selectedText, selection)
                  ├─ Copy text to clipboard
                  ├─ Create PromptArgs with shard info
                  ├─ Call prepareSubPromptInput callback
                  └─ Updates HomeScreen state
12. Sub-Prompt Preparation
  HomeScreen.prepareSubPromptInput(PromptArgs promptArgs)
    1. Validate selectedText
    2. setState:
      - Clear errorMessage
      - Update this.promptArgs with new values:
        * currentPointId
        * isShardChild = true
        * selectedText, startPosition, endPosition
    3. Show SnackBar: "Text copied to input..."
    4. Post frame callback:
      - PromptInputFieldKey.currentState.setText(selectedText)
      - PromptInputFieldKey.currentState.focusInput()
13. Node Selection
  HomeScreen.onNodeSelected(String currentPointId)
    1. setState:
      - Update promptArgs with selected node's pointId
      - Set isShardChild = false (continuing from this point)
    2. Focus input field for user to type next prompt
14. Data Models
  # Point Model (lib/models/point.dart)
    Point:
      - id: String
      - parentPointId: String
      - pointChildren: List<String>
      - parentShardId: String?
      - shardsList: List<Shard>
      - exchangesList: List<Exchange>
      - metadata: Metadata?

    Shard:
      - shardId: String
      - shardChildren: List<String>
      - anchor: Anchor (startPosition, endPosition, selectedText)

    Exchange:
      - exchangeId: String
      - prompt: Prompt (model, promptMessage, maxTokens)
      - response: Response? (with choices containing message content)
  # TreeNodeModel (lib/models/tree_node_model.dart)
    TreeNodeModel:
      - id: String (unique identifier for display)
      - pointId: String (actual Point.id)
      - parentId: String?
      - parentShardId: String?
      - exchangeId: String?
      - shardId: String?
      - level: int (indentation level)
      - isExpanded: bool
      - hasChildren: bool
      - promptContent: String?
      - responseContent: String?
      - promptRole: String?
      - responseRole: String?
      - nodeType: NodeType (exchange, shard, shardResponse)
      - children: List<TreeNodeModel>
      - shardStartPosition: int?
      - shardEndPosition: int?
      - choiceIndex: int?
15. Complete Flow Summary for Screenshot Example
  Scenario: "What's Earth?" conversation
    1. User types "What's Earth?" → HomeScreen.onPrompt()
      ↓
    2. HomeScreenManager.handlePrompt()
      ↓
    3. ThreadManager.updateThreadMap() → Creates Point T001
      ↓
    4. ApiService.sendPromptToAi() → Returns response "Third planet from the Sun"
      ↓
    5. TreeViewManager.updateTreeViewList() → [T001]
      ↓
    6. TreeSliverManager.buildFlatTreeList()
      → Creates TreeNodeModel for T001 (prompt + response)
      ↓
    7. TreeSliverThread renders TreeSliverItem
      → Shows blue "You" box: "What's Earth?"
      → Shows green "AI Assistant" box: "Third planet from the Sun"

    8. User selects "Sun" text → Right-click → "Sub-prompt"
      ↓
    9. HomeScreen.prepareSubPromptInput()
      → Sets promptArgs with shard info
      → Fills input field with "Sun - what's this?"
      ↓
    10. User types sub-prompt → HomeScreen.onPrompt()
        ↓
    11. ThreadManager.updateThreadMap() with isShardChild=true
        ↓
    12. ShardManager.addShardToParentPoint()
        → Creates Shard S01 in Point T001
        → Creates new Point T002 (parentShardId = S01)
        ↓
    13. TreeSliverManager rebuilds tree:
        Point T001 now split into:
        ├─ Prompt: "What's Earth?"
        ├─ Response segment: "Third planet from the "
        ├─ Shard segment (with children):
        │  ├─ "Sun"
        │  └─ Child: T002
        │      ├─ Prompt: "Sun - what's this?"
        │      └─ Response: "The star" + "A luminous sphere" + "of plasma..."
        └─ After segment: " [remaining text]"

    14. Each subsequent sub-prompt repeats steps 8-13,
        creating nested tree structure as shown in screenshot




Key Architectural Patterns

1. State Management: Centralized in HomeScreen with callbacks to update UI
2. Manager Pattern: Separate managers for different concerns (Thread, Shard, Point, Context, TreeView)
3. Immutable Updates: Points are copied with modifications, original data preserved
4. Tree Flattening: 3D tree structure flattened to 1D list for rendering
5. Lazy Loading: Only visible items rendered via SliverList.builder
6. Error Handling: Try-catch blocks throughout with fallbacks to prevent crashes
7. Reactive UI: setState() triggers rebuild only when data changes