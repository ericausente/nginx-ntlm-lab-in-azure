
# NGINX Plus + NTLM Lab on Azure

## 🧾 Objective
Deploy a test environment on Azure where NGINX Plus proxies requests to Windows IIS VMs protected with NTLM authentication. The setup uses Azure Load Balancer in front of multiple NGINX Plus instances.

---

## 🔧 Architecture Overview

```
[Client]
   ↓
[Azure Load Balancer]
   ↓
[NGINX Plus VMs (Ubuntu)]
   ↓
[Windows IIS VMs with NTLM Auth]
```

---

## 🧱 Step-by-Step Setup

### 1. Resource Group

```bash
az group create --name ntlm-lab-rg --location eastus
```

---

### 2. Networking

#### Virtual Network and Subnets
```bash
az network vnet create --resource-group ntlm-lab-rg --name ntlm-vnet --address-prefix 10.0.0.0/16 --subnet-name nginx-subnet --subnet-prefix 10.0.1.0/24
az network vnet subnet create --resource-group ntlm-lab-rg --vnet-name ntlm-vnet --name iis-subnet --address-prefix 10.0.2.0/24
```

#### NSGs
```bash
az network nsg create --resource-group ntlm-lab-rg --name nginx-nsg
az network nsg create --resource-group ntlm-lab-rg --name iis-nsg
```

Allow rules:
```bash
az network nsg rule create --resource-group ntlm-lab-rg --nsg-name nginx-nsg --name AllowHTTP --priority 100 --destination-port-ranges 80 443 22 3000 --access Allow --protocol Tcp --direction Inbound
```

---

### 3. Windows IIS VMs with NTLM

#### Provision
```bash
az vm create --resource-group ntlm-lab-rg --name iis-vm01 --image Win2019Datacenter --size Standard_B2s --vnet-name ntlm-vnet --subnet iis-subnet --admin-username azureuser --admin-password <YourSecurePassword> --nsg iis-nsg
```

#### Configure
- Enable IIS via PowerShell:
```powershell
Install-WindowsFeature -name Web-Server -IncludeManagementTools
```
- In IIS Manager:
  - Enable **Windows Authentication**
  - Disable **Anonymous Authentication**
  - Set **NTLM** as top provider

---

### 4. NGINX Plus VMs

#### Provision
```bash
az vm create --resource-group ntlm-lab-rg --name nginx-vm01 --image Ubuntu2204 --size Standard_B2s --vnet-name ntlm-vnet --subnet nginx-subnet --admin-username azureuser --ssh-key-values ~/.ssh/id_rsa.pub --nsg nginx-nsg
```

#### Install NGINX Plus
```bash
sudo mkdir -p /etc/ssl/nginx
sudo cp nginx-repo.crt nginx-repo.key /etc/ssl/nginx/

sudo tee /etc/apt/sources.list.d/nginx-plus.list <<EOF
deb [signed-by=/etc/ssl/nginx/nginx-repo.crt] https://plus-pkgs.nginx.com/ubuntu jammy nginx-plus
EOF

sudo apt update
sudo apt install nginx-plus
```

#### Configure `/etc/nginx/conf.d/default.conf`

```nginx
upstream iis_backend {
    server 10.0.2.4;
    server 10.0.2.5;
    ntlm;
}

server {
    listen 80;

    location / {
        proxy_pass http://iis_backend;
        proxy_http_version 1.1;
        proxy_set_header Connection "";
        proxy_set_header Host $host;
        proxy_set_header Authorization $http_authorization;
    }
}
```

---

### 5. Azure Load Balancer

#### Create LB
```bash
az network lb create --resource-group ntlm-lab-rg --name nginx-lb --sku Standard --frontend-ip-name nginxFrontend --backend-pool-name nginxBackendPool --vnet-name ntlm-vnet --subnet nginx-subnet
```

#### Add NGINX VMs to Backend Pool
```bash
az network nic ip-config address-pool add --address-pool nginxBackendPool --ip-config-name ipconfig1 --nic-name nginx-vm01VMNic --resource-group ntlm-lab-rg --lb-name nginx-lb
```

#### Health Probe
```bash
az network lb probe create --resource-group ntlm-lab-rg --lb-name nginx-lb --name nginxHealthProbe --protocol Http --port 80 --path /
```

#### Load Balancing Rule
```bash
az network lb rule create --resource-group ntlm-lab-rg --lb-name nginx-lb --name nginxHttpRule --protocol Tcp --frontend-port 80 --backend-port 80 --frontend-ip-name nginxFrontend --backend-pool-name nginxBackendPool --probe-name nginxHealthProbe
```

#### Enable Persistence
```bash
az network lb rule update --resource-group ntlm-lab-rg --lb-name nginx-lb --name nginxHttpRule --load-distribution SourceIP
```

---

### 6. Validation Checklist

| Checkpoint | Description |
|-----------|-------------|
| NTLM Config | Enabled in IIS |
| NGINX Config | Uses `ntlm` directive, no `keepalive` |
| Load Balancer | Source IP persistence enabled |
| Connectivity | Curl or browser access returns authenticated page |

---

## ✅ Next Steps

- Add TLS termination or passthrough
- Add monitoring (NGINX Agent or NIM)
- Automate with Terraform (coming next)
