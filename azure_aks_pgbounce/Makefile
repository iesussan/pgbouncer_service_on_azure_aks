# Variables Generales
TENANT_ID = 3949b967-1cb4-4105-b7d9-4fd1a7878ac3
#TENANT_ID ?= cbc2c381-2f2e-4d93-91d1-506c9316ace7
subscription_id = 98fc3001-5ba8-4215-acde-94ca7d64d2fa
#subscription_id ?= 1bba1d8d-f619-4f6d-8cc5-4f328ac370bd
RESOURCE_GROUP ?= az-sena-prd-aks-moodle-e1-000
LOCATION ?= eastus

# Variables Log Analytics Workspace
AZURE_LOG_ANALYTICS_WORKSPACE_NAME ?= law-$(LOCATION)

# Variables Generales de AKS
AKS_CLUSTER_NAME ?= az-sena-prd-aks-pgbouncer-e1-000
KUBERNETES_VERSION ?= 1.28.5

#AZURE CONTAINER REGISTRY
AZURE_CONTAINER_REGISTRY_NAME ?= azsenaprdcrpgbouncer
AZURE_CONTAINER_REGISTY_SKU ?= Premium

#AKS VNET
AKS_VNET_NAME ?= az-sena-prd-vnet-pgbouncer-e1-000
VNET_ADDRESS_SPACE ?= 172.24.100.0/22


#SYSTEM NODE POOL
SYSTEM_NODE_VM_SIZE ?= Standard_D4ds_v5
SYSTEM_NODE_COUNT ?= 2
SYSTEM_NODE_MIN_COUNT ?= 2
SYSTEM_NODE_MAX_COUNT ?= 5
SYSTEM_NODE_ZONES ?= 1 2 3
SYSTEM_NODE_POOL_VNET_SUBNET_NAME ?= subn-aks-system-01
SYSTEM_NODE_POOL_VNET_NODES_ADDRESS_SPACE ?= 172.24.100.0/24

#USER NODE POOL FOR PGBOUNCER
USER_NODE_PGBOUNCER_VM_SIZE ?= Standard_D4ds_v5
USER_NODE_PGBOUNCER_COUNT ?= 3
USER_NODE_PGBOUNCER_MIN_COUNT ?= 3
USER_NODE_PGBOUNCER_MAX_COUNT ?= 10
USER_NODE_PGBOUNCER_ZONES ?= 1 2 3
USER_NODE_POOL_PGBOUNCER_VNET_SUBNET_NAME = subn-user-pgbouncer-01
USER_NODE_POOL_PGBOUNCER_NAME ?= pgbouncerp01
USER_NODE_POOL_PGBOUNCER_VNET_NODES_ADDRESS_SPACE ?= 172.24.101.0/24

#AZURE BASTION
BASTION_PUBLIC_IP_NAME ?= bastion-public-ip
BASTION_ADDRESS_PREFIXES ?= 172.24.102.0/27

#JUMP SERVER
JUMP_SERVER_NAME ?= jumpserver
JUMP_SERVER_IMAGE ?= Ubuntu2204
JUMP_SERVER_ADMIN_USERNAME ?= azureuser
JUMP_SERVER_SSH_KEY ?= ~/.ssh/id_rsa.pub
JUMP_SERVER_VNET_SUBNET_NAME ?= subn-jumpserver-aks
JUMP_SERVER_VNET_SUBNET_ADDRESS_PREFIX ?= 172.24.102.32/27 
JUMP_SERVER_NSG_NAME ?= jumpserver-aks-nsg

#INGRESS CONTROLLER
INGRESS_CONTROLLER_NAMESPACE ?= ingress-controller

#DNS ZONE Y CETIFICADO
DNS_ZONE_NAME ?= senaprivatecloud.int
INGRESS_CERTIFICATE_NAME ?= senaprivatecloudint

#AZURE MANAGED GRAFANA
AZURE_MANAGED_GRAFANA_NAME ?= azsenaprdgfnpgb1000

