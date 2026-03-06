---
name: Code Ghost
description: Advanced static analysis engine that detects hidden architectural patterns, subtle anti-patterns, and provides contextual refactoring guidance for legacy code modernization
version: "2.4.1"
author: SMOUJBOT Analysis Engine
requires:
  - clang-tidy>=15.0
  - semgrep>=1.50.0
  - codeql>=2.13.0
  - jscpd>=3.8.0
  - radon>=5.1.0
conflicts:
  - simple-linter
  - basic-code-checker
tags:
  - analysis
  - patterns
  - anti-patterns
  - code-quality
  - refactoring
  - legacy-modernization
---

# Code Ghost

Deep-pattern static analysis tool that surfaces hidden code smells, architectural mismatches, and subtle quality issues that traditional linters miss. Operates across JavaScript/TypeScript, Python, Go, and Java.

## Purpose

Code Ghost enables:
- **Archeological reconstruction** of how a codebase evolved by analyzing commit history + pattern accumulation
- **Detection of zombie dependencies** - imported but never used, or used only in dead code paths
- **Identification of signature patterns** that indicate specific developer handoffs or emergency patches
- **Mapping of implicit coupling** between modules through shared constants, error messages, or business logic duplication
- **Finding "ghost code"** - code that appears to be active but is unreachable due to conditional compilation dead branches
- **Recovery of lost intent** by analyzing unnatural control flow that suggests workarounds for now-removed features

## Scope

```bash
# Complete multi-language analysis with historical reconstruction
code-ghost analyze --depth=full --history=90d ./src

# Targeted archeological dig: find patterns related to specific bug/feature
code-ghost trace --pattern=circular-dependency --after="2024-01-15" --until="2024-02-01"

# Generate architectural dependency graph with "ghost edges" (implicit coupling)
code-ghost graph --format=graphviz --show-implicit ./

# Detect zombie code across the entire repo
code-ghost zombie-scan --threshold=0.3 --exclude-tests --export-zombies=./zombies.json

# Analyze temporal pattern: "when did this anti-pattern emerge and why"
code-ghost timeline --pattern=god-object --file=src/services/UserService.ts

# Check for "pattern ghosts" - code that matches multiple mutually-exclusive patterns
code-ghost conflict-detect --aggressive ./src

# Refactoring impact simulation: "what breaks if I extract this?"
code-ghost impact-sim --refactor=extract-class --target=src/utils/price-calculator.ts::PriceCalculator

# Export detailed pattern inventory for knowledge transfer
code-ghost export --format=csv --include-confidence --output=./pattern-inventory.csv
```

## Work Process

### Phase 1: Corpus Ingestion & Normalization
1. Scans all tracked files, respects `.codeghostignore` patterns
2. Extracts Abstract Syntax Trees with language-specific parsers
3. Builds semantic model: types, scopes, control flow, data flow
4. If Git history available: reconstructs timeline of each symbol's modification
5. If no Git: creates snapshot signature for baseline comparison

### Phase 2: Pattern Library Application
Applies pattern detectors in this order:
1. **Structural**: cyclic dependencies, diamond dependencies, layer violations
2. **Behavioral**: shotgun surgery, feature envy, primitive obsession, switch statements on type codes
3. **Temporal**: rapid-fire commits (>5 changes to same file in 24h), long-term TODOs (>180 days), commented-out code blocks
4. **Social**: code ownership drift (many authors, no clear owner), bus factor >3 on critical module
5. **Technical debt accrual**: complexity growth without corresponding test coverage increase

### Phase 3: Ghost Detection
Special algorithms identify:
- **ZOMBIE FUNCTIONS**: Called only from dead code paths, or only via reflection/dynamic import
- **PHANTOM DEPENDENCIES**: Imports that appear used but are satisfied by package bundler tree-shaking
- **PATTERN CONFLICTS**: Same code region matches contradictory patterns (e.g., both "God Object" and "Data Class" - indicates confusion)
- **TEMPORAL ANOMALIES**: Sudden drops in cyclomatic complexity after a commit suggesting feature kill-switch
- **ARCHITECTURAL STRAIGHT-JACKETS**: Code that's clean but resists extension (all methods final/private, no extension points)

