# JMeter Performance Test Setup for API Groups

## Overview
This guide provides a step-by-step approach to creating a **performance test in JMeter** for **5 groups of APIs**, where each group has multiple APIs with unique data (e.g., endpoint, headers, payload, requests per hour, etc.).

---

## Step 1: Prepare CSV Files
Create 5 separate CSV files, one for each group (e.g., `Group1.csv`, `Group2.csv`, ..., `Group5.csv`).

### Example CSV File (`Group1.csv`)
```csv
api_number,endpoint,headers,auth_token,payload,expected_response,content_type,correlation_id,end_type,end_iteration,requests_per_hour
1,/api/v1/resource1,"{""key"":""value""}",token1,{...},200,application/json,123,type1,iteration1,100
2,/api/v1/resource2,"{""key"":""value""}",token2,{...},201,application/xml,456,type2,iteration2,200
```

---

## Step 2: Create a JMeter Test Plan
1. Open **JMeter** and create a new **Test Plan**.
2. Add **5 Thread Groups** (one for each API group):
   - **Right-click** on the Test Plan → **Add** → **Threads (Users)** → **Thread Group**.
   - Name each Thread Group (`Group 1`, `Group 2`, etc.).

---

## Step 3: Configure Thread Groups
For each **Thread Group**:
- **Number of Threads (Users)**: Set to the number of APIs in the group (e.g., `50` threads for `50` APIs).
- **Ramp-Up Period**: Set to `3600` seconds (distributes the load evenly over `1 hour`).
- **Loop Count**: Set to `1` (each API runs once per iteration).

---

## Step 4: Add CSV Data Set Config
For each **Thread Group**:
1. **Right-click** on the Thread Group → **Add** → **Config Element** → **CSV Data Set Config**.
2. Configure the CSV Data Set Config:
   - **Filename**: Select the corresponding CSV file (`Group1.csv`, `Group2.csv`, etc.).
   - **Variable Names**: Enter column names:
     ```
     api_number,endpoint,headers,auth_token,payload,expected_response,content_type,correlation_id,end_type,end_iteration,requests_per_hour
     ```
   - **Delimiter**: `,` (comma).
   - **Ignore first line**: `true`.
   - **Recycle on EOF**: `false`.
   - **Stop thread on EOF**: `false`.
   - **Sharing mode**: `All threads`.

---

## Step 5: Add JSR223 PreProcessor for Throughput Calculation
For each **Thread Group**:
1. **Right-click** on the Thread Group → **Add** → **Pre Processors** → **JSR223 PreProcessor**.
2. Use the following **Groovy script**:

```groovy
// Read requests_per_hour from CSV
int requestsPerHour = Integer.parseInt(vars.get("requests_per_hour"));

// Calculate throughput (requests per minute)
double throughput = requestsPerHour / 60;

// Store throughput in a JMeter variable
vars.put("throughput", throughput.toString());

// Log the calculated throughput for debugging
log.info("API " + vars.get("api_number") + " Throughput: " + throughput + " requests per minute");
```

---

## Step 6: Add Constant Throughput Timer
For each **Thread Group**:
- **Right-click** on the Thread Group → **Add** → **Timer** → **Constant Throughput Timer**.
- Set **Target Throughput** to `${throughput}`.

---

## Step 7: Add HTTP Request
For each **Thread Group**:
1. **Right-click** on the Thread Group → **Add** → **Sampler** → **HTTP Request**.
2. Configure the **HTTP Request**:
   - **Server Name or IP**: Enter the base URL of your API.
   - **Path**: `${endpoint}` (from CSV file).
   - **Body Data**: `${payload}` (for POST/PUT requests).

---

## Step 8: Add HTTP Header Manager
For each **Thread Group**:
1. **Right-click** on the Thread Group → **Add** → **Config Element** → **HTTP Header Manager**.
2. Use a **JSR223 PreProcessor** to dynamically set headers.

### Example Groovy Script for Headers:
```groovy
import groovy.json.JsonSlurper

// Read headers from CSV
String headersJson = vars.get("headers");

// Parse headers JSON
def headers = new JsonSlurper().parseText(headersJson);

// Clear existing headers
sampler.getHeaderManager().clear();

// Add headers to HTTP Header Manager
headers.each { key, value ->
    sampler.getHeaderManager().add(new org.apache.jmeter.protocol.http.control.Header(key, value));
}

// Log headers for debugging
log.info("Headers for API " + vars.get("api_number") + ": " + headers);
```

---

## Step 9: Add Listeners
Add listeners to monitor and analyze test results:
- **Right-click** on the Test Plan → **Add** → **Listener** → **View Results Tree**.
- **Right-click** on the Test Plan → **Add** → **Listener** → **Summary Report**.

---

## Step 10: Run the Test
- Save your test plan.
- Click the **Start** button (green play icon) to run the test.
- Monitor results in the listeners.

---

## Final JMeter Structure
```
Test Plan
  ├── Thread Group 1 (Group 1)
  │     ├── CSV Data Set Config (Group1.csv)
  │     ├── JSR223 PreProcessor (Groovy script for throughput calculation)
  │     ├── Loop Controller (loop count = 1)
  │     │     ├── HTTP Request
  │     │     │     ├── HTTP Header Manager
  │     │     │     ├── Response Assertion
  │     │     │     └── Constant Throughput Timer (Target Throughput = ${throughput})
  │     │     └── View Results Tree
  ├── Thread Group 2 (Group 2)
  │     ├── CSV Data Set Config (Group2.csv)
  │     ├── JSR223 PreProcessor (Groovy script for throughput calculation)
  │     ├── Loop Controller (loop count = 1)
  │     │     ├── HTTP Request
  │     │     │     ├── HTTP Header Manager
  │     │     │     ├── Response Assertion
  │     │     │     └── Constant Throughput Timer (Target Throughput = ${throughput})
  │     │     └── View Results Tree
  └── ...
```

---

## Key Points
✅ **Each Thread Group** corresponds to one API group.
✅ **CSV Data Set Config** reads API data dynamically.
✅ **JSR223 PreProcessor** calculates throughput.
✅ **Constant Throughput Timer** distributes requests evenly.

By following this guide, you can create a **performance test in JMeter** for **5 groups of APIs**, ensuring each API runs at its specified rate and distributes its requests evenly. 🚀

