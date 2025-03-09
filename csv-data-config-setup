# JMeter CSV Data Set Config for Parallel API Execution

## Overview
This guide provides a step-by-step approach to configuring **CSV Data Set Config** in JMeter to run **5 groups of APIs in parallel**, with each API within a group executing in parallel. Each group uses its own CSV file.

---

## Step 1: Create Separate CSV Files
Create 5 separate CSV files for each group (e.g., `Group1.csv`, `Group2.csv`, ..., `Group5.csv`). Each file should contain API details for a single group.

### Example `Group1.csv`
```csv
api_number,endpoint,headers,auth_token,payload,expected_response,content_type,correlation_id,end_type,end_iteration,requests_per_hour
1,/api/v1/resource1,"{""key"":""value""}",token1,"{""param1"":""value1""}",200,application/json,123,type1,iteration1,1200
2,/api/v1/resource2,"{""key"":""value""}",token2,"{""param1"":""value1""}",201,application/xml,456,type2,iteration2,600
```

---

## Step 2: Add CSV Data Set Config for Each Group
For each group (Thread Group), add a **CSV Data Set Config** component and configure it as follows:

- **Filename**: Select the corresponding CSV file (`Group1.csv` for Thread Group 1, `Group2.csv` for Thread Group 2, etc.).
- **File Encoding**: Leave as `UTF-8` (default).
- **Variable Names**: Enter the column names from your CSV file, separated by commas:
  ```
  api_number,endpoint,headers,auth_token,payload,expected_response,content_type,correlation_id,end_type,end_iteration,requests_per_hour
  ```
- **Delimiter**: Use `,` (comma) as the delimiter.
- **Ignore first line**: Set to `true` (if your CSV file has a header row).
- **Recycle on EOF**: Set to `false` (to avoid reusing data after reaching the end of the file).
- **Stop thread on EOF**: Set to `false` (default).
- **Sharing mode**: Set to `All threads` (to ensure all threads in the Thread Group share the same CSV data).

---

## Step 3: Configure Thread Groups for Parallel Execution
- Create **5 Thread Groups** (one for each API group).
- Configure each Thread Group as follows:
  - **Number of Threads (Users)**: Set to the number of APIs in the group.
  - **Ramp-Up Period**: Set to `0` (start all threads at the same time for parallel execution).
  - **Loop Count**: Set to `1` (or as needed).

---

## Step 4: Use Loop Controller for Each API
Inside each **Thread Group**, add a **Loop Controller** with the following settings:
- **Loop Count**: `1` (each API should run once per iteration).
- Inside the Loop Controller, add:
  - **HTTP Request Sampler** (configured dynamically using CSV variables).
  - **HTTP Header Manager** (for passing authentication and headers).
  - **Response Assertion** (to validate response status and body).
  - **Constant Throughput Timer** (to control request distribution).

---

## Step 5: Example JMeter Structure
```
Test Plan
  ├── Thread Group 1 (Group 1)
  │     ├── CSV Data Set Config (Group1.csv)
  │     ├── Loop Controller (loop count = 1)
  │     │     ├── HTTP Request
  │     │     │     ├── HTTP Header Manager
  │     │     │     ├── Response Assertion
  │     │     │     └── Constant Throughput Timer
  │     │     └── View Results Tree
  ├── Thread Group 2 (Group 2)
  │     ├── CSV Data Set Config (Group2.csv)
  │     ├── Loop Controller (loop count = 1)
  │     │     ├── HTTP Request
  │     │     │     ├── HTTP Header Manager
  │     │     │     ├── Response Assertion
  │     │     │     └── Constant Throughput Timer
  │     │     └── View Results Tree
  └── ...
```

---

## Step 6: Run and Validate
- Execute the test plan in JMeter.
- Use **View Results Tree** or **Summary Report** to analyze execution results.
- Ensure all APIs in each group run in parallel without delays.

---

## Key Points to Remember
✅ **Recycle on EOF**: `false` (avoids reusing data).

✅ **Sharing mode**: `All threads` (ensures all threads share the same CSV data).

✅ **Parallel Execution**:
- Multiple Thread Groups run simultaneously.
- Each API within a group runs in parallel.

By following this setup, all **5 API groups execute in parallel**, ensuring efficient performance testing. 🚀

---

## Need Help?
For further questions, feel free to reach out!
