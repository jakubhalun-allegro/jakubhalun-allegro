# Fixed Date/Time WAIT Block — Code Change Analysis

## Overview

Currently the WAIT block stores a **relative time offset** (`timeOffset`, type `waitTime`) — an integer
representing seconds to delay relative to journey start or previous stage. The new feature adds
a **fixed date/time mode** where the WAIT block specifies an exact `Instant` to wait until.
Some WAIT blocks in a path may remain relative while others use fixed dates.
Validation will ensure fixed-date WAITs on the same path are chronologically ordered.

---

## 1. Block Definition (JSON + loading)

### `batch-core/src/main/resources/blocks/wait.json`
- Add a **new version** of the WAIT block (version 2) with additional parameters:
  - A mode selector (e.g. `waitMode`: `"relative"` | `"fixed"`)
  - A `fixedDateTime` parameter (type to be determined, likely a new param type)
- Existing version 1 remains for backward compatibility.

### `batch-core/src/main/kotlin/.../blocks/BlocksImporter.kt`
- **Class:** `BlocksImporter`
- **Method:** `readBlocksFromFiles()`
- Review: may need no change if the JSON schema is a superset, but verify the ObjectMapper deserialization handles new fields.

### `batch-core/src/main/kotlin/.../blocks/BlockDefinition.kt`
- **Class:** `BlockDefinition`
- `params: LinkedHashMap<String, Any>` — generic map, likely no structural change needed, but verify it handles the new parameter shape.

---

## 2. Domain Model — Node & Parameter Access

### `batch-core/src/main/kotlin/.../scenario/domain/ScenarioStructure.kt`
- **Class:** `Node`
- **Methods:** `getParamValue()`, `getParamValueOrDefault()`, `getOptionalParam()`
- Currently `timeOffset` is read as an `Int`. New code must also read `waitMode` and `fixedDateTime` from `Node.params`.
- Consider adding a dedicated helper (e.g. `getWaitMode()`, `getFixedDateTime()`) or keeping it generic.

---

## 3. ScenarioHelpers — Slice Node Creation

### `batch-core/src/main/kotlin/.../utils/ScenarioHelpers.kt`
- **Object:** `ScenarioHelpers`
- **Constant:** `TIME_OFFSET = "timeOffset"` — add constants for new params (e.g. `WAIT_MODE`, `FIXED_DATE_TIME`).
- **Method:** `createSliceNode(node: Node)` (line 106)
  - Copies `node.params` verbatim to the SLICE node. This should continue to work if new params are added to the map, but verify.

---

## 4. WAIT → SLICE Conversion

### `batch-core/src/main/kotlin/.../converters/blocks/WaitOrUnionToSliceConverter.kt`
- **Class:** `WaitOrUnionToSliceConverter`
- **Method:** `convertWait(source: Input)` (line 45)
  - Currently creates Write → Slice → Read nodes. The slice node inherits params from the WAIT node (including `timeOffset`).
  - May need adjustment if fixed-date mode requires different intermediate handling.
- **Method:** `validate(source: Input)` (line 150)
  - Currently validates parent count. May need to validate the new `waitMode`/`fixedDateTime` parameter presence.

---

## 5. Sliced Graph → Stages Graph (offset extraction)

### `batch-core/src/main/kotlin/.../converters/SlicedGraphToStagesGraph.kt`
- **Class:** `SlicedGraphToStagesGraph`
- **Method:** `convert(source: Graph<Node, Edge>)` (line 29)
  - **Line 62:** `.withOffset(slice.getParamValueOrDefault(TIME_OFFSET, 0))`
    - This is the **key extraction point**. For fixed-date WAITs, the stage needs to carry
      the absolute datetime instead of (or in addition to) a relative offset.
  - **Lines 79-84:** Edge weight calculation: `stagesGraph.setEdgeWeight(..., -1.0 * nodeToStage[child]!!.offset.toDouble())`
    - Edge weight logic assumes relative offsets. Fixed-date stages need different treatment (e.g. edge weight = 0 or computed from the fixed date minus journey.runAt).

---

## 6. ScenarioStage Domain — Stage Model

### `batch-core/src/main/kotlin/.../scenario/domain/ScenarioStage.kt`
- **Sealed class:** `ScenarioStage`
  - **Field:** `abstract val offset: Int`
  - Must be extended to support fixed date/time. Options:
    - Add `abstract val fixedDateTime: Instant?` (null for relative stages)
    - Add `abstract val waitMode: WaitMode` enum
  - Both need to be added to all subclasses.
- **Class:** `DataPreparationScenarioStage` — add new fields
- **Class:** `SendingScenarioStage` — add new fields
- **Class:** `ScenarioStageBuilder`
  - **Method:** `withOffset(offset: Int)` (line 82) — add method for fixed date (e.g. `withFixedDateTime(...)`)
  - **Method:** `build()` (line 87) — pass new fields to stage constructors

