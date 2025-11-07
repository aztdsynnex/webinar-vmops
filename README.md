# Fedora VM Hello-World Demo on OpenShift with GitOps

This repository demonstrates deploying a **Virtual Machine–based web server** on **OpenShift** using **GitOps** principles.
It shows how infrastructure as code can define, deploy, and manage virtualized workloads through OpenShift Virtualization and OpenShift GitOps (Argo CD).

The example provisions a **Fedora VM** running a simple Python-based web server that serves a “Hello from a VM on OpenShift 👋” page.
This highlights how Git-driven workflows can manage virtual machines just like containerized workloads.

> ⚠️ **Disclaimer – Not for Production Use**
> This setup is intended **for demonstration and learning purposes only**.
> The provided YAML files contain hardcoded credentials (e.g., user and password defined in cloud-init).
> In production environments, you must follow security best practices:
>
> * Never commit credentials in plaintext YAML or Git.
> * Use **OpenShift Secrets** or external secret management solutions (Vault, External Secrets Operator).
> * Enforce RBAC and limit namespace access.
> * Rotate and manage credentials securely.

---

## 1. Prerequisites

Ensure the following are available before deployment:

* An **OpenShift cluster** with a RWX-capable storage class
* **OpenShift Virtualization** operator installed and enabled
* **OpenShift GitOps (Argo CD)** operator installed
* CLI tools: `oc`, `git`, and (optionally) `argocd` installed on your workstation

---

## 2. Manual Test Deployment

Before automating with GitOps, the Fedora VM can be tested manually to confirm the configuration works.

### 2.1. Create a project (namespace)

```bash
oc new-project webinar-vmops
oc label namespace webinar-vmops argocd.argoproj.io/managed-by=openshift-gitops
```

### 2.2. Apply YAML manifests

The repository contains three key YAML files:

* `webinar-vm.yaml` – defines the Fedora VM with cloud-init startup script
* `webinar-service.yaml` – exposes the VM internally on port 80
* `webinar-route.yaml` – exposes the web service externally via OpenShift Route

Apply them manually:

```bash
oc apply -f webinar-vm.yaml
oc apply -f webinar-service.yaml
oc apply -f webinar-route.yaml
```

### 2.3. Verify VM status

Check that the virtual machine and its underlying pod are running:

```bash
oc -n webinar-vmops get vm,vmi,pod
```

Wait until the VM phase is `Running`.

### 2.4. Access the web server

Get the route URL:

```bash
oc -n webinar-vmops get route webapp-route -o jsonpath='{.spec.host}{"\n"}'
```

Open the URL in your browser.
You should see:

```
Hello from a VM on OpenShift 👋
```

---

## 3. Cloud-Init configuration

The VM’s behavior is driven by **cloud-init**, which bootstraps the Fedora OS on first boot.

Key actions performed:

* Creates the user `artzoul` with password `redhat123`
* Writes a small HTML file to `/opt/hello/index.html`
* Creates and enables a systemd service `/etc/systemd/system/hello-web.service`
* Starts a Python-based HTTP server on port 80

This ensures the web server is running automatically after each boot without manual login.

> 🔒 **Security Note:**
> In a real environment, user credentials should be injected via Kubernetes Secrets and mounted at runtime.
> Cloud-init can reference Secrets dynamically instead of embedding static passwords.

---

## 4. Moving to GitOps (Argo CD Integration)

Once manual verification succeeds, the same YAMLs can be managed automatically via **Argo CD**.

Example `Application` manifest:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: webinar-vmops
spec:
  destination:
    namespace: webinar-vmops
    server: https://kubernetes.default.svc
  source:
    repoURL: https://github.com/aztdsynnex/webinar-vmops.git
    path: .
    targetRevision: main
  project: default
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
```

Deploy the Application:

```bash
oc apply -f argo-application.yaml
```

Then log into the **Argo CD UI** to monitor synchronization and health of the deployed VM.

---

## 5. Interesting facts about this deployment

* The Fedora VM runs fully virtualized on OpenShift nodes using KubeVirt.
* The entire VM definition and configuration are stored in Git.
* The VM self-configures using cloud-init, requiring no manual intervention.
* OpenShift Services and Routes make the VM’s web service accessible externally.
* Argo CD continuously monitors and reconciles the desired state in Git with the cluster.
* The example intentionally uses static credentials to demonstrate configuration flow — **do not reuse this in production**.

---

**Author:** Artzoul
**Purpose:** Demonstrating VM-based application deployment through GitOps on OpenShift Virtualization.
**Security:** Demo only – replace all hardcoded credentials before using in any real environment.
