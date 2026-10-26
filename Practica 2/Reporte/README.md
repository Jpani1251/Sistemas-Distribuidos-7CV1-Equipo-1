# Práctica 2. Acuerdo de nivel de servicio: análisis de un SLA real y diseño de los SLOs

**Equipo 1:**
* Bernal Jardon Saul Andres
* Hernández Villeda Carlos Antonio
* Paniagua Escamilla Jorge Alberto


## 1. Tabla 1. Anatomía del SLA (AWS)

| Integrante | Servicio y URL (fecha) | Compromiso por configuración | Definición de inactividad | Exclusiones | Créditos | Reclamación | Min/mes permitidos |
| :--- | :--- | :--- | :--- | :--- | :--- | :--- | :--- |
| **Bernal Jardon Saul Andres** | **Amazon ECS/Fargate**<br>`aws.amazon.com/compute/sla`<br>(23 Octubre 2026) | Multizona $\ge 99.99\%$<br>Individual $\ge 99.5\%$ | Pérdida de conectividad externa o incapacidad de lanzar tareas de reemplazo durante más de 1 minuto. | Uso de preview, problemas en la red del cliente, fallas fuera del control de AWS. | 10% (<99.99%), 30% (<99.0%), 100% (<95%). Remedio único. | Aviso a soporte adjuntando logs y horas exactas del corte. | Multizona: 4.3 min<br>Individual: 216 min |
| **Hernández Villeda Carlos Antonio** | **Amazon RDS**<br>`aws.amazon.com/rds/sla`<br>(24 Octubre 2026) | Multi-AZ $\ge 99.95\%$<br>Single-DB $\ge 99.5\%$ | Incapacidad de conectar a la BD o transacciones caídas por 1 o más minutos consecutivos. | Mantenimientos programados, superación de cuotas por picos de reserva, consultas bloqueadas por el cliente. | 10% (<99.95%), 25% (<99.0%), 100% (<95%). Remedio único. | Abrir ticket en ciclo de facturación con evidencia de errores de conexión. | Multi-AZ: 21.6 min<br>Single-DB: 216 min |
| **Paniagua Escamilla Jorge Alberto** | **Amazon S3**<br>`aws.amazon.com/s3/sla`<br>(21 Octubre 2026) | S3 Standard $\ge 99.9\%$ | Tasa de error superior al promedio en peticiones válidas en intervalos de 5 minutos. | Errores 4xx generados por falta de permisos del usuario o bloqueos por abusos. | 10% (<99.9%), 25% (<99.0%), 100% (<95%). Remedio único. | Solicitar crédito a soporte con métricas CloudWatch de aumento `5xx`. | Standard: 43.2 min |

---

## 2. Disponibilidad compuesta del proyecto

**Cadena de dependencias:** `DNS (Route 53) -> Balanceador (ALB) -> Cómputo (Fargate) -> Base de Datos (RDS) -> Objetos (S3)`
*(Diagrama en `docs/arquitectura/diagrama_disponibilidad.png`)*

| Componente | SLA individual | SLA redundante | En serie o paralelo | Costo extra redundancia (USD/mes) |
| :--- | :--- | :--- | :--- | :--- |
| DNS (Route 53) | 100% | 100% | Serie | $0 (Global) |
| Balanceador (ALB) | 99.99% | 99.99% | Serie | $0.0225 |
| Cómputo (Fargate) | 99.5% | 99.99% | Serie (Paralelo a nivel AZ) | $0 |
| Base de datos (RDS) | 99.5% | 99.95% | Serie (Paralelo a nivel AZ) | $0 |
| Objetos (S3) | 99.9% | 99.9% | Serie | $0 |
| **Compuesto sin/con redundancia** | **~98.80%** | **~99.83%** | | **Total extra:** $0.0225 |
| **Minutos al mes sin/con** | **518.4 min** | **73.4 min** | | |

---

## 3. Tabla 3. SLOs del proyecto

| SLO | SLI (buenos / válidos) | Punto de medición | Ventana | Objetivo | Presupuesto (min) | Alertas (50% / 100%) | Herramienta |
| :--- | :--- | :--- | :--- | :--- | :--- | :--- | :--- |
| **Disponibilidad** | Peticiones HTTP de reservas respondidas sin error `5xx` / Total peticiones HTTP | Balanceador (ALB) | 30 d | 99.5% | 216 min | 108 min / 216 min | AWS CloudWatch + UptimeRobot |
| **Latencia p95** | Reservas completadas en $<200$ ms / Total de reservas procesadas | API Gateway / ALB | 30 d | 99.0% | 432 min (lentos) | 216 min / 432 min | AWS X-Ray / CloudWatch |
| **De dominio (Integridad)** | Reservas exitosas sin concurrencia (sobreventa) / Total intentos de reserva válidos | Base de Datos (RDS) | 30 d | 99.99% | 0.01% fallos | Al primer caso reportado | Logs de App (Métricas CloudWatch) |

---

## 4. Cláusula de SLA del Equipo

