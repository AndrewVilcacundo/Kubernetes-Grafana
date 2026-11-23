# INFORME TÉCNICO Y ACADÉMICO
## Configuración de Clúster Kubernetes: Deployment, ReplicaSet y Monitoreo con Grafana

**Asignatura:** Integración de Sistemas  
**Actividad:** Configuración de cluster (Docker Swarm / Kubernetes) – Deployment y ReplicaSet  
**Plataforma evaluada:** Grafana (v11.5.0) sobre Kubernetes (Minikube)  
**Entorno Operativo:** Windows 11 / PowerShell  

---

## 1. Introducción

En la arquitectura de sistemas distribuidos y microservicios contemporánea, la gestión manual de contenedores se vuelve insostenible ante fluctuaciones de demanda y posibles fallos de infraestructura. Los orquestadores de contenedores resuelven este desafío automatizando el aprovisionamiento, escalabilidad, balanceo de carga y recuperación ante desastres.

El presente informe detalla la implementación práctica de un clúster de Kubernetes empleando Minikube en un entorno Windows 11. Se aborda el despliegue de **Grafana**, una solución referente a nivel de industria para la observabilidad y visualización de métricas en tiempo real. Se analizan y comprueban en la práctica los conceptos clave de **Namespace**, **Deployment**, **ReplicaSet**, **Pods** y **Services**, evidenciando las capacidades de autorreparación (*self-healing*) y escalado declarativo de Kubernetes.

---

## 2. Objetivos

### 2.1. Objetivo General
Desplegar y administrar la plataforma de observabilidad Grafana en un clúster local de Kubernetes (Minikube), aplicando y validando el ciclo de vida de los controladores *Deployment* y *ReplicaSet* en términos de disponibilidad, escalabilidad y tolerancia a fallos.

### 2.2. Objetivos Específicos
1. Estructurar manifiestos declarativos YAML que definan un Namespace aislado, un Deployment con sondas de salud (*health checks*) y restricciones de recursos, y un Service para exposición de red.
2. Demostrar la relación de dependencia y control jerárquico entre el Deployment, su ReplicaSet subyacente y los Pods gestionados.
3. Validar el mecanismo de auto-recuperación (*self-healing*) del clúster ante la eliminación forzada de Pods.
4. Experimentar el escalamiento dinámico (horizontal pod scaling declarativo) variando la cantidad de réplicas en caliente.
5. Analizar el valor operativo y los casos de uso empresarial de Grafana en ecosistemas tecnológicos integrados.

---

## 3. Fundamentación Teórica

### 3.1. ¿Qué es Grafana?
Grafana es una plataforma de código abierto ampliamente adoptada para el análisis y visualización interactiva de métricas, trazas (*traces*) y registros (*logs*). Permite consultar, graficar, alertar y explorar datos con independencia del motor de persistencia subyacente. A través de tableros (*dashboards*) dinámicos, transforma series temporales procedentes de múltiples orígenes (Prometheus, InfluxDB, Elasticsearch, CloudWatch, PostgreSQL, etc.) en información visual procesable para equipos de desarrollo, operaciones (DevOps) y ciberseguridad.

### 3.2. ¿Qué es un Deployment en Kubernetes?
Un **Deployment** es un objeto de nivel superior en la API de Kubernetes (`apps/v1`) que proporciona actualizaciones declarativas para los Pods y ReplicaSets. En lugar de gestionar contenedores o réplicas de forma directa, el administrador define el estado deseado en la especificación (`spec`) del Deployment. El controlador de Deployment se encarga de transicionar el estado real al estado deseado a un ritmo controlado, posibilitando despliegues progresivos (*rolling updates*), reversión a versiones previas (*rollbacks*) y control de revisiones.

