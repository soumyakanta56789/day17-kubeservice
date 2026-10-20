# Kubernetes Service Routing & Diagnostics Guide (`hello`)

This project demonstrates declarative service routing, DNS resolution, load balancing, and endpoint lifecycles using the `hello` Service inside a Kubernetes cluster.

---

## 🛠️ Service Architecture

The project deploys a stable **ClusterIP** network interface to balance internal container traffic.

* **Service Name:** `hello`
* **Network Mode:** `ClusterIP` (Internal cluster access only)
* **Selector:** `app: hello`
* **Routing State:** Routes traffic to **three healthy backend addresses** managed via EndpointSlices.

### 🔌 Understanding Ports

| Port Type | Configuration | Purpose |
| :--- | :--- | :--- |
| **`port`** | `80` | The stable external port exposed by the Service to the rest of the cluster. |
| **`targetPort`** | `8080` | The internal backend port where your container application is listening for traffic. |

---

## 📋 Service Definition (`service.yaml`)

```yaml
apiVersion: v1
kind: Service
metadata:
  name: hello
spec:
  type: ClusterIP
  selector:
    app: hello
  ports:
    - port: 80
      targetPort: 8080
```

---

## 🛰️ Core Concepts & Components

### 📦 EndpointSlices
* **Automated Scaling:** Tracks the real-time IP network addresses of ready backend pods.
* **Fidelity:** Replaces legacy, monolithic endpoint structures to support high-scale clusters.

### 🌐 Service DNS Lookup
* **CoreDNS Resolution:** Resolves short names natively from inside any running pod.
* **Internal FQDN Format:** Resolves fully via `hello.default.svc.cluster.local`.

---

## 🔬 Diagnostic & Verification Scenarios

### 1. Launching a Temporary Testing Pod
To spin up an interactive debugging environment, deploy a busybox test container:
```bash
kubectl run tmp --image=busybox:1.36 --restart=Never --rm -it -- sh
```

### 2. Testing Internal DNS
* Run `nslookup hello` to verify internal cluster resolution.
* Ensure CoreDNS returns the exact `ClusterIP` mapped to the Service.

### 3. Verification Loop & Load Balancing
Run this request sequence inside your test container to evaluate endpoint routing:
```bash
for i in {1..6}; do 
  wget -qO- http://hello/hostname
  echo " - Request $i done"
done
```
* **Expected Result:** The loop prints out varying pod hostnames, proving that traffic balances across all available backends evenly.

### ♻️ Endpoint Lifecycle & Recovery
* **Pod Deletion:** Deleting an active backend pod immediately detaches its IP from the EndpointSlice.
* **Self-Healing:** A replacement pod spins up, passes readiness checks, and attaches automatically.

### ❌ Misconfiguration Recovery (Wrong Selector)
* **The Fault:** Applying a non-matching selector (e.g., `app=wrong`) empties the EndpointSlice values.
* **The Symptom:** Traffic requests fail instantly because the Service has zero active backend addresses.
* **The Fix:** Restoring `app: hello` to the manifest reconciles the configuration and updates traffic paths smoothly.
