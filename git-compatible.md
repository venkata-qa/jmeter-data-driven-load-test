https://github.com/venkata-qa/jmeter-data-driven-load-test/blob/main/git-compatible.md# 🚀 Complete JMeter Test Plan Structure with Script Placement

## 📌 JMeter Test Plan Structure

```
Test Plan
 ├── Thread Group (API Load Test)
 │   ├── CSV Data Set Config (Reads API Data)  ✅
 │   ├── HTTP Request Sampler (Executes API Calls)
 │   │   ├── JSR223 PreProcessor (Calculate Threads & Distribution)  ✅
 │   │   ├── JSR223 PreProcessor (Load Headers & Payload) ✅
 │   │   ├── HTTP Header Manager (For Common Headers) ✅
 │   │   ├── Constant Throughput Timer (Ensure Correct RPS) ✅
 │   │   ├── JSR223 PostProcessor (Track & Validate Requests) ✅
 │   ├── View Results Tree (For Debugging)
 │   ├── Summary Report (For Test Metrics)
```

✅ Below is a detailed explanation of each component with correct script placement.

---

## 🔹 1️⃣ CSV Data Set Config (Loads API Data)

📍 **Position:**
✅ Inside **Thread Group** (Before any samplers run).

📍 **Configuration:**

| Field         | Value                           |
|--------------|---------------------------------|
| Filename     | `config/api_data.csv`          |
| Variable Names | `api_number, api_method, end_point, api_folder, requests_per_hour, expected_response` |
| Sharing Mode | `All Threads` |

🔹 Ensures that each thread gets API data from the CSV file.

---

## 🔹 2️⃣ JSR223 PreProcessor - Calculate Threads & Distribution

📍 **Position:**
✅ Inside **HTTP Request Sampler** (Executes before the request runs).

📍 **Script:** `scripts/thread_calculator.groovy`

```groovy
// Calculate Threads per API & Ensure Even Distribution
int requestsPerHour = vars.get("requests_per_hour").toInteger()
int testDuration = 3600
int rampUpTime = 300
int rampDownTime = 300
int steadyStateTime = testDuration - (rampUpTime + rampDownTime)
int threadsRequired = (int) Math.ceil(requestsPerHour / (double) testDuration)
double exactDelayBetweenRequests = 3600.0 / requestsPerHour

vars.put("threadsRequired", String.valueOf(threadsRequired))
vars.put("rampUpTime", String.valueOf(rampUpTime))
vars.put("rampDownTime", String.valueOf(rampDownTime))
vars.put("steadyStateTime", String.valueOf(steadyStateTime))
vars.put("exactDelayBetweenRequests", String.valueOf(exactDelayBetweenRequests))

log.info("✅ API " + vars.get("api_number") + " | Threads: " + threadsRequired)
```

---

## 🔹 3️⃣ JSR223 PreProcessor - Load Headers & Payload

📍 **Position:**
✅ Inside **HTTP Request Sampler** (Executes before sending request).

📍 **Script:** `scripts/headers_payload_loader.groovy`

```groovy
import groovy.json.JsonSlurper
import org.apache.jmeter.protocol.http.control.Header
import org.apache.jmeter.protocol.http.control.HeaderManager

String apiFolder = vars.get("api_folder")

// Check if apiFolder is null or empty
if (apiFolder == null || apiFolder.trim().isEmpty()) {
    log.error("❌ Error: 'api_folder' variable is not set or is empty!")
    throw new IllegalArgumentException("api_folder variable is missing")
}

File folder = new File(apiFolder).getCanonicalFile()
File headersFile = new File(folder, "headers.json")

// Ensure sampler is not null and has a HeaderManager
if (sampler == null) {
    log.error("❌ Error: 'sampler' is null. Ensure this script runs inside an HTTP Request sampler.")
    throw new IllegalStateException("sampler is null")
}

def headerManager = sampler.getHeaderManager()

// If headerManager is null, initialize a new one
if (headerManager == null) {
    log.warn("⚠️ Warning: 'HeaderManager' was null. Creating a new HeaderManager.")
    headerManager = new HeaderManager()
    sampler.setHeaderManager(headerManager)
}

// Clear existing headers
headerManager.removeAll()

// Load API-Specific Headers
if (headersFile.exists()) {
    def headersMap = new JsonSlurper().parseText(headersFile.text)
    headersMap.each { key, value ->
        headerManager.add(new Header(key, value.toString()))
        log.info("✅ Added API-Specific Header: " + key + " = " + value)
    }
} else {
    log.warn("⚠️ Headers file not found: " + headersFile.absolutePath)
}

// Load Payload (For POST/PUT Requests)
String methodType = vars.get("api_method")
if (methodType?.equalsIgnoreCase("POST") || methodType?.equalsIgnoreCase("PUT")) {
    File payloadFile = new File(folder, "payload.json")
    if (payloadFile.exists()) {
        vars.put("payload", payloadFile.text)
        log.info("✅ Loaded Payload for API: " + vars.get("api_number"))
    } else {
        log.warn("⚠️ Payload file not found: " + payloadFile.absolutePath)
    }
}

```