### Phase 4: Confidence Calculation
Each finding gets confidence score (0.0-1.0) based on:
- Number of pattern detectors that flagged same region (consensus)
- Matches across multiple languages (cross-language smell)
- Historical stability (pattern present for >N commits)
- Author consensus (multiple authors wrote similar code)
- Test coverage correlation (smell in low-coverage zones)

### Phase 5: Actionable Insight Generation
Transforms findings into specific, contextual suggestions:
- Exact code transformation (include before/after snippets)
- Estimated lines changed and affected files
- Risk assessment with rollback plan
- Phased migration strategy (if complex)
- Related findings that should be grouped

## Golden Rules

1. **Confidence Threshold**: Only report findings with confidence ≥0.65 for production code, ≥0.45 for exploratory analysis. Override with `--aggressive` for legacy rescue missions.

2. **False Positive Minimization**: If pattern matches >80% of codebase (e.g., "no tests"), treat as systemic issue, not per-location findings. Report once at project level.

3. **Context Preservation**: Never suggest refactoring that breaks debug stack traces or error message contracts. Maintain public API stability unless explicitly asked for breaking changes (`--breaking`).

4. **Historical Respect**: If pattern emerges from deliberate architectural decision (referenced in ADRs or README), flag as "Intentional" and document justification. Do not suggest removal without `--force`.

5. **Impact Tracking**: All high-confidence findings must include "ripple impact" calculation: other files/features that would be affected by fix.

6. **Gradual Ramp-Up**: Large refactoring (>5 files, >200 loc) must be broken into numbered steps with verification checkpoints between each.

7. **Test Coverage Correlation**: Never suggest removal/modification of code with >90% test coverage without `--dangerous` flag. Such code is likely well-tested for a reason.

8. **Zombie Code Neutrality**: Zombie detection should be neutral - zombie code may be intentional backup systems, staged rollouts, or canary deployments. Provide context, not judgments.

## Examples

### Example 1: Detecting Shotgun Surgery in a Price Calculation Module

```bash
$ code-ghost analyze --depth=full --patterns=shotgun-surgery ./src
```

**Input code pattern found**:
```typescript
// src/utils/price-calculator.ts
export function calculateTotal(items: Item[]): number {
  let total = 0;
  for (const item of items) {
    total += item.price;
    if (item.country === 'US') total += item.price * 0.08; // tax logic scattered
    if (item.country === 'EU') total += item.price * 0.20;
    if (item.country === 'JP') total += item.price * 0.10;
  }
  return total;
}

// src/services/tax-service.ts
export function applyTax(subtotal: number, country: string): number {
  if (country === 'US') return subtotal * 0.08;
  if (country === 'EU') return subtotal * 0.20;
  if (country === 'JP') return subtotal * 0.10;
  return 0;
}

// src/controllers/checkout.ts
if (order.country === 'US') tax = price * 0.08; // COPY-PASTE
if (order.country === 'EU') tax = price * 0.20;
if (order.country === 'JP') tax = price * 0.10;
```

**Code Ghost output**:
```
Finding #CG-2024-0342: Shotgun Surgery (Confidence: 0.89)
Files: src/utils/price-calculator.ts, src/services/tax-service.ts, src/controllers/checkout.ts
Root Cause: Tax calculation logic duplicated across 3 modules with identical country rates.
Impact: Adding new country requires changes in 3 places (high change coupling).
Historical: Pattern first appeared 11 months ago in commit 5f8a2b1 (initial checkout), duplicated in 7 locations over time.

Suggested Refactoring (Phased):
Phase 1: Create TaxEngine module with single source of truth
  src/services/tax-engine.ts (NEW)
    - class TaxEngine with calculate(country, amount)
    - centralized rate table
    - validation and logging

  Impact: +45 loc, -22 loc duplicated
  Risk: Low (pure function, no side effects)
  Rollback: Keep old functions temporarily, delegate to TaxEngine after deployment

Phase 2: Migrate callers one-by-one with feature flag
  - Deprecate old functions gradually
  - 3-week migration window with dual-write validation

Verification:
  - Run: code-ghost --check-only --pattern=shotgun-surgery post-migration
  - Ensure no other shotgun instances of tax calculation remain
  - Validate tax calculation parity with 1000 sample orders

Breaking? No - maintains backward compatibility through adapter functions.
```

