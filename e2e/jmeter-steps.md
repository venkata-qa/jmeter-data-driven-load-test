# JMeter Dynamic API Load Testing

## 📌 Overview
This JMeter test plan dynamically handles multiple API load tests using a **single Thread Group**. Instead of manually creating multiple Thread Groups per API, this approach ensures efficient execution while dynamically adjusting the thread count and request distribution.

## 🚀 Key Features
- **Single Thread Group** for all APIs, reducing complexity.
- **Dynamic thread allocation** based on API request load.
- **CSV Data Set Config** for dynamically reading API details.
- **Throughput Controller** to ensure correct request per second (RPS).
- **JSR223 PreProcessor** for loading API headers & payloads dynamically.
- **Scalable** solution that supports 100s of APIs without additional Thread Groups.

---

## 📜 JMeter Test Plan Structure
```
Test Plan
 ├── Setup Thread Group (Calculates Threads for Each API) ✅
 │   ├── JSR223 Sampler (Reads CSV & Stores `threadsRequired` per API) ✅
 ├── Thread Group (Handles All APIs) ✅
 │   ├── CSV Data Set Config (Reads API Data) ✅
 │   ├── Throughput Controller (Ensures API Execution) ✅
 │   ├── JSR223 PreProcessor (Load Headers & Payload) ✅
 │   ├── HTTP Request Sampler (Executes API Calls) ✅
 │   │   ├── HTTP Header Manager (For Common Headers) ✅
 │   │   ├── Constant Throughput Timer (Ensure Correct RPS) ✅
 │   │   ├── JSR223 PostProcessor (Track & Validate Requests) ✅
```

## 🔹 Step 1: Process CSV File & Store Thread Counts (Setup Thread Group)
### **JSR223 Sampler - Setup Thread Group**
The **Setup Thread Group** reads API data from `config/api_data.csv` and dynamically determines the required threads per API.

```groovy
import java.io.BufferedReader
import java.io.FileReader

int testDuration = 3600 // 1 hour test
int rampUpTime = 300
int rampDownTime = 300
int steadyStateTime = testDuration - (rampUpTime + rampDownTime)

BufferedReader br = new BufferedReader(new FileReader("config/api_data.csv"))
String line = br.readLine() // Skip header

while ((line = br.readLine()) != null) {
    String[] data = line.split(",")
    String apiNumber = data[0].trim()
    int requestsPerHour = Integer.parseInt(data[12].trim())
    int threadsRequired = (int) Math.ceil(requestsPerHour / (double) testDuration)
    
    props.put("threadsRequired_" + apiNumber, String.valueOf(threadsRequired))
    props.put("rampUpTime_" + apiNumber, String.valueOf(rampUpTime))
    props.put("rampDownTime_" + apiNumber, String.valueOf(rampDownTime))
    props.put("steadyStateTime_" + apiNumber, String.valueOf(steadyStateTime))
    
    log.info("✅ API " + apiNumber + " | Threads: " + threadsRequired)
}
br.close()
```
📌 **Each API's thread count is stored using** `props.put("threadsRequired_API-XXXX")`.

---

## 🔹 Step 2: Single Thread Group Runs All APIs Dynamically

Instead of multiple Thread Groups, we use **one** that:
- Reads API details from **CSV Data Set Config**.
- Dynamically assigns the number of threads per API.
- Uses **Throughput Controller** to regulate execution.

📍 **Thread Group Settings:**

| Parameter               | Value                        |
|-------------------------|----------------------------|
| Number of Threads       | `${__P(threadsRequired)}`  |
| Ramp-Up Period (seconds)| `${__P(rampUpTime)}`       |
| Loop Count             | Forever (or defined number)|
| Test Duration (seconds) | `${__P(steadyStateTime)}`  |

📌 This ensures **correct thread execution** per API dynamically.

---

## 🔹 Step 3: CSV Data Set Config Reads API Data
📍 **CSV Data Set Config Settings:**

| Field         | Value                        |
|--------------|-----------------------------|
| Filename     | `config/api_data.csv`       |
| Variable Names | `api_number, api_method, env_type, end_point, ...` |
| Sharing Mode | Current Thread              |

📌 Ensures **each thread gets the correct API data dynamically**.

---

## 🔹 Step 4: Use Throughput Controller to Control API Execution
📍 **Throughput Controller Settings:**

| Setting       | Value                         |
|--------------|------------------------------|
| Execution Mode | Percent Execution           |
| Throughput (%) | `${requests_per_hour}`      |

📌 Ensures **each API executes at its required request rate**.

---

## 🔹 Step 5: Load Headers & Payload Per API
📍 **JSR223 PreProcessor - Load API Headers & Payload**

```groovy
import groovy.json.JsonSlurper
import org.apache.jmeter.protocol.http.control.Header
import org.apache.jmeter.protocol.http.control.HeaderManager
import org.apache.jmeter.services.FileServer

String apiFolder = vars.get("api_folder")?.trim()?.replace("\\", "/")?.replaceAll(/^"|"$/, '')
if (apiFolder == null || apiFolder.trim().isEmpty()) {
    log.error("❌ 'api_folder' variable is missing or empty!")
    throw new IllegalArgumentException("api_folder variable is missing")
}

String baseDir = FileServer.getFileServer().getBaseDir()
File folder = new File(baseDir, apiFolder).getCanonicalFile()
File headersFile = new File(folder, "headers.json").getCanonicalFile()
File payloadFile = new File(folder, "payload.json").getCanonicalFile()

log.info("📂 Resolved API folder path: " + folder.absolutePath)

def headerManager = sampler.getHeaderManager()
if (headerManager == null) {
    log.warn("⚠️ HeaderManager was null. Creating a new one.")
    headerManager = new HeaderManager()
    sampler.setHeaderManager(headerManager)
}

headerManager.clear()
if (headersFile.exists()) {
    def headersMap = new JsonSlurper().parseText(headersFile.text)
    headersMap.each { key, value ->
        headerManager.add(new Header(key, value.toString()))
        log.info("✅ Added Header: " + key + " = " + value)
    }
}

String methodType = vars.get("api_method")
if (methodType?.equalsIgnoreCase("POST") || methodType?.equalsIgnoreCase("PUT")) {
    if (payloadFile.exists()) {
        vars.put("payload", payloadFile.text)
        log.info("✅ Loaded Payload for API: " + vars.get("api_number"))
    } else {
        log.warn("⚠️ Payload file not found: " + payloadFile.absolutePath)
    }
}
```

---

## 🎯 Final Outcome
✔ Dynamically handles multiple APIs in **one** Thread Group.
✔ Uses **CSV Data Set Config** to execute APIs correctly.
✔ **Throughput Controller** ensures correct request distribution.
✔ **JSR223 PreProcessor** loads API headers & payloads dynamically.
✔ **Scalable solution** for testing 100s of APIs without extra Thread Groups.

🚀 **JMeter now dynamically adjusts load per API in an optimized test plan!** 🎯🔥