#PGBOUNCER VARIABLES
PGBOUNCER_IMAGE ?= sena-pgbouncer:1.22.1

.PHONY: check_azure_cli validate_resource_group create_resource_group create_azure_container_registry

check_azure_cli:
ifeq (, $(shell which az))
	@echo "Error: Azure CLI no se encuentra en el PATH del sistema. Por favor, instálelo y vuelva a intentarlo."
	exit 1
else
	@echo "Azure CLI encontrado"
endif

validate_resource_group:
ifndef RESOURCE_GROUP
	@echo "Error: La variable RESOURCE_GROUP no está definida. Por favor, defínala y vuelva a intentarlo."
	exit 1
else
	@echo "La variable RESOURCE_GROUP está definida como: $(RESOURCE_GROUP)"
endif

create_resource_group: check_azure_cli validate_resource_group
	@echo "Creando grupo de recursos $(RESOURCE_GROUP) en la ubicación $(LOCATION)..."
	az group create --name $(RESOURCE_GROUP) --location $(LOCATION)
	@echo "Grupo de recursos creado con éxito."

create_azure_container_registry:
	az acr create --name $(AZURE_CONTAINER_REGISTRY_NAME) --resource-group $(RESOURCE_GROUP) --sku $(AZURE_CONTAINER_REGISTY_SKU)

create_aks_vnet: 
	@echo "Creando la VNET $(AKS_VNET_NAME) para el node pool system del cluster de AKS $(AKS_CLUSTER_NAME)"
	az network vnet create \
		--name $(AKS_VNET_NAME) \
		--resource-group $(RESOURCE_GROUP) \
		--location $(LOCATION) \
 		--address-prefixes $(VNET_ADDRESS_SPACE) -o none 
	
	az network vnet subnet create \
		--name $(SYSTEM_NODE_POOL_VNET_SUBNET_NAME) \
		--resource-group $(RESOURCE_GROUP) \
		--vnet-name $(AKS_VNET_NAME) \
		--address-prefixes $(SYSTEM_NODE_POOL_VNET_NODES_ADDRESS_SPACE)

	az network vnet subnet create \
		--name $(USER_NODE_POOL_PGBOUNCER_VNET_SUBNET_NAME) \
		--resource-group $(RESOURCE_GROUP) \
		--vnet-name $(AKS_VNET_NAME) \
		--address-prefixes $(USER_NODE_POOL_PGBOUNCER_VNET_NODES_ADDRESS_SPACE)
	
create_log_analytics_workspace:
	@echo "Creando Log Analytics workspace..."

	az monitor log-analytics workspace create \
		--resource-group $(RESOURCE_GROUP) \
		--name law-$(LOCATION)

create_control_plane_managed_identity:
	@echo "Creando Managed Identity para el control plane..."
	az identity create \
		--resource-group $(RESOURCE_GROUP) \
		--name $(AKS_CLUSTER_NAME)-control-plane-identity \
		--location $(LOCATION)

create_kubelet_manage_identity:
	@echo "Creando Managed Identity para los nodos..."
	az identity create \
		--resource-group $(RESOURCE_GROUP) \
		--name $(AKS_CLUSTER_NAME)-kubelet-identity \
		--location $(LOCATION)

add_role_assignment_control_plane_manage_identity:
	@echo "Obteniendo la ID de la identidad manejada para el control plane..."
	$(eval CONTROLPLANE_MANAGE_IDENTITY_PRINCIPAL_ID=$(shell az identity show --name $(AKS_CLUSTER_NAME)-control-plane-identity --resource-group $(RESOURCE_GROUP) --query principalId  -o tsv))

	@echo "Obteniendo el ID del resource group..."
	$(eval RESOURCE_GROUP_ID=$(shell az group show --name $(RESOURCE_GROUP) --query id --output tsv))
	
	@echo "Asignando rol de Contributor a la Managed Identity del control plane..."
	az role assignment create --assignee "$(CONTROLPLANE_MANAGE_IDENTITY_PRINCIPAL_ID)" \
	--role "Contributor" \
	--scope "$(RESOURCE_GROUP_ID)"

