# Understanding LEFT JOIN with INNER JOIN

## 🔍 Overview
When combining `LEFT JOIN` and `INNER JOIN` in a SQL query, the execution order significantly impacts the final result. This document explains the logic and potential issues that may arise when using both joins together.

---

## 📌 SQL Query Structure
```sql
SELECT *
FROM TableA A
LEFT JOIN TableB B ON A.ID = B.A_ID
INNER JOIN TableC C ON A.ID = C.A_ID;
```

### 🔹 How It Works
1. **`LEFT JOIN TableB`**:
   - Includes all records from `TableA`, even if no match exists in `TableB`.
   - If no match is found in `TableB`, the columns from `TableB` will contain `NULL` values.

2. **`INNER JOIN TableC`**:
   - This join filters out records from `TableA` that do not have a match in `TableC`.
   - Since this join is on `TableA`, it affects all previous joins, meaning even some records included in `LEFT JOIN` may get removed.

---

## 📊 Example Data

### **TableA (Customers)**
| ID  | Name  |
|-----|-------|
| 1   | Ali   |
| 2   | Reza  |
| 3   | Sara  |

### **TableB (Orders)**
| A_ID | OrderID |
|------|---------|
| 1    | 101     |
| 2    | 102     |

### **TableC (Payments)**
| A_ID | PaymentID |
|------|----------|
| 1    | 201      |
| 3    | 202      |

---

## 🔥 Execution Flow

### ✅ Expected Output
```sql
SELECT A.ID, A.Name, B.OrderID, C.PaymentID
FROM TableA A
LEFT JOIN TableB B ON A.ID = B.A_ID
INNER JOIN TableC C ON A.ID = C.A_ID;
```

| ID  | Name  | OrderID | PaymentID |
|-----|-------|---------|-----------|
| 1   | Ali   | 101     | 201       |

### ❌ Why Are Some Records Missing?
- **Reza (ID = 2) is removed** because he has no payment in `TableC`.
- **Sara (ID = 3) is removed** because `LEFT JOIN` kept her, but `INNER JOIN` on `TableC` required a match.

---

## ⚡ Optimizing the Query
### 🔹 If You Need All `TableA` Records:
Use `LEFT JOIN` on `TableC` instead:
```sql
SELECT *
FROM TableA A
LEFT JOIN TableB B ON A.ID = B.A_ID
LEFT JOIN TableC C ON A.ID = C.A_ID;
```

### 🔹 If `TableB` Data Is Not Needed:
Simply remove the unnecessary `LEFT JOIN`:
```sql
SELECT *
FROM TableA A
INNER JOIN TableC C ON A.ID = C.A_ID;
```
This removes the extra overhead and improves query performance.

---

## 🎯 Key Takeaways
- `LEFT JOIN` retains unmatched records, while `INNER JOIN` removes them.
- Using `INNER JOIN` after `LEFT JOIN` can **nullify the effect of `LEFT JOIN`**, making it redundant.
- If `TableB` is not essential to the output, **removing `LEFT JOIN` results in a more optimized query**.
- Be mindful of the join order to avoid unintended data loss.

---

