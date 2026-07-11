1. Crear carpeta y entrar
cd $HOME\Documents
mkdir aks-project
cd aks-project

2. Revisar herramientas y entrar a Azure
az version
terraform version
kubectl version --client
git --version

az login
az account show --output table

Verificar que todo esté instalado y entrar a suscripción de Azure.
3. Guardar variables
$LOCATION = "eastus"
$RG_NAME = "rg-aks-devops-rudy"
$AKS_NAME = "aks-devops-rudy"
$DNS_PREFIX = "aks-devops-rudy"
$SP_NAME = "sp-aks-devops-rudy"

$SUBSCRIPTION_ID = az account show --query id -o tsv
$TENANT_ID = az account show --query tenantId -o tsv

Se definen nombres de cada variable
4. Habilitar servicios necesarios de Azure
az provider register --namespace Microsoft.ContainerService --wait
az provider register --namespace Microsoft.Compute --wait
az provider register --namespace Microsoft.Network --wait
az provider register --namespace Microsoft.ManagedIdentity --wait

5. Crear permisos para Terraform
$SP_JSON = az ad sp create-for-rbac --name $SP_NAME --role Contributor --scopes "/subscriptions/$SUBSCRIPTION_ID" --output json | ConvertFrom-Json

$env:ARM_CLIENT_ID = $SP_JSON.appId
$env:ARM_CLIENT_SECRET = $SP_JSON.password
$env:ARM_TENANT_ID = $TENANT_ID
$env:ARM_SUBSCRIPTION_ID = $SUBSCRIPTION_ID

Se crea un Service Principal para que Terraform pueda crear recursos en Azure.
6. Crear llave SSH
ssh-keygen -t rsa -b 4096 -f .\aks-devops_rsa

7. Crear main.tf
Primero abre el archivo:
notepad main.tf

Pega esto dentro de main.tf y guarda:
terraform {
  required_providers {
    azurerm = {
      source  = "hashicorp/azurerm"
      version = "~> 4.0"
    }
  }
}

provider "azurerm" {
  features {}
}

resource "azurerm_resource_group" "main" {
  name     = "rg-aks-devops-rudy"
  location = "eastus"
}

resource "azurerm_kubernetes_cluster" "main" {
  name                = "aks-devops-rudy"
  location            = azurerm_resource_group.main.location
  resource_group_name = azurerm_resource_group.main.name
  dns_prefix          = "aks-devops-rudy"

  default_node_pool {
    name       = "system"
    node_count = 2
    vm_size    = "Standard_D4as_v5"
    os_sku     = "Ubuntu"
  }

  linux_profile {
    admin_username = "azureuser"

    ssh_key {
      key_data = file("./aks-devops_rsa.pub")
    }
  }

  identity {
    type = "SystemAssigned"
  }
}

output "resource_group_name" {
  value = azurerm_resource_group.main.name
}

output "aks_cluster_name" {
  value = azurerm_kubernetes_cluster.main.name
}

output "kube_admin_config" {
  value     = azurerm_kubernetes_cluster.main.kube_admin_config_raw
  sensitive = true
}


8. Crear el cluster con Terraform
terraform init
terraform fmt
terraform validate
terraform plan -out aks.tfplan
terraform apply aks.tfplan

9. Conectar kubectl al cluster
az aks list --resource-group rg-aks-devops-rudy --output table
az aks get-credentials --resource-group rg-aks-devops-rudy --name aks-devops-rudy --admin --file .\kubeconfig --overwrite-existing
Test-Path .\kubeconfig

$env:KUBECONFIG = (Resolve-Path .\kubeconfig).Path

kubectl get nodes


10. Crear nginx.yaml
mkdir k8s
notepad .\k8s\nginx.yaml

Pega esto dentro de nginx.yaml y guarda:
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
spec:
  replicas: 2
  selector:
    matchLabels:
      app: nginx-demo
  template:
    metadata:
      labels:
        app: nginx-demo
    spec:
      containers:
      - name: nginx
        image: nginx:stable
        ports:
        - containerPort: 80
---
apiVersion: v1
kind: Service
metadata:
  name: nginx-service
spec:
  type: LoadBalancer
  selector:
    app: nginx-demo
  ports:
  - port: 80
    targetPort: 80


11. Desplegar nginx y probar IP
kubectl apply -f .\k8s\nginx.yaml
kubectl get pods
kubectl get svc nginx-service

$EXTERNAL_IP = kubectl get svc nginx-service -o jsonpath="{.status.loadBalancer.ingress[0].ip}"
echo $EXTERNAL_IP

Invoke-WebRequest -Uri "http://$EXTERNAL_IP"
start "http://$EXTERNAL_IP"

Si EXTERNAL-IP sale pending, espera 1-3 minutos y vuelve a ejecutar kubectl get svc nginx-service.
12. Crear checklist para la entrega
notepad checklist.md

Pega esto dentro de checklist.md y guarda:
# AKS Project Checklist

1. Environment configured: Azure CLI, Terraform, kubectl, Git, az login.
2. Permissions granted: Service Principal with Contributor role.
3. Resource Group created with Terraform.
4. Storage account: optional, not used.
5. SSH keys created with ssh-keygen -t rsa -b 4096.
6. Terraform script includes azurerm provider, resource group, AKS and kubeconfig output.
7. Terraform executed with init, plan and apply. Cluster verified with kubectl get nodes.
8. Application deployed with Deployment + LoadBalancer Service. External IP tested.

13. Subir a GitHub
Primero crea un repo vacío en GitHub llamado aks-project. Después ejecuta:
notepad .gitignore

Pega esto en .gitignore y guarda:
.terraform/
*.tfstate
*.tfstate.*
kubeconfig
*.tfplan

Luego corre estos comandos:
git init
git branch -M main
git add .
git commit -m "Create AKS cluster with Terraform and deploy nginx"

git remote add origin https://github.com/rudysan91-crypto/aks-project.git 
git push -u origin main

14. Borrar recursos al terminar
terraform destroy -auto-approve

Esto borra el AKS para que no siga consumiendo crédito.


