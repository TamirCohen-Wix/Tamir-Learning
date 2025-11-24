# Cursor Rules Summary

This document provides an overview of the agent instructions created for the bookings-catalog service using the `AGENTS.md` format as recommended by [Cursor documentation](https://cursor.com/docs/context/rules).

## Files Created

### 1. External Libraries Documentation
**File**: `EXTERNAL_LIBRARIES.md`
- Comprehensive list of all external libraries used
- Descriptions of each library's purpose
- Usage patterns and examples
- Categorized by functionality

### 2. Root Agent Instructions
**File**: `AGENTS.md`
- Project overview and architecture patterns
- Core principles and code style guidelines
- Library usage guidelines (AutoMapper, gRPC, Babel, etc.)
- Common patterns and best practices
- Security and performance considerations
- References to nested module instructions

### 3. Module-Specific Agent Instructions

#### Adapters Module
**File**: `src/com/wixpress/bookings/bookings/catalog/adapters/AGENTS.md`
- Adapter pattern guidelines
- Interface design principles
- Error handling in adapters
- Query building patterns
- Examples and common pitfalls

#### Mappers Module
**File**: `src/com/wixpress/bookings/bookings/catalog/mappers/AGENTS.md`
- AutoMapper usage patterns
- Transformer definition guidelines
- Builder pattern for complex objects
- Field mapping best practices
- Error handling in builders

#### Domain Module
**File**: `src/com/wixpress/bookings/bookings/catalog/domain/AGENTS.md`
- Domain model design principles
- Immutability and purity guidelines
- Sealed trait hierarchies (ADTs)
- SDL annotations usage
- Type safety patterns

#### Utils Module
**File**: `src/com/wixpress/bookings/bookings/catalog/utils/AGENTS.md`
- Utility function design
- Pure vs. impure function guidelines
- Stateless function patterns
- Translation and formatting utilities
- Complex adapter utilities

#### Errors Module
**File**: `src/com/wixpress/bookings/bookings/catalog/errors/AGENTS.md`
- Custom exception patterns
- Wix error framework usage
- Status code selection guidelines
- Error message and details best practices
- Error handling patterns

## How Cursor Uses These Instructions

Cursor will automatically read these `AGENTS.md` files when:
1. You're working in a specific directory - it reads the nearest `AGENTS.md` file and combines with parent directories
2. You ask questions about code in that module - it uses module-specific instructions
3. You request code generation - it follows the patterns and guidelines
4. You ask for refactoring suggestions - it respects the established patterns

According to the [Cursor documentation](https://cursor.com/docs/context/rules), nested `AGENTS.md` files are automatically applied when working with files in that directory or its children. Instructions from nested files are combined with parent directories, with more specific instructions taking precedence.

## Key Features of These Rules

### 1. Specific and Concrete
- Each rule includes concrete examples
- Clear do's and don'ts
- Real code patterns from the codebase

### 2. Module-Aware
- Rules are organized by module
- Each module has its own concerns
- Rules reference each other appropriately

### 3. Library-Specific
- Rules reference actual libraries used
- AutoMapper patterns
- gRPC best practices
- Babel translation patterns

### 4. Best Practices
- Based on industry standards
- Adapted for Wix's internal frameworks
- Includes common pitfalls to avoid

## Usage Examples

### When Working on Adapters
Cursor will:
- Suggest using `Future[Option[T]]` return types
- Recommend error handling patterns
- Guide query building with QueryMapper
- Enforce adapter pattern principles

### When Working on Mappers
Cursor will:
- Suggest AutoMapper transformer patterns
- Recommend builder classes for complex objects
- Guide field mapping strategies
- Enforce immutability

### When Working on Domain Models
Cursor will:
- Suggest case class patterns
- Recommend sealed trait hierarchies
- Guide Option usage
- Enforce immutability and purity

### When Working on Utilities
Cursor will:
- Suggest pure function patterns
- Recommend stateless design
- Guide error handling
- Enforce single responsibility

### When Working on Errors
Cursor will:
- Suggest Wix error framework patterns
- Recommend appropriate status codes
- Guide error message formatting
- Enforce error detail inclusion

## Maintenance

These rules should be updated when:
1. New libraries are added to the project
2. Patterns change in the codebase
3. Best practices evolve
4. New modules are added

## Integration with Cursor

Cursor will automatically:
- Read `AGENTS.md` from the current directory and parent directories
- Apply the most specific instructions first (nested files take precedence)
- Combine instructions from multiple levels
- Use instructions to guide code generation and suggestions

The `AGENTS.md` format is simpler than Project Rules (`.cursor/rules` with MDC format) and doesn't require metadata or complex configurations. It's perfect for straightforward, readable instructions.

## Migration Notes

We've migrated from the legacy `.cursorrules` format to the recommended `AGENTS.md` format:
- ✅ Removed all `.cursorrules` files (legacy format, will be deprecated)
- ✅ Created `AGENTS.md` files in root and all module directories
- ✅ Maintained all existing content and patterns
- ✅ Simplified format - no metadata, just plain markdown

## Next Steps

1. Review the instructions to ensure they match your team's preferences
2. Update instructions as patterns evolve
3. Add module-specific instructions for new modules
4. Share instructions with team members for consistency

## Questions?

If you have questions about:
- **Library usage**: See `EXTERNAL_LIBRARIES.md`
- **General patterns**: See root `.cursorrules`
- **Module-specific**: See module-specific `.cursorrules` files