add_role_assignment_kubelet_manage_identity:
	@echo "Obteniendo la ID de la identidad manejada para los nodos..."
	$(eval KUBELET_MANAGE_IDENTITY_PRINCIPAL_ID=$(shell az identity show --name $(AKS_CLUSTER_NAME)-kubelet-identity --resource-group $(RESOURCE_GROUP) --query principalId  -o tsv))

	@echo "Obteniendo el ID del resource group..."
	$(eval RESOURCE_GROUP_ID=$(shell az group show --name $(RESOURCE_GROUP) --query id --output tsv))
	
	@echo "Asignando rol de Managed Identity Operator a la Managed Identity de los nodos..."
	az role assignment create --assignee "$(KUBELET_MANAGE_IDENTITY_PRINCIPAL_ID)" \
	--role "Managed Identity Operator" \
	--scope "$(RESOURCE_GROUP_ID)"

create_azure_kubernetes_services:
	@echo "Obteniendo el ID de la Subnet del node pool system..."
	$(eval SYSTEM_SUBNET_NODE_ID = $(shell az network vnet subnet show --resource-group $(RESOURCE_GROUP) --vnet-name $(AKS_VNET_NAME) --name $(SYSTEM_NODE_POOL_VNET_SUBNET_NAME) --query id -o tsv))

	$(eval CONTROLPLANE_MANAGE_IDENTITY_PRINCIPAL_ID=$(shell az identity show --name $(AKS_CLUSTER_NAME)-control-plane-identity --resource-group $(RESOURCE_GROUP) --query id  -o tsv))
	$(eval KUBELET_MANAGE_IDENTITY_PRINCIPAL_ID=$(shell az identity show --name $(AKS_CLUSTER_NAME)-kubelet-identity --resource-group $(RESOURCE_GROUP) --query id  -o tsv))

	
	@echo "Creando el clúster AKS $(AKS_CLUSTER_NAME) con Managed Identity..."
	
	az aks create \
		--resource-group $(RESOURCE_GROUP) \
		--name $(AKS_CLUSTER_NAME) \
		--location $(LOCATION) \
		--kubernetes-version $(KUBERNETES_VERSION) \
		--zones $(SYSTEM_NODE_ZONES) \
		--node-count $(SYSTEM_NODE_COUNT) \
		--node-vm-size $(SYSTEM_NODE_VM_SIZE) \
		--network-plugin azure \
		--network-plugin-mode overlay \
		--vnet-subnet-id $(SYSTEM_SUBNET_NODE_ID) \
		--enable-managed-identity \
		--assign-identity $(CONTROLPLANE_MANAGE_IDENTITY_PRINCIPAL_ID) \
    	--assign-kubelet-identity $(KUBELET_MANAGE_IDENTITY_PRINCIPAL_ID) \
		--generate-ssh-keys \
		--load-balancer-sku standard \
    	--enable-private-cluster \
		--attach-acr $(AZURE_CONTAINER_REGISTRY_NAME) \
		--disable-public-fqdn \
		--enable-oidc-issuer \
		--tier Standard \
		--enable-cluster-autoscaler \
		--min-count $(SYSTEM_NODE_MIN_COUNT) \
		--max-count $(SYSTEM_NODE_MAX_COUNT) \
		--enable-image-cleaner \
		--disable-local-account \
		--enable-aad \
		--enable-azure-monitor-metrics