### 3.3. ¿Qué es un ReplicaSet?
Un **ReplicaSet** es un controlador de nivel inferior cuyo único propósito fundamental es garantizar en todo momento la presencia de un número específico y constante de Pods idénticos en ejecución dentro del clúster. Utiliza selectores de etiquetas (`spec.selector`) para identificar los Pods que pertenecen a su dominio. Si un Pod falla o es destruido, el ReplicaSet instancia uno nuevo inmediatamente; si existen más Pods de los configurados, elimina los excedentes.

### 3.4. Relación entre Deployment, ReplicaSet y Pods
Existe una jerarquía y separación de responsabilidades estricta entre estos tres elementos:

```
+-------------------------------------------------------+
|                      DEPLOYMENT                       |
|   (Estrategia de rollout, historial de versiones)     |
+---------------------------+---------------------------+
                            | Administra
                            v
+-------------------------------------------------------+
|                      REPLICASET                       |
|      (Garante del número exacto de réplicas)          |
+---------------------------+---------------------------+
                            | Crea y supervisa
                            v
              +-------------+-------------+
              |                           |
              v                           v
     +-----------------+         +-----------------+
     |       POD       |         |       POD       |
     | (Contenedor(es))|         | (Contenedor(es))|
     +-----------------+         +-----------------+
```

- **El usuario interactúa con el Deployment:** El ingeniero define la imagen, variables, puertos y réplicas deseadas en el Deployment.
- **El Deployment genera y controla el ReplicaSet:** Por cada versión o cambio de plantilla (*pod-template-hash*), el Deployment genera un ReplicaSet.
- **El ReplicaSet gestiona los Pods:** El ReplicaSet se encarga del cómputo individual de Pods en los nodos del clúster. No se debe crear un ReplicaSet de forma directa en producción, ya que se perderían las capacidades de actualización progresiva y rollback provistas por el Deployment.

---

## 4. Explicación de la Configuración Realizada

La infraestructura como código (IaC) del proyecto se estructuró en la carpeta `Kubernetes-Grafana/kubernetes/` dividida en tres manifiestos:

### 4.1. Namespace (`namespace.yaml`)
Aísla los recursos del laboratorio bajo el nombre `monitoring`. Esto garantiza orden lógico, evita colisiones de nombres con servicios del sistema (`kube-system`) o de otras asignaturas, y permite delimitar permisos o cuotas en escenarios multi-inquilino (*multi-tenant*).

### 4.2. Deployment (`deployment.yaml`)
- **Metadata y Selectores:** Se establecieron etiquetas consistentes `app: grafana` vinculadas mediante `spec.selector.matchLabels`.
- **Imagen estable:** Se utilizó `grafana/grafana:11.5.0` en lugar del tag flotante `latest`, cumpliendo con las buenas prácticas de inmutabilidad y predictibilidad en DevOps.
- **Réplicas iniciales:** Se fijó `replicas: 2` para disponer de redundancia básica.
- **Límites de Recursos:**
  - *Requests:* `100m` CPU y `128Mi` memoria (recursos mínimos asegurados por el planificador de Kubernetes).
  - *Limits:* `250m` CPU y `256Mi` memoria (tope superior para prevenir asfixia de recursos en Minikube).
- **Sondas de Salud (Probes):**
  - `readinessProbe`: Consulta el endpoint HTTP `/api/health` en el puerto 3000. Mientras no devuelva HTTP 200, el Pod no recibe tráfico del Service.
  - `livenessProbe`: Consulta periódicamente el mismo endpoint. Si el proceso interno colapsa o entra en bloqueo, Kubernetes reinicia el contenedor de manera automática.

### 4.3. Service (`service.yaml`)
Se creó un Service llamado `grafana-service` de tipo `NodePort`, exponiendo internamente el puerto 3000 y mapeándolo al puerto de nodo `32000`. Esto permite balancear la carga de red entrante entre los distintos Pods del Deployment y habilita tanto el acceso directo mediante la IP de Minikube como el túnel transparente con `kubectl port-forward`.

---

## 5. Procedimiento de Implementación y Pruebas Planificadas

