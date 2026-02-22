---
name: add-room-database
description: Replace an in-memory local data source with Room KMP for persistent storage
argument-hint: <feature-name>
allowed-tools:
  - Read
  - Write
  - Edit
  - Glob
  - Grep
---

# Add Room Database

Replace `InMemory<Feature>LocalDataSource` with Room KMP for persistent local storage. See [persistence-patterns.md](../../references/persistence-patterns.md) for templates.

## Input

`$ARGUMENTS` — the feature name (e.g., `cart`, `favorites`, `settings`)

## Instructions

### 1. Detect Project Context

Read `settings.gradle.kts` → `rootProject.name` → lowercase = `{resource_prefix}`. Read `composeApp/build.gradle.kts` → `namespace` → `{package_base}` (strip `.app`). Derive `{package_base_path}` (dots → `/`).

### 2. Validate Existing Module

Verify the feature has `InMemory<Feature>LocalDataSource` in `<feature>/data/impl/datasource/local/`. If missing, tell the user to run `/cmp-scaffold:scaffold-feature <feature>` first.

### 3. Read Existing Files

Read the local data source interface, model classes, and DI module. Understand all methods on `<Feature>LocalDataSource` — the Room implementation must implement the same interface.

### 4. Create Entity

Create `<feature>/data/impl/datasource/local/db/<Feature>Entity.kt`:
- `@Entity` with `tableName` matching `<feature>s`
- `@PrimaryKey` on `id` field
- Fields matching the domain model

### 5. Create Entity Mappers

In the same `db/` directory, add `toDomain()` and `toEntity()` extension functions.

### 6. Create DAO

Create `<feature>/data/impl/datasource/local/db/<Feature>Dao.kt`:
- `@Dao` interface
- `@Query` for reads returning `Flow<List<Entity>>`
- `@Insert(onConflict = REPLACE)` for writes
- `@Query("DELETE ...")` for deletes

### 7. Create Database

Create `<feature>/data/impl/datasource/local/db/<Feature>Database.kt`:
- `@Database(entities = [...], version = 1)`
- `@ConstructedBy(<Feature>DatabaseConstructor::class)` — required for non-Android KSP
- Abstract function returning the DAO
- `expect object <Feature>DatabaseConstructor : RoomDatabaseConstructor<<Feature>Database>` — KSP generates the `actual object`

### 8. Create Platform Database Module (Koin DI)

Create `expect`/`actual` Koin Module for platform-specific database construction:
- **commonMain**: `expect val <feature>DatabaseModule: Module`
- **androidMain**: Uses `Room.databaseBuilder()` with `Context` from Koin `get()`
- **iosMain**: Uses `Room.databaseBuilder()` with `NSHomeDirectory()` AND `.setDriver(BundledSQLiteDriver())` — non-Android requires explicit SQLite driver

### 9. Create Room Local Data Source

Create `Room<Feature>LocalDataSource` implementing the existing `<Feature>LocalDataSource` interface. Delegate to DAO, mapping between Entity and domain model.

### 10. Update DI Module

Swap the local data source binding in the existing `<feature>DataModule`:
```kotlin
singleOf(::Room<Feature>LocalDataSource) bind <Feature>LocalDataSource::class
```

> Database creation and DAO provision live in the separate `<feature>DatabaseModule` (Step 8). Both modules must be included in Koin initialization.

### 10b. Ensure `startKoin` Initialization

Room KMP on Android requires `androidContext()` in Koin. Verify `startKoin` is used in platform entry points (Android `Application.onCreate()` and iOS `MainViewController`). If the project still uses `KoinApplication` composable, it must be migrated to `startKoin`. Add `<feature>DatabaseModule` to the modules list in both entry points. See persistence-patterns.md for templates.

### 11. Update build.gradle.kts

Add Room dependencies to `<feature>/data/impl/build.gradle.kts`:
- Plugins: `ksp`, `room`
- commonMain: `room-runtime`, `sqlite-bundled`
- KSP dependencies per target: `room-compiler`
- Room schema directory configuration
- Add version catalog entries if not present

### 12. Preserve InMemory

Keep `InMemory<Feature>LocalDataSource` in the same file — useful for unit testing.

### 13. Verify

Before reporting, confirm each item — fix any violations:

- [ ] `@Entity` with `tableName`, `@PrimaryKey` on id field
- [ ] `@Dao` interface with `Flow<List<Entity>>` reads and `@Insert(onConflict = REPLACE)` writes
- [ ] `@Database` with `@ConstructedBy` and `expect object` constructor
- [ ] Platform `actual val` database modules: Android uses `Room.databaseBuilder()`, iOS adds `.setDriver(BundledSQLiteDriver())`
- [ ] `Room<Feature>LocalDataSource` implements existing `<Feature>LocalDataSource` interface
- [ ] DI module swapped: `singleOf(::Room<Feature>LocalDataSource) bind <Feature>LocalDataSource::class`
- [ ] `<feature>DatabaseModule` included in `startKoin` modules list (both platform entry points)
- [ ] `InMemory<Feature>LocalDataSource` preserved for testing

### 14. Report

Output a summary listing files created, files modified, and next steps:
- Add database migrations for schema changes
- Consider pre-populating database with initial data
- Add indices for frequently queried columns
- `/cmp-scaffold:scaffold-tests <feature>` to generate tests for the persistent data layer (InMemory impl available as test double)
- `/cmp-scaffold:polish-ui <feature>` to enhance the UI with shimmer loading and rich visual design