### Example 2: Finding Ghost Dependencies

```bash
$ code-ghost zombie-scan --threshold=0.3 --exclude-tests ./my-app
```

**Output**:
```
Zombie Scan Results (7 zombies detected, confidence ≥0.3):

1. src/components/LegacyChart.tsx (Confidence: 0.73)
   Imported but never rendered. Detected via:
   - No usage in JSX templates (semgrep)
   - No dynamic import() calls (static analysis)
   - Not exported from index barrels
   Historical: Last rendered 6 months ago (commit abc123), removed from UI but kept in build.
   
   Action: Safe to delete after confirming no external package imports it.
   Check: grep -r "LegacyChart" ./packages/external | wc -l  # Should be 0

2. src/lib/old-auth-provider.ts (Confidence: 0.92)
   Imported by 1 file (src/main.ts) but that import is conditional under FEATURE_FLAG_X=false which is never true in production.
   
   Action: Remove unused import, delete file after 30-day flag observation period.

3. src/types/__tests__/migrations/v1.d.ts (Confidence: 0.41)
   Type definition from 2019 migration. No direct usage but referenced in 3 JSDoc @typedef comments in legacy files.
   
   Action: Keep (low confidence, but documentation value). Mark with @deprecated and link to V2 types.

Zombie Summary:
- 4 high-confidence (≥0.7): ready for removal
- 3 medium-confidence (0.4-0.7): investigate further
- 0 low-confidence (<0.4): none

Command to batch-remove high-confidence zombies (after manual review):
  code-ghost zombie-apply --dry-run --confidence=0.7 > changes.txt
  # Review changes.txt, then:
  code-ghost zombie-apply --confidence=0.7
```

### Example 3: Temporal Pattern Analysis - "When did this God Object emerge?"

```bash
$ code-ghost timeline --pattern=god-object --file=src/models/Order.ts
```

**Output** (in markdown format):
```markdown
# Pattern Timeline: God Object in src/models/Order.ts

## Current State (HEAD)
Confidence: 0.94 - 47 methods, 32 instance variables, 2 responsibilities merged (order processing + invoicing + shipping)

## Historical Evolution

### 2024-12-01 - Commit 9f8a2b1 (Confidence: 0.31 → 0.94)
- Added: `generateInvoice()`, `calculateShipping()`, `trackShipment()`, 12 new fields
- Change: +214 lines, -0 deletions
- Author: alice@company.com
- Reason: "Quick implementation for Q4 shipping promo"

### 2024-10-15 - Commit 3d2c4e5 (Confidence: 0.18 → 0.31)
- Added: `applyDiscounts()`, `validateCoupon()`, `sendConfirmationEmail()`
- Change: +87 lines
- Author: bob@company.com  
- Reason: "Black Friday feature"

### 2023-08-20 - Commit 7b1a2c3 (Initial detection)
- First time pattern confidence exceeded 0.15
- File had 12 methods, was clean Domain Model
- Reason: "Refactor for multi-channel support"

## Pattern Trigger Analysis
- **Sprint pressure markers**: 70% of additions occurred 2 days before sprint deadline
- **Author handoff**: Alice added core shipping, Bob added promo logic - each extended for their needs without coordination
- **Test coverage correlation**: Coverage dropped from 87% to 43% over same period

## Recommended Intervention Points (roll back to these commits):
1. 2024-10-15: Split before discount/invoice logic entered
2. 2024-12-01: Split before shipping logic entrenched

## Rollback simulation:
If we revert commit 9f8a2b1 and refactor at 2024-10-15 state:
- Extract: `OrderShipping` (8 methods)
- Extract: `OrderInvoicing` (6 methods)  
- Extract: `OrderPromotion` (4 methods)
- Result: Order model reduces from 47 → 15 methods, focuses on core aggregate root duties

Impact: Would require migration of ~250 call sites but reduces complexity by 62%.

Next step: code-ghost refactor-plan --target=src/models/Order.ts --to-state=2024-10-15
```

