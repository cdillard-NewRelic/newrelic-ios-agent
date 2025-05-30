# New Relic iOS Agent: Implementation Description

This document provides a technical overview of the New Relic iOS Agent, detailing its internal architecture and how it collects and transmits data for mobile application performance monitoring.

The New Relic iOS Agent is designed to provide insight into your iOS application's performance, helping you identify and diagnose issues, understand user interactions, and track overall application health. It achieves this by instrumenting key parts of your application, collecting various types of data (including performance metrics, network requests, crashes, and user interactions), and sending this data to the New Relic platform for analysis and visualization.

## Core Components

The agent is composed of several core components that work together to monitor your application:

*   **Data Collection:** This is the foundation of the agent and includes several sub-components:
    *   `Instrumentation`: Dynamically modifies your application code at runtime (e.g., using method swizzling and `NSURLProtocol`) to intercept relevant events like method calls, network requests, and UI interactions.
    *   `Measurements`: Gathers various performance metrics and other data points from the instrumented code.
    *   `ActivityTracing`: Focuses on tracking sequences of operations or user interactions to provide detailed traces.
    *   `UserActions`: Specifically monitors and records discrete user actions within the application.
    *   `CrashHandler`: Captures information about application crashes, including stack traces and device context.
*   **Data Processing:** Once data is collected, it needs to be processed:
    *   `Analytics`: Aggregates and prepares collected data into events that are suitable for analysis in New Relic.
    *   `Harvester`: Periodically collects processed data from various components and prepares it for transmission to the New Relic backend.
*   **Data Transmission:** Handles the communication with New Relic servers:
    *   `Connectivity`: Manages the secure sending of harvested data to the New Relic collector endpoints.
*   **Configuration:** Allows for dynamic adjustments to the agent's behavior:
    *   `FeatureFlags`: Enables or disables specific agent features and functionalities based on configuration received from New Relic or set locally.
*   **Utilities:** A collection of helper classes and functions that support the various activities of the agent, such as JSON handling, Base64 encoding, and device information retrieval.

## Data Flow

The following diagram illustrates the general flow of data through the New Relic iOS Agent:

```
+--------------------------------------------------------------------------+
| Your iOS Application                                                     |
+--------------------------------------------------------------------------+
|          | (App Interactions, Network Requests, Crashes, etc.)           |
|          V                                                               |
+--------------------------------------------------------------------------+
| New Relic iOS Agent                                                      |
|                                                                          |
|    +-----------------------+     +----------------------+                |
|    |    Instrumentation    |<--->|     Measurements     |                |
|    | (Method Swizzling,    |     | (Performance Data,   |                |
|    |  NSURLProtocol, etc.) |     |  Custom Metrics)     |                |
|    +-----------------------+     +----------------------+                |
|              |                           |                               |
|              |                           V                               |
|              |                 +--------------------+                    |
|              +----------------->   ActivityTracing  |                    |
|              |                 +--------------------+                    |
|              |                           |                               |
|              |                           V                               |
|              |                  +-----------------+                      |
|              +------------------&gt;   UserActions   |                      |
|              |                  +-----------------+                      |
|              |                           |                               |
|              |                           V                               |
|    +-----------------------+     +----------------------+                |
|    |     Crash Handler     |<--->|      Analytics       |                |
|    | (Captures crash data) |     | (Event Aggregation)  |                |
|    +-----------------------+     +----------------------+                |
|              |                           |                               |
|              V                           V                               |
|    +---------------------------------------------------+                 |
|    |                    Harvester                      |                 |
|    | (Collects & prepares data for transmission)       |                 |
|    +---------------------------------------------------+                 |
|                              |                                           |
|                              V                                           |
|    +---------------------------------------------------+                 |
|    |                   Connectivity                    |                 |
|    | (Sends data to New Relic backend via HTTPS)       |                 |
|    +---------------------------------------------------+                 |
|                                                                          |
+--------------------------------------------------------------------------+
|                              |                                           |
|                              V                                           |
+--------------------------------------------------------------------------+
| New Relic Platform (Collector, Data Analysis, Visualization)             |
+--------------------------------------------------------------------------+
```

**Explanation of Flow:**

1.  **Events Occur in Application:** User interactions, network requests, method executions, and unfortunately, crashes, happen within your running application.
2.  **Instrumentation Captures Data:** The `Instrumentation` component, through techniques like method swizzling and custom `NSURLProtocol` implementations, intercepts these events and gathers raw data.
3.  **Data Enrichment and Aggregation:**
    *   `Measurements` takes this raw data and converts it into structured performance metrics (e.g., response times, view loading times) and custom metrics.
    *   `ActivityTracing` and `UserActions` focus on specific sequences or discrete user activities, creating detailed trace information.
    *   `CrashHandler` captures detailed information if a crash occurs.
    *   The `Analytics` component takes data from these various sources and aggregates it into events that are reported to New Relic.
