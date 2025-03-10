Best Solution: Use a Single Thread Group & Handle APIs Dynamically
Instead of manually creating a Thread Group per API, you can:

Use a single Thread Group
Control thread count dynamically per API
Ensure APIs are executed separately
📌 Final JMeter Test Plan Structure
java
Copy
Edit
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
📌 One Thread Group will run all APIs while controlling execution dynamically.

🔹 Step 1: Process CSV File & Store Thread Counts (Setup Thread Group)
📍 JSR223 Sampler inside Setup Thread Group

groovy
Copy
Edit
import java.io.BufferedReader
import java.io.FileReader

// Test Duration
int testDuration = 3600 // 1 hour test
int rampUpTime = 300
int rampDownTime = 300
int steadyStateTime = testDuration - (rampUpTime + rampDownTime)

// Read CSV file
BufferedReader br = new BufferedReader(new FileReader("config/api_data.csv"))
String line = br.readLine() // Skip header

while ((line = br.readLine()) != null) {
    String[] data = line.split(",") // CSV is comma-separated

    String apiNumber = data[0].trim() // API Number
    int requestsPerHour = Integer.parseInt(data[12].trim()) // requests_per_hour column

    // Calculate threads for this API
    int threadsRequired = (int) Math.ceil(requestsPerHour / (double) testDuration)

    // Store values as JMeter properties
    props.put("threadsRequired_" + apiNumber, String.valueOf(threadsRequired))
    props.put("rampUpTime_" + apiNumber, String.valueOf(rampUpTime))
    props.put("rampDownTime_" + apiNumber, String.valueOf(rampDownTime))
    props.put("steadyStateTime_" + apiNumber, String.valueOf(steadyStateTime))

    log.info("✅ API " + apiNumber + " | Threads: " + threadsRequired)
}
br.close()
📌 Each API's thread count is stored using props.put("threadsRequired_API-XXXX").

🔹 Step 2: Single Thread Group Runs All APIs Dynamically
Instead of separate Thread Groups, we use one Thread Group that:

Reads each API’s details from CSV.
Controls how many threads are created per API dynamically.
Uses "Throughput Controller" to control API execution rate.
📍 Thread Group Settings:

Parameter	Value
Number of Threads	${__P(threadsRequired)}
Ramp-Up Period (seconds)	${__P(rampUpTime)}
Loop Count	Forever (or a defined number)
Test Duration (seconds)	${__P(steadyStateTime)}
📌 This ensures the correct number of threads execute per API.

🔹 Step 3: CSV Data Set Config Reads API Data
📍 CSV Data Set Config Settings:

Field	Value
Filename	config/api_data.csv
Variable Names	api_number, api_method, env_type, env_iteration, end_point, auth_token, content_type, correlation_id, api_folder, headers_file, payload_file, expected_response, requests_per_hour
Sharing Mode	Current Thread
📌 Each thread gets the correct API data dynamically.

🔹 Step 4: Use Throughput Controller to Control API Execution
📍 Add a Throughput Controller inside Thread Group

This ensures each API gets its correct number of requests per hour.
Setting	Value
Execution Mode	Percent Execution
Throughput (%)	${requests_per_hour}
📌 This ensures each API executes in proportion to its requests per hour.

🔹 Step 5: Load Headers & Payload Per API
📍 JSR223 PreProcessor inside HTTP Request Sampler

groovy
Copy
Edit
import groovy.json.JsonSlurper
import org.apache.jmeter.protocol.http.control.Header
import org.apache.jmeter.protocol.http.control.HeaderManager
import org.apache.jmeter.services.FileServer

// Get API folder from CSV variable
String apiFolder = vars.get("api_folder")?.trim()?.replace("\\", "/")?.replaceAll(/^"|"$/, '')

if (apiFolder == null || apiFolder.trim().isEmpty()) {
    log.error("❌ 'api_folder' variable is missing or empty!")
    throw new IllegalArgumentException("api_folder variable is missing")
}

// Construct file paths
String baseDir = FileServer.getFileServer().getBaseDir()
File folder = new File(baseDir, apiFolder).getCanonicalFile()
File headersFile = new File(folder, "headers.json").getCanonicalFile()
File payloadFile = new File(folder, "payload.json").getCanonicalFile()

log.info("📂 Resolved API folder path: " + folder.absolutePath)

// Load API-Specific Headers
def headerManager = sampler.getHeaderManager()
if (headerManager == null) {
    log.warn("⚠️ HeaderManager was null. Creating a new one.")
    headerManager = new HeaderManager()
    sampler.setHeaderManager(headerManager)
}

// Clear existing headers and load new ones
headerManager.clear()
if (headersFile.exists()) {
    def headersMap = new JsonSlurper().parseText(headersFile.text)
    headersMap.each { key, value ->
        headerManager.add(new Header(key, value.toString()))
        log.info("✅ Added Header: " + key + " = " + value)
    }
}

// Load Payload for POST/PUT Requests
String methodType = vars.get("api_method")
if (methodType?.equalsIgnoreCase("POST") || methodType?.equalsIgnoreCase("PUT")) {
    if (payloadFile.exists()) {
        vars.put("payload", payloadFile.text)
        log.info("✅ Loaded Payload for API: " + vars.get("api_number"))
    } else {
        log.warn("⚠️ Payload file not found: " + payloadFile.absolutePath)
    }
}
🎯 Final Outcome
✔ Dynamically handles multiple APIs in a single Thread Group.
✔ Uses CSV Data Set Config to ensure correct API execution per thread.
✔ Throughput Controller ensures correct request distribution per API.
✔ JSR223 PreProcessor dynamically loads API-specific headers & payloads.
✔ Fully scalable—supports 100s of APIs without creating multiple Thread Groups.

🚀 Now JMeter dynamically adjusts load per API in a single optimized test! Let me know if you need further optimizations! 🎯🔥