**Acuerdo de Nivel de Servicio para Hoteles (SLA)**
Nos comprometemos a mantener una disponibilidad mensual del sistema de reservas del **99.0%**. La inactividad se define como la pérdida total de conectividad externa a nuestra API de reservas por un periodo consecutivo mayor a 5 minutos. Este compromiso **excluye** caídas causadas por: ventanas de mantenimiento programadas semanalmente (notificadas con 48 hrs de anticipación), errores de configuración de red del hotel (cliente), y eventos de fuerza mayor. 

Si no cumplimos con este SLA, el hotel afectado recibirá un crédito del 15% sobre su cuota mensual (si la disponibilidad es de 95% a <99.0%) o un 50% (si es <95%). Para reclamar, el cliente debe contactar a soporte dentro de los 30 días posteriores al incidente. 

Nuestro SLO de equipo es del 99.5%, lo que nos hace ser más exigentes internamente para reaccionar a incidentes antes de violar la promesa legal del 99.0% dada a los clientes.

---

## 5. Preguntas de análisis

**1. El SLA de tu proveedor define «no disponible» de forma muy específica. Describe una falla que dejaría tu aplicación fuera de servicio y que no contaría como inactividad para el proveedor. ¿De qué lado de la responsabilidad compartida cae y qué SLI tuyo la detectaría?**
Si nuestra lógica de bloqueos optimistas en la base de datos falla y permitimos "sobreventa" de una misma habitación a dos huéspedes distintos, el sistema sigue "arriba" para AWS (RDS y Fargate operan con normalidad). Al ser un bug de software, cae totalmente de nuestro lado de responsabilidad compartida. Esto sería detectado por nuestro SLI de dominio (Integridad) que mide las reservas exitosas vs reservas traslapadas en la base de datos.

**2. Con la factura mensual estimada en la Práctica 1, ¿cuánto crédito recibirías por una caída de 8 horas en un mes? Compáralo con una estimación de lo que perdería el negocio. ¿Por qué el SLA no es un seguro?**
Una caída de 8 horas equivale a 480 minutos. El SLA de Amazon RDS/EC2 otorga un crédito del 10% si la disponibilidad baja del 99.0% (que equivale a 432 minutos). Por tanto, se aplicaría al 10% de crédito de nuestra factura. Esto es insignificante en comparación a las ganancias perdidas por reservas caídas en temporada alta. El SLA no es un seguro, es solo un descuento parcial sobre el uso de su infraestructura.

**3. Google/AWS publican compromisos por región. Si tu sistema necesitara 99.99% y residir en México (`mx-central-1`), ¿qué arquitectura lo resolvería y cuánto costaría al mes?**
Para asegurar un SLA del 99.99% bajo AWS en la región de México, tendríamos que desplegar nuestros contenedores Fargate distribuidos al menos en 2 Zonas de Disponibilidad (Multi-AZ) y habilitar la misma replicación Multi-AZ para Amazon RDS. Lograr esto costaría aproximadamente $16.20 USD al mes, debido principalmente a la duplicación de nodos de base de datos y balanceo de tráfico cruzado.

**4. Explica por qué tu SLO no puede ser mayor que la disponibilidad compuesta de la tabla 2, y propón dos cambios concretos de arquitectura que subirían ese techo.**
El SLO (99.5%) no puede superar nuestro compuesto en serie (99.83%) porque probabilísticamente nuestro sistema fallará cada que falle su eslabón más débil (en este caso S3 con 99.9% o RDS con 99.95%). Para subir este límite físico podríamos: 
1) Cambiar RDS PostgreSQL por Amazon DynamoDB (Global Tables), que ofrece un SLA del 99.999%. 
2) Configurar un balanceo activo-activo hacia otra región de AWS para sortear caídas completas de la región `mx-central-1`.

**5. Elige un incidente público reciente. ¿El SLA cubrió la caída? ¿Cuál de tus SLIs la habría detectado primero y cuánto presupuesto consumiría?**
En la interrupción de AWS us-east-1 del 20 de octubre de 2025, servicios core cayeron durante horas. Aunque el SLA otorgó el crédito del 10% a los clientes afectados, el daño estaba hecho. Nuestro SLI de *Disponibilidad (Peticiones exitosas sin 5xx)* hubiera detectado de inmediato los tiempos de espera agotados (Timeouts). Una caída de más de 4 horas (240 minutos) habría consumido por completo nuestro presupuesto de error mensual de 216 minutos.

**6. Si en la primera semana del mes consumiste el 80% del presupuesto de error, ¿qué decides sobre el despliegue programado para el viernes? Justifica con tu política.**
Decidiríamos congelar el despliegue del viernes. Nuestra política de presupuesto de error dicta que, al superar el umbral del 50% de consumo, las alertas se disparan y los pases a producción de nuevas funciones se detienen. El equipo debe enfocar el 100% de su esfuerzo en tareas de confiabilidad, reparar bugs de control de concurrencia y estabilizar el sistema antes de inyectar nuevos riesgos.

## 6. Uso de IA
Para este práctica se uso la Inteligecia Artificial para la obtencion de la formula para calcular la inactividad que se puede permitir en 30 dias y la adaptacion del SLO conforme a nuestro proyecto al documento del mismo nombre.