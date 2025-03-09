# JMeter Test Plan Structure

## Final JMeter Structure
```
Test Plan
  ├── Thread Group 1 (Group 1)
  │     ├── CSV Data Set Config (Group1.csv)
  │     ├── JSR223 PreProcessor (Groovy script for Thread Group configuration)
  │     ├── Loop Controller (loop count = number of APIs in Group 1)
  │     │     ├── JSR223 PreProcessor (Groovy script for API-specific throughput calculation)
  │     │     ├── Throughput Controller (Throughput = ${throughput_${api_number}})
  │     │     │     ├── HTTP Request
  │     │     │     ├── Response Assertion
  │     │     │     └── Constant Throughput Timer (Target Throughput = ${throughput_${api_number}})
  │     │     └── View Results Tree
```

---

## Explanation of Each Component

### 1. CSV Data Set Config
Reads API details from a CSV file (e.g., `Group1.csv`).

**Columns in CSV:**
- `api_number`
- `endpoint`
- `headers`
- `auth_token`
- `payload`
- `expected_response`
- `content_type`
- `correlation_id`
- `end_type`
- `end_iteration`
- `requests_per_hour`

### 2. JSR223 PreProcessor (Thread Group Configuration)
Calculates the Number of Threads and Ramp-Up Period based on the total requests per hour for all APIs in the group.

**Outputs:**
- `numberOfThreads`
- `rampUpPeriod`

### 3. Loop Controller
- Iterates through each API in the group.
- Loop count = Number of APIs in the group.

### 4. JSR223 PreProcessor (API-Specific Throughput Calculation)
- Calculates the throughput (requests per minute) for each API using `requests_per_hour`.
- Stores calculated throughput in a JMeter variable (e.g., `throughput_1`, `throughput_2`).

### 5. Throughput Controller
- Controls the request rate per API based on calculated throughput (`throughput_${api_number}`).

### 6. HTTP Request
- Sends API requests using details from the CSV file.

### 7. Response Assertion
- Validates API responses (e.g., status code, response body).

### 8. Constant Throughput Timer
- Ensures that requests are evenly distributed over the test duration.

### 9. View Results Tree
- Displays API request results for debugging purposes.

---

## Groovy Scripts

### 1. JSR223 PreProcessor (Thread Group Configuration)
Calculates the number of threads and ramp-up period.
```groovy
// Initialize total requests per hour
int totalRequestsPerHour = 0;

// Loop through all APIs in the group (assuming CSV has multiple rows)
while (vars.get("requests_per_hour") != null) {
    totalRequestsPerHour += Integer.parseInt(vars.get("requests_per_hour"));
    // Move to the next row in the CSV
    ctx.getCurrentSampler().getProperty("CSVDataSet").next();
}

// Calculate number of threads and ramp-up period
int testDuration = 3600; // 1 hour
int multiplier = 10; // Adjustable multiplier
int numberOfThreads = (int) Math.ceil(totalRequestsPerHour / testDuration * multiplier);
int rampUpPeriod = testDuration;

// Store calculated values in JMeter variables
vars.put("numberOfThreads", numberOfThreads.toString());
vars.put("rampUpPeriod", rampUpPeriod.toString());

// Log calculated values
log.info("Number of Threads: " + numberOfThreads);
log.info("Ramp-Up Period: " + rampUpPeriod);
```

### 2. JSR223 PreProcessor (API-Specific Throughput Calculation)
Calculates throughput for each API.
```groovy
// Read requests_per_hour from CSV
int requestsPerHour = Integer.parseInt(vars.get("requests_per_hour"));

// Calculate throughput (requests per minute)
int throughput = (int) (requestsPerHour / 60);

// Store throughput in a JMeter variable
vars.put("throughput_" + vars.get("api_number"), throughput.toString());

// Log calculated throughput
log.info("API " + vars.get("api_number") + " Throughput: " + throughput + " requests per minute");
```

---

## How It Works

1. The `CSV Data Set Config` reads API details, including `requests_per_hour`.
2. The `JSR223 PreProcessor (Thread Group Configuration)` calculates the **Number of Threads** and **Ramp-Up Period**.
3. The `Loop Controller` iterates through each API in the group.
4. The `JSR223 PreProcessor (API-Specific Throughput Calculation)` calculates throughput for each API.
5. The `Throughput Controller` ensures that each API runs at its specified throughput.
6. The `Constant Throughput Timer` evenly distributes requests over the hour.

---

## Example

Consider a scenario where `Group 1` has the following APIs:

| API  | Requests per Hour | Throughput (Requests per Minute) |
|------|------------------|--------------------------------|
| API 1 | 100              | 1.67                           |
| API 2 | 1000             | 16.67                          |

### Thread Group Configuration:
- **Number of Threads**: 10 (calculated based on total requests per hour)
- **Ramp-Up Period**: 3600 seconds (1 hour)

### Throughput Controllers:
- **API 1 Throughput** = 1.67 requests per minute
- **API 2 Throughput** = 16.67 requests per minute

---

## Benefits of This Approach

### ✅ Dynamic Configuration
- The Thread Group and API-specific throughput are **calculated automatically**.

### ✅ Scalability
- Supports **any number of APIs** with different load requirements.

### ✅ Flexibility
- Easily **adjust the multiplier** and other parameters based on test results.

---

## Conclusion
This JMeter test plan dynamically calculates and manages API request load to ensure realistic testing while maintaining flexibility and scalability.