create_azure_kubernetes_services_user_node_pool_for_pgbouncer:
	$(eval SUBNET_USER_NODE_POOL_PGBOUNCER_ID = $(shell az network vnet subnet show --resource-group $(RESOURCE_GROUP) --vnet-name $(AKS_VNET_NAME) --name $(USER_NODE_POOL_PGBOUNCER_VNET_SUBNET_NAME) --query id -o tsv))
	
	az aks nodepool add \
			--resource-group $(RESOURCE_GROUP) \
			--cluster-name $(AKS_CLUSTER_NAME)  \
			--name $(USER_NODE_POOL_PGBOUNCER_NAME) \
			--node-count $(USER_NODE_PGBOUNCER_COUNT) \
			--vnet-subnet-id $(SUBNET_USER_NODE_POOL_PGBOUNCER_ID) \
			--mode User \
			--zones $(USER_NODE_PGBOUNCER_ZONES) \
			--enable-cluster-autoscaler \
			--min-count $(USER_NODE_PGBOUNCER_MIN_COUNT) \
			--max-count $(USER_NODE_PGBOUNCER_MAX_COUNT) \
			--node-vm-size $(USER_NODE_PGBOUNCER_VM_SIZE) \
			--labels "workload-type=pgbouncer"

activate_azure_kubernetes_services_monitoring:
	
	$(eval LAW_ID = $(shell az monitor log-analytics workspace show --resource-group $(RESOURCE_GROUP) --workspace-name $(AZURE_LOG_ANALYTICS_WORKSPACE_NAME) --query id -o tsv))
	
	az aks enable-addons \
	-a monitoring --name $(AKS_CLUSTER_NAME) \
	--resource-group $(RESOURCE_GROUP) \
	--workspace-resource-id $(LAW_ID)

create_azure_managed_grafana_linked_to_azure_monitor:
	az grafana create --name $(AZURE_MANAGED_GRAFANA_NAME) --resource-group $(RESOURCE_GROUP)


create_azure_dns_private_zone:
	az network private-dns zone create \
		--resource-group $(RESOURCE_GROUP) \
		--name $(DNS_ZONE_NAME)

assing_kubelet_permission_to_dns_zone:
	@echo "Obteniendo la ID de la zona DNS privada..."
	$(eval PRIVATE_DNS_ZONE_ID=$(shell az network private-dns zone show --resource-group $(RESOURCE_GROUP) --name $(DNS_ZONE_NAME) --query id -o tsv))
	$(eval KUBELET_MANAGE_IDENTITY_PRINCIPAL_ID=$(shell az identity show --name $(AKS_CLUSTER_NAME)-kubelet-identity --resource-group $(RESOURCE_GROUP) --query principalId  -o tsv))
	$(eval RESOURCE_GROUP_ID=$(shell az group show --name $(RESOURCE_GROUP) --query id --output tsv))

	az role assignment create --role "DNS Zone Contributor" \
	--assignee $(KUBELET_MANAGE_IDENTITY_PRINCIPAL_ID) \
	--scope $(PRIVATE_DNS_ZONE_ID)

	az role assignment create --role "Private DNS Zone Contributor" \
	--assignee $(KUBELET_MANAGE_IDENTITY_PRINCIPAL_ID) \
	--scope $(PRIVATE_DNS_ZONE_ID)

	az role assignment create --role "Reader" \
	--assignee $(KUBELET_MANAGE_IDENTITY_PRINCIPAL_ID) \
	--scope $(RESOURCE_GROUP_ID)

create_azure_bastion:
	az network public-ip create \
		--resource-group $(RESOURCE_GROUP) \
		--name $(BASTION_PUBLIC_IP_NAME) \
		--sku Standard \
		--location $(LOCATION)
	
	az network vnet subnet create \
		--name AzureBastionSubnet \
		--resource-group $(RESOURCE_GROUP) \
		--vnet-name $(AKS_VNET_NAME) \
		--address-prefixes $(BASTION_ADDRESS_PREFIXES)

	az network bastion create \
		--name bastion-$(LOCATION) \
		--resource-group $(RESOURCE_GROUP) \
		--location $(LOCATION) \
		--vnet-name $(AKS_VNET_NAME) \
		--public-ip-address $(BASTION_PUBLIC_IP_NAME)

