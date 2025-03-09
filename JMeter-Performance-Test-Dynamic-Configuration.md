# JMeter Performance Test - Dynamic Configuration

## Overview
This guide provides an end-to-end approach for creating a **JMeter script** that:
- Reads **dynamic headers** for each API.
- Loads **payloads from separate files** dynamically.
- Allocates **unique throughput** based on API requests per hour.
- Dynamically assigns **threads per API** for realistic load distribution.
- Configures **Thread Groups for parallel execution**.

---

## Step 1: Prepare Folder Structure
Organize the data directory as follows:
```
data/
  ├── grp1/
  │     ├── api1/
  │     │     ├── headers.json
  │     │     ├── payload.json
  │     ├── api2/
  │     │     ├── headers.json
  │     │     ├── payload.json
  │     └── ...
  ├── grp2/
  │     ├── api1/
  │     │     ├── headers.json
  │     │     ├── payload.json
  │     ├── api2/
  │     │     ├── headers.json
  │     │     ├── payload.json
  │     └── ...
```
- **Each API has its own folder (`api_number`).**
- **Each API folder contains:**
  - `headers.json` → Stores headers for the API.
  - `payload.json` → Stores payload for the API.

---

## Step 2: Prepare CSV Files
Create CSV files for each group (e.g., `Group1.csv`, `Group2.csv`).

### Example CSV (`Group1.csv`):
```csv
api_number,endpoint,auth_token,api_folder,expected_response,content_type,correlation_id,end_type,end_iteration,requests_per_hour,method_type
1,/api/v1/resource1,token1,data/grp1/api1,200,application/json,123,type1,iteration1,100,POST
2,/api/v1/resource2,token2,data/grp1/api2,201,application/xml,456,type2,iteration2,200,GET
```
- **`api_folder`** points to the API-specific folder containing `headers.json` and `payload.json`.

---

## Step 3: Create a JMeter Test Plan
- Open **JMeter**.
- Create a **new Test Plan**.
- Add **5 Thread Groups** (one for each API group):
  - **Right-click on Test Plan** → **Add** → **Threads (Users)** → **Thread Group**.

---

## Step 4: Configure Thread Groups
For each **Thread Group**:
- **Number of Threads (Users)**: **Calculated dynamically** in Step 6.
- **Ramp-Up Period**: Set to `3600` seconds (distributes load over 1 hour).
- **Loop Count**: `1` (each API runs once per iteration).

---

## Step 5: Add CSV Data Set Config
For each **Thread Group**:
1. **Right-click on the Thread Group** → **Add** → **Config Element** → **CSV Data Set Config**.
2. Configure **CSV Data Set Config**:
   - **Filename**: Select the CSV file (`Group1.csv`, `Group2.csv`, etc.).
   - **Variable Names**: Enter column names:
     ```
     api_number,endpoint,auth_token,api_folder,expected_response,content_type,correlation_id,end_type,end_iteration,requests_per_hour,method_type
     ```
   - **Ignore first line**: `true`.
   - **Recycle on EOF**: `false`.
   - **Sharing mode**: `All threads`.

---

## Step 6: JSR223 PreProcessor - Thread Group Configuration
For each **Thread Group**:

```groovy
// Initialize total requests per hour
int totalRequestsPerHour = 0;

// Loop through all APIs in the group
while (vars.get("requests_per_hour") != null) {
    totalRequestsPerHour += Integer.parseInt(vars.get("requests_per_hour"));
    ctx.getCurrentSampler().getProperty("CSVDataSet").next();
}

// Calculate number of threads and ramp-up period
int testDuration = 3600;
int multiplier = 10;
int numberOfThreads = (int) Math.ceil(totalRequestsPerHour / testDuration * multiplier);
int rampUpPeriod = testDuration;

// Store values in JMeter variables
vars.put("numberOfThreads", numberOfThreads.toString());
vars.put("rampUpPeriod", rampUpPeriod.toString());

// Log calculated values
log.info("Threads: " + numberOfThreads);
log.info("Ramp-Up Period: " + rampUpPeriod);
```

---

## Step 7: JSR223 PreProcessor - Throughput Calculation
For each **Thread Group**:

```groovy
// Read requests_per_hour from CSV
int requestsPerHour = Integer.parseInt(vars.get("requests_per_hour"));

// Calculate throughput (requests per minute)
double throughput = requestsPerHour / 60;

// Store in a JMeter variable
vars.put("throughput", throughput.toString());

// Log for debugging
log.info("API " + vars.get("api_number") + " Throughput: " + throughput + " requests per minute");
```

---

## Step 8: JSR223 PreProcessor - Read Headers and Payload
```groovy
import groovy.json.JsonSlurper

// Read API folder path from CSV
String apiFolder = vars.get("api_folder");

// Read headers.json if it exists
File headersFile = new File(apiFolder + "/headers.json");
if (headersFile.exists()) {
    String headersJson = headersFile.text;
    vars.put("headers", headersJson);
    log.info("Headers for API " + vars.get("api_number") + ": " + headersJson);
} else {
    log.warn("Headers file not found for API " + vars.get("api_number"));
}

// Read payload.json if method is POST/PUT
if (vars.get("method_type") == "POST" || vars.get("method_type") == "PUT") {
    File payloadFile = new File(apiFolder + "/payload.json");
    if (payloadFile.exists()) {
        String payloadJson = payloadFile.text;
        vars.put("payload", payloadJson);
        log.info("Payload for API " + vars.get("api_number") + ": " + payloadJson);
    } else {
        log.warn("Payload file not found for API " + vars.get("api_number"));
    }
}
```

---

## Step 9: HTTP Request Configuration
- **Path**: `${endpoint}`
- **Body Data**: `${payload}` (for POST/PUT requests)

---

## Step 10: Add Listeners
To monitor test results:
- **View Results Tree**
- **Summary Report**

---

## Final JMeter Structure
```
Test Plan
  ├── Thread Group 1
  │     ├── CSV Data Set Config
  │     ├── JSR223 PreProcessor (Thread Group configuration)
  │     ├── Loop Controller
  │     │     ├── JSR223 PreProcessor (Throughput calculation)
  │     │     ├── JSR223 PreProcessor (Read headers and payload)
  │     │     ├── HTTP Request
  │     │     ├── Response Assertion
  │     │     ├── Constant Throughput Timer
  │     │     ├── View Results Tree
```

---

## Key Takeaways
✅ **Dynamic Headers** from JSON
✅ **Dynamic Payloads** for POST/PUT requests
✅ **Dynamic Throughput Calculation**
✅ **Dynamic Thread Allocation**

By following this guide, you ensure efficient, scalable, and maintainable JMeter test execution. 🚀

