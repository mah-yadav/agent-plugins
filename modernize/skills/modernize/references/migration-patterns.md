# Migration Patterns

Use these patterns during Step 5 (Create the replacement plan) to choose the right approach for the replacement.

**Contents:** Pattern 1 — Direct 1:1 API Swap · Pattern 2 — Adapter/Wrapper · Pattern 3 — Gradual Module-by-Module · Pattern 4 — Strangler Fig (Parallel Run) · Pattern 5 — Big Bang · Choosing the Right Pattern · Common Pitfalls

## Pattern 1: Direct 1:1 API Swap

**When to use**: The new component/library has a nearly identical API surface. Most calls can be replaced mechanically.

**Approach**:
1. Update imports across the codebase.
2. Rename function calls, props, or class references according to the mapping table.
3. Adjust minor parameter differences (renamed options, changed defaults).
4. Update build tooling configs that reference the old library.
5. Remove the old dependency.

**Risk level**: Low
**Example**: Replacing `moment.js` with `dayjs` (similar API, mostly drop-in).

**Watch out for**:
- Format string token differences (e.g., `YYYY` vs `yyyy`).
- Subtle behavioral differences in edge cases (timezone handling, locale loading).
- Plugins that extend the old library — the new library may need equivalent plugins enabled.

---

## Pattern 2: Adapter/Wrapper Pattern

**When to use**: The new API differs significantly from the old one, or you need to support both old and new simultaneously during a transition period.

**Approach**:
1. Create an adapter module that exposes the old API but internally uses the new library.
2. Update all imports to point to the adapter instead of the old library directly.
3. Remove the old library dependency.
4. Over time, migrate callers from the adapter to the new API directly.
5. Remove the adapter once all callers are migrated.

**Risk level**: Medium
**Tradeoff**: Adds an indirection layer, but minimizes blast radius and allows incremental migration.

**Watch out for**:
- The adapter can become a permanent fixture if there's no follow-up plan. Set a deadline or track adapter removal as a separate task.
- Performance overhead from the indirection layer (usually negligible, but measure for hot paths).
- Type safety — the adapter's type signatures need to match both old callers and new internals.

---

## Pattern 3: Gradual Module-by-Module Migration

**When to use**: The codebase is large, the replacement is complex, and you can't afford to change everything at once.

**Approach**:
1. Pick one module or feature area as the pilot.
2. Replace all usages in the pilot module.
3. Verify the pilot (tests, manual review).
4. If successful, proceed to the next module.
5. Repeat until all modules are migrated.
6. Remove the old dependency once no usages remain.

**Risk level**: Low to Medium
**Tradeoff**: Slower overall, but each step is small and reversible. Both old and new may coexist temporarily.

**Watch out for**:
- Shared utilities or wrappers that are used across modules — migrating one module may break others if the wrapper is changed.
- Both old and new libraries must coexist in the dependency tree. Check for version conflicts or global state collisions.
- Test isolation — tests for unmigrated modules must still pass with the new library installed alongside the old one.

---

## Pattern 4: Strangler Fig (Parallel Run)

**When to use**: The replacement is high-risk, the old and new components can coexist, and you want to validate behavior before fully committing.

**Approach**:
1. Introduce the new component alongside the old one.
2. Route new features or new code paths to the new component.
3. Optionally run both in parallel and compare outputs for critical paths.
4. Gradually migrate existing code paths from old to new.
5. Remove the old component once all paths use the new one.

**Risk level**: Medium to High (complexity of running both)
**Tradeoff**: Safest for critical systems, but requires maintaining two implementations temporarily.

**Watch out for**:
- Global state or singletons — two libraries managing the same global state (e.g., HTTP client defaults, database connection pools) can interfere with each other.
- Increased bundle size / memory usage during the parallel run period.
- Developer confusion — clear naming conventions and documentation are essential to prevent accidental use of the wrong implementation.

---

## Pattern 5: Big Bang Replacement

**When to use**: The codebase is small, the replacement is well-understood, and you have good test coverage to catch regressions.

**Approach**:
1. Replace everything at once across the entire codebase.
2. Run the full test suite to validate.
3. Fix any issues immediately.

**Risk level**: High (if test coverage is low)
**Tradeoff**: Fastest approach, but risky without strong tests. No coexistence period.

**Watch out for**:
- Ensure the replacement is truly well-understood — hidden behavioral differences surface only after the full swap.
- Have a rollback plan ready (feature branch, revert commit).
- Run integration tests, not just unit tests — behavioral differences often manifest at integration boundaries.

---

## Choosing the Right Pattern

| Factor                       | Direct Swap | Adapter | Module-by-Module | Strangler Fig | Big Bang |
| ---------------------------- | :---------: | :-----: | :--------------: | :-----------: | :------: |
| APIs are similar             |      ✅      |    —    |        —         |       —       |    ✅     |
| APIs differ significantly    |      ❌      |    ✅    |        ✅         |       ✅       |    ❌     |
| Small codebase (≤ 20 files)  |      ✅      |    —    |        —         |       —       |    ✅     |
| Large codebase (100+ files)  |      ❌      |    ✅    |        ✅         |       ✅       |    ❌     |
| Need backward compatibility  |      ❌      |    ✅    |        ✅         |       ✅       |    ❌     |
| High-risk / critical system  |      ❌      |    ✅    |        ✅         |       ✅       |    ❌     |
| Good test coverage           |      ✅      |    ✅    |        ✅         |       ✅       |    ✅     |
| Low test coverage            |      ❌      |    ✅    |        ✅         |       ✅       |    ❌     |
| Behavioral differences       |      ❌      |    ✅    |        ✅         |       ✅       |    ❌     |
| Build tooling changes needed |      ✅      |    ✅    |        ⚠️         |       ⚠️       |    ✅     |
| Framework integrations exist |      ⚠️      |    ✅    |        ✅         |       ✅       |    ⚠️     |

**Legend**: ✅ = Good fit, ❌ = Poor fit, ⚠️ = Possible but needs extra care, — = Neutral

## Common Pitfalls

- **Forgetting transitive usages**: The old library may be re-exported or wrapped. Trace through abstraction layers.
- **Missing type-level references**: Interfaces, generics, and type aliases that reference the old library are easy to overlook.
- **Test mocks out of sync**: Tests may mock the old library. These mocks need to be updated or replaced, not just the production code.
- **Configuration drift**: Config files (e.g., babel plugins, webpack loaders, ESLint rules) may reference the old library.
- **Implicit dependencies**: Some libraries register global side effects (polyfills, prototype extensions). Removing them may break unrelated code.
- **Dynamic/string references**: Plugin configs, DI bindings, and reflection-based code may reference the library by string name. These won't be caught by import-based search alone.
- **Behavioral assumptions in callers**: Callers may depend on behavioral details (return types, error shapes, async vs sync, default values) that differ between old and new. Syntax-correct replacement can still be semantically wrong.
- **Build tooling left behind**: CI/CD configs, Dockerfiles, and build scripts referencing the old library by name cause failures after the library is removed.
- **Lock file drift**: After updating dependency manifests, failing to regenerate the lock file can cause inconsistent installs across environments.
- **Missing rollback plan**: Starting a replacement without a clear rollback strategy turns every issue into a crisis.
