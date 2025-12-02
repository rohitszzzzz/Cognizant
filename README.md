# Employee Asset Management — Backend Blueprint (Git README)

**Purpose:** Single-day, microservices-based Spring Boot backend using JPA (H2 for quick setup). Three services: User, Asset, Issue. Optional API Gateway and Eureka. No security.

---

## Repo layout (suggested)

```
employee-asset-backend/
├── gateway/                     # Spring Cloud Gateway (optional)
├── eureka-server/               # Eureka Service Registry (optional)
├── user-service/                # Dev 1 (8081)
├── asset-service/               # Dev 2 (8082)
├── issue-service/               # Dev 3 (8083)
└── README.md                    # this file
```

---

## Quick run strategy

* Each developer clones the repo and works inside their service folder.
* Use H2 (in-memory) for each service so no DB setup is needed. If you prefer MySQL, change application.properties.
* Use `mvn spring-boot:run` for each service (or run from IDE).
* If using Gateway/Eureka, start Eureka → services → gateway.

---

# Service: USER SERVICE (Dev 1)

**PORT:** `8081`

### Tables (JPA Entities)

**Employee**

* `empId` (Integer, Primary Key)
* `empName` (String)
* `empPassword` (String)
* `empEmail` (String)
* `dob` (LocalDate)
* `empAddress` (String)
* `role` (String) // optional: 'USER' or 'DEPT'

> JPA: `EmployeeRepository extends JpaRepository<Employee, Integer>`

### Endpoints

* `POST /login`
  *Request:* `{ "empId": 101, "password": "pass" }` or `{ "empEmail": "a@b.com", "password":"pass" }`
  *Response:* `200 OK` `{ "status":"OK", "empId":101, "empName":"Rohit" }`

* `GET /employees/{id}`
  *Response:* `Employee` JSON (empName, empEmail, dob, empAddress)

* `GET /employees/search?name={name}`
  *Response:* list of employees matching the name (used by Department Login screen)

### Sample SQL (data.sql for H2)

```sql
INSERT INTO employee(emp_id, emp_name, emp_password, emp_email, dob, emp_address, role) VALUES
(101, 'Rohit', 'pass', 'rohit@example.com', '1999-01-01', 'Chennai', 'USER');
```

---

# Service: ASSET SERVICE (Dev 2)

**PORT:** `8082`

### Tables (JPA Entities)

**Asset**

* `assetId` (Integer, PK)
* `assetName` (String)
* `assetType` (String) // Laptop, Desktop, Monitor, Keyboard, Mouse, Headphones, IP Phone

**AssetAlloc**

* `assetAllocId` (Integer, PK)
* `assetId` (Integer) // foreign key to Asset (store id only)
* `allocatedTo` (Integer) // empId
* `allocationStartDate` (LocalDate)
* `allocationEndDate` (LocalDate)
* `status` (String) // Assigned, Pending, Issues

> JPA: `AssetRepository extends JpaRepository<Asset, Integer>`
> JPA: `AssetAllocRepository extends JpaRepository<AssetAlloc, Integer>`

### Endpoints

* `GET /assets/user/{empId}`

  * Returns list of assets allocated to `empId` with allocation info

* `GET /assets/{assetId}`

  * Returns asset + latest allocation info

* `POST /assets/{assetId}/release`

  * Request: `{ "empId": 101 }`
  * Action: remove or update allocation (set allocation_end_date = today)
  * Response: updated list or success message

* `POST /assets/{assetId}/replace`

  * Request: `{ "empId":101, "newAssetId": 202 }`
  * Action: create new allocation record for newAssetId and set old allocation end date

* `GET /assets/search?user=&type=&status=`

  * Used by Department screen to show table: Asset Name, Asset Status, Asset Type, Assigned To, Allocated Since

### Sample SQL (data.sql)

```sql
INSERT INTO asset(asset_id, asset_name, asset_type) VALUES
(201, 'Dell XPS 13', 'Laptop'), (202, 'HP Monitor', 'Monitor');

INSERT INTO asset_alloc(asset_alloc_id, asset_id, allocated_to, allocation_start_date, allocation_end_date, status) VALUES
(1, 201, 101, '2024-10-01', NULL, 'Assigned');
```

---

# Service: ISSUE SERVICE (Dev 3)

**PORT:** `8083`

### Tables (JPA Entities)

**AssetManagement**