---

## 7. MongoDB Persistence — Stage Serialization

### `batch-core/src/main/kotlin/.../scenario/persistence/MongoScenarioStage.kt`
- **Sealed class:** `MongoScenarioStage` — add `fixedDateTime: Instant?` (or similar) field
- **Class:** `MongoDataPreparationScenarioStage` — add new field
- **Class:** `MongoSendingScenarioStage` — add new field
- **Class:** `MongoDummyScenarioStage` — add new field with default

### `batch-core/src/main/kotlin/.../scenario/domain/ScenarioMapper.kt`
- **Interface:** `ScenarioMapper` — MapStruct mappings; verify auto-mapping handles new fields
- **Interface:** `ScenarioStageMapper` — same concern
- **Class:** `ScenarioStageMapperHelper`
  - **Method:** `toScenarioStage(stage: MongoScenarioStage)` (line 61)
  - **Method:** `toMongoScenarioStage(stage: ScenarioStage)` (line 71)

### `batch-core/src/main/kotlin/.../journey/StageGraphMapper.kt`
- **Class:** `StageGraphMapper`
- **Method:** `toStageGraph(input: MongoStageGraph)` (line 17)
  - **Line 25:** `g.setEdgeWeight(source, target, -1.0 * target.offset.toDouble())`
  - For fixed-date stages the edge weight calculation must account for absolute timestamps.

---

## 8. Offset Calculation (Bellman-Ford)

### `batch-core/src/main/kotlin/.../journey/OffsetHelpers.kt`
- **Object:** `OffsetHelpers`
- **Method:** `calculate(graph, stage): Long` (line 11)
  - Uses Bellman-Ford with negative edge weights derived from `stage.offset`.
  - **Critical change needed:** For fixed-date stages, the offset is not relative — it's an absolute time.
    The algorithm must handle mixed edges (relative offsets vs. fixed dates).
    A fixed-date stage's execution time is the fixed date itself, not `journey.runAt + accumulated_offset`.

---

## 9. Run Scheduling — Timestamp Calculation

### `batch-core/src/main/kotlin/.../runs/NextStageRunner.kt`
- **Class:** `DataAvailabilityAwareNextStageRunner`
- **Method:** `calculateTimeForStage(journey, stage, clock): Instant` (line 23)
  - Currently: `journey.runAt.plusSeconds(offset)`
  - For fixed-date stages: return the fixed datetime directly instead of computing from offset.

### `batch-core/src/main/kotlin/.../runs/DependencyExecutionTimeHelper.kt`
- **Class:** `DependencyExecutionTimeHelper`
- **Method:** `calculateExpectedDependencyStartTime(journey, current): Long?` (line 31)
  - Uses `journey.runAt.plusSeconds(offset)` — must handle fixed dates.
- **Method:** `dfsMinOffset(stage, memo): Long?` (line 53)
  - Accumulates `child.offset + childCost` — a fixed-date child breaks this additive model.

### `batch-core/src/main/kotlin/.../runs/schedule/WaitingForDataRunsSchedulingService.kt`
- **Class:** `WaitingForDataRunsSchedulingService`
- **Method:** `shouldStartRunToday(run: Run): Boolean` (line 29)
  - Uses `run.timestamp` which is already computed. Should work if upstream timestamp calculation is correct.
  - Review to ensure no implicit assumptions about relative-only offsets.

---

## 10. Journey Builder — Estimated Dispatch Time

### `batch-core/src/main/kotlin/.../journey/JourneyBuilder.kt`
- **Class:** `JourneyBuilder`
- **Method:** `generateEstimatedDispatchTime(): List<EstimatedDispatchTimeParameter>` (line 58)
  - Uses `OffsetHelpers.calculate(stages, it)` and `runAt.plusSeconds(offset)`.
  - Must handle fixed-date stages (use fixed datetime directly for dispatch estimate).

---

## 11. SLA & Delay Notifications

### `batch-core/src/main/kotlin/.../sla/ComaSlaEventService.kt`
- **Class:** `ComaSlaEventService`
- **Method:** `calculateRunAt(journey, stage): Instant` (line 161)
  - Currently: `journey.runAt.plusSeconds(offset)` — must handle fixed dates.

### `batch-core/src/main/kotlin/.../journey/notifier/JourneyDelayNotifier.kt`
- **Class:** `JourneyDelayNotifier`
- **Method:** `calculateRunAt(journey, stage): Instant` (line 98)
  - Same pattern: `journey.runAt.plusSeconds(offset)` — must handle fixed dates.

---

## 12. Validation — NEW: Fixed-Date Chronological Order

