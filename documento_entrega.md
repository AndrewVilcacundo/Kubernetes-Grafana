# INFORME DE PRÁCTICA DE LABORATORIO
# INTEGRACIÓN DE SISTEMAS

---

**TEMA:** Configuración de Clúster (Kubernetes) – Deployment y ReplicaSet con Grafana  
**ESTUDIANTE:** Andrew Vilcacundo  
**DOCENTE:** [Nombre del Docente]  
**FECHA:** [Fecha de Entrega]  
**REPOSITORIO GITHUB:** [https://github.com/AndrewVilcacundo/Kubernetes-Grafana](https://github.com/AndrewVilcacundo/Kubernetes-Grafana)  

---

## 1. INTRODUCCIÓN

En la arquitectura de software contemporánea orientada a microservicios y sistemas distribuidos, la gestión manual de instancias de contenedores resulta inviable ante fallos de hardware o picos de demanda. Los orquestadores de contenedores resuelven este desafío automatizando el ciclo de vida completo de las aplicaciones, proporcionando balanceo de carga, escalamiento horizontal y mecanismos autónomos de autorreparación (*self-healing*).

El presente informe documenta el desarrollo práctico del despliegue y administración de la plataforma de observabilidad **Grafana** (v11.5.0) sobre un clúster local de **Kubernetes** administrado mediante **Minikube** en un entorno Windows 11. A través de este laboratorio se comprueba el funcionamiento y la jerarquía de los controladores **Deployment** y **ReplicaSet**, evaluando la resiliencia del sistema ante caídas forzadas y la capacidad de escalamiento dinámico.

---

## 2. OBJETIVOS

### 2.1. Objetivo General
Desplegar, configurar y evaluar la plataforma de observabilidad Grafana en un clúster local de Kubernetes (Minikube), validando el ciclo de vida de los controladores *Deployment* y *ReplicaSet* en términos de tolerancia a fallos, autorreparación y escalabilidad declarativa.

### 2.2. Objetivos Específicos
1. Diseñar e implementar manifiestos declarativos YAML para Namespace, Deployment (con sondas *liveness* y *readiness*, y límites de recursos) y Service (NodePort).
2. Evidenciar la relación de control entre el Deployment, su ReplicaSet subyacente y los Pods generados.
3. Demostrar el mecanismo de *self-healing* de Kubernetes simulando una falla mediante la eliminación manual de un Pod.
4. Ejecutar operaciones de escalamiento horizontal (de 2 a 3 réplicas y retorno a 2 réplicas) sin interrupción del servicio.
5. Analizar el rol estratégico y los casos de uso empresarial de Grafana en ecosistemas tecnológicos integrados.

---

## 3. MARCO TEÓRICO

### 3.1. Plataforma Seleccionada: Grafana
Grafana es una plataforma de software libre líder en la industria para el análisis, consulta y visualización de métricas, registros (*logs*) y trazas de telemetría. Diseñada para operar de manera desacoplada del motor de almacenamiento, permite conectar múltiples orígenes de datos simultáneos (Prometheus, InfluxDB, Elasticsearch, PostgreSQL, AWS CloudWatch) y estructurar tableros (*dashboards*) dinámicos en tiempo real indispensables para la ingeniería DevOps y SRE.

### 3.2. Conceptos Fundamentales de Kubernetes

#### Namespace
Mecanismo de aislamiento lógico dentro de un mismo clúster físico o virtual. Permite separar entornos (por ejemplo, desarrollo, producción o monitoreo), evitando conflictos de nombres entre recursos y facilitando la asignación de cuotas de cómputo.

#### Deployment
Controlador declarativo de alto nivel (`apps/v1`) que define el estado deseado de una aplicación (versión de imagen, número de réplicas, variables de entorno, puertos, sondas de salud). Gestiona actualizaciones progresivas (*rolling updates*) y reversiones (*rollbacks*) sin pérdida de servicio.

#### ReplicaSet
Controlador de nivel intermedio encargado de garantizar que un número exacto de Pods idénticos se mantenga en ejecución constante según los selectores de etiquetas (`matchLabels`). Si un Pod colapsa, el ReplicaSet lo detecta y lo reemplaza automáticamente.

#### Relación Arquitectónica: Deployment vs. ReplicaSet vs. Pods

```
+-------------------------------------------------------+
|                      DEPLOYMENT                       |
|          (Estrategia de despliegue y versiones)       |
+---------------------------+---------------------------+
                            | Administra
                            v
+-------------------------------------------------------+
|                      REPLICASET                       |
|        (Garantiza el número exacto de réplicas)       |
+---------------------------+---------------------------+
                            | Supervisa y crea
                            v
              +-------------+-------------+
              |                           |
              v                           v
     +-----------------+         +-----------------+
     |       POD       |         |       POD       |
     | (Contenedor(es))|         | (Contenedor(es))|
     +-----------------+         +-----------------+
```

---

## 4. ESPECIFICACIÓN TÉCNICA DE LA CONFIGURACIÓN REALIZADA

La infraestructura como código (IaC) fue estructurada en tres manifiestos YAML:

1. **`kubernetes/namespace.yaml`:**
   - Nombre: `monitoring`. Aísla completamente el stack de observabilidad de los recursos del sistema.
2. **`kubernetes/deployment.yaml`:**
   - Nombre: `grafana` (Namespace: `monitoring`).
   - Imagen explícita: `grafana/grafana:11.5.0` (evitando tags mutables como `latest`).
   - Réplicas iniciales: `2`.
   - Puerto del contenedor: `3000` (TCP).
   - Recursos asignados:
     - *Requests:* `100m` CPU, `128Mi` memoria.
     - *Limits:* `250m` CPU, `256Mi` memoria.
   - Sondas de salud:
     - `readinessProbe`: Endpoint HTTP `/api/health` en puerto 3000 (impide enviar tráfico a réplicas no listas).
     - `livenessProbe`: Endpoint HTTP `/api/health` en puerto 3000 (reinicia el contenedor ante bloqueos).
3. **`kubernetes/service.yaml`:**
   - Nombre: `grafana-service`.
   - Tipo: `NodePort` con mapeo de puerto de servicio `3000` y puerto de nodo `32000`.

---

## 5. PROCEDIMIENTO, PRUEBAS Y EVIDENCIAS FOTOGRÁFICAS

A continuación se presenta el desarrollo paso a paso de las pruebas operativas en el clúster con sus respectivas evidencias fotográficas.

---

### EVIDENCIA 1: Verificación e Inicio del Clúster Minikube
- **Comandos ejecutados:**
  ```powershell
  minikube start --driver=docker
  minikube status
  ```
- **Descripción:** Se inicializa el clúster local de Kubernetes sobre el hipervisor/driver de Docker Desktop y se comprueba que el plano de control (`host`, `kubelet`, `apiserver`) esté en estado `Running`.

> #### 📸 CAPTURA DE PANTALLA 1: ESTADO ACTIVO DE MINIKUBE
> *(Pega aquí la captura donde se observe la ejecución de `minikube status` mostrando todos los componentes en Running)*
> 
> ```
> +-------------------------------------------------------------------------+
> |                                                                         |
> |               [ PEGAR AQUÍ IMAGEN 1: MINIKUBE STATUS ]                  |
> |                                                                         |
> +-------------------------------------------------------------------------+
> ```
> **Figura 1.** Verificación del estado operativo del clúster Minikube.

---

### EVIDENCIA 2: Despliegue de Recursos y Verificación del Deployment
- **Comandos ejecutados:**
  ```powershell
  cd c:\Users\ASUS\Downloads\procesos\Kubernetes-Grafana
  kubectl apply -f kubernetes/
  kubectl get deployments -n monitoring
  ```
- **Descripción:** Se aplican los manifiestos YAML y se verifica que el controlador de Deployment registre 2 réplicas solicitadas y 2 réplicas disponibles (`2/2 READY`).

> #### 📸 CAPTURA DE PANTALLA 2: DEPLOYMENT DE GRAFANA CREADO
> *(Pega aquí la captura de la consola con la salida de `kubectl get deployments -n monitoring`)*
> 
> ```
> +-------------------------------------------------------------------------+
> |                                                                         |
> |              [ PEGAR AQUÍ IMAGEN 2: KUBECTL GET DEPLOYMENTS ]           |
> |                                                                         |
> +-------------------------------------------------------------------------+
> ```
> **Figura 2.** Deployment `grafana` desplegado satisfactoriamente con 2 réplicas activas.

---

### EVIDENCIA 3: Comprobación del ReplicaSet Automático
- **Comandos ejecutados:**
  ```powershell
  kubectl get replicasets -n monitoring
  ```
- **Descripción:** Se constata que el ReplicaSet fue creado automáticamente por el Deployment (asumiendo el formato de nombre `grafana-<hash>`), manteniendo las 2 réplicas deseadas y sincronizadas.

> #### 📸 CAPTURA DE PANTALLA 3: REPLICASET GENERADO POR KUBERNETES
> *(Pega aquí la captura con la salida de `kubectl get rs -n monitoring`)*
> 
> ```
> +-------------------------------------------------------------------------+
> |                                                                         |
> |              [ PEGAR AQUÍ IMAGEN 3: KUBECTL GET REPLICASETS ]           |
> |                                                                         |
> +-------------------------------------------------------------------------+
> ```
> **Figura 3.** ReplicaSet creado y supervisado por el Deployment.

---

### EVIDENCIA 4: Verificación de los Pods Iniciales en Ejecución
- **Comandos ejecutados:**
  ```powershell
  kubectl get pods -n monitoring -o wide
  ```
- **Descripción:** Se comprueba que existan exactamente 2 Pods en ejecución, ambos con estado `Running`, `1/1` en la columna `READY` y sus respectivas direcciones IP asignadas por la red CNI de Minikube.

> #### 📸 CAPTURA DE PANTALLA 4: DOS PODS DE GRAFANA EN RUNNING
> *(Pega aquí la captura con la salida de `kubectl get pods -n monitoring -o wide`)*
> 
> ```
> +-------------------------------------------------------------------------+
> |                                                                         |
> |                  [ PEGAR AQUÍ IMAGEN 4: KUBECTL GET PODS ]              |
> |                                                                         |
> +-------------------------------------------------------------------------+
> ```
> **Figura 4.** Instancias de Pods de Grafana activas y superando las sondas de salud.

---

### EVIDENCIA 5: Acceso a la Interfaz Web de Grafana
- **Comandos ejecutados:**
  ```powershell
  kubectl port-forward -n monitoring svc/grafana-service 3000:3000
  ```
- **URL de acceso:** `http://localhost:3000`
- **Credenciales por defecto:** Usuario: `admin` | Contraseña: `admin`
- **Descripción:** Se expone el servicio mediante redirección de puertos y se valida en el navegador web el acceso exitoso a la interfaz gráfica de Grafana.

> #### 📸 CAPTURA DE PANTALLA 5: GRAFANA EN EL NAVEGADOR
> *(Pega aquí la captura del navegador web mostrando la pantalla de login o el panel inicial de Grafana)*
> 
> ```
> +-------------------------------------------------------------------------+
> |                                                                         |
> |             [ PEGAR AQUÍ IMAGEN 5: NAVEGADOR HTTP://LOCALHOST:3000 ]    |
> |                                                                         |
> +-------------------------------------------------------------------------+
> ```
> **Figura 5.** Interfaz web de Grafana operativa y recibiendo tráfico a través del Service.

---

### EVIDENCIA 6: Escalamiento Horizontal a 3 Réplicas
- **Comandos ejecutados:**
  ```powershell
  kubectl scale deployment grafana -n monitoring --replicas=3
  kubectl get pods -n monitoring
  ```
- **Descripción:** Se modifica declarativamente la cantidad de réplicas en caliente. El ReplicaSet crea un tercer Pod de manera inmediata, alcanzando 3 instancias concurrentes en estado `Running`.

> #### 📸 CAPTURA DE PANTALLA 6: ESCALAMIENTO A 3 PODS
> *(Pega aquí la captura mostrando los 3 Pods en estado Running tras el comando scale)*
> 
> ```
> +-------------------------------------------------------------------------+
> |                                                                         |
> |             [ PEGAR AQUÍ IMAGEN 6: ESCALAMIENTO A 3 RÉPLICAS ]          |
> |                                                                         |
> +-------------------------------------------------------------------------+
> ```
> **Figura 6.** Escalamiento horizontal del Deployment a tres réplicas activas.

---

### EVIDENCIA 7: Eliminación Manual de un Pod (Simulación de Fallo)
- **Comandos ejecutados:**
  ```powershell
  $POD = (kubectl get pods -n monitoring -o jsonpath='{.items[0].metadata.name}')
  Write-Host "Eliminando pod: $POD"
  kubectl delete pod $POD -n monitoring
  ```
- **Descripción:** Se simula la caída forzada de una instancia eliminando uno de los Pods activos para poner a prueba el mecanismo de resiliencia del clúster.

> #### 📸 CAPTURA DE PANTALLA 7: ELIMINACIÓN MANUAL DE UN POD
> *(Pega aquí la captura de la consola con la confirmación de eliminación del Pod)*
> 
> ```
> +-------------------------------------------------------------------------+
> |                                                                         |
> |              [ PEGAR AQUÍ IMAGEN 7: ELIMINACIÓN MANUAL DE POD ]         |
> |                                                                         |
> +-------------------------------------------------------------------------+
> ```
> **Figura 7.** Confirmación de la eliminación manual de un Pod para evaluar tolerancia a fallos.

---

### EVIDENCIA 8: Demostración de Auto-Recuperación (*Self-Healing*)
- **Comandos ejecutados:**
  ```powershell
  kubectl get pods -n monitoring
  ```
- **Descripción:** Al consultar los Pods inmediatamente después de la eliminación, se observa que el Pod destruido desapareció y el ReplicaSet generó automáticamente un nuevo Pod con un identificador único y una antigüedad (*AGE*) de pocos segundos, restableciendo el estado deseado de forma totalmente autónoma.

> #### 📸 CAPTURA DE PANTALLA 8: AUTO-RECUPERACIÓN DEL REPLICASET
> *(Pega aquí la captura donde se aprecie el nuevo Pod creado con edad de segundos)*
> 
> ```
> +-------------------------------------------------------------------------+
> |                                                                         |
> |              [ PEGAR AQUÍ IMAGEN 8: NUEVO POD CREADO POR REPLICASET ]   |
> |                                                                         |
> +-------------------------------------------------------------------------+
> ```
> **Figura 8.** Mecanismo de auto-recuperación (Self-Healing) restableciendo las réplicas requeridas.

---

### EVIDENCIA ADICIONAL: Retorno a 2 Réplicas y Estado Final del Clúster
- **Comandos ejecutados:**
  ```powershell
  kubectl scale deployment grafana -n monitoring --replicas=2
  kubectl get all -n monitoring
  ```
- **Descripción:** Se reduce la escala nuevamente a 2 réplicas y se comprueba el estado consolidado de todos los recursos (`Deployment`, `ReplicaSet`, `Pods` y `Service`) en el namespace `monitoring`.

> #### 📸 CAPTURA DE PANTALLA 9: ESTADO FINAL DEL CLÚSTER
> *(Pega aquí la salida de `kubectl get all -n monitoring`)*
> 
> ```
> +-------------------------------------------------------------------------+
> |                                                                         |
> |                [ PEGAR AQUÍ IMAGEN 9: KUBECTL GET ALL ]                 |
> |                                                                         |
> +-------------------------------------------------------------------------+
> ```
> **Figura 9.** Visión general consolidada del namespace `monitoring` con 2 réplicas estables.

---

## 6. USOS DE GRAFANA EN ENTORNOS EMPRESARIALES

Grafana se ha convertido en el estándar de facto para la observabilidad en arquitecturas corporativas debido a sus amplias capacidades de integración:

1. **Monitoreo de Infraestructura y Nube Híbrida:**  
   Supervisión continua de consumo de CPU, memoria, I/O de disco y ancho de banda en centros de datos locales y nubes públicas (AWS, Azure, GCP), conectándose a fuentes como Prometheus Node Exporter.
2. **Observabilidad del Rendimiento de Aplicaciones (APM):**  
   Seguimiento de tiempos de respuesta, latencias percentiles (p95, p99), tasas de error HTTP y rendimiento transaccional por segundo en APIs y sistemas distribuidos.
3. **Gestión Proactiva de Alertas:**  
   Disparo automatizado de notificaciones críticas a través de canales corporativos (Slack, Microsoft Teams, Telegram, PagerDuty) cuando las métricas superan umbrales predefinidos, previniendo indisponibilidades de servicio.
4. **Tableros de Indicadores Clave de Negocio (KPIs):**  
   Consolidación de información comercial en tiempo real (ventas por minuto, usuarios conectados, volumen de transacciones) integrando directamente bases de datos transaccionales y analíticas.
5. **Ciberseguridad y Auditoría Forense:**  
   Análisis y correlación de registros de auditoría y tráfico malicioso mediante la integración con motores de logs como Grafana Loki o Elasticsearch.

---

## 7. CONCLUSIONES

1. **Gestión Declarativa Eficiente:** El uso de Kubernetes mediante archivos YAML permite implementar el paradigma de Infraestructura como Código (IaC), haciendo que el aprovisionamiento de plataformas complejas como Grafana sea 100% reproducible y auditable.
2. **Rol Crucial del ReplicaSet:** Se comprobó que el ReplicaSet es el componente responsable directo de la alta disponibilidad en Kubernetes; ante fallos imprevistos o eliminación de instancias, el controlador reconcilia automáticamente el estado real con el estado deseado sin intervención humana.
3. **Jerarquía y Abstracción:** El Deployment aporta una capa de control superior indispensable sobre el ReplicaSet, gestionando estrategias de versión, historial de cambios y escalabilidad de manera desacoplada de los Pods.
4. **Resiliencia y Monitoreo Integrado:** La incorporación de sondas de salud (*readiness* y *liveness probes*) garantiza que el tráfico de los usuarios sólo sea dirigido a instancias listas para procesar solicitudes, elevando los estándares de confiabilidad operativa del sistema.
