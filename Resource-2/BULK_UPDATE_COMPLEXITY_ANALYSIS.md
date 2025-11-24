# Runtime Complexity Analysis: `bulkUpdateResources`

## Overview
This document provides a detailed line-by-line runtime complexity analysis of the `bulkUpdateResources` method in `ResourcesService.scala`, where **N = length of `request.resources`**.

---

## Method Signature
```scala
override def bulkUpdateResources(request: BulkUpdateResourcesRequest)(implicit callScope: CallScope): Future[BulkUpdateResourcesResponse]
```

---

## Line-by-Line Complexity Analysis

### **Lines 145-147: Initial Domain Conversion**
```scala
val resourceDomainsWithMasks: Seq[(ResourceDomain, FieldMask)] = request.resources.map { maskedResource =>
  (maskedResource.getResource.toResourceDomain, maskedResource.getFieldMask)
}
```

**Complexity: O(N)**
- **Time**: O(N) - Iterates through all N resources
- **Space**: O(N) - Creates N tuples
- **Operations per element**: 
  - `toResourceDomain`: O(1) - Simple field mapping
  - `getFieldMask`: O(1) - Direct field access
- **Notes**: This is a pure transformation with no I/O operations

---

### **Line 151: Validation**
```scala
_ <- validator.validate(request)
```

**Complexity: O(N) work, O(1) parallelism (limited to 3 concurrent)**
- **Time**: O(N) - Must validate all N resources
- **Parallelism**: Limited to 3 concurrent validations (`ParallelismLimitForBulkVerification = 3`)
- **Per-resource validation** (`validateResourceForUpdate`):
  - `sdlHandler.getResource(resource.getId)`: **O(1) database call** - Single resource fetch
  - `constructResourceToValidate`: O(1) - Field mask merge operation
  - Validation checks (immutability, name, location, schedules): **O(1)** - Simple field checks
- **Total database calls**: N (one per resource)
- **Wall-clock time**: O(N) due to limited parallelism, but could be O(N/3) with perfect parallelization
- **Notes**: This is a **critical bottleneck** - each resource requires a database read for validation

---

### **Lines 155-160: Schedule Update Futures Creation**
```scala
scheduleUpdateFutures: Seq[Future[Option[Throwable]]] = resourceDomainsWithMasks.zipWithIndex.map {
  case ((resourceDomain, fieldMask), idx) =>
    updatedEventsScheduleIfNeeded(resourceDomain, fieldMask)
      .map(_ => None: Option[Throwable])
      .recover { case e => Some(e) }
}
```

**Complexity: O(N)**
- **Time**: O(N) - Creates N futures (lazy, no immediate execution)
- **Space**: O(N) - Stores N future objects
- **Notes**: Futures are created but not executed yet. The actual work happens in the next step.

---

### **Line 161: Schedule Updates Execution**
```scala
scheduleUpdateResults <- Future.sequence(scheduleUpdateFutures)
```

**Complexity: O(N) work, O(N) parallelism (subject to execution context)**
- **Time**: O(N) - Executes N futures concurrently
- **Parallelism**: Depends on execution context, but typically high (all N can run in parallel)
- **Per-resource work** (`updatedEventsScheduleIfNeeded`):
  - `sdlHandler.getResource(updatedResource.id.toString)`: **O(1) database call** - Fetch existing resource
  - `updateScheduleOwnerNameIfNeeded`: 
    - If name in field mask: **O(1) RPC call** to schedules service
    - Otherwise: O(1) - Simple check
  - `updateLinkedScheduleIfNeeded`:
    - If working hours schedules in field mask: **O(1) RPC call** to schedules service
    - Otherwise: O(1) - Simple check
- **Total database calls**: N (one `getResource` per resource)
- **Total RPC calls**: Up to 2N (name update + schedule link/unlink, but only if field masks require it)
- **Worst case**: O(N) sequential if all operations must be sequential (unlikely with proper async execution)
- **Best case**: O(1) wall-clock time if all N operations execute in parallel
- **Notes**: This is another **major bottleneck** - each resource requires at least one database read, potentially more RPC calls

---