## Environment Variables

```bash# Required for CodeQL analysis
CODE_GHOST_CODEQL_DB=/path/to/codeql-db  # Optional: pre-built database for faster analysis

# Optional: Increase Java heap for large codebases
CODE_GHOST_JAVA_OPTS="-Xmx8g -Xms2g"

# Optional: Semgrep configuration
SEMGREP_RULES=https://github.com/your-org/custom-rules.git
SEMGREP_TIMEOUT=300

# Optional: Git history depth (default 365 days)
CODE_GHOST_HISTORY_DAYS=180

# Optional: Exclude patterns
CODE_GHOST_EXCLUDE="**/node_modules/**,**/dist/**,**/*.min.js"
```

## Dependencies & Requirements

**Mandatory tools** (installed in PATH):
- `clang-tidy >= 15.0` - C/C++/Java analysis
- `semgrep >= 1.50.0` - multi-language pattern matching  
- `codeql >= 2.13.0` - deep semantic analysis
- `jscpd >= 3.8.0` - copy-paste detection
- `radon >= 5.1.0` - Python complexity metrics
- `git >= 2.20` - historical reconstruction (optional but recommended)

**Disk space**: 2GB minimum for CodeQL database cache

**Network**: Optional for Semgrep rule updates and CodeQL database download

**Filesystem**: Read access to entire project, `.codeghostignore` respected

## Verification

After running any analysis:

```bash
# 1. Check that all mandatory tools are available
code-ghost --version

# 2. Verify parser health on sample files  
code-ghost test-parser --sample=src/main.ts --sample=src/models/User.js

# 3. Ensure Git history is accessible (if using history features)
git log --oneline -1 > /dev/null || echo "Git not available, running in snapshot-only mode"

# 4. Validate findings are fresh (re-run same query to check determinism)
code-ghost analyze ./src --output=baseline.json
sleep 2
code-ghost analyze ./src --output=recheck.json
diff baseline.json recheck.json && echo "Deterministic" || echo "Non-deterministic - investigate"

# 5. Confirm no false positives on known-clean code
code-ghost analyze ./tests --patterns=god-object,feature-envy | grep -i "FOUND" && echo "False positives!" || echo "Clean test suite"
```

## Troubleshooting

**Issue: "Parser failed for file X.py"**
- Check Python version compatibility (Code Ghost needs same major version as project)
- Ensure no syntax errors in file: `python -m py_compile X.py`
- Add to `.codeghostignore` if generated/third-party code

**Issue: "Analysis taking >2 hours on medium codebase"**
- Reduce history depth: `CODE_GHOST_HISTORY_DAYS=30 code-ghost analyze`
- Use pre-built CodeQL database: `CODE_GHOST_CODEQL_DB=/path/to/db`
- Exclude large directories: `--exclude=src/gen/,src/vendor/`

**Issue: "Confidence scores all low (<0.3)"**
- Check that files have sufficient Git history (>50 commits)
- Ensure code is not minified/obfuscated
- For new projects: `--force` to bypass confidence threshold for baseline

**Issue: "Pattern conflict: file matches both X and Y"**
- This is expected for complex code. Review `code-ghost conflict-details <file>` to see regions
- Typically indicates need for extraction - different parts of file have different responsibilities
- Use `--resolve-conflicts` to get suggested partition

