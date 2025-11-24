# LocationOptions Usage Without SingleResourceDomain.locationOptions

This document identifies all places in the codebase where `locationOptions` is NOT taken from `SingleResourceDomain.locationOptions` (i.e., from `CompositionDetails.SingleResourceDomain.locationOptions`).

## Summary

According to the architecture, `locationOptions` should ALWAYS be accessed from `CompositionDetailsDomain.SingleResourceDomain.locationOptions` in the domain layer. However, there are places where:
1. `locationOptions` is accessed from the deprecated root `Resource.locationOptions` proto field
2. `locationOptions` is set on the deprecated root `Resource.locationOptions` proto field (even if correctly extracted from SingleResourceDomain)
3. `locationOptions` is accessed without going through the `SingleResourceDomain` structure

## Issues Found

### 1. ResourceDomainToProtoMappers.scala (Line 46)
**File:** `src/com/wixpress/bookings/resources/v2/mappers/ResourceDomainToProtoMappers.scala`

**Issue:** Even though `resourceDomain.locationOptions` correctly extracts from `SingleResourceDomain.locationOptions` (via the method on line 65-68 of ResourceDomain.scala), it's being set on BOTH:
- The deprecated root `Resource.locationOptions` field (line 46) ❌
- In `CompositionDetails.SingleResource.locationOptions` (line 39 → 132 → 56) ✅

```scala
def toResource = Resource(
  // ... other fields ...
  compositionDetails = resourceDomain.compositionDetails.toCompositionDetails,  // ✅ Sets in SingleResource
  // ...
  locationOptions = resourceDomain.locationOptions.map(_.toLocationOptions),  // ❌ ALSO sets deprecated root field
  // ...
)
```

**Note:** `resourceDomain.locationOptions` correctly extracts from `SingleResourceDomain.locationOptions`, but the problem is it's being duplicated on the deprecated root field.

**Impact:** This creates duplicate data - `locationOptions` exists in both `CompositionDetails.SingleResource.locationOptions` (correct) and `Resource.locationOptions` (deprecated). The root field should NOT be set.

---

### 2. ResourceProtoToDomainMappers.scala (Lines 70-75)
**File:** `src/com/wixpress/bookings/resources/v2/mappers/ResourceProtoToDomainMappers.scala`

**Issue:** Falls back to deprecated root `resource.locationOptions` when `singleResource.locationOptions` is empty.

```scala
locationOptions = singleResource.locationOptions
  .map(_.toLocationOptionsDomain)
  .orElse(
    resource.locationOptions  // ❌ Falls back to deprecated root field
      .map(_.toLocationOptionsDomain),
  )
  .orElse(Some(LocationOptionsDomain.Default)),
```

**Impact:** For backward compatibility, this accesses the deprecated root field. However, it should check `CompositionDetails` structure first.

---

### 3. ResourceProtoToDomainMappers.scala (Line 161)
**File:** `src/com/wixpress/bookings/resources/v2/mappers/ResourceProtoToDomainMappers.scala`

**Issue:** Checks deprecated root `resource.locationOptions` when `CompositionDetails` is Empty.

```scala
case CompositionDetails.Empty =>
  if (
    resource.workingHoursSchedules.isEmpty && 
    resource.eventsSchedule.isEmpty && 
    resource.locationOptions.isEmpty  // ❌ Accesses deprecated root field
  )
```

**Impact:** This check uses the deprecated root field instead of checking `CompositionDetails.SingleResource.locationOptions`.

---

### 4. ResourceProtoToDomainMappers.scala (Line 172)
**File:** `src/com/wixpress/bookings/resources/v2/mappers/ResourceProtoToDomainMappers.scala`

**Issue:** Uses deprecated root `resource.locationOptions` when `CompositionDetails` is Empty but other fields are present.

