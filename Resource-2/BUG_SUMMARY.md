# Bug Summary: Search Endpoint Error with Multi-Location Resources

## Issue
Search endpoint fails for users with location permissions (e.g., Business Manager) when querying multi-location resources. Query endpoint works correctly.

## Root Cause
Nile Search schema was not updated since ~Nov 2024 and lacks the `LocationOptions` field. ABAC constraint rules reference `locationOptions.specificLocationOptions.businessLocations.locationId`, but the schema only contains the deprecated `SingleResource.LocationOptions` field.

## Why Search vs Query?
- **Query**: Uses `ResourceTranslator()` which maps `locationOptions...` → `CompositionDetails.SingleResource.locationOptions...` (works in Domain space)
- **Search**: Nile schema is based on public API structure without mapping, so it doesn't match Domain definition

## Solution
Update Nile Search schema via "Update Scheme" button in Nile search dev portal.

## Remaining Issues
1. **Empty Results**: All service data is stored in `SingleResource.LocationOptions`, so `LocationOptions` remains empty → Search returns valid but empty results
2. **API Duplication**: Resource API still supports both deprecated `SingleResource.LocationOptions` and `LocationOptions` fields
3. **Timing Mystery**: ABAC constraints added in July, but error only reported last month (no public user reports before)

## Next Steps
- Create action item for API cleanup (remove deprecated fields)
- Decide on reindexing strategy for Search schema
- Update constraint rule to access `SingleResource.LocationOptions` until above resolved
- Investigate why error appeared recently despite July deployment