**Issue: "Out of memory during CodeQL analysis"**
- Set `CODE_GHOST_JAVA_OPTS="-Xmx16g"` (or appropriate for system)
- Split analysis: `code-ghost analyze --language=javascript ./src` and repeat for other languages
- Use cached database: `code-ghost analyze --codeql-cache-db=/path/to/cached-db`

**Issue: "False positive: flagged code that is intentionally complex"**
- Add `// code-ghost-ignore: pattern-name` comment above the code
- Or add to `.codeghostignore` with `# reason: intentional X pattern`
- Document in ADR why pattern is acceptable

**Issue: "No Git history available (bare repository)"**
- Code Ghost still works in snapshot-only mode but confidence will be 0.3-0.4 lower
- Enable `--history=skip` to avoid warnings
- Consider `git clone --mirror` to preserve history if analysis is critical

## Rollback Commands

### Undo Code Ghost analysis output (not source changes)
```bash
# Remove generated reports and temporary databases
code-ghost clean --all
# or manually:
rm -rf .codeghost/ .codeghost-cache/
```

### Undo applied zombie removals (if you used `code-ghost zombie-apply`)
```bash
# If you committed the removals:
git revert <commit-hash>  # Revert entire zombie-apply commit

# If not committed but changes staged:
git reset --hard HEAD  # WARNING: discards all uncommitted changes

# If you want selective rollback:
git checkout HEAD -- src/components/LegacyChart.tsx  # Restore one file
```

### Rollback from simulated refactoring impact
```bash
# After running code-ghost impact-sim and before applying:
# If you applied changes manually:
git checkout -b refactor-rollback
git reset --hard $(git rev-parse HEAD~N)  # N = number of commits made

# If using code-ghost apply-refactor (planned feature):
code-ghost refactor-rollback --plan=./plan-2024-0342.json
```

### Disable specific pattern detectors globally
```bash
# Create .codeghost/config.yaml:
# detectors:
#   disabled:
#     - "design-patterns/recommended"  # Too noisy
#     - "performance/n-plus-one"       # Handled by separate tool

# Then re-run analysis - disabled patterns won't appear
```

### Restore previous analysis baseline comparison
```bash
# Code Ghost keeps last 5 analysis runs in .codeghost/history/
code-ghost compare --from=.codeghost/history/2024-03-01.json --to=.codeghost/history/2024-03-15.json

# To restore to previous state (revert all changes shown in comparison):
git checkout $(git rev-list -n 1 --before="2024-03-01" main)  # if analysis-informed changes committed
```

## Advanced: Pattern Development

Define custom pattern detectors:

```yaml
# .codeghost/patterns/custom-patterns.yaml
pattern:
  name: "magic-number-cluster"
  description: "Multiple unrelated magic numbers in same file (indicates hardcoded business rules)"
  languages: ["javascript", "typescript", "python", "java"]
  detection:
    - Find integer/float literals not assigned to constants
    - Find 3+ distinct literals within 50 lines
    - Exclude common values: 0, 1, -1, 2, 100, 1000
  confidence_boost:
    - If literals match known business constants (currency, tax rates) +0.2
    - If scattered across multiple functions +0.3  
  suggestion:
    extract_to: "config/constants.{js,ts}"
    message: "Extract magic numbers to named constants in centralized config"
```

Register with: `code-ghost patterns register .codeghost/patterns/custom-patterns.yaml`

## Exit Codes

- `0`: Analysis completed successfully, findings within acceptable thresholds
- `1`: Findings exceed configured thresholds (normal for violations mode)
- `2`: Invalid arguments or configuration
- `3`: Missing required dependencies
- `4`: Internal analysis error (check logs in `.codeghost/logs/`)
- `5`: Pattern detector crashed - run with `--verbose` for details

## Logging

Verbose logging: `CODE_GHOST_LOG=debug code-ghost analyze ./src`
Logs written to `.codeghost/logs/analysis-YYYY-MM-DD-HHMMSS.log`

Structured JSON logs: `CODE_GHOST_LOG=json:file=.codeghost/logs/structured.log`

```