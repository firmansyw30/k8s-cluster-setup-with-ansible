# 🧠 Kubernetes Bare-Metal Nginx Gateway API Demo

This project showcases a production-grade approach to routing and exposing an Nginx application on a custom, bare-metal Kubernetes cluster. Moving away from the legacy Ingress standard, this demo utilizes the modern **Kubernetes Gateway API** alongside **Envoy Gateway** and **MetalLB** to handle North-South traffic securely and scalably.

It provides a step-by-step progression from initial cluster bootstrap via Ansible, to internal Layer-2 network routing, and finally to full public internet exposure using an external Cloud Load Balancer.

## 🚀 Features
- **Modern Gateway API**: Replaces legacy Ingress controllers with the CNCF-recommended Gateway API for robust, role-oriented traffic routing.
- **Bare-Metal Load Balancing**: Utilizes MetalLB in Layer 2 mode to dynamically assign routable private IPs to services without a managed cloud provider.
- **Envoy Proxy Edge Routing**: Implements Envoy Gateway to act as the highly performant edge proxy handling incoming HTTP requests.
- **Progressive Exposure**: Demonstrates three tiers of access: internal port-forwarding, direct NodePort access, and High-Availability Cloud Load Balancing.
- **Infrastructure as Code**: Fully automated native Kubernetes cluster bootstrapping using Ansible.

## 🛠️ Tech Stack
- **Kubernetes (v1.34)**: Core container orchestration.
- **Ansible**: Infrastructure-as-code bootstrap for the native K8s cluster.
- **MetalLB**: Internal network load balancer for bare-metal environments.
- **Envoy Gateway**: Gateway API controller and proxy fleet.
- **Nginx**: Target web application (sample deployment).
- **IDCloudHost (or any Cloud Provider)**: External infrastructure and public load balancing.

---

## 📦 Installation & Setup Runbook

### Prerequisites
- Bare-metal VMs (1 Control Plane, 1+ Worker Nodes) running Ubuntu.
- `ansible` and `kubectl` installed on your local control machine.

### Phase 0: Cluster Bootstrap (Ansible)
*This phase provisions the raw bare-metal servers into a functional Kubernetes cluster.*

1. **Clone the repository:**
   ```bash
   git clone https://github.com/firmansyw30/kubernetes-nginx-gateway-demo.git
   cd kubernetes-nginx-gateway-demo
2. **Run the Ansible Playbook:**
  Update the inventory.ini file with your specific node IP addresses, then run the bootstrap playbook to install Containerd, Kubeadm, Kubelet, and the Calico CNI:
  ```bash
  ansible-playbook -i inventory.ini setup-k8s.yaml
  ```
3. **Configure Local Kubeconfig:**
  The playbook automatically fetches the kubeconfig to your local directory. Set it as your default so you can interact with the cluster:
  ```bash
  mkdir -p ~/.kube
  cp ./kubeconfig ~/.kube/config
  ```

### Phase 1: Internal Network Routing (MetalLB)
*This phase assigns private, routable IPs within the internal cloud network.*

1. **Install MetalLB (Controller & Speaker):**
   ```bash
   kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.14.3/config/manifests/metallb-native.yaml
   ```
2. **Apply the MetalLB IP Pool configuration:**
  Customize based on requirement, then apply using
  ```bash
  kubectl apply -f metallb-config.yaml
  ```
3. **Deploy the Nginx application:**
  Test deployment using nginx
  ```bash
  kubectl apply -f nginx-demo.yaml
  ```
4. **Quick Verification (Local Development):**
  Use port-forwarding to securely tunnel into the cluster:
  ```bash
  kubectl port-forward svc/nginx-loadbalancer 8080:80
  ```
*Note: Access http://localhost:8080 locally, or SSH into a node and curl the MetalLB EXTERNAL-IP.*

### Phase 2: Public Internet Exposure via NodePort (Gateway API)
*This phase routes public traffic directly to the VMs using the Envoy Gateway.*

1. **Install the stable Gateway API CRDs:**
   ```bash
   kubectl apply -f https://github.com/kubernetes-sigs/gateway-api/releases/download/v1.2.1/standard-install.yaml
   ```
2. **Install the matching stable Envoy Gateway Controller:**
  (Note: --server-side is required to bypass annotation size limits for large CRDs).
  ```bash
  kubectl apply --server-side -f https://github.com/envoyproxy/gateway/releases/download/v1.2.0/install.yaml
  ```
3. **Convert Nginx to an Internal Service:**
  Free up the MetalLB IP by patching the service to ClusterIP
  ```bash
  kubectl patch svc nginx-loadbalancer -p '{"spec": {"type": "ClusterIP"}}'
  ```
  Or change it manually on change it in the file `gateway-nginx-demo.yaml` the services from `LoadBalancer` into `ClusterIP` Then apply using 
  ```bash
  kubectl apply -f gateway-nginx-demo.yaml
  ```
4. **Test NodePort Access**
  Identify the NodePort assigned to the envoy-default-external-gateway service (e.g., 31241).
  Access the application via http://<Node-Public-IP>:31241. (Ensure node firewalls allow this port).

### Phase 3: Production Public Ingress (Cloud Load Balancer)
*This phase achieves High Availability and clean URLs by placing a Cloud LB in front of the Gateway.*

1. **Provision an External Load Balancer via your cloud provider (e.g., IDCloudHost).**
2. **Configure Target Group & Listeners:**
  - Add your K8s Worker Nodes to the target pool.
  - Set the LB to listen on TCP Port 80 and forward traffic to your Gateway's TCP NodePort (e.g., 31241).
3. **Access the Application:**
  Traffic now flows seamlessly from the public internet: http://<Load-Balancer-Public-IP>.

## 📖 Architecture Glossary
- *North-South Traffic*: Traffic flowing from the outside internet (South) into the cluster, and responses flowing back (North) to the user.
- *Gateway API*: The modern K8s standard replacing Ingress. It splits routing into GatewayClass (infrastructure), Gateway (listeners), and HTTPRoute (routing rules).
- *Envoy Gateway*: The software implementation of the Gateway API that physically proxies the network packets.
- *MetalLB*: A software-based load balancer for bare-metal K8s. It assigns routable internal network IPs to services so they can be accessed across the local LAN without NodePorts.
- *NodePort*: A primitive way to expose a service. Kubernetes opens a specific high-numbered port on every single node in the cluster to forward traffic to the service.

## 📂 Project Structure
.
├── inventory.ini          # Ansible inventory file
├── setup-k8s.yaml         # Ansible playbook for cluster bootstrap
├── metallb-config.yaml    # MetalLB IPAddressPool and L2Advertisement rules
├── nginx-demo.yaml        # Nginx Deployment and ClusterIP Service
├── gateway-nginx.yaml     # GatewayClass, Gateway, and HTTPRoute manifests
└── README.md

## 🤝 Contributing
Contributions are welcome! If you have ideas, bug fixes, or want to add TLS/Cert-Manager support, please submit a pull request.

## 📬 Contact
For any questions or concerns, please reach out at firmansyahwicaksono30@gmail.com or connect on LinkedIn.
