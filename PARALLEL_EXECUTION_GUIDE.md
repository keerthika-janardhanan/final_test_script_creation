# Parallel Test Execution with Different Data Rows

## Overview

Run the same test flow multiple times in parallel, each execution using a different data row from Excel. This enables efficient data-driven testing where each test worker picks its own data row.

## How It Works

```
Excel File (PayablesData.xlsx):
┌───────────┬──────────────────────────────┬──────────────────┬────────────┐
│ Invoice ID│         Supplier             │     Number       │   Amount   │
├───────────┼──────────────────────────────┼──────────────────┼────────────┤
│   9998    │ PrimeSource Distributors     │ CM-SHEZ2233198   │   100.00   │
│   9999    │ EverBright Traders           │ CM-SHEZ2233199   │   100.00   │
│   10000   │ Nova Industrial Solutions    │ CM-SHEZ2233200   │   100.00   │
│   10001   │ TEST_Sup_001                 │ CM-SHEZ2233201   │   100.00   │
│   10002   │ TEST_Sup_002                 │ CM-SHEZ2233202   │   100.00   │
└───────────┴──────────────────────────────┴──────────────────┴────────────┘
                                    ↓
                    Playwright loads all 5 rows
                                    ↓
        ┌──────────────┬──────────────┬──────────────┬──────────────┬──────────────┐
        │   Worker 1   │   Worker 2   │   Worker 3   │   Worker 4   │   Worker 5   │
        ├──────────────┼──────────────┼──────────────┼──────────────┼──────────────┤
        │ Row 9998     │ Row 9999     │ Row 10000    │ Row 10001    │ Row 10002    │
        │ PrimeSource  │ EverBright   │ Nova         │ TEST_Sup_001 │ TEST_Sup_002 │
        └──────────────┴──────────────┴──────────────┴──────────────┴──────────────┘
                          All execute simultaneously
```

## File Structure

### 1. New Test File: `payables-parallel.spec.ts`

This file loads ALL data rows and creates individual tests for each:

```typescript
const allDataRows = getAllDataRowsForTest(dataFilePath, '');
// Result: [
//   { "Invoice ID": 9998, "Supplier": "PrimeSource Distributors", ... },
//   { "Invoice ID": 9999, "Supplier": "EverBright Traders", ... },
//   { "Invoice ID": 10000, "Supplier": "Nova Industrial Solutions", ... },
//   ...
// ]

// Create a test for each row
allDataRows.forEach((dataRow, index) => {
  test(`payables-${invoiceId} (${supplier})`, async ({ page }) => {
    // Each test uses its own dataRow
    await payablespage.applyData(dataRow, ["Supplier"]);
    // ...
  });
});
```

### 2. Helper Function: `getAllDataRowsForTest()`

Added to `csvFileManipulation.ts`:

```typescript
export function getAllDataRowsForTest(
  filePath: string, 
  sheetName: string
): Array<Record<string, any>> {
  const workBook = XLSX.readFile(filePath);
  const sheet = sheetName || workBook.SheetNames[0];
  const workSheet = workBook.Sheets[sheet];
  const data = XLSX.utils.sheet_to_json(workSheet, { defval: '' });
  return data; // Returns ALL rows
}
```

## Usage

### Option 1: Run All Data Rows in Parallel

```bash
# Run the parallel test file
npx playwright test payables-parallel.spec.ts --workers=5

# Output:
# ✓ payables-9998 (PrimeSource Distributors)
# ✓ payables-9999 (EverBright Traders)
# ✓ payables-10000 (Nova Industrial Solutions)
# ✓ payables-10001 (TEST_Sup_001)
# ✓ payables-10002 (TEST_Sup_002)
```

### Option 2: Run Specific Data Rows

Use Playwright's test filtering:

```bash
# Run only TEST_Sup_001
npx playwright test payables-parallel.spec.ts -g "10001"

# Run first 3 rows
npx playwright test payables-parallel.spec.ts -g "9998|9999|10000"
```

### Option 3: Run in Headed Mode (see each execution)

```bash
npx playwright test payables-parallel.spec.ts --headed --workers=3
```

## Configuration

### Playwright Workers

Control parallelism in `playwright.config.ts`:

```typescript
export default defineConfig({
  workers: process.env.CI ? 1 : 5, // 5 parallel workers locally
  fullyParallel: true, // Each test runs independently
});
```

### Test Execution Matrix

| Workers | Data Rows | Execution Time | Use Case                    |
|---------|-----------|----------------|-----------------------------|
| 1       | 10        | ~10 min        | Debugging, sequential       |
| 5       | 10        | ~2 min         | Balanced parallel execution |
| 10      | 10        | ~1 min         | Maximum speed (if resources)|
| 3       | 100       | ~35 min        | Batch processing            |

## Test Naming

Each test gets a unique name based on its data:

```typescript
test(`payables-${invoiceId} (${supplier})`, async ({ page }) => {
  // Test execution
});
```

Examples:
- `payables-9998 (PrimeSource Distributors)`
- `payables-10001 (TEST_Sup_001)`
- `payables-10007 (TEST_Sup_007)`