create_jump_server_for_aks:

	az network vnet subnet create \
        --name $(JUMP_SERVER_VNET_SUBNET_NAME) \
        --resource-group $(RESOURCE_GROUP) \
        --vnet-name $(AKS_VNET_NAME) \
        --address-prefixes $(JUMP_SERVER_VNET_SUBNET_ADDRESS_PREFIX)
	
	az network nsg create \
        --resource-group $(RESOURCE_GROUP) \
        --name $(JUMP_SERVER_NSG_NAME)
	
	az network nsg rule create \
        --resource-group $(RESOURCE_GROUP) \
        --nsg-name $(JUMP_SERVER_NSG_NAME) \
        --name AllowSSH \
        --protocol Tcp \
        --priority 1000 \
        --destination-port-range 22 \
        --access Allow \
        --direction Inbound
	
	@echo "Creando la máquina virtual..."
	az vm create \
		--resource-group $(RESOURCE_GROUP) \
		--name $(JUMP_SERVER_NAME) \
		--image $(JUMP_SERVER_IMAGE) \
		--size Standard_DS4_v2 \
		--admin-username azureuser \
		--admin-password MyPassword123 \
		--public-ip-address "" \
		--nsg "" \
		--vnet-name $(AKS_VNET_NAME) \
		--subnet $(JUMP_SERVER_VNET_SUBNET_NAME)

	@echo "Instalando az cli y kubectl..."
	az vm run-command invoke \
        --resource-group $(RESOURCE_GROUP) \
        --name $(JUMP_SERVER_NAME) \
        --command-id RunShellScript \
        --scripts "curl -sL https://aka.ms/InstallAzureCLIDeb | sudo bash" \
                  "&& sudo az aks install-cli"


link_azure_dns_private_zone_to_vnet:
	$(eval PRIVATE_DNS_ZONE_ID=$(shell az network private-dns zone show --resource-group $(RESOURCE_GROUP) --name $(DNS_ZONE_NAME) --query id -o tsv))
	$(eval JUMP_SERVER_VNET_ID=$(shell az network vnet show --resource-group $(RESOURCE_GROUP) --name $(AKS_VNET_NAME) --query id -o tsv))

	az network private-dns link vnet create \
		--resource-group $(RESOURCE_GROUP) \
		--zone-name $(DNS_ZONE_NAME) \
		--name $(JUMP_SERVER_NAME) \
		--virtual-network $(JUMP_SERVER_VNET_ID) \
		--registration-enabled false

deploy_external_dns_service:

	$(eval IDENTITY_CLIENT_ID=$(shell az aks show --resource-group $RESOURCE_GROUP --name $AKS_CLUSTER_NAME \
 		--query "identityProfile.kubeletidentity.clientId" --output tsv))

	cat <<-EOF > azure.json
	{
	"tenantId": "$(az account show --query tenantId -o tsv)",
	"subscriptionId": "$(az account show --query id -o tsv)",
	"resourceGroup": "$RESOURCE_GROUP",
	"useManagedIdentityExtension": true,
	"userAssignedIdentityID": "$IDENTITY_CLIENT_ID"
	}
	EOF

	kubectl create namespace external-dns
	kubectl create secret generic azure-config-file --namespace "external-dns" --from-file azure.json


create_azure_dns_private_zone_deployment:
	@echo "Creating Azure DNS Private Zone Config..."
	@echo "apiVersion: v1\n\
kind: ServiceAccount\n\
metadata:\n\
  name: external-dns\n\
  namespace: external-dns\n\
---\n\
apiVersion: rbac.authorization.k8s.io/v1\n\
kind: ClusterRole\n\
metadata:\n\
  name: external-dns\n\
rules:\n\
  - apiGroups: [\"\"]\n\
    resources: [\"services\",\"endpoints\",\"pods\", \"nodes\"]\n\
    verbs: [\"get\",\"watch\",\"list\"]\n\
  - apiGroups: [\"extensions\",\"networking.k8s.io\"]\n\
    resources: [\"ingresses\"]\n\
    verbs: [\"get\",\"watch\",\"list\"]\n\
---\n\
apiVersion: rbac.authorization.k8s.io/v1\n\
kind: ClusterRoleBinding\n\
metadata:\n\
  name: external-dns-viewer\n\
roleRef:\n\
  apiGroup: rbac.authorization.k8s.io\n\
  kind: ClusterRole\n\
  name: external-dns\n\
subjects:\n\
  - kind: ServiceAccount\n\
    name: external-dns\n\
    namespace: external-dns\n\
---\n\
apiVersion: apps/v1\n\
kind: Deployment\n\
metadata:\n\
  name: external-dns\n\
  namespace: external-dns\n\
spec:\n\
  strategy:\n\
    type: Recreate\n\
  selector:\n\
    matchLabels:\n\
      app: external-dns\n\
  template:\n\
    metadata:\n\
      labels:\n\
        app: external-dns\n\
    spec:\n\
      serviceAccountName: external-dns\n\
      containers:\n\
        - name: external-dns\n\
          image: registry.k8s.io/external-dns/external-dns:v0.14.1\n\
          args:\n\
            - \"--source=service\"\n\
            - \"--source=ingress\"\n\
            - \"--provider=azure-private-dns\"\n\
            - \"--domain-filter=$(DNS_ZONE_NAME)\"\n\
            - \"--azure-resource-group=$(RESOURCE_GROUP)\"\n\
            - \"--azure-subscription-id=$(subscription_id)\"\n\
          volumeMounts:\n\
            - name: azure-config-file\n\
              mountPath: /etc/kubernetes\n\
              readOnly: true\n\
      volumes:\n\
        - name: azure-config-file\n\
          secret:\n\
            secretName: azure-config-file" > azure-external-dns-manifest.yaml 
	kubectl apply -f azure-external-dns-manifest.yaml
	@echo "Configuration file created successfully."


build_docker_image_for_pgbouncer:
	docker build -t $(AZURE_CONTAINER_REGISTRY_NAME).azurecr.io/$(PGBOUNCER_IMAGE) --platform linux/amd64 .

push_docker_image_to_azure_container_registry:
	# az acr update --name $(AZURE_CONTAINER_REGISTRY_NAME) --admin-enabled true
	
	# $(eval ACR_USERNAME=$(shell az acr credential show --name $(AZURE_CONTAINER_REGISTRY_NAME) --query username -o tsv))
	# $(eval ACR_PASSWORD=$(shell az acr credential show --name $(AZURE_CONTAINER_REGISTRY_NAME) --query passwords[0].value -o tsv))
	
	# @echo "ACR Username: $(ACR_USERNAME)"
	# @echo "ACR Password: $(ACR_PASSWORD)"
	
	az acr login --name $(AZURE_CONTAINER_REGISTRY_NAME)
	docker push $(AZURE_CONTAINER_REGISTRY_NAME).azurecr.io/$(PGBOUNCER_IMAGE)

create_pgbouncer_namespace:
	@echo "Creando el namespace para PgBouncer..."
	kubectl create namespace pgbouncer

create_pgbouncer_secret_for_configuration:
	@echo "Creando el secreto para la configuración de PgBouncer..."

	kubectl create secret generic pgbouncer-config --namespace pgbouncer \
		--from-literal=DB_NAME="moodle" \
		--from-literal=DB_HOST="testingdbmoodle.postgres.database.azure.com" \
		--from-literal=DB_PORT="5432" \
		--from-literal=DB_USER="azureuser" \
		--from-literal=DB_PASSWORD="MyPassword123" \
		--from-literal=MAX_CLIENT_CONN="20000" \
		--from-literal=POOL_SIZE="235" \
		--from-literal=POOL_MODE="transaction" \
		--from-literal=IGNORE_STARTUP_PARAMETERS="options" \
		--from-literal=SERVER_TLS_SSLMODE="require" \
		--from-literal=MIN_POOL_SIZE="235" \
		--from-literal=AUTH_TYPE="plain"

create_pgbouncer_deployment:
	@echo "Creando el deployment de PgBouncer..."
	kubectl apply -f pgbouncer/pgbouncer-deployment.yaml

create_pgbouncer_service:
	@echo "Creando el servicio de PgBouncer..."
	kubectl apply -f pgbouncer/pgbouncer-service.yaml