4.  **Harvesting:** The `Harvester` periodically collects all the processed data (metrics, events, traces, crash reports) from the different components.
5.  **Transmission:** The `Connectivity` component takes the harvested data payload and securely transmits it to the New Relic backend servers via HTTPS.
6.  **Backend Processing:** Once received by New Relic, the data is processed, analyzed, and made available for visualization and alerting in the New Relic UI.

## Key Mechanisms

Several key mechanisms underpin the agent's ability to collect and report data:

*   **Instrumentation Engine:**
    *   **Method Swizzling:** For Objective-C and Swift methods that are exposed to Objective-C, the agent uses method swizzling. This technique allows the agent to replace the original implementation of a method with its own wrapper implementation at runtime. This wrapper executes some monitoring logic (e.g., starting a timer, recording attributes) before and/or after calling the original method implementation. This is primarily used for tracking method execution times and user interactions.
    *   **`NSURLProtocol` and Network Instrumentation:** To monitor network requests made via `NSURLConnection` and `NSURLSession`, the agent historically used a custom `NSURLProtocol`. This allowed it to intercept outgoing HTTP requests and incoming responses to capture details like URL, status code, response time, and data transfer sizes. For newer versions and specific configurations, direct instrumentation of `NSURLSession` methods might also be employed for more granular control and to support newer networking features.
    *   **WKWebView Instrumentation:** The agent can also instrument `WKWebView` instances to capture information about web views, such as page load times and network requests originating from the web view.

*   **Harvest Cycle:**
The agent doesn't send data to New Relic immediately as it's collected. Instead, it employs a harvest cycle:
    1.  **Data Buffering:** Collected metrics, events, and traces are stored in memory buffers.
    2.  **Periodic Harvest:** The `Harvester` component runs on a timer (typically every 60 seconds by default, but configurable). When the timer fires, or when certain data limits are reached (e.g., max event pool size), the harvester collects all buffered data.
    3.  **Data Packaging:** The collected data is formatted into a JSON payload.
    4.  **Transmission:** The `Connectivity` component sends this payload to the New Relic collector endpoint.
    5.  **Confirmation & Reset:** Upon successful transmission, the local buffers are typically cleared. If transmission fails, data might be retried or discarded based on configuration and error type.
    This batching approach is more efficient in terms of network usage and device battery life compared to sending individual data points.

*   **Crash Reporting:**
    1.  **Exception Handlers:** The agent registers handlers for uncaught Objective-C exceptions and signals (like `SIGSEGV`, `SIGBUS`, etc.) that indicate a crash.
    2.  **Crash Data Collection:** When a crash occurs, the handler collects as much information as possible, including:
        *   Stack traces for all threads.
        *   Device information (OS version, model, etc.).
        *   Application information (bundle ID, version).
        *   Session attributes and breadcrumbs leading up to the crash.
    3.  **Persistent Storage:** The crash report is saved to disk immediately to ensure it survives an app restart.
    4.  **Reporting on Next Launch:** On the subsequent application launch, the agent detects the saved crash report and sends it to the New Relic crash collector endpoint.
    5.  **Symbolication:** To make stack traces human-readable (showing method and file names instead of memory addresses), dSYM (debug symbol) files corresponding to the crashed app version must be uploaded to New Relic. The agent itself does not perform symbolication.

*   **Distributed Tracing:**
    To enable tracing of requests across multiple services (e.g., from your mobile app to a backend service), the agent supports W3C Trace Context headers:
    1.  **Header Generation:** When an instrumented network request is about to be made, the agent can generate or propagate W3C trace context headers (`traceparent` and `tracestate`).
    2.  **Header Injection:** These headers are automatically injected into outgoing HTTP requests.
    3.  **Backend Correlation:** If the receiving backend service is also New Relic instrumented, it will recognize these headers and use them to link the mobile request with the backend transaction, providing an end-to-end trace in the New Relic UI.
    The agent also participates in trace context by accepting and continuing traces initiated by other New Relic agents (e.g., in a hybrid app scenario).

## How to Contribute

We welcome contributions to improve the New Relic iOS Agent! If you're interested in contributing, please review our [CONTRIBUTING.md](CONTRIBUTING.md) file for guidelines on how to:

*   Report bugs and request features.
*   Submit pull requests.
*   Sign the Contributor License Agreement (CLA).

Your contributions are highly appreciated!

## License

The New Relic iOS Agent is licensed under the [Apache 2.0 License](LICENSE). Please see the [LICENSE](LICENSE) file for full license text.

This project also uses source code from third-party libraries. Full details on which libraries are used and the terms under which they are licensed can be found in the main [README.md](README.md) and within the respective source code directories.