* `assetMgmtId` (Integer, PK)
* `assetId` (Integer)
* `issueType` (String) // 'issue' or 'request'
* `raisedOn` (LocalDateTime or LocalDate)
* `raisedBy` (Integer) // empId
* `issueDesc` (String)
* `status` (String) // Pending/Resolved/Assigned

> JPA: `IssueRepository extends JpaRepository<AssetManagement, Integer>`

### Endpoints

* `GET /issues/user/{empId}`

  * Returns list of issues and requests raised by or related to the user. FE uses it for UL top-right pane (pending/issue counts)

* `GET /issues/asset/{assetId}`

  * Returns issues for the asset

* `POST /issues`

  * Request body: `{ "assetId":201, "issueType":"issue", "raisedBy":101, "issueDesc":"Keyboard not working" }`
  * Inserts a new record

* `GET /issues/search?status=&user=&type=`

  * For Department to filter issues (optional join with Asset service by FE)

### Sample SQL

```sql
INSERT INTO asset_management(asset_mgmt_id, asset_id, issue_type, raised_on, raised_by, issue_desc, status) VALUES
(1, 201, 'issue', '2025-01-10', 101, 'Battery not charging', 'Pending');
```

---

# API Gateway (optional)

If you add Spring Cloud Gateway, put a minimal `application.yml` with routes:

```yaml
spring:
  cloud:
    gateway:
      routes:
        - id: user-service
          uri: http://localhost:8081
          predicates:
            - Path=/user/**
        - id: asset-service
          uri: http://localhost:8082
          predicates:
            - Path=/asset/**
        - id: issue-service
          uri: http://localhost:8083
          predicates:
            - Path=/issue/**
```

**FE calls through gateway**: e.g. `GET http://<gateway-host>/user/employees/101`

---

# Eureka (optional)

If you use Eureka, set each service with `spring.application.name` and `eureka.client` config pointing to Eureka server. Gateway can then route by service name.

---

# Contracts (Example JSONs)

**Employee JSON**

```json
{
  "empId": 101,
  "empName": "Rohit",
  "empEmail": "rohit@example.com",
  "dob": "1999-01-01",
  "empAddress": "Chennai"
}
```

**AssetAlloc JSON (as returned by asset service)**

```json
{
  "assetAllocId": 1,
  "assetId": 201,
  "assetName": "Dell XPS 13",
  "assetType": "Laptop",
  "allocatedTo": 101,
  "allocationStartDate": "2024-10-01",
  "allocationEndDate": null,
  "status": "Assigned"
}
```

**Issue JSON**

```json
{
  "assetMgmtId": 1,
  "assetId": 201,
  "issueType": "issue",
  "raisedOn": "2025-01-10",
  "raisedBy": 101,
  "issueDesc": "Battery not charging",
  "status": "Pending"
}
```

---

# Integration Notes / Best Practices for Today

* **No DB sharing**. Each service owns its tables.

* **FE calls gateway** routes (or directly to services if no gateway).

* If you need employee names in the Issue or Asset responses, **either**:

  * FE makes an extra call to User Service to fetch names (recommended), **or**
  * Store `empName` in the other services when creating allocations/issues (simpler, duplicates data).

* Keep DTOs minimal. Use simple POJOs and JPA entities.

* Use `data.sql` for sample data that loads at startup.

---

# Minimal spring-boot dependencies (pom.xml snippets)

```xml
<!-- Spring Boot Starter Web, JPA, H2 -->
<dependency>
  <groupId>org.springframework.boot</groupId>
  <artifactId>spring-boot-starter-web</artifactId>
</dependency>
<dependency>
  <groupId>org.springframework.boot</groupId>
  <artifactId>spring-boot-starter-data-jpa</artifactId>
</dependency>
<dependency>
  <groupId>com.h2database</groupId>
  <artifactId>h2</artifactId>
  <scope>runtime</scope>
</dependency>

<!-- Optional: Eureka Discovery, Gateway -->
<dependency>
  <groupId>org.springframework.cloud</groupId>
  <artifactId>spring-cloud-starter-netflix-eureka-client</artifactId>
</dependency>
<dependency>
  <groupId>org.springframework.cloud</groupId>
  <artifactId>spring-cloud-starter-gateway</artifactId>
</dependency>
```

---

# Final checklist for each developer (to finish today)

**All devs:**

* Create Spring Boot project with Web + JPA + H2.
* Create Entity classes and JPA repository.
* Create Controller with the endpoints listed above.
* Add `data.sql` to pre-populate a few rows.
* Run and test with Postman.

**Gateway/Eureka (optional):**
