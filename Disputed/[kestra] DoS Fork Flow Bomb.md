# Security Advisory: Resource Exhaustion via API Recursion in Kestra

## Summary
A Denial of Service (DoS) situation exists in Kestra's execution engine due to an architectural bypass of recursion checks. While Kestra effectively prevents direct recursive calls between flows using native subflow tasks, this protection can be bypassed by using the `io.kestra.plugin.scripts.http.Request` task to trigger the same flow via Kestra's own API. This allows the creation of a "fork bomb" scenario where each execution triggers multiple new instances, leading to exponential resource consumption.

## Technical Details
The situation allows a user with flow-creation privileges to circumvent built-in recursion detection:

1. **Native Check Bypass:** Kestra’s engine detects and blocks circular dependencies when using native tasks. However, when a flow triggers itself via an externalized HTTP request to `/api/v1/executions`, the engine treats it as a new, independent request.
2. **Exponential Growth:** If a flow is configured to send multiple HTTP requests to its own trigger endpoint, it creates a "fork bomb" effect.
3. **Resource Starvation:** The rapid influx of tasks can exhaust Worker threads, CPU, RAM, and database connections, potentially rendering the entire Kestra instance unresponsive for all users.

## Vendor Response
The vendor has reviewed the report and considers this **expected behaviour** within an orchestration platform. Their stance is that Kestra is designed for high-throughput automation and provides flexibility to trigger flows via multiple channels, including the API. They emphasize that security is a **shared responsibility** and recommend that organizations manage risks through the principle of least privilege and infrastructure hardening.

## Proof of Concept (Educational Purposes)
> **Warning:** This PoC is intended for internal security testing in isolated environments only.

1. **Bypass Method:** Create a flow that utilizes an **HTTP Request task**.
2. **Target:** Configure the task to POST to the Kestra API endpoint for its own execution.
3. **Execution:** Upon triggering, the flow bypasses native recursion checks because the call is processed as an inbound web request rather than a task dependency.
4. **Observation:** Monitor server metrics to observe the rapid consumption of system resources as executions scale exponentially.

## Impact
- **Service Availability:** Risk of total instance downtime, affecting all scheduled tasks and users.
- **Infrastructure Costs:** In auto-scaling environments (e.g., Kubernetes, AWS), this could trigger massive, unintended resource provisioning and costs.
- **Resource Exhaustion:** Rapid depletion of CPU, Memory, and Worker pool availability.

## Remediation & Hardening (Recommended)
As this is a design-level consideration under the shared responsibility model, administrators should implement the following **Hardening** measures:

* **Strict RBAC:** Limit flow creation and editing permissions to trusted users only, as per the vendor's recommendation.
* **Concurrency Limits:** Always define `concurrency` limits at the flow level to act as a circuit breaker against unintended execution spikes.
* **API Rate Limiting:** Implement rate limiting at the Load Balancer or Reverse Proxy level (e.g., Nginx, Traefik) specifically for internal requests originating from the Worker nodes to the API.
* **Execution Depth Monitoring:** Implement custom monitoring or alerts to detect unusual spikes in execution volume for specific flow IDs.
* **Resource Quotas:** Enforce CPU and Memory limits at the container/pod level to ensure a fork bomb cannot impact the host or other services.

---
**Disclaimer:** This advisory is provided for educational and defensive purposes. It aims to assist Kestra administrators in securing their automation infrastructure by following the recommended shared responsibility and hardening practices.