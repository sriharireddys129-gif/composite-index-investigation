# Composite Index Ordering Investigation

## 1. Objective

The objective of this investigation was to understand how the column order of a composite B-tree index affects query execution in PostgreSQL.

The query filters employees by department and salary:

```sql
SELECT *
FROM employees
WHERE department = 'Sales'
  AND salary > 50000;
```

## 2. Original Index

The original index was defined as:

```sql
CREATE INDEX idx_salary_department
ON employees(salary, department);
```

The salary column appears first, followed by the department column.

### Observations

The initial execution plan used an Index Scan on `idx_salary_department`. After running `ANALYZE employees`, PostgreSQL selected a Sequential Scan because the original table contained only eight rows.

This demonstrated that the query planner considers table size and estimated execution costs when choosing a plan.

## 3. Dataset Used for Investigation

To investigate the indexes with a larger dataset, 100,000 additional employee records were inserted using PostgreSQL's `generate_series` function.

The original eight records were retained, resulting in 100,008 employee records.

The table structure was not modified.

After inserting the additional records, the table statistics were refreshed using:

```sql
ANALYZE employees;
```

## 4. Reordered Composite Index

The following index was created:

```sql
CREATE INDEX idx_department_salary
ON employees(department, salary);
```

The department column appears first, followed by the salary column.

The same query was executed with the original index and then with the reordered index. Each index was tested separately, while using the same dataset and query.

## 5. Execution Plan Comparison

Both index configurations returned 19,002 matching rows.

| Observation | Original Index | Reordered Index |
|---|---|---|
| Index column order | `(salary, department)` | `(department, salary)` |
| Index searches reported | 61 | 1 |
| Bitmap Index Scan time | Approximately 3.54 ms | Approximately 2.44 ms |
| Bitmap Heap Scan time | Approximately 3.83–10.37 ms | Approximately 2.59–6.36 ms |
| Matching rows | 19,002 | 19,002 |

In the observed execution plans, the reordered index required fewer index searches and showed lower reported execution times.

These measurements are observations from individual runs. They do not establish a universal performance improvement because execution time can vary with cache state, system load, data distribution, and query-planner decisions.

## 6. Left-Most Prefix Rule

A multicolumn B-tree index stores its indexed columns in the specified order.

The Left-Most Prefix Rule describes how queries can use the leading columns of a multicolumn index. Conditions on the leading column or columns generally provide the most direct access to the relevant portion of the index.

For this query:

- `department = 'Sales'` is an equality condition.
- `salary > 50000` is a range condition.

The reordered index, `(department, salary)`, places the equality-filtered column first and the range-filtered column second. This is a suitable index design for this query pattern because PostgreSQL can narrow the search by department before applying the salary range.

The original index, `(salary, department)`, places the range-filtered column first. In the observed execution plan, PostgreSQL reported more index searches for this index.

The rule is a useful design principle, not an absolute guarantee of which index PostgreSQL will choose. The optimizer considers the query, data distribution, statistics, estimated costs, and available indexes.

## 7. Conclusion

This investigation demonstrated that composite index column order can affect PostgreSQL query execution.

For the tested query, the reordered index `(department, salary)` produced an execution plan with one reported index search, compared with 61 for `(salary, department)`. Both configurations returned the same 19,002 matching rows.

The observations support using `(department, salary)` as the more suitable index design for this particular equality-plus-range query. Further repeated tests and measurements would be needed to establish a reliable performance difference.