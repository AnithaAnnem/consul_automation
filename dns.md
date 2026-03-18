# LDC Infrastructure: Dynamic Service Discovery Migration

**Project Objective:** Transition the LDC environment from a static, IP-based routing model to a dynamic Service Discovery architecture. By utilizing **Consul** and **Envoy**, the infrastructure now supports automatic node registration, health-based routing, and zero-touch configuration updates for services like Keycloak, Minio, and Wazuh.

---

## 🏗️ Architecture Overview

The system follows a **"Register -> Discover -> Automate"** lifecycle:
1.  **Registration:** Backend nodes (Keycloak, etc.) announce their presence via a local Consul Agent.
2.  **Discovery:** Consul creates a dynamic DNS record (`.service.consul`) for all healthy nodes.
3.  **Automation:** `consul-template` on the Envoy server watches the Consul catalog and rewrites the Envoy configuration in real-time.
4.  **Routing:** Envoy uses `STRICT_DNS` to resolve the backends via the local Consul DNS interface.

---

## 🛠️ Implementation Stages

### Stage 1: The "Reporter" (Service Registration)
Each backend server (e.g., **.30** and **.163**) must tell the cluster it is alive.

**1. Install & Join Agent:**
The Consul agent must be running and joined to the LDC cluster.
```bash
# Verify agent status
consul members
```


**2. Define the Service (/etc/consul.d/keycloak.json):**
This file tells Consul exactly what service is running and how to check its health.
```
{
  "service": {
    "name": "keycloak",
    "port": 8080,
    "token": "fec50ca6-6b33-99fe-d522-cb80ec89d6f4",
    "check": {
      "id": "keycloak-check",
      "name": "Keycloak Port Check",
      "tcp": "localhost:8080",
      "interval": "10s",
      "timeout": "2s"
    }
  }
}
```

### - Apply: consul reload


### Stage 2: The "Writer" (Consul Templating)
On the Envoy server (** .154**), we eliminate manual YAML edits using consul-template.

**1. The Template Logic (envoy.yaml.ctmpl):**
The template uses a loop to find all registered services and map them to DNS names.

```
{{ range services }}
  - name: "{{ .Name }}"
    type: STRICT_DNS
    lb_policy: ROUND_ROBIN
    load_assignment:
      cluster_name: "{{ .Name }}"
      endpoints:
      - lb_endpoints:
        - endpoint:
            address:
              socket_address:
                address: "{{ .Name }}.service.consul"
                port_value: {{ .Port }}
{{ end }}
```

### Stage 3: The "Gateway" (Envoy Routing)
Envoy is configured to treat the Consul DNS name as the source of truth.

**1. STRICT_DNS Configuration:**
By setting the cluster type to STRICT_DNS, Envoy automatically queries the local Consul DNS (port 8600) to find the current healthy IPs for any given service.

**2. Proof of Concept Verification:**
Run these commands to confirm the bridge is working:
```
dig @127.0.0.1 -p 8600 keycloak.service.consul +short
```
### 🚀 Scaling Protocol (New Servers)
To add a new server (e.g., Minio or Wazuh):

1. Install Consul Agent on the new node.
2. Place the appropriate <service>.json in /etc/consul.d/.
3. **Result**: Consul DNS updates $\rightarrow$ consul-template detects change $\rightarrow$ Envoy reloads $\rightarrow$ Traffic flows. Zero manual IP changes required.
