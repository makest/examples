# Java @SpringBootTest testing against DBs with init scripts

Create an application-test.yml (or .properties) under src/test/resources that overrides your custom properties:
```yaml
spring:
  oracle-datasource:
    jdbc-url: jdbc:h2:mem:oracle_db;MODE=Oracle;DB_CLOSE_DELAY=-1;DATABASE_TO_UPPER=false
    username: sa
    password: ""
    driver-class-name: org.h2.Driver
    # Hikari-specific properties if your config maps them
    maximum-pool-size: 2
  sybase-datasource:
    jdbc-url: jdbc:h2:mem:sybase_db;MODE=MSSQLServer;DB_CLOSE_DELAY=-1;DATABASE_TO_UPPER=false
    username: sa
    password: ""
    driver-class-name: org.h2.Driver
    maximum-pool-size: 2
```
H2 supports compatibility modes — MODE=Oracle and MODE=MSSQLServer (closest to Sybase ASE) — that emulate vendor-specific SQL syntax reasonably well.
Key URL flags:

-    MODE=Oracle / MODE=MSSQLServer — dialect emulation (MSSQLServer is the closest H2 has for Sybase T-SQL).
-    DB_CLOSE_DELAY=-1 — keeps the in-memory DB alive across connections in the pool.
-    DATABASE_TO_UPPER=false — preserves case in identifiers (helpful if your real schema uses mixed/lower case).

Add scripts:

- `src/test/resources/init-oracle.sql`

```sql
CREATE TABLE customers (
    id NUMBER PRIMARY KEY,
    name VARCHAR2(100) NOT NULL
);

INSERT INTO customers (id, name) VALUES (1, 'Alice');
INSERT INTO customers (id, name) VALUES (2, 'Bob');
```

- `src/test/resources/init-sybase.sql`

```sql
CREATE TABLE orders (
    id INT PRIMARY KEY,
    customer_id INT NOT NULL,
    amount DECIMAL(10,2)
);

INSERT INTO orders (id, customer_id, amount) VALUES (100, 1, 49.99);
INSERT INTO orders (id, customer_id, amount) VALUES (101, 2, 19.50);
```

> H2 in Oracle mode accepts NUMBER/VARCHAR2. In MSSQLServer mode it accepts INT/DECIMAL natively.

Add dependency for H2:
```xml
<dependency>
    <groupId>com.h2database</groupId>
    <artifactId>h2</artifactId>
    <scope>test</scope>
</dependency>
```

Write SpringBootTest annotated test:
```java
@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
@ActiveProfiles("test")
@SqlGroup({
    @Sql(
        scripts = "/init-oracle.sql",
        config = @SqlConfig(dataSource = "oracleDataSource", transactionMode = SqlConfig.TransactionMode.ISOLATED),
        executionPhase = Sql.ExecutionPhase.BEFORE_TEST_CLASS
    ),
    @Sql(
        scripts = "/init-sybase.sql",
        config = @SqlConfig(dataSource = "sybaseDataSource", transactionMode = SqlConfig.TransactionMode.ISOLATED),
        executionPhase = Sql.ExecutionPhase.BEFORE_TEST_CLASS
    )
})
class CustomerOrderControllerIT {

    @Autowired
    private TestRestTemplate restTemplate;

    @Test
    void shouldReturnCustomerWithOrders() {
        ResponseEntity<CustomerOrdersResponse> response =
            restTemplate.getForEntity("/api/customers/1/orders", CustomerOrdersResponse.class);

        assertThat(response.getStatusCode()).isEqualTo(HttpStatus.OK);
        assertThat(response.getBody().customerName()).isEqualTo("Alice");
        assertThat(response.getBody().orders()).hasSize(1);
    }

    @Test
    void shouldHaveBothDataSourcesConfigured(@Autowired @Qualifier("oracleDataSource") DataSource oracle,
                                             @Autowired @Qualifier("sybaseDataSource") DataSource sybase) throws SQLException {
        try (Connection c = oracle.getConnection()) {
            assertThat(c.getMetaData().getURL()).contains("oracle_db");
        }
        try (Connection c = sybase.getConnection()) {
            assertThat(c.getMetaData().getURL()).contains("sybase_db");
        }
    }
}
```

If your DatabaseConfig sets a Hikari connection-test-query like SELECT 1 FROM DUAL (Oracle) or SELECT 1 (Sybase), this will fail against H2 in some cases:

    `SELECT 1 FROM DUAL` works in Oracle mode (H2 has a DUAL table).
    Plain `SELECT 1` works in MSSQLServer mode.

If your config hardcodes a query that doesn't work in H2, override it in application-test.yml, or better — rely on JDBC4 isValid() (Hikari's default when no test query is set).


## Testcontainers approach

If Docker is available we can use testcontainers to run DBs:

```java
@SpringBootTest
@Testcontainers
class MyControllerIntegrationTest {
    @Container static OracleContainer oracle = new OracleContainer("gvenzl/oracle-free:slim");
    @Container static GenericContainer<?> sybase = new GenericContainer<>("datagrip/sybase:16.0").withExposedPorts(5000);

    @DynamicPropertySource
    static void props(DynamicPropertyRegistry r) {
        r.add("spring.oracle-datasource.jdbc-url", oracle::getJdbcUrl);
        r.add("spring.oracle-datasource.username", oracle::getUsername);
        // ...
    }
}
```

> Init scripts can be applied via .withInitScript() on the container or via @Sql.

