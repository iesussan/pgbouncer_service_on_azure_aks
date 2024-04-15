# Documentación de Implementación de Azure Kubernetes Service y PgBouncer

Este documento proporciona una guía paso a paso para desplegar un clúster de Kubernetes en Azure utilizando un Makefile, con la implementación de un servicio de PgBouncer para optimizar conexiones de bases de datos PostgreSQL. El enfoque detalla cómo utilizar diferentes recursos de Azure como AKS, VNETs, y servicios relacionados.

## Estructura del Proyecto

```
azure_aks_pgbounce/
├── Dockerfile
├── Makefile
└── entrypoint.sh
pgbouncer/
├── pgbouncer-deployment.yaml
├── pgbouncer-service.yaml
└── pod
```

## Requisitos Previos

- Azure CLI instalado localmente.
- Credenciales de Azure configuradas para acceso mediante Azure CLI.
- Docker instalado para la construcción de imágenes.

## Descripción de Variables en el Makefile

Las variables se configuran en el Makefile para adaptarse a los diferentes aspectos del despliegue, como la ubicación de los recursos, dimensiones del clúster, configuración de red, y más. Las variables incluyen configuraciones para el grupo de recursos, el clúster de AKS, la red VNET, las identidades manejadas, y más.

## Proceso de Despliegue

### 1. Validación de Azure CLI

Primero, se valida si Azure CLI está instalado en el sistema. Este paso es crucial para asegurar que todos los comandos subsecuentes se ejecuten correctamente.

```bash
cd azure_aks_pgbounce
make check_azure_cli
```

### 2. Creación de Grupo de Recursos

Se crea un grupo de recursos en Azure, donde todos los recursos asociados con el clúster y el despliegue se mantendrán.

```bash
make create_resource_group
```

### 3. Creación de Azure Container Registry

Se establece un registro de contenedores en Azure para alojar imágenes como la de PgBouncer.

```bash
make create_azure_container_registry
```

### 4. Configuración de la Red Virtual (VNET)

Se configura una VNET y subredes para el clúster de AKS, lo que incluye subredes para los nodos del sistema y los nodos de usuario específicos para PgBouncer.

```bash
make create_aks_vnet
```

### 5. Creación de Managed Identities

Se crean identidades manejadas para el plano de control y los nodos de kubelet, lo que ayuda a administrar permisos y accesos dentro de Azure sin necesidad de gestionar credenciales explícitas.

```bash
make create_control_plane_managed_identity
make create_kubelet_manage_identity
```

### 6. Despliegue del Clúster de AKS

Se despliega el clúster de Kubernetes especificando versiones, tamaños de nodos, y más. Este paso integra las identidades manejadas y configura el clúster para operar dentro de la VNET creada.

```bash
make create_azure_kubernetes_services
```

### 7. Adición de Pool de Nodos para PgBouncer

Se agrega un pool de nodos específico para PgBouncer en el clúster de AKS, configurado para autoescalado y con etiquetas específicas para su función.

```bash
make create_azure_kubernetes_services_user_node_pool_for_pgbouncer
```

### 8. Activación de Monitoreo de AKS

Se activa el monitoreo para el clúster utilizando Azure Monitor, lo que facilita la supervisión y gestión del rendimiento del clúster.

```bash
make activate_azure_kubernetes_services_monitoring
```

### 9. Despliegue de PgBouncer

Se construye la imagen de PgBouncer utilizando Docker, se sube al registro de Azure, y luego se despliegan el servicio y el deployment de PgBouncer en el clúster de AKS.

```bash
make build_docker_image_for_pgbouncer
make push_docker_image_to_azure_container_registry
make create_pgbouncer_namespace
make create_pgbouncer_secret_for_configuration
make create_pgbouncer_deployment
make create_pgbouncer_service
```

Este flujo de trabajo encapsula la configuración, despliegue y administración de un clúster de Kubernetes en Azure con PgBouncer como un componente clave para gestionar conexiones de base de datos, asegurando un enfoque estructurado y reproducible gracias al uso de un Makefile.