### `batch-core/src/main/kotlin/.../scenario/domain/validation/ScenarioValidator.kt`
- **Class:** `DefaultScenarioValidator`
- **Method:** `validate(scenario: Scenario)` (line 25)
  - Currently checks cycles. **Must add new validation:** on every path through the graph,
    fixed-date WAIT blocks must be in chronological order (a later WAIT on the same path
    must have a later or equal fixed date).
  - This requires graph traversal (DFS/BFS per path) checking fixed-date ordering.

### `batch-core/src/main/kotlin/.../scenario/domain/validation/NodeValidator.kt`
- **Class:** `SelectedNodeValidator`
- **Method:** `validate(node: Node)` (line 21)
  - Currently WAIT falls through to `DefaultNodeValidator`. Add a dedicated WAIT validator case:
    - `BlockTypes.WAIT -> WaitNodeValidator.validate(node)`
  - The new validator should check: if `waitMode == "fixed"`, then `fixedDateTime` must be present and valid.

### `batch-core/src/main/kotlin/.../scenario/domain/validation/` (NEW FILE)
- **New class:** `WaitNodeValidator`
  - Validate parameter completeness for both relative and fixed modes.

### `batch-core/src/main/kotlin/.../converters/ScenarioToJourneyConverter.kt`
- **Class:** `ScenarioToJourneyConverter`
- **Method:** `validate(scenario: Scenario)` (line 114)
  - Currently calls `nodeValidator.validate(node)` per node and `unsupportedBlocksCheck`.
  - Consider adding the path-level fixed-date ordering validation here or in `ScenarioValidator`.

---

## 13. API Layer — DTOs

### `batch-core/src/main/kotlin/.../scenario/api/ScenarioResponse.kt`
- **Class:** `NodeResponse` — `params: Map<String, Any>` — generic map, likely transparent to new params.

### `batch-core/src/main/kotlin/.../blocks/Block.kt`
- **Class:** `Block` — `params: LinkedHashMap<String, Any>` — transparent, but frontend must understand the new param types.

### `batch-core/src/main/kotlin/.../blocks/BlockEndpoint.kt`
- **Class:** `BlockEndpoint`
- **Method:** `list(authentication): ResponseEntity<List<Block>>` — no change needed if block JSON is correct.

---

## 14. Test Files to Update

| Test file | What needs updating |
|---|---|
| `converters/NodeTestFactories.kt` | Add `createFixedDateWaitNode(id, dateTime)` factory method alongside existing `createWaitNode(id, offset)` |
| `converters/blocks/WaitOrUnionToSliceConverterTest.kt` | Add tests for fixed-date WAIT conversion |
| `converters/SlicedGraphToStagesGraphTest.kt` | Add tests for stages with fixed-date offsets |
| `converters/ScenarioToJourneyConverterTest.kt` | Add tests for journeys with mixed relative/fixed WAITs |
| `converters/ByBlockTypeSliceConverterTest.kt` | Verify fixed-date WAITs routed correctly |
| `runs/DataAvailabilityAwareNextStageRunnerTest.kt` | Test `calculateTimeForStage` with fixed-date stages |
| `runs/DependencyExecutionTimeHelperTest.kt` | Test dependency time with fixed-date in the path |
| `scenario/scheduling/WaitingForDataRunsSchedulingServiceTest.kt` | Verify scheduling with fixed-date run timestamps |
| `converters/StageSizeConstraintEnforcerTest.kt` | Verify size enforcement still works with new params |
| `utils/JourneyUtils.kt` | Test utility may need helpers for fixed-date stages |

---

## Summary: Minimal Change Set (ordered by dependency)

1. **wait.json** — new block version with `waitMode` + `fixedDateTime` params
2. **ScenarioHelpers** — new constants
3. **ScenarioStage** + **ScenarioStageBuilder** — add fixed-date field
4. **MongoScenarioStage** subclasses — add fixed-date field for persistence
5. **ScenarioMapper** / **ScenarioStageMapper** — verify mapping
6. **SlicedGraphToStagesGraph.convert()** — extract fixed-date from slice params into stage
7. **StageGraphMapper.toStageGraph()** — edge weight for fixed-date stages
8. **OffsetHelpers.calculate()** — handle mixed relative/fixed stages
9. **NextStageRunner.calculateTimeForStage()** — return fixed date when applicable
10. **DependencyExecutionTimeHelper** — handle fixed-date in DFS offset accumulation
11. **JourneyBuilder.generateEstimatedDispatchTime()** — use fixed date for dispatch estimate
12. **ComaSlaEventService.calculateRunAt()** — handle fixed date
13. **JourneyDelayNotifier.calculateRunAt()** — handle fixed date
14. **WaitNodeValidator** (new) — validate fixed-date params
15. **SelectedNodeValidator** — register WAIT case
16. **DefaultScenarioValidator.validate()** — add path-level chronological order check
17. **NodeTestFactories** + all test files listed above
