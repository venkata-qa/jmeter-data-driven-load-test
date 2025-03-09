# JMeter Performance Test - Updated Structure

## Overview
This guide details the updated **JMeter test plan structure** with enhanced **dynamic configurations**, including:
- **Thread Group Configuration PreProcessor** for calculating the number of threads dynamically.
- **API-Level PreProcessors** for throughput calculation, dynamic headers, and conditional payload loading.
- **Efficient Execution** using `Loop Controller`, `If Controller`, and `JSR223 PreProcessors`.

---

## Updated JMeter Structure
```
Test Plan
  ├── Thread Group 1 (Group 1)
  │     ├── CSV Data Set Config (Group1.csv)
  │     ├── JSR223 PreProcessor (Thread Group configuration)  // At Thread Group level
  │     ├── Loop Controller (loop count = number of APIs in Group 1)
  │     │     ├── JSR223 PreProcessor (Throughput calculation)  // At API level
  │     │     ├── JSR223 PreProcessor (Dynamic headers)  // At API level
  │     │     ├── If Controller (Condition: ${__groovy(vars.get("method_type") == "POST" || vars.get("method_type") == "PUT")})
  │     │     │     └── JSR223 PreProcessor (Read payload file)  // Only for POST/PUT
  │     │     ├── HTTP Request
  │     │     │     ├── HTTP Header Manager
  │     │     │     ├── Response Assertion
  │     │     │     └── Constant Throughput Timer (Target Throughput = ${throughput})
  │     │     └── View Results Tree
  ├── Thread Group 2 (Group 2)
  │     ├── CSV Data Set Config (Group2.csv)
  │     ├── JSR223 PreProcessor (Thread Group configuration)  // At Thread Group level
  │     ├── Loop Controller (loop count = number of APIs in Group 2)
  │     │     ├── JSR223 PreProcessor (Throughput calculation)  // At API level
  │     │     ├── JSR223 PreProcessor (Dynamic headers)  // At API level
  │     │     ├── If Controller (Condition: ${__groovy(vars.get("method_type") == "POST" || vars.get("method_type") == "PUT")})
  │     │     │     └── JSR223 PreProcessor (Read payload file)  // Only for POST/PUT
  │     │     ├── HTTP Request
  │     │     │     ├── HTTP Header Manager
  │     │     │     ├── Response Assertion
  │     │     │     └── Constant Throughput Timer (Target Throughput = ${throughput})
  │     │     └── View Results Tree
  └── ...
```

---

## Step-by-Step Explanation

### 1. Thread Group Configuration PreProcessor
**Location:** At the Thread Group level.

**Purpose:**
- Calculates the **Number of Threads** and **Ramp-Up Period** for the Thread Group.

#### **Script:**
```groovy
// Initialize total requests per hour
int totalRequestsPerHour = 0;

// Loop through all APIs in the group (assuming CSV has multiple rows)
while (vars.get("requests_per_hour") != null) {
    totalRequestsPerHour += Integer.parseInt(vars.get("requests_per_hour"));
    ctx.getCurrentSampler().getProperty("CSVDataSet").next();
}

// Calculate number of threads and ramp-up period
int testDuration = 3600; // 1 hour
int multiplier = 10; // Adjustable multiplier
int numberOfThreads = (int) Math.ceil(totalRequestsPerHour / testDuration * multiplier);
int rampUpPeriod = testDuration;

// Store values in JMeter variables
vars.put("numberOfThreads", numberOfThreads.toString());
vars.put("rampUpPeriod", rampUpPeriod.toString());

// Log calculated values
log.info("Number of Threads: " + numberOfThreads);
log.info("Ramp-Up Period: " + rampUpPeriod);
```

---

### 2. API-Level PreProcessors
These are inside the **Loop Controller** and run for **each API** in the group.

#### a. **Throughput Calculation PreProcessor**
**Purpose:** Calculate the throughput (requests per minute) for each API.

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

#### b. **Dynamic Headers PreProcessor**
**Purpose:** Dynamically set headers for each API.

```groovy
import groovy.json.JsonSlurper;

// Read headers from CSV
String headersJson = vars.get("headers");

// Parse headers JSON
def headers = new JsonSlurper().parseText(headersJson);

// Clear existing headers
sampler.getHeaderManager().clear();

// Add headers dynamically
headers.each { key, value ->
    sampler.getHeaderManager().add(new org.apache.jmeter.protocol.http.control.Header(key, value));
}

// Log headers
log.info("Headers for API " + vars.get("api_number") + ": " + headers);
```

#### c. **Read Payload File PreProcessor**
**Purpose:** Read the payload file **only for POST/PUT requests**.

**If Controller Condition:**
```groovy
${__groovy(vars.get("method_type") == "POST" || vars.get("method_type") == "PUT")}
```

**Script:**
```groovy
// Read payload file path from CSV
String payloadFilePath = vars.get("payload_file");

// Read payload file content
String payloadContent = new File(payloadFilePath).text;

// Store in a JMeter variable
vars.put("payload", payloadContent);

// Log payload content
log.info("Payload for API " + vars.get("api_number") + ": " + payloadContent);
```

---

### 3. HTTP Request Configuration
- Use `${endpoint}` variable for API path.
- Use `${payload}` for Body Data (only for POST/PUT requests).

---

### 4. Add Listeners
To monitor results, add:
- **View Results Tree**
- **Summary Report**

---

## Key Points
✅ **Thread Group Configuration PreProcessor:** Runs **once per Thread Group**.

✅ **API-Level PreProcessors:** Execute **for each API**.

✅ **Payload File PreProcessor:** Runs **only for POST/PUT requests**.

✅ **Dynamic Headers & Throughput:** Applied per API.

---

## Benefits of This Approach
- **Scalability:** Each API runs with its unique configuration.
- **Flexibility:** Handles different request types dynamically.
- **Efficiency:** Only reads payloads when necessary.

By implementing this **optimized JMeter structure**, you ensure **dynamic execution, efficient throughput distribution, and proper API configuration** for robust performance testing. 🚀

