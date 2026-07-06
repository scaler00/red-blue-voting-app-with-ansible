# TechCorp Kubernetes Lab — Ansible Automation

End-to-end Ansible project that provisions a 3-node Kubernetes cluster on AWS EC2 and deploys the classic Voting Application — designed for a **90-minute live coding class**.

---

## Cluster Layout

| Host        | Role                     | Min Spec      |
|-------------|--------------------------|---------------|
| master01    | Kubernetes Control Plane | 2 vCPU / 4 GB |
| worker-red  | Worker Node              | 2 vCPU / 4 GB |
| worker-blue | Worker Node              | 2 vCPU / 4 GB |

OS: Ubuntu 24.04 LTS | Runtime: containerd | CNI: Calico

---

## Project Structure

```
techcorp-k8s/
├── ansible.cfg
├── inventory.ini
├── site.yml
├── vault.yml
├── group_vars/
│   └── all.yml
├── templates/
│   ├── flask-secret.yaml.j2
│   └── namespace.yaml.j2
└── roles/
    ├── common/          ← OS prep + containerd + K8s packages
    │   ├── tasks/
    │   ├── handlers/
    │   └── templates/
    ├── cluster/         ← kubeadm init (master) + join (workers)
    │   ├── tasks/
    │   └── handlers/
    └── application/     ← git clone + Vault + kubectl apply
        └── tasks/
```

---

## Playbook Flow

```
Play 1 — k8s_cluster
    common  →  OS baseline, containerd, kubeadm/kubelet/kubectl
         ↓
Play 2 — k8s_cluster
    cluster →  master: kubeadm init, Calico, join command
               workers: kubeadm join
         ↓
Play 3 — k8s_master
    application → git clone, Secret, Namespace, kubectl apply
```

---

## Quick Start

### 1 — Prerequisites (controller machine)

```bash
sudo apt update && sudo apt install ansible git sshpass python3-pip -y
ansible --version
```

### 2 — Configure Inventory

Edit `inventory.ini` and replace placeholder IPs with your EC2 public IPs:

```ini
master01    ansible_host=<MASTER_IP>
worker-red  ansible_host=<WORKER_RED_IP>
worker-blue ansible_host=<WORKER_BLUE_IP>
```

### 3 — SSH Access

```bash
ssh-keygen
ssh-copy-id ubuntu@<MASTER_IP>
ssh-copy-id ubuntu@<WORKER_RED_IP>
ssh-copy-id ubuntu@<WORKER_BLUE_IP>
```

### 4 — Test Connectivity

```bash
ansible all -m ping
```

### 5 — Encrypt the Vault

```bash
ansible-vault encrypt vault.yml
```

### 6 — Run the Playbook

```bash
ansible-playbook site.yml --ask-vault-pass
```

---

## Files Students Touch During Class (~10 files)

```
inventory.ini
ansible.cfg
site.yml
group_vars/all.yml
vault.yml
roles/common/tasks/main.yml
roles/cluster/tasks/main.yml
roles/application/tasks/main.yml
templates/flask-secret.yaml.j2
README.md
```

---

## Class Timeline (90 Minutes)

| Time   | Topic                                                                               |
|--------|-------------------------------------------------------------------------------------|
| 10 min | Architecture, inventory, SSH, project structure                                     |
| 20 min | `common` role — OS prep, containerd, kubeadm/kubelet/kubectl, loops, handlers       |
| 25 min | `cluster` role — `kubeadm init`, `register`, `set_fact`, `hostvars`, workers join   |
| 20 min | `application` role — git clone, Vault, Jinja2 Secret template, `kubectl apply`      |
| 10 min | Run the playbook, verify nodes and pods                                             |
|  5 min | Re-run to demonstrate idempotency                                                   |

---

## AWS Security Group Rules

| Protocol | Port Range   | Purpose              |
|----------|--------------|----------------------|
| TCP      | 22           | SSH                  |
| TCP      | 6443         | Kubernetes API       |
| TCP      | 2379–2380    | etcd                 |
| TCP      | 10250–10259  | Kubelet / Controller |
| TCP      | 30000–32767  | NodePort Services    |
| ICMP     | All          | Ping (optional)      |

---

## Ansible Concepts Covered

| Concept       | Where Used                                         |
|---------------|----------------------------------------------------|
| Inventory     | `inventory.ini`                                    |
| Host groups   | `k8s_master`, `k8s_workers`                        |
| Variables     | `group_vars/all.yml`, `defaults/main.yml`          |
| Facts         | `ansible_default_ipv4.address`                     |
| Roles         | `common`, `cluster`, `application`                 |
| Loops         | Package installation in `common`                   |
| Handlers      | Restart containerd / kubelet                       |
| Conditionals  | `when: inventory_hostname in groups['k8s_master']` |
| `register`    | Capture `kubeadm init` output, join command        |
| `set_fact`    | Store join command for workers to consume          |
| `hostvars`    | Workers read join command from master01            |
| Templates     | Kubernetes Secret and Namespace manifests          |
| Jinja2        | `{{ flask_secret_key }}` from vault in template    |
| Vault         | Encrypt `FLASK_SECRET_KEY` and `db_password`       |
| Git module    | Clone the Voting App repository                    |
| Idempotency   | Safe re-execution — most tasks report **ok**       |

---

## Verify the Cluster

```bash
kubectl get nodes        # master01, worker-red, worker-blue
kubectl get pods -A      # all system and app pods
kubectl get svc          # services + NodePorts
kubectl get ns           # voting namespace
```

---

## Idempotency Demo

```bash
ansible-playbook site.yml --ask-vault-pass
```

Most tasks report **ok** instead of **changed** — demonstrating Ansible's idempotent design.
