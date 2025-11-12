### **Flow 1: Prompt (child of root Point, no shards)**
```
HomeScreen.onPrompt()
├─> HomeScreenManager.handlePrompt()
    ├─> ThreadManager.updateThreadMap()
    │   ├─> PointManager.createNewPoint()
    │   │   └─> Creates Point with parentPointId = currentPointId
    │   └─> PointManager.updateParentPoint()
    │       └─> Adds newPointId to parent.pointChildren[]
    ├─> ApiService.sendPromptToAi()
    │   └─> Updates Point with response
    ├─> threadMap[updatedNewPoint.id] = updatedNewPoint
    └─> onMapUpdated()
        └─> HomeScreen.setState()
            └─> FlatTreeManager.buildFlatTreeList()
                └─> _addRootPoint() → ISSUE HERE ⚠️
                    ├─> _addPrompt(rootPoint)
                    ├─> _addResponse(rootPoint)
                    └─> _addPointWithDescendants(childPoint) ✅
```

### **Flow 2: Prompt (child of Point WITH shards)**
```
HomeScreen.onPrompt()
├─> HomeScreenManager.handlePrompt()
    ├─> ThreadManager.updateThreadMap()
    │   ├─> PointManager.createNewPoint()
    │   └─> PointManager.updateParentPoint()
    │       └─> Adds newPointId to parent.pointChildren[]
    ├─> ApiService.sendPromptToAi()
    ├─> threadMap[updatedNewPoint.id] = updatedNewPoint
    └─> onMapUpdated()
        └─> HomeScreen.setState()
            └─> FlatTreeManager.buildFlatTreeList()
                └─> _addRootPoint() → ISSUE HERE ⚠️
                    ├─> _addPrompt(rootPoint)
                    ├─> _addShardSegments(rootPoint)
                    │   └─> (shards shown, but children NOT shown)
                    └─> _addPointWithDescendants(childPoint) ✅
```

### **Flow 3: Sub-prompt (child of root Point)**
```
HomeScreen.prepareSubPromptInput() → copies selected text
HomeScreen.onPrompt()
├─> HomeScreenManager.handlePrompt()
    ├─> ThreadManager.updateThreadMap()
    │   ├─> PointManager.createNewPoint()
    │   │   └─> Creates Point with parentShardId
    │   └─> ShardManager.addShardToParentPoint()
    │       └─> Creates Shard, adds newPointId to shard.shardChildren[]
    ├─> ApiService.sendPromptToAi()
    ├─> threadMap[updatedNewPoint.id] = updatedNewPoint
    └─> onMapUpdated()
        └─> HomeScreen.setState()
            └─> FlatTreeManager.buildFlatTreeList()
                └─> _addRootPoint()
                    ├─> _addPrompt(rootPoint)
                    └─> _addShardSegments(rootPoint) ✅
                        └─> _addPointWithDescendants(childPoint) ✅
```

### **Flow 4: Sub-prompt (child of Point WITH shards)**
```
Same as Flow 3 - works correctly ✅