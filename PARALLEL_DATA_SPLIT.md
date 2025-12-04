# Parallel Test Data Split Guide

## Problem
When running tests in parallel with multiple workers, if the same ReferenceID is used, both workers try to access the same data row, causing:
- Concatenated IDs like `'10005,10003'` that don't exist in Excel
- Empty data rows leading to test failures

## Solution
The `parallelDataResolver` utility automatically splits comma-separated reference IDs and assigns one per worker.

## Usage

### 1. In testmanager.xlsx
Set ReferenceID as comma-separated values for parallel execution:

```
TestCaseID          | ReferenceID      | Execute
--------------------|------------------|--------
create_inv_payable  | 10005,10003      | yes
```

### 2. Trial Run (Automatic)
The trial adapter automatically injects the resolver:

```bash
# Run with 2 parallel workers
npx playwright test --workers=2
```

Worker 0 gets `10005`, Worker 1 gets `10003`

### 3. Manual Integration
If not using trial adapter, add to your test script:

```typescript
import { resolveParallelReferenceId } from "../util/parallelDataResolver.ts";

// Replace this:
const dataReferenceId = String(testRow?.['ReferenceID'] ?? '').trim() || defaultReferenceId;

// With this:
const rawReferenceId = String(testRow?.['ReferenceID'] ?? '').trim() || defaultReferenceId;
const dataReferenceId = resolveParallelReferenceId(
  rawReferenceId, 
  testinfo.parallelIndex ?? 0, 
  testinfo.config.workers ?? 1
);
```

## How It Works
- Splits `"10005,10003"` → `["10005", "10003"]`
- Worker 0 (parallelIndex=0) → `"10005"`
- Worker 1 (parallelIndex=1) → `"10003"`
- Round-robin if more workers than IDs

## Example Data Sheet
```
Invoice ID | Supplier        | Number          | Amount
-----------|-----------------|-----------------|-------
10005      | TEST_Sup_005    | CM-SHEZ2233205  | 100.00
10003      | TEST_Sup_003    | CM-SHEZ2233203  | 100.00
```

Each worker gets unique data, avoiding conflicts.
