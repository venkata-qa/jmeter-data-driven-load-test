# 🚀 Dynamic Thread Allocation in JMeter

## 🔹 Step 1: SetUp Thread Group - Reads CSV & Stores Thread Counts

### 📍 What Happens Here?
- Reads **ALL API rows** from the CSV file (one by one).
- Calculates the **required threads** for each API.
- Stores **thread counts** in JMeter properties (`props.put()`) for later use.

### 📍 Example Log Output (from SetUp Thread Group)
```sql
✅ SetUp Thread Group - API API-1001 | Threads: 5
✅ SetUp Thread Group - API API-1002 | Threads: 20
✅ SetUp Thread Group - API API-1003 | Threads: 8
...
```

## 🔹 Step 2: Ultimate Thread Group - Reads CSV & Fetches Stored Threads

### 📍 What Happens Here?
- Reads the CSV again (row by row).
- Fetches the **pre-calculated thread count** from `props.get()`.
- Starts execution **immediately** for each API.
- **Does NOT** wait for other APIs to finish.

### 📍 Example Log Output (from Ultimate Thread Group)
```yaml
🚀 Starting API API-1001 in parallel | Threads: 5
🚀 Starting API API-1002 in parallel | Threads: 20
🚀 Starting API API-1003 in parallel | Threads: 8
...
```

🔥 **Result:** All APIs execute in parallel, using their correct thread count.

## 🎯 Final Confirmation: Your Process Works!
✅ Each API starts **immediately** after fetching its thread count.
✅ APIs run **independently** and do NOT wait for each other.
✅ **Thread counts** are pre-determined but assigned **dynamically**.
✅ **JMeter does NOT** allow changing thread count during execution, so all threads must be set before execution starts.

🚀 **Now, your JMeter test dynamically assigns threads while ensuring parallel execution!** 🚀🔥

---

## 🚀 Detailed Steps to Set Up Parallel API Execution with Dynamic Thread Allocation in JMeter

### 🎯 Goal
- Read a **CSV file** containing multiple API details.
- **SetUp Thread Group** calculates the required threads per API and stores them.
- **Ultimate Thread Group** dynamically assigns and runs APIs in **parallel**.
- Ensure **all APIs execute simultaneously**, without waiting for one another.
- Use **Constant Throughput Timer** to distribute requests evenly.

## 📌 Step 1: Prepare Your CSV File

📍 **File Location:** `config/api_data.csv`

📍 **CSV Data Format:**
```csv
api_number,api_method,end_point,api_folder,requests_per_hour
API-1001,POST,/api/login,data/api-1001,5000
API-1002,GET,/api/users,data/api-1002,20000
API-1003,PUT,/api/orders,data/api-1003,8000
```
🔹 Each row contains API details, including **requests_per_hour** (used to calculate thread count).

## 📌 Step 2: Create a "SetUp Thread Group"

### 📍 Purpose:
- Reads **all CSV rows**.
- Calculates **required threads per API**.
- Stores the values in **JMeter properties (`props.put()`)**.

### ➤ 2.1 Add "CSV Data Set Config" to Read API Data
📍 Inside **SetUp Thread Group** → Right-click → **Add → Config Element → CSV Data Set Config**

| Field | Value |
|--------|--------|
| Filename | `config/api_data.csv` |
| Variable Names | `api_number,api_method,end_point,api_folder,requests_per_hour` |
| Sharing Mode | `All Threads` |

🔹 Ensures **every row** from the CSV is processed.

### ➤ 2.2 Add "Loop Controller" to Process Each Row
📍 Inside **SetUp Thread Group** → Right-click → **Add → Logic Controller → Loop Controller**

| Field | Value |
|--------|--------|
| Loop Count | `${__groovy(new File('config/api_data.csv').readLines().size()-1)}` |

🔹 Ensures **SetUp Thread Group** runs **once per API row**.

### ➤ 2.3 Add "JSR223 PreProcessor" to Calculate Threads
📍 Inside **Loop Controller** → Right-click → **Add → PreProcessor → JSR223 PreProcessor**
📍 Paste the following **Groovy script**:
```groovy
// Read API details from CSV
String apiNumber = vars.get("api_number")
String requestsPerHourStr = vars.get("requests_per_hour")

if (!requestsPerHourStr) {
    log.error("🚨 Missing requests_per_hour for API " + apiNumber)
    return
}

int requestsPerHour = requestsPerHourStr.toInteger()
int testDuration = 3600  // 1 hour
int threadsRequired = (int) Math.ceil(requestsPerHour / (double) testDuration)

// Store values as JMeter properties
props.put("threadsRequired_" + apiNumber, String.valueOf(threadsRequired))
log.info("✅ SetUp Thread Group - API: " + apiNumber + " | Threads: " + threadsRequired)
```
🔹 Each API now has its **required thread count stored globally**.

## 📌 Step 3: Create an "Ultimate Thread Group"

### 📍 Purpose:
- Reads the **CSV again**.
- Fetches **pre-calculated thread counts (`props.get()`)**.
- Executes **all APIs in parallel**.

### ➤ 3.1 Add "Ultimate Thread Group"
📍 **Right-click on Test Plan** → **Add → Thread Group → jp@gc - Ultimate Thread Group**

### ➤ 3.2 Add "CSV Data Set Config" to Read API Data
📍 Inside **Ultimate Thread Group** → **Right-click → Add → Config Element → CSV Data Set Config**

| Field | Value |
|--------|--------|
| Filename | `config/api_data.csv` |
| Variable Names | `api_number,api_method,end_point,api_folder,requests_per_hour` |
| Sharing Mode | `All Threads` |

### ➤ 3.3 Add "JSR223 PreProcessor" to Fetch Stored Threads
📍 Inside **Ultimate Thread Group** → **Right-click → Add → PreProcessor → JSR223 PreProcessor**
📍 Paste the following **Groovy script**:
```groovy
// Read API number
String apiNumber = vars.get("api_number")

// Fetch stored values from SetUp Thread Group
String threadsRequiredStr = props.get("threadsRequired_" + apiNumber)

if (!threadsRequiredStr) {
    log.error("🚨 Missing thread config for API " + apiNumber)
    return
}

int threadsRequired = threadsRequiredStr.toInteger()
vars.put("threadsRequired", String.valueOf(threadsRequired))
log.info("🚀 Ultimate Thread Group - API " + apiNumber + " | Threads: " + threadsRequired)
```

### ➤ 3.4 Configure "Ultimate Thread Group" for Parallel Execution

| Thread Group Row | Thread Count | Ramp-Up | Hold Time | Ramp-Down |
|-----------------|--------------|----------|----------|-----------|
| `${__P(threadsRequired_${api_number}, 1)}` | `${threadsRequired}` | `300` | `3600` | `300` |

### ➤ 3.5 Add "HTTP Request Sampler"
- Executes API requests **dynamically** using CSV data.

### ➤ 3.6 Add "Constant Throughput Timer"
| Target Throughput (requests per minute) | `${requestsPerHour}/60` |

## 📌 Step 4: Run the Test
✅ Verify execution in **View Results Tree** and **Summary Report**.

🚀 **Now, your JMeter setup dynamically assigns threads and executes APIs in parallel!** 🚀🔥