```scala
else
  SingleResourceDomain(
    workingHoursSchedules = resource.workingHoursSchedules.map(_.toWorkingHoursSchedulesDomain),
    locationOptions =
      resource.locationOptions.map(_.toLocationOptionsDomain).orElse(Some(LocationOptionsDomain.Default)),  // ❌ Uses deprecated root field
    eventsSchedule = resource.eventsSchedule.map(_.toScheduleDomain),
  )
```

**Impact:** Accesses the deprecated root field instead of checking `CompositionDetails.SingleResource.locationOptions`.

---

### 5. ResourceValidator.scala (Lines 144-149)
**File:** `src/com/wixpress/bookings/resources/v2/validation/ResourceValidator.scala`

**Issue:** Falls back to deprecated root `resource.locationOptions` when field mask doesn't contain `SingleResource`.

```scala
val maybeLocationsOptions =
  if (fieldMask.exists(isFieldNameInFieldMaskDeep(ResourceFieldNames.SingleResource, _))) {
    resource.getSingleResource.locationOptions  // ✅ Correct
  } else {
    resource.locationOptions  // ❌ Falls back to deprecated root field
  }
```

**Impact:** The validation logic should always check `CompositionDetails.SingleResource.locationOptions` first, and only fall back to the deprecated root field if `CompositionDetails` is Empty or not properly set.

---

## Correct Usage Examples

### ResourceLocationsHelper.scala ✅
**File:** `src/com/wixpress/bookings/resources/v2/helpers/ResourceLocationsHelper.scala`

This file correctly accesses `locationOptions` through `CompositionDetails`:

```scala
def getBusinessLocationIds(resource: Resource): Seq[String] = {
  resource.compositionDetails.singleResource  // ✅ Goes through CompositionDetails
    .flatMap(_.locationOptions.flatMap(...))
    .getOrElse(Seq.empty)
    .flatten
}
```

---

## Summary of Issues

All issues involve accessing or setting `locationOptions` from/on the deprecated root `Resource.locationOptions` field instead of exclusively using `CompositionDetailsDomain.SingleResourceDomain.locationOptions`:

1. **ResourceDomainToProtoMappers.scala (line 46)**: Sets `locationOptions` on deprecated root field (even though it correctly extracts from SingleResourceDomain)
2. **ResourceProtoToDomainMappers.scala (lines 70-75)**: Falls back to deprecated root field when SingleResource.locationOptions is empty
3. **ResourceProtoToDomainMappers.scala (line 162)**: Checks deprecated root field when CompositionDetails is Empty
4. **ResourceProtoToDomainMappers.scala (line 174)**: Uses deprecated root field when CompositionDetails is Empty but other fields exist
5. **ResourceValidator.scala (lines 144-149)**: Falls back to deprecated root field when field mask doesn't contain SingleResource

## Recommendations

1. **ResourceDomainToProtoMappers.scala (line 46)**: **REMOVE** the line that sets `locationOptions` on the root Resource proto. It should ONLY be set in `CompositionDetails.SingleResource` (which already happens via line 39 → 132 → 56).

2. **ResourceProtoToDomainMappers.scala**: 
   - **Lines 70-75**: Should NOT fall back to `resource.locationOptions`. If `singleResource.locationOptions` is empty, use default.
   - **Line 162**: Should check `resource.compositionDetails.singleResource.flatMap(_.locationOptions).isEmpty` instead of `resource.locationOptions.isEmpty`
   - **Line 174**: Should check `resource.compositionDetails.singleResource.flatMap(_.locationOptions)` instead of `resource.locationOptions`

3. **ResourceValidator.scala (lines 144-149)**: Should ALWAYS check `resource.compositionDetails.singleResource.flatMap(_.locationOptions)` first, and only fall back to root field if `CompositionDetails` is Empty (for backward compatibility with old data).

4. **Deprecation path**: Since the root `locationOptions` field is deprecated, these fallbacks should be temporary and eventually removed once all clients migrate to using `CompositionDetails.SingleResource.locationOptions`.