### **Lines 165-166: Filter Successful Schedule Updates**
```scala
sdlInputWithOriginalIndices: Seq[((ResourceDomain, FieldMask), Int)] = resourceDomainsWithMasks.zipWithIndex.zip(scheduleUpdateResults)
  .collect { case ((rfm, idx), None) => (rfm, idx) }
```

**Complexity: O(N)**
- **Time**: O(N) - Iterates through all N results
- **Space**: O(M) where M ≤ N (only successful updates)
- **Operations**:
  - `zipWithIndex`: O(N) - Creates index pairs
  - `zip`: O(N) - Combines with schedule results
  - `collect`: O(N) - Filters successful updates
- **Notes**: M = number of resources with successful schedule updates

---

### **Line 168: Extract SDL Input**
```scala
sdlInput: Seq[(ResourceDomain, FieldMask)] = sdlInputWithOriginalIndices.map(_._1)
```

**Complexity: O(M)** where M ≤ N
- **Time**: O(M) - Maps M successful resources
- **Space**: O(M) - Creates new sequence
- **Notes**: M = number of successful schedule updates

---

### **Line 170: Extract Original Indices**
```scala
sdlInputOriginalIndices: Seq[Int] = sdlInputWithOriginalIndices.map(_._2)
```

**Complexity: O(M)** where M ≤ N
- **Time**: O(M) - Maps M indices
- **Space**: O(M) - Creates new sequence
- **Notes**: M = number of successful schedule updates

---

### **Lines 173-180: SDL Bulk Update**
```scala
sdlResults <- if (sdlInput.nonEmpty) {
  sdlHandler.bulkUpdateResources(sdlInput, request.returnEntity)
} else {
  Future.successful(PatchBulkResultsWithEntities[ResourceDomain](...))
}
```

**Complexity: O(M)** where M ≤ N
- **Time**: O(M) - Processes M resources
- **Database operations**: 
  - `sdl.patch.bulk.withFieldMaskEntries(...).execute()`: **O(M) database operations** - Bulk update
  - SDL likely optimizes this to a single batch operation, but internally processes M resources
- **Space**: O(M) - Stores results for M resources
- **Notes**: 
  - If M = 0 (all schedule updates failed), this is O(1)
  - SDL bulk operations are typically optimized but still scale with M

---

### **Lines 183-187: Map SDL Results to Original Indices**
```scala
val sdlResultsByOriginalIndex: Map[Int, PatchBulkResultWithEntity[ResourceDomain]] =
  sdlResults.results.map { result =>
    val originalIdx = sdlInputOriginalIndices(result.entityMetadata.originalIndex)
    originalIdx -> result.copy(entityMetadata = result.entityMetadata.copy(originalIndex = originalIdx))
  }.toMap
```

**Complexity: O(M)** where M ≤ N
- **Time**: O(M) - Iterates through M results
- **Space**: O(M) - Creates map with M entries
- **Operations**:
  - `map`: O(M) - Transforms M results
  - `toMap`: O(M) - Builds hash map (amortized O(1) per insertion)
- **Notes**: Creates a lookup map for O(1) access in the next step

---

### **Lines 190-207: Merge Results Preserving Order**
```scala
finalResults: Seq[PatchBulkResultWithEntity[ResourceDomain]] = resourceDomainsWithMasks.indices.map { idx =>
  scheduleUpdateResults(idx) match {
    case Some(scheduleError) => // Create error result
    case None => sdlResultsByOriginalIndex(idx)
  }
}
```

**Complexity: O(N)**
- **Time**: O(N) - Iterates through all N original indices
- **Space**: O(N) - Creates result sequence
- **Operations per element**:
  - Array access: O(1)
  - Map lookup: O(1) amortized
  - Result construction: O(1)
- **Notes**: Ensures results are in the same order as the input request

---

### **Lines 209-214: Build Response**
```scala
yield BulkUpdateResourcesResponse(
  results = finalResults.map(_.toBulkResourceResult),
  bulkActionMetadata = Some(BulkActionMetadata(
    totalSuccesses = finalResults.count(_.entityMetadata.success),
    totalFailures = finalResults.count(!_.entityMetadata.success),
    undetailedFailures = 0
  ))
)
```

