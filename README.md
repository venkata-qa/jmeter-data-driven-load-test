# jmeter-data-driven-load-test
Load test using Jmeter by driving the data from CSV


Final JMeter Structure
Copy
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
Explanation of Each Component
1. CSV Data Set Config
Reads the API details from the CSV file (e.g., Group1.csv).

Columns include api_number, endpoint, headers, auth_token, payload, expected_response, content_type, correlation_id, end_type, end_iteration, requests_per_hour.

2. JSR223 PreProcessor (Thread Group Configuration)
Calculates the Number of Threads and Ramp-Up Period for the Thread Group based on the total requests per hour for all APIs in the group.

Stores the calculated values in JMeter variables (numberOfThreads, rampUpPeriod).

3. Loop Controller
Iterates through each API in the group.

The loop count is set to the number of APIs in the group.

4. JSR223 PreProcessor (API-Specific Throughput Calculation)
Calculates the throughput (requests per minute) for each API based on its requests_per_hour.

Stores the calculated throughput in a JMeter variable (e.g., throughput_1, throughput_2).

5. Throughput Controller
Controls the request rate for each API based on its calculated throughput (throughput_${api_number}).

6. HTTP Request
Sends the API request using the details from the CSV file.

7. Response Assertion
Validates the API response (e.g., status code, response body).

8. Constant Throughput Timer
Ensures that requests are distributed evenly over the hour.

9. View Results Tree
Displays the results of each API request for debugging.

Groovy Scripts
1. JSR223 PreProcessor (Thread Group Configuration)
This script calculates the Number of Threads and Ramp-Up Period for the Thread Group.

groovy
Copy
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
int multiplier = 10; // Adjust based on your requirements
int numberOfThreads = (int) Math.ceil(totalRequestsPerHour / testDuration * multiplier);
int rampUpPeriod = testDuration;

// Store calculated values in JMeter variables
vars.put("numberOfThreads", numberOfThreads.toString());
vars.put("rampUpPeriod", rampUpPeriod.toString());

// Log the calculated values for debugging
log.info("Number of Threads: " + numberOfThreads);
log.info("Ramp-Up Period: " + rampUpPeriod);
2. JSR223 PreProcessor (API-Specific Throughput Calculation)
This script calculates the throughput for each API.

groovy
Copy
// Read requests_per_hour from CSV
int requestsPerHour = Integer.parseInt(vars.get("requests_per_hour"));

// Calculate throughput (requests per minute)
int throughput = (int) (requestsPerHour / 60);

// Store throughput in a JMeter variable
vars.put("throughput_" + vars.get("api_number"), throughput.toString());

// Log the calculated throughput for debugging
log.info("API " + vars.get("api_number") + " Throughput: " + throughput + " requests per minute");
How It Works
The CSV Data Set Config reads the API details, including requests_per_hour.

The JSR223 PreProcessor (Thread Group Configuration) calculates the Number of Threads and Ramp-Up Period for the Thread Group.

The Loop Controller iterates through each API in the group.

The JSR223 PreProcessor (API-Specific Throughput Calculation) calculates the throughput for each API.

The Throughput Controller ensures that each API runs at its specified throughput.

The Constant Throughput Timer ensures that requests are distributed evenly over the hour.

Example
Let’s say Group 1 has the following APIs:

API 1: 100 requests per hour

Throughput = 100 / 60 = 1.67 requests per minute

API 2: 1000 requests per hour

Throughput = 1000 / 60 = 16.67 requests per minute

Thread Group Configuration:
Number of Threads: 10 (calculated based on total requests per hour)

Ramp-Up Period: 3600 seconds (1 hour)

Throughput Controllers:
API 1: Throughput = 1.67 requests per minute

API 2: Throughput = 16.67 requests per minute

Benefits of This Approach
Dynamic Configuration: The Thread Group and API-specific throughput are calculated automatically.

Scalability: You can handle any number of APIs with different load requirements.

Flexibility: Easily adjust the multiplier and other parameters based on your test results.