🔹 Ensures each API has its correct headers & payload before sending a request.

---

## 🔹 4️⃣ HTTP Header Manager - Set Common Headers

📍 **Position:**
✅ Directly inside **HTTP Request Sampler**

📍 **Common Headers Example:**

| Header Name       | Value                  |
|------------------|------------------------|
| Content-Type     | `application/json`      |
| Authorization    | `Bearer ${auth_token}`  |
| X-Correlation-ID | `${correlation_id}`     |

🔹 Ensures all APIs get these common headers. API-specific headers from `headers.json` will be merged.

---

## 🔹 5️⃣ Constant Throughput Timer - Control Request Rate

📍 **Position:**
✅ Inside **HTTP Request Sampler**

📍 **Configuration:**

| Field                     | Value |
|---------------------------|---------------------------------|
| Target Throughput (RPM)   | `${requests_per_hour}/60`      |
| Threads to Schedule       | `${threadsRequired}`           |
| Precision                 | `High`                         |

🔹 Ensures exact `requests_per_hour` are met evenly.

---

## 🔹 6️⃣ JSR223 PostProcessor - Track & Validate Requests

📍 **Position:**
✅ Inside **HTTP Request Sampler** (Runs after the request).

📍 **Script:** `scripts/request_tracker.groovy`

```groovy
import groovy.json.JsonSlurper

// Extract API Response
String responseBody = prev.getResponseDataAsString()
int responseCode = prev.getResponseCode().toInteger()
String expectedResponse = vars.get("expected_response")

log.info("📌 API " + vars.get("api_number") + " | Response Code: " + responseCode)

// Track Processed Requests
String apiNum = vars.get("api_number")
int processedRequests = vars.get("processed_requests_" + apiNum)?.toInteger() ?: 0
processedRequests += 1
vars.put("processed_requests_" + apiNum, processedRequests.toString())

// Validate API Response
if (expectedResponse && responseCode.toString() != expectedResponse) {
    log.error("❌ Expected: " + expectedResponse + " but got: " + responseCode)
} else {
    log.info("✅ API " + apiNum + " successfully processed " + processedRequests + " requests.")
}
```

🔹 Tracks requests per API & validates response codes.

---

## 📌 Summary of Script Placement

| Component | Placement in JMeter |
|-----------|----------------------|
| CSV Data Set Config | Inside **Thread Group** |
| JSR223 PreProcessor (Threads Calculation) | Inside **HTTP Request Sampler** |
| JSR223 PreProcessor (Headers & Payload Loader) | Inside **HTTP Request Sampler** |
| HTTP Header Manager (Common Headers) | Inside **HTTP Request Sampler** |
| Constant Throughput Timer (Even Load Distribution) | Inside **HTTP Request Sampler** |
| JSR223 PostProcessor (Request Tracking & Validation) | Inside **HTTP Request Sampler** |

🎯 **Expected Outcome**
✅ Each API executes exactly `requests_per_hour` over 1 hour.
✅ Requests are evenly distributed & controlled via Timers.
✅ Headers & payloads are dynamically loaded per API.
✅ Common headers are applied automatically.
✅ Ramp-up & ramp-down work smoothly.
✅ All scripts & configurations are version-controlled in Git.

🚀 Now you have a fully scalable & Git-compatible JMeter API testing framework! 🔥

