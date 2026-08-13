# XAP SQL & JDBC

XAP exposes two SQL interfaces:
- **`SQLQuery<T>`** — native space API; strongly typed; best for latency-critical paths
- **GigaSpaces JDBC Driver** (`jdbc:gigaspaces:v3://`) — standard JDBC; supports JOIN, GROUP BY, EXPLAIN ANALYZE, `DYNAMIC_FILTER` hint

---

## SQLQuery API (prefer for performance-critical code)

### Basic Usage

```java
import com.j_spaces.core.client.SQLQuery;

// Parameterized query — ALWAYS use parameters, never string concatenation
SQLQuery<Payment> query = new SQLQuery<>(Payment.class, "status = ? AND amount > ?");
query.setParameter(1, TransactionStatus.PENDING);
query.setParameter(2, 100.0);

Payment[] results = gigaSpace.readMultiple(query, 500); // always set a limit
```

Always use `?` placeholders, never inline a value into the query string. A parameterized query's
text is identical across calls, so the engine can cache the parsed/analyzed query plan and reuse
it — inlining values makes each call a distinct query string, forcing re-parsing and re-analysis
every time. See `space-operations.md` § SQLQuery.

### Projections (return only needed fields)

```java
// Only fetch the transactionFee property — avoids deserializing the full object
SQLQuery<Contract> query = new SQLQuery<>(Contract.class, "merchantAccountId = ?")
        .setProjections("transactionPrecentFee");
query.setParameter(1, merchantId);
Contract contract = gigaSpace.read(query);
// Only transactionPrecentFee will be populated; all other fields will be null
```

### Routing in SQLQuery

```java
// If you know the routing value, set it to avoid scatter-gather across all partitions
SQLQuery<Payment> query = new SQLQuery<>(Payment.class, "paymentId = ?");
query.setParameter(1, paymentId);
query.setRouting(merchantId); // routes to the single partition owning merchantId
Payment payment = gigaSpace.read(query);
```

### Supported Operators

```sql
-- Comparison
status = ?   status != ?   amount > ?   amount >= ?   amount < ?   amount <= ?

-- Range
amount BETWEEN 100 AND 500

-- NULL checks
field IS NULL    field IS NOT NULL

-- String matching
name LIKE 'John%'   name RLIKE '(?i)john.*'   -- RLIKE = Java regex

-- IN list
status IN ('PENDING', 'ACTIVE')

-- Logical
field1 = ? AND field2 > ?
field1 = ? OR  field2 = ?
NOT (field1 = ?)
```

### SpaceIterator (large result sets)

```java
import com.gigaspaces.client.iterator.SpaceIterator;
import com.gigaspaces.client.iterator.SpaceIteratorConfiguration;

// Never use readMultiple with MAX_VALUE for large datasets — use iterator
SQLQuery<Payment> query = new SQLQuery<>(Payment.class, "status = ?");
query.setParameter(1, TransactionStatus.PENDING);

SpaceIteratorConfiguration config = new SpaceIteratorConfiguration().setBatchSize(200);
SpaceIterator<Payment> iterator = gigaSpace.iterator(query, config);

while (iterator.hasNext()) {
    Payment payment = iterator.next();
    // process one at a time
}
iterator.close(); // always close to release server-side resources
```

---

## JDBC Driver

### Maven Dependency

```xml
<dependency>
    <groupId>com.gigaspaces</groupId>
    <artifactId>xap-jdbc</artifactId>
    <version>17.3.0</version>
</dependency>
```

### Connection URL Formats

```java
// Remote space via manager (recommended for production)
"jdbc:gigaspaces:v3://localhost:4174/mySpace"

// Embedded space (same JVM)
"jdbc:gigaspaces:v3://localhost/mySpace"
```

### Basic JDBC Connection

```java
import java.sql.*;
import java.util.Properties;

public Connection getConnection(String spaceName) throws Exception {
    Properties props = new Properties();
    props.put("com.gs.embeddedQP.enabled", "true"); // enables embedded query processing
    return DriverManager.getConnection(
            "jdbc:gigaspaces:v3://localhost:4174/" + spaceName, props);
}
```

---

## SQL Syntax Reference

### Table Names

Table names are **fully qualified class names** in double quotes:

```sql
SELECT * FROM "com.example.model.Payment" WHERE status = 'PENDING'
SELECT * FROM "com.example.model.User"    WHERE balance > 1000
```

### SELECT with WHERE

```sql
SELECT paymentId, amount, status
FROM "com.example.model.Payment"
WHERE status = 'PENDING' AND amount > 100.0
ORDER BY amount DESC
LIMIT 50
```