## Benefits

### 1. **True Parallel Execution**
- Each worker processes a different data row
- No data conflicts or race conditions
- Linear speedup with more workers

### 2. **Data Isolation**
- Each test has its own `dataRow` variable
- No shared state between tests
- Independent screenshots and reports

### 3. **Flexible Filtering**
```bash
# Run only specific suppliers
npx playwright test -g "TEST_Sup"

# Run specific invoice ranges
npx playwright test -g "1000[1-5]"
```

### 4. **Easy to Extend**
```typescript
// Add more data rows in Excel
// Automatic test generation for new rows
```

## Comparison: Original vs Parallel

### Original `payables.spec.ts`
```typescript
// Single test execution
test("payables", async ({ page }) => {
  // Reads ONE specific row (ReferenceID from testmanager.xlsx)
  const dataRow = readExcelData(..., "10001", "Invoice ID");
  // Uses: Invoice 10001 (TEST_Sup_001)
});
```

**Result**: 1 test, 1 execution, uses 1 data row

### Parallel `payables-parallel.spec.ts`
```typescript
// Multiple test executions
allDataRows.forEach((dataRow) => {
  test(`payables-${id}`, async ({ page }) => {
    // Uses: This specific dataRow
  });
});
```

**Result**: 10 tests, 10 executions, uses all 10 data rows in parallel

## Execution Flow

```
1. Test file loads
   └─> getAllDataRowsForTest() reads entire Excel
       └─> Returns array of ALL rows

2. forEach creates tests
   └─> test("payables-9998") created
   └─> test("payables-9999") created
   └─> test("payables-10000") created
   └─> ... one test per row

3. Playwright scheduler
   └─> Assigns tests to workers
       ├─> Worker 1: payables-9998
       ├─> Worker 2: payables-9999
       ├─> Worker 3: payables-10000
       ├─> Worker 4: payables-10001
       └─> Worker 5: payables-10002

4. Parallel execution
   └─> All workers run simultaneously
       └─> Each uses its own dataRow
           └─> No data conflicts
```

## Advanced: Conditional Data Loading

Filter data rows before creating tests:

```typescript
// Only load rows with Amount > 50
const allDataRows = getAllDataRowsForTest(dataFilePath, '')
  .filter(row => parseFloat(row['Amount']) > 50);

// Only load TEST suppliers
const testDataRows = getAllDataRowsForTest(dataFilePath, '')
  .filter(row => row['Supplier']?.startsWith('TEST_'));

// Load specific invoice range
const rangeDataRows = getAllDataRowsForTest(dataFilePath, '')
  .filter(row => row['Invoice ID'] >= 10001 && row['Invoice ID'] <= 10005);
```

## Report Output

Each execution generates separate results:

```
test-results/
├── payables-9998-PrimeSource-Distributors/
│   ├── test-failed-1.png
│   └── trace.zip
├── payables-9999-EverBright-Traders/
│   ├── test-passed-1.png
│   └── trace.zip
└── payables-10001-TEST-Sup-001/
    ├── test-passed-1.png
    └── trace.zip
```

HTML Report shows each test separately:
```
✓ payables-9998 (PrimeSource Distributors) [2.3s]
✓ payables-9999 (EverBright Traders) [2.1s]
✓ payables-10000 (Nova Industrial Solutions) [2.4s]
✓ payables-10001 (TEST_Sup_001) [2.2s]
```

## Best Practices

### 1. **Independent Data**
Ensure each data row creates unique entities (different supplier names, invoice numbers):
```
Invoice ID | Supplier      | Number
10001      | TEST_Sup_001  | CM-SHEZ2233201  ✅ Unique
10002      | TEST_Sup_002  | CM-SHEZ2233202  ✅ Unique
10003      | TEST_Sup_001  | CM-SHEZ2233201  ❌ Duplicate (will conflict)
```

### 2. **Worker Limits**
Don't exceed your system's capacity:
```typescript
// Good for 8-core machine
workers: 6

// May cause performance issues
workers: 20
```

### 3. **Test Stability**
Ensure tests don't interfere with each other:
- Use unique invoice numbers
- Use unique supplier names (or append timestamp)
- Clean up test data after execution

### 4. **Debugging Single Test**
```bash
# Debug specific data row
npx playwright test payables-parallel.spec.ts -g "10001" --headed --debug
```

## Migration Path

1. ✅ Keep `payables.spec.ts` for single-run execution
2. ✅ Use `payables-parallel.spec.ts` for batch testing
3. Run based on need:
   - CI/CD: Use parallel for regression testing
   - Development: Use single for quick validation
   - Data validation: Use parallel with all data rows

## Summary

- **Original file**: `payables.spec.ts` - Single test, one data row
- **Parallel file**: `payables-parallel.spec.ts` - Multiple tests, all data rows
- **Key function**: `getAllDataRowsForTest()` - Loads all Excel rows
- **Execution**: Each test = one data row, all run in parallel
- **Benefit**: Test 10 data rows in ~2 minutes instead of ~10 minutes
