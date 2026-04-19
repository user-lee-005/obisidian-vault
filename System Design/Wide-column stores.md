- Popular DB - Cassandra, Cosmos DB
- This is designed to handle massive, sparse and distributed datasets efficiently.
- The core idea of the wide column store is storing the Map<Map<String, String>>
	Row key -> Column Family -> Columns (key-value pairs)
- Because of this
	1. Rows don't need the same columns.
	2. Columns can be added dynamically.
	3. Data is stored by column families, not strictly rows.

---

**Logical View**

	UserID     Personal:Name      Personal:Age      Activity:last_login
	-------------------------------------------------------------------
	1          Leela              25                today              
	2          Venkatesh          25                -----              

- The missing columns are fine, no _NULL_ storage overhead like SQL
- Columns are grouped into families like Personal, Activity

---

**Physical View**

	RowKey: 101
		ColumnFamily: Personal
			name -> Leela
			age -> 25
		ColumnFamily: Activity
			last_login -> today

---

**Sparse Storage**

If a row doesn’t have a column:
- It is **NOT stored at all**
- No wasted space
This is why wide-column stores dominate:
- Logs
- IoT data
- Analytics workloads

---

**Key Characteristics**
- **Schema-flexible** - Don't predefine the columns
- **Column Family based** - Data grouped logically
- **Highly scalable** - Designed for distributed clusters
- **Fast writes** - Optimized for append-heavy workloads
- **Eventually consistent (usually)** - Not strict like SQL
- **Scaling** - Horizontal

---

**!! DONT USE IT IF**
1. You need Complex Joins
2. You want **Strict ACID transactions**
3. Your data is small or structured -> SQL is simpler and better