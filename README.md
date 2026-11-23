# Despliegue de Grafana en Kubernetes con Minikube

Este proyecto implementa el despliegue de **Grafana** sobre un clúster local de **Kubernetes (Minikube)**, demostrando de manera práctica el funcionamiento de los conceptos fundamentales de **Deployment**, **ReplicaSet** y **Service (NodePort)**.

---

## 1. Requisitos Previos

- **Sistema Operativo:** Windows 10/11
- **Terminal:** PowerShell (ejecutado preferentemente como Administrador)
- **Virtualización / Driver:** Docker Desktop (con integración WSL2) o Hyper-V
- **Herramientas Instaladas:**
  - `minikube` (v1.30+ recomendado)
  - `kubectl` (v1.28+ recomendado)
  - `docker` (en ejecución si se usa el driver Docker)

---

## 2. Estructura del Proyecto

```text
Kubernetes-Grafana/
├── kubernetes/
│   ├── namespace.yaml      # Definición del namespace 'monitoring'
│   ├── deployment.yaml     # Deployment de Grafana (2 réplicas, probes, límites de recursos)
│   └── service.yaml        # Service tipo NodePort para exponer el puerto 3000
├── README.md               # Guía técnica y operativa del laboratorio
└── informe.md              # Informe académico formal para Integración de Sistemas
```

---

## 3. Preparación del Clúster (Minikube)

Antes de aplicar los manifiestos, asegúrate de que Minikube esté activo:

```powershell
# Iniciar Minikube (usando el driver de docker)
minikube start --driver=docker

# Verificar el estado del clúster
minikube status
kubectl get nodes
```

---

## 4. Instalación y Despliegue

Aplica los manifiestos YAML en orden lógico o mediante la carpeta completa:

```powershell
# Navegar a la carpeta del proyecto
cd c:\Users\ASUS\Downloads\procesos\Kubernetes-Grafana

# 1. Crear el Namespace 'monitoring'
kubectl apply -f kubernetes/namespace.yaml

# 2. Desplegar Grafana (Deployment y ReplicaSet automático)
kubectl apply -f kubernetes/deployment.yaml

# 3. Exponer el servicio (NodePort)
kubectl apply -f kubernetes/service.yaml

# Alternativa en un solo comando:
# kubectl apply -f kubernetes/
```

---

## 5. Acceso a Grafana

Tienes dos alternativas para acceder a la interfaz web de Grafana:

### Opción A: Mediante `kubectl port-forward` (Recomendada y universal en Windows)
```powershell
kubectl port-forward -n monitoring svc/grafana-service 3000:3000
```
- **URL en el navegador:** [http://localhost:3000](http://localhost:3000)
- **Credenciales por defecto:**
  - **Usuario:** `admin`
  - **Contraseña:** `admin` (solicitará cambiarla en el primer ingreso o puedes presionar "Skip").

### Opción B: Mediante URL de servicio de Minikube
```powershell
minikube service grafana-service -n monitoring --url
```

---

## 6. Guía de Ejecución de Pruebas (Paso a Paso)

Ejecuta los siguientes comandos en PowerShell para comprobar el ciclo de vida y la auto-recuperación de Kubernetes:

### Prueba 1: Desplegar recursos
```powershell
kubectl apply -f kubernetes/
```

### Prueba 2: Consultar Deployments
```powershell
kubectl get deployments -n monitoring
```
*Resultado esperado:* Deployment `grafana` con `2/2` réplicas READY.

### Prueba 3: Consultar ReplicaSets
```powershell
kubectl get replicasets -n monitoring
# o la forma abreviada:
kubectl get rs -n monitoring
```
*Resultado esperado:* Un ReplicaSet con prefijo `grafana-<hash>` gestionando 2 réplicas DESIRED y CURRENT.

### Prueba 4: Consultar Pods
```powershell
kubectl get pods -n monitoring -o wide
```
*Resultado esperado:* Dos Pods con estado `Running` y `1/1` en READY.

### Prueba 5: Comprobar el estado detallado de las réplicas
```powershell
kubectl describe deployment grafana -n monitoring
```

### Prueba 6: Acceso a la interfaz web
```powershell
kubectl port-forward -n monitoring svc/grafana-service 3000:3000
```
Abre en el navegador `http://localhost:3000` y valida la pantalla de bienvenida de Grafana.

### Prueba 7: Escalar el Deployment de 2 a 3 réplicas
```powershell
kubectl scale deployment grafana -n monitoring --replicas=3
kubectl get pods -n monitoring
```
*Resultado esperado:* Verás aparecer un tercer Pod en estado `ContainerCreating` y luego `Running`.

### Prueba 8: Eliminar manualmente uno de los Pods (Simular fallo)
```powershell
# Obtén el nombre exacto de uno de los pods
$POD_NAME = (kubectl get pods -n monitoring -o jsonpath='{.items[0].metadata.name}')
Write-Host "Eliminando pod: $POD_NAME"
kubectl delete pod $POD_NAME -n monitoring
```

### Prueba 9: Comprobar la auto-reparación (Self-Healing)
```powershell
kubectl get pods -n monitoring
```
*Resultado esperado:* El Pod seleccionado entra en estado `Terminating`, pero inmediatamente el ReplicaSet crea un Pod nuevo con un nombre distinto para mantener el número de réplicas en 3.

### Prueba 10: Restablecer a 2 réplicas
```powershell
kubectl scale deployment grafana -n monitoring --replicas=2
kubectl get pods -n monitoring
```
*Resultado esperado:* Uno de los tres Pods pasa a `Terminating` y quedan exactamente 2 Pods activos.

### Prueba 11: Comprobar el estado final del clúster
```powershell
kubectl get all -n monitoring
```

---

## 7. Eliminación de Recursos (Limpieza)

Para desmontar el laboratorio sin afectar otros servicios:

```powershell
# Eliminar todos los recursos del proyecto mediante los manifiestos
kubectl delete -f kubernetes/

# O eliminar directamente el namespace completo
kubectl delete namespace monitoring

# Opcional: Detener Minikube cuando termines de estudiar
minikube stop
```