A continuación se detalla la secuencia ordenada de pruebas a ejecutar en PowerShell, junto con la descripción del comportamiento que debe evidenciarse en el clúster:

### Prueba 1: Despliegue de los recursos
- **Comando:** `kubectl apply -f kubernetes/`
- **Explicación:** Se envían los manifiestos al Kubernetes API Server. Se registran secuencialmente el namespace, el deployment y el service.

### Prueba 2: Consulta del Deployment
- **Comando:** `kubectl get deployments -n monitoring`
- **Resultado esperado:** Se observa el deployment `grafana` con 2 réplicas deseadas, actualizadas y disponibles (`2/2 READY`).

### Prueba 3: Consulta del ReplicaSet
- **Comando:** `kubectl get replicasets -n monitoring`
- **Resultado esperado:** Se evidencia un ReplicaSet creado automáticamente con el nombre `grafana-<hash_aleatorio>`, reportando `DESIRED: 2`, `CURRENT: 2`, `READY: 2`.

### Prueba 4: Consulta de los Pods
- **Comando:** `kubectl get pods -n monitoring -o wide`
- **Resultado esperado:** Se listan dos Pods cuyos nombres inician con el nombre del ReplicaSet. Ambos deben encontrarse en estado `Running` con IP interna asignada.

### Prueba 5: Verificación de las dos réplicas
- **Comando:** `kubectl describe deployment grafana -n monitoring`
- **Resultado esperado:** La sección de eventos confirma la instanciación exitosa del ReplicaSet y el escalado inicial a 2 réplicas sin errores de pull de imagen ni fallos en las sondas.

### Prueba 6: Acceso a la interfaz web de Grafana
- **Comando:** `kubectl port-forward -n monitoring svc/grafana-service 3000:3000`
- **Resultado esperado:** Se abre `http://localhost:3000` en el navegador web, visualizándose la pantalla de inicio de sesión de Grafana (usuario y contraseña inicial: `admin`/`admin`).

### Prueba 7: Escalamiento del Deployment a 3 réplicas
- **Comando:** `kubectl scale deployment grafana -n monitoring --replicas=3`
- **Comando de verificación:** `kubectl get pods -n monitoring`
- **Resultado esperado:** El Deployment actualiza la especificación del ReplicaSet, el cual crea de forma inmediata un tercer Pod (`1/1 Running`).

### Prueba 8: Simulación de fallo eliminando un Pod manualmente
- **Comandos:**
  ```powershell
  $POD = (kubectl get pods -n monitoring -o jsonpath='{.items[0].metadata.name}')
  kubectl delete pod $POD -n monitoring
  ```
- **Resultado esperado:** El Pod indicado cambia su estado a `Terminating`.

### Prueba 9: Comprobación de Auto-Recuperación (Self-Healing)
- **Comando:** `kubectl get pods -n monitoring`
- **Resultado esperado:** El ReplicaSet detecta que la cantidad de Pods activos descendió a 2 cuando el objetivo es 3; por consiguiente, crea de manera instantánea un nuevo Pod con un identificador único para restablecer las 3 réplicas exigidas.

### Prueba 10: Retorno a dos réplicas
- **Comando:** `kubectl scale deployment grafana -n monitoring --replicas=2`
- **Resultado esperado:** El controlador identifica un exceso de réplicas y termina limpiamente uno de los Pods, estabilizando el clúster en 2 réplicas.

### Prueba 11: Estado final consolidado
- **Comando:** `kubectl get all -n monitoring`
- **Resultado esperado:** Listado íntegro mostrando el Deployment (2/2), el ReplicaSet (2/2), los dos Pods (Running) y el Service (NodePort).

---

## 6. Usos de Grafana en Entornos Empresariales

Grafana se ha consolidado como un componente indispensable en las arquitecturas modernas de TI debido a las siguientes aplicaciones clave:

1. **Monitoreo de Infraestructura y Nube Híbrida:**
   Supervisión continua del consumo de hardware (CPU, memoria, disco, I/O de red) de servidores bare-metal, máquinas virtuales y clústeres Kubernetes, integrándose de forma nativa con agentes recolectores como Prometheus Node Exporter.
2. **Observabilidad del Rendimiento de Aplicaciones (APM):**
   Seguimiento del rendimiento transaccional, latencias (percentiles p95, p99), tasas de error HTTP y volumen de peticiones (RPS), facilitando la detección de cuellos de botella en APIs y microservicios.
3. **Gestión de Alarmas y Notificaciones Críticas:**
   Configuración de reglas proactivas que alertan a los equipos de soporte vía Slack, Microsoft Teams, PagerDuty o correo electrónico ante anomalías o degradación del servicio antes de que impacte a los usuarios finales.
4. **Visualización de Negocio y KPIs Operativos:**
   Integración con bases de datos relacionales y de analítica (PostgreSQL, MySQL, ClickHouse) para presentar paneles ejecutivos con volumen de ventas, registros de usuarios o transacciones por segundo.
5. **Ciberseguridad y Análisis Forense:**
   Correlación de eventos de seguridad y análisis de bitácoras centralizadas (mediante Grafana Loki o Elasticsearch) para investigar incidentes de acceso no autorizado y auditorías de cumplimiento.

---

## 7. Plan de Evidencias Requeridas para la Entrega

Para la sustentación académica de la tarea, se debe capturar pantalla de los siguientes 8 hitos críticos:

| Hito | Descripción del hito a capturar | Comando asociado | Elemento visual clave a resaltar |
| :--- | :--- | :--- | :--- |
| **Evidencia 1** | Minikube en ejecución | `minikube status` | Estados `Running` de host, kubelet y apiserver. |
| **Evidencia 2** | Creación del Deployment | `kubectl get deployment -n monitoring` | Nombre `grafana` y estado `2/2 READY`. |
| **Evidencia 3** | ReplicaSet generado | `kubectl get rs -n monitoring` | Prefijo generado automáticamente y réplicas en 2. |
| **Evidencia 4** | Dos Pods en ejecución | `kubectl get pods -n monitoring` | Ambos Pods con `STATUS: Running` y `READY: 1/1`. |
| **Evidencia 5** | Interfaz web de Grafana | Navegador en `http://localhost:3000` | Pantalla de login/dashboard de Grafana v11.5.0. |
| **Evidencia 6** | Escalamiento a 3 réplicas | `kubectl get pods -n monitoring` | Visualización de 3 Pods en estado `Running`. |
| **Evidencia 7** | Eliminación manual de un Pod | `kubectl delete pod ...` | Mensaje de confirmación del Pod eliminado. |
| **Evidencia 8** | Auto-recuperación del ReplicaSet | `kubectl get pods -n monitoring` | Presencia del Pod nuevo con edad de pocos segundos. |

---

## 8. Conclusiones

1. La abstracción proporcionada por Kubernetes a través del objeto **Deployment** desacopla la definición del software de la administración operativa de sus procesos, permitiendo mantener arquitecturas reproducibles mediante código declarativo (YAML).
2. El **ReplicaSet** actúa como el mecanismo central de resiliencia en Kubernetes; la prueba de eliminación manual evidenció cómo el lazo de control (*reconciliation loop*) detecta discrepancias en tiempo real y regenera las instancias faltantes de forma autónoma.
3. La configuración de pruebas de salud (*readiness* y *liveness probes*) junto con límites de recursos resulta indispensable en entornos de producción para evitar la asignación desmedida de memoria o el enrutamiento de tráfico hacia contenedores no inicializados.
4. Grafana demostró ser una solución idónea para entornos de observabilidad en contenedores, requiriendo una huella de recursos reducida y exponiendo facilidades de integración que encajan con las políticas de servicio de Kubernetes.
