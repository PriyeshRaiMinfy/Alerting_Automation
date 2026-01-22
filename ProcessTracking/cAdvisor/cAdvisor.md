### Architecture
---
```
+-----------------------------------------------------------------------+
|   SERVER 1: MONITORING HUB                                            |
|   (Prometheus + Alertmanager)                                         |
|                                                                       |
|   1. Prometheus Server                                                |
|      Running "scraper"                                                |
|      Config: "Go to Server 2:8080 and get data"                       |
|           |                                                           |
+-----------|-----------------------------------------------------------+
            |  (HTTP GET /metrics)
            |
            v
+-----------------------------------------------------------------------+
|   SERVER 2: APPLICATION NODE (Linux)                                  |
|                                                                       |
|   +--------------------------+    +-------------------------------+   |
|   |  Docker Engine           |    |  Linux Kernel (The OS)        |   |
|   |                          |    |                               |   |
|   |  [Container A: Python]---|--->|  Cgroup: /docker/python_id    |   |
|   |        (PID 101)         |    |  (CPU: 40%, RAM: 200MB)       |   |
|   |                          |    |                               |   |
|   |  [Container B: Java]-----|--->|  Cgroup: /docker/java_id      |   |
|   |        (PID 102)         |    |  (CPU: 10%, RAM: 500MB)       |   |
|   |                          |    +-------------------------------+   |
|   |                          |                   ^                    |
|   |  [Container C: cAdvisor] |                   |                    |
|   |   (The Monitoring Agent) |-------------------+                    |
|   |   Exposes Port: 8080     |    (Reads Stats Directly)              |
|   +--------------------------+                                        |
+-----------------------------------------------------------------------+
```