### JOIN (requires routing alignment for efficiency)

```sql
-- Efficient: both tables share a common routing key (userId/studentId)
SELECT u.name, p.amount
FROM "com.example.model.User"    AS u
JOIN "com.example.model.Payment" AS p ON u.userAccountId = p.payingAccountId
WHERE u.userAccountId = ?
```

### GROUP BY / Aggregation

```sql
SELECT category, COUNT(*) AS cnt, SUM(amount) AS total
FROM "com.example.model.Payment"
GROUP BY category
ORDER BY total DESC
```

### DYNAMIC_FILTER Hint (Join Optimization)

When joining across tables without explicit routing alignment, the `DYNAMIC_FILTER` hint tells XAP to push down filters partition-by-partition after resolving the leading table, reducing cross-partition data movement:

```sql
SELECT /*+ DYNAMIC_FILTER */
    s.id, s.firstName, s.lastName, sc.sem, c.name
FROM "com.gs.Student" AS s
JOIN "com.gs.StudentCourses" AS sc ON s.id = sc.studentId
JOIN "com.gs.Courses"        AS c  ON c.id = sc.courseId
WHERE s.id = ?
```

**When to use `DYNAMIC_FILTER`:**
- JOIN where the leading table has a routing condition but joined tables don't
- Avoids full-partition scatter on joined tables
- Contrast with explicit routing: adding `WHERE sc.studentId = ?` explicitly achieves similar results without the hint

### EXPLAIN ANALYZE

Use `EXPLAIN ANALYZE FOR <query>` to inspect execution plan and index usage:

```java
String explain = "EXPLAIN ANALYZE FOR SELECT * FROM \"com.example.model.Payment\" WHERE status = ?";
PreparedStatement ps = connection.prepareStatement(explain);
ps.setString(1, "PENDING");
ResultSet rs = ps.executeQuery();
// Print explain plan columns
ResultSetMetaData meta = rs.getMetaData();
while (rs.next()) {
    for (int i = 1; i <= meta.getColumnCount(); i++) {
        System.out.print(meta.getColumnName(i) + "=" + rs.getString(i) + "  ");
    }
    System.out.println();
}
```

---

## Spring JdbcTemplate with XAP

```java
import org.springframework.jdbc.core.JdbcTemplate;
import com.zaxxer.hikari.HikariDataSource; // or any DataSource

// Configure DataSource pointing at XAP JDBC
HikariDataSource ds = new HikariDataSource();
ds.setJdbcUrl("jdbc:gigaspaces:v3://localhost:4174/mySpace");
ds.setDriverClassName("com.gigaspaces.jdbc.Driver");

JdbcTemplate jdbc = new JdbcTemplate(ds);

// Query returning a list of maps
List<Map<String, Object>> rows = jdbc.queryForList(
        "SELECT paymentId, amount FROM \"com.example.model.Payment\" WHERE status = ?",
        "PENDING");

// Query mapping to a POJO
List<Payment> payments = jdbc.query(
        "SELECT * FROM \"com.example.model.Payment\" WHERE amount > ?",
        (rs, rowNum) -> {
            Payment p = new Payment();
            p.setPaymentId(rs.getString("paymentId"));
            p.setPaymentAmount(rs.getDouble("paymentAmount"));
            return p;
        },
        100.0);
```

---

## SQLQuery vs JDBC Decision Guide

| Factor | Use SQLQuery | Use JDBC |
|--------|-------------|---------|
| Latency-critical path | ✅ | ❌ |
| Simple get-by-routing | ✅ | ❌ |
| JOIN across types | ❌ | ✅ |
| GROUP BY / ORDER BY | ❌ (use aggregators) | ✅ |
| Dynamic ad-hoc queries | ❌ | ✅ |
| Existing JDBC ecosystem (Spring JdbcTemplate, connection pools) | ❌ | ✅ |
| Explain plan / query analysis | ❌ | ✅ (`EXPLAIN ANALYZE FOR`) |

---

## Anti-Patterns

| ❌ Don't | ✅ Do |
|---|---|
| `readMultiple(template, Integer.MAX_VALUE)` | Set a finite limit, or use `SpaceIterator` |
| String-concatenated SQL (`"amount > " + val`) | Always use parameterized queries (`?`) |
| Full table scan JOIN without routing | Add routing condition or use `DYNAMIC_FILTER` hint |
| Return JDBC `ResultSet` outside try-with-resources | Always close `ResultSet`, `Statement`, `Connection` |