**Complexity: O(N)**
- **Time**: O(N) - Maps N results + counts successes/failures
- **Space**: O(N) - Creates response with N results
- **Operations**:
  - `map`: O(N) - Transforms N results
  - `count` (x2): O(N) each - Iterates through results
- **Notes**: Could be optimized to count in a single pass

---

## Overall Complexity Summary

### **Time Complexity**
- **Best Case**: O(N) - All operations are O(N) and can be parallelized
- **Worst Case**: O(N) - Sequential execution, but still linear
- **Wall-clock Time**: 
  - **Ideal parallelization**: O(1) for concurrent operations, but limited by:
    - Validation: O(N/3) due to parallelism limit of 3
    - Schedule updates: O(1) if fully parallelized
    - SDL bulk update: O(1) if batched efficiently
  - **Realistic**: O(N) due to database I/O and limited validation parallelism

### **Space Complexity**
- **O(N)** - Stores domain objects, futures, results throughout the process

### **I/O Operations**
1. **Validation**: N database reads (`getResource` for each resource)
2. **Schedule Updates**: 
   - N database reads (`getResource` for existing resource)
   - Up to 2N RPC calls (name updates + schedule link/unlink, conditional)
3. **SDL Bulk Update**: M database writes (where M ≤ N, successful schedule updates)

**Total I/O**: 
- **Database reads**: 2N (validation + schedule update preparation)
- **RPC calls**: Up to 2N (conditional schedule operations)
- **Database writes**: M (bulk update, where M ≤ N)

---

## Critical Bottlenecks

### 1. **Validation Phase (Line 151)**
- **Issue**: N sequential database reads with limited parallelism (3 concurrent)
- **Impact**: High - Each resource requires a database fetch
- **Optimization Potential**: 
  - Batch fetch all resources upfront: O(1) database call instead of N
  - Increase parallelism limit (currently 3)

### 2. **Schedule Update Phase (Lines 155-161)**
- **Issue**: N database reads (one per resource) + up to 2N RPC calls
- **Impact**: High - Each resource requires at least one database read
- **Optimization Potential**:
  - Batch fetch all existing resources: O(1) database call instead of N
  - Batch schedule operations if possible

### 3. **Redundant Database Reads**
- **Issue**: Resources are fetched twice:
  - Once during validation (`validateResourceForUpdate`)
  - Once during schedule update (`updatedEventsScheduleIfNeeded`)
- **Impact**: Medium - Doubles database read operations
- **Optimization Potential**: Cache validation results and reuse in schedule update phase

---

## Optimization Recommendations

1. **Batch Resource Fetching**: 
   - Fetch all resources needed for validation and schedule updates in a single batch query
   - Reduces 2N database reads to O(1) batch operation

2. **Increase Validation Parallelism**:
   - Consider increasing `ParallelismLimitForBulkVerification` from 3 to a higher value
   - Or remove the limit if database can handle the load

3. **Cache Validation Results**:
   - Store fetched resources from validation phase
   - Reuse in schedule update phase to avoid duplicate fetches

4. **Optimize Response Building**:
   - Count successes/failures in a single pass instead of two separate `count` operations

5. **Consider Transaction Batching**:
   - If SDL supports it, batch all database operations together

---

## Asymptotic Analysis

| Phase | Time | Space | I/O Operations |
|-------|------|-------|----------------|
| Domain Conversion | O(N) | O(N) | 0 |
| Validation | O(N) | O(N) | N reads |
| Schedule Updates | O(N) | O(N) | N reads + up to 2N RPCs |
| SDL Bulk Update | O(M) | O(M) | M writes |
| Result Merging | O(N) | O(N) | 0 |
| Response Building | O(N) | O(N) | 0 |
| **Total** | **O(N)** | **O(N)** | **2N reads + 2N RPCs + M writes** |

Where:
- **N** = total number of resources in request
- **M** = number of resources with successful schedule updates (M ≤ N)

---

## Conclusion

The `bulkUpdateResources` method has **O(N) time complexity** with **O(N) space complexity**. The main performance bottlenecks are:

1. **2N database reads** (validation + schedule updates)
2. **Limited validation parallelism** (3 concurrent operations)
3. **Up to 2N RPC calls** for schedule operations

The method is well-structured for correctness but has significant optimization potential, particularly in reducing redundant database operations and increasing parallelism.


