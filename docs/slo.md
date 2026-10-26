# Objetivos de Nivel de Servicio (SLO)
**Proyecto:** Sistema de Reservas de Hoteles en AWS

---

## SLO 1: Disponibilidad de la API de Reservas
**Servicio:** `api-reservas` (Microservicio en ECS Fargate detrás del Application Load Balancer)
**SLI (definición):** Proporción de peticiones HTTP de reservas válidas respondidas sin error `5xx`.
**SLI (implementación):** `(RequestCount - HTTPCode_Target_5XX_Count) / RequestCount` medido directamente en AWS CloudWatch (métricas del ALB). Se excluyen los *health checks* y errores `4xx` causados por el cliente (ej. credenciales inválidas).
**Ventana:** 30 días móviles.
**Objetivo (SLO):** 99.5%
**Presupuesto de error:** 0.5% × 43,200 min = **216 min** por ventana.
* **Alertas:** Aviso al consumir 50% del presupuesto (108 min caídos); Crítica al 100% (216 min).
**Política:** Al agotarse el 100% del presupuesto, se declara el estado de contingencia. Se congelan los despliegues de funciones nuevas y todo el equipo debe enfocarse en tareas de confiabilidad (SRE) y resolución de bugs hasta recuperar al menos el 25% del presupuesto.
**Verificación externa:** Monitor sintético externo (UptimeRobot) haciendo un `GET /health` cada 5 minutos.
**Responsable:** Bernal Jardon Saul Andres.
**Revisión:** Cada cierre de sprint / cierre parcial.

---

## SLO 2: Latencia p95 (Experiencia de Búsqueda y Reserva)
**Servicio:** `api-reservas` y `catalogo-hoteles`
**SLI (definición):** Proporción de peticiones de búsqueda y reserva procesadas en menos de 200 milisegundos.
**SLI (implementación):** Medición del percentil 95 (p95) de la métrica `TargetResponseTime` extraída del ALB mediante AWS CloudWatch y rastreos de AWS X-Ray.
**Ventana:** 30 días móviles.
**Objetivo (SLO):** 99.0%
**Presupuesto de error:** 1.0% × 43,200 min = **432 min** de respuestas lentas permitidas por ventana.
* **Alertas:** Aviso al consumir 50% del presupuesto (216 min de respuestas lentas); Crítica al 100%.
**Política:** Al agotarse el presupuesto, la siguiente iteración de desarrollo debe destinar el 50% del esfuerzo a optimizar las consultas a la base de datos (Amazon RDS) o a implementar caché local, antes de liberar nuevas funcionalidades web.
**Verificación externa:** Pruebas de carga periódicas simulando picos de demanda en temporada vacacional.
**Responsable:** Hernández Villeda Carlos Antonio.
**Revisión:** Cada cierre de sprint / cierre parcial.

---

## SLO 3: Integridad de Datos (Regla de Dominio: No Sobreventa)
**Servicio:** `db-reservas` (Lógica transaccional sobre Amazon RDS PostgreSQL)
**SLI (definición):** Proporción de reservas confirmadas de manera exitosa sin conflictos de concurrencia (no vender la misma habitación en fechas traslapadas).
**SLI (implementación):** `(Total de reservas confirmadas - Reservas duplicadas/sobrevendidas) / Total de reservas confirmadas`. Medido a través de logs de aplicación enviados a CloudWatch Logs y consultas de auditoría de inconsistencias en la base de datos.
**Ventana:** 30 días móviles.
**Objetivo (SLO):** 99.99% (Prácticamente tolerancia cero a la pérdida de consistencia).
**Presupuesto de error:** 0.01% de fallos transaccionales por ventana.
* **Alertas:** Alerta crítica inmediata (SEV-1) al detectarse 1 solo caso reportado de sobreventa.
**Política:** Al ocurrir un fallo de concurrencia, es un incidente crítico. Se detiene cualquier otro desarrollo, se contacta a los huéspedes afectados y se depuran los bloqueos transaccionales (locks optimistas/pesimistas) en PostgreSQL hasta garantizar la atomicidad.
**Verificación externa:** Scripts diarios (cron jobs) de auditoría que cruzan las fechas de reserva en la base de datos para buscar traslapes.
**Responsable:** Paniagua Escamilla Jorge Alberto.
**Revisión:** Continua / Diaria.

---

### Extra: SLO de Disponibilidad en formato OpenSLO (YAML)
*(Este bloque define el SLO 1 como código estándar de la industria, compatible con herramientas de SRE como Nobl9 o Sloth).*

```yaml
apiVersion: openslo/v1
kind: SLO
metadata:
  name: api-reservas-disponibilidad
  displayName: Disponibilidad de la API de Reservas
spec:
  description: Peticiones validas de reservas respondidas sin error 5xx en AWS
  service: api-reservas
  indicator:
    metadata:
      name: peticiones-exitosas-alb
    spec:
      ratioMetric:
        counter: true
        good:
          metricSource:
            type: CloudWatch
            spec:
              query: SUM(RequestCount) - SUM(HTTPCode_Target_5XX_Count)
        total:
          metricSource:
            type: CloudWatch
            spec:
              query: SUM(RequestCount)
  timeWindow:
    - duration: 30d
      isRolling: true
  budgetingMethod: Occurrences
  objectives:
    - displayName: Objetivo Mensual 99.5%
      target: 0.995