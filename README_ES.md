# Taller-Disponibilidad-Balanceador-AWS-ARSW

## Alta Disponibilidad con Application Load Balancer en AWS

---

## Evidencias del Laboratorio

### Instancias EC2 Creadas

![Instancias EC2](Images/Instances.png)

### Instancia A - web-ha-a

![Instancia A](Images/Instancia_A.png)

### Instancia B - web-ha-b

![Instancia B](Images/Instancia_B.png)

### Security Groups Configurados

![Security Groups](Images/Security_Groups.png)

### Target Group - tg-ha-web

![Target Groups](Images/Target_Groups.png)

### Respuesta desde Instancia A vía ALB

![Respuesta Instancia A](Images/Respuesta_I_A.png)

### Respuesta desde Instancia B vía ALB

![Respuesta Instancia B](Images/Respuesta_I_B.png)

### Simulación de Falla - Solo responde Instancia B

![Solo responde Instancia B](Images/Solo_respuesta_I_B.png)

### Target Group - Instancia A detenida (Unhealthy)

![Target Group Error](Images/Target_Groups_Error.png)

---

## Actividad 1: Análisis del Balanceo

### 1. ¿Qué instancia respondió primero?
Respondió primero la Instancia B, luego la Instancia A al recargar la página.

### 2. ¿El balanceador alternó entre ambas instancias?
Sí, generó respuestas diferentes cada vez que se recargaba el DNS del Application Load Balancer.

### 3. ¿Qué información permite confirmar que hay más de una instancia activa?
Las diferentes respuestas por parte de las Instancias A y B, ya que ambas se configuraron bajo el mismo balanceador de carga y se alternaban al consultar el DNS del ALB. Cada página muestra el Instance ID y la Availability Zone, lo que permite identificar claramente qué instancia respondió.

### 4. ¿Qué papel cumple el Target Group?
El Target Group registra las instancias como destinos, define el puerto (80) y protocolo (HTTP), y ejecuta los health checks para determinar qué instancias están saludables. El Application Load Balancer usa el Target Group para saber a qué instancias enviar el tráfico entrante.

### 5. ¿Qué papel cumplen los health checks?
Los health checks son verificaciones periódicas (cada 15 segundos a la ruta `/health`) que confirman que la instancia responde correctamente con código 200. Si una instancia falla el health check, el ALB deja de enviarle tráfico automáticamente, manteniendo la disponibilidad del sistema al redirigir las solicitudes solo a las instancias saludables.

### 6. ¿Por qué el usuario no necesita conocer las IP públicas de las instancias?
Porque el balanceador de carga actúa como **punto único de entrada** a través de su DNS público. El usuario solo necesita conocer el DNS del ALB; el ALB se encarga internamente de distribuir el tráfico entre las instancias. Esto permite:

- **Abstracción**: las instancias pueden cambiar (caerse, reemplazarse) sin afectar al usuario.
- **Alta disponibilidad**: si una instancia falla, el ALB redirige a otra sin que el usuario lo note.
- **Balanceo de carga**: la carga se distribuye automáticamente entre las instancias disponibles.

---

## Actividad 2: Análisis de Falla

### 1. ¿Qué ocurrió cuando se detuvo la instancia A?
Cuando se detuvo la instancia A (web-ha-a), el health check del Target Group dejó de recibir respuesta exitosa en la ruta `/health`. Como resultado, el Application Load Balancer marcó la instancia A como **Unhealthy** y automáticamente dejó de enviarle tráfico. La instancia A quedó fuera del pool de servidores disponibles.

### 2. ¿El sistema completo dejó de estar disponible?
No. El sistema se mantuvo operativo porque la instancia B (web-ha-b) continuó recibiendo y respondiendo a las solicitudes. El usuario final no experimentó ninguna interrupción del servicio, ya que el ALB redirigió todo el tráfico únicamente hacia la instancia saludable.

### 3. ¿Qué hizo el Load Balancer cuando detectó la falla?
El Application Load Balancer, a través de los health checks configurados en el Target Group, detectó que la instancia A ya no respondía correctamente. Automáticamente la eliminó del grupo de destino y redirigió todo el tráfico entrante exclusivamente hacia la instancia B, que se mantenía saludable.

### 4. ¿Qué diferencia habría si solo existiera una instancia?
Si solo existiera una instancia, al detenerse el sistema completo habría quedado fuera de servicio. Los usuarios habrían recibido errores de conexión o timeout, y la aplicación no habría estado disponible hasta que la instancia se reiniciara manualmente. La redundancia proporcionada por la segunda instancia evitó este escenario.

### 5. ¿Qué atributo de calidad mejora esta arquitectura?
La arquitectura mejora la **disponibilidad** del sistema. Al distribuir las instancias en dos zonas de disponibilidad diferentes y utilizar un balanceador de carga con health checks, el sistema puede tolerar la falla de una instancia sin afectar la experiencia del usuario. También mejora la **resiliencia** y la **tolerancia a fallos**, ya que el sistema se recupera automáticamente del fallo de un componente sin intervención manual inmediata.

---

## Actividad 3: Análisis de Recuperación

### 1. ¿Qué ocurrió cuando la instancia A volvió a estar saludable?
Se volvió a verificar en el Target Group que ambas instancias estuvieran en estado **Healthy**. El health check confirmó que la instancia A respondía correctamente en la ruta `/health`, por lo que el ALB la reincorporó automáticamente al pool de servidores disponibles.

### 2. ¿El balanceador volvió a enviarle tráfico?
Sí. Una vez que la instancia A apareció como **Healthy** en el Target Group, el Application Load Balancer reanudó el envío de solicitudes hacia ella, alternando nuevamente el tráfico entre ambas instancias.

### 3. ¿Por qué es importante que la recuperación sea automática desde el punto de vista del usuario?
Para que no haya pérdida de disponibilidad del servicio. Si una instancia se cae, otra responde inmediatamente sin que el usuario lo perciba. Esto garantiza que la aplicación permanezca accesible en todo momento, manteniendo la continuidad del servicio sin interrupciones.

### 4. ¿Qué limitaciones tiene esta arquitectura si la instancia no se reinicia manualmente?
Si la instancia no se reinicia manualmente, el sistema dependería únicamente de una sola instancia para atender todo el tráfico, perdiendo redundancia y aumentando el riesgo de caída total si esa instancia también falla. Además, se debe verificar que al reiniciar la instancia, el health check la confirme como saludable para que el balanceador pueda reincorporarla al balanceo de carga.

---

## Validación Arquitectónica (Sección 24)

| Elemento | Función en la arquitectura |
|----------|---------------------------|
| **EC2 instancia A** | Ejecuta el servidor web Apache y aloja la aplicación. Ubicada en una zona de disponibilidad (us-east-1c) para proporcionar redundancia. Responde a las solicitudes del ALB cuando está saludable. |
| **EC2 instancia B** | Ejecuta el servidor web Apache y aloja la aplicación. Ubicada en una zona de disponibilidad diferente (us-east-1a) para garantizar tolerancia a fallos. Si la instancia A falla, B continúa atendiendo el tráfico. |
| **Application Load Balancer** | Punto único de entrada para los usuarios. Recibe las solicitudes HTTP y las distribuye entre las instancias registradas en el Target Group, basándose en los resultados de los health checks. |
| **Target Group** | Agrupa las instancias EC2 como destinos del balanceador. Define el puerto (80), el protocolo (HTTP) y la configuración de health checks para monitorear el estado de las instancias. |
| **Health Check** | Verificación periódica (GET /health cada 15 segundos) que determina si una instancia está disponible para recibir tráfico. Si falla, el ALB elimina la instancia del pool automáticamente. |
| **Security Group del ALB** | Permite tráfico HTTP (puerto 80) desde cualquier origen (0.0.0.0/0) hacia el balanceador, actuando como primera capa de seguridad. |
| **Security Group de EC2** | Permite tráfico HTTP únicamente desde el Security Group del ALB, aislando las instancias del acceso directo desde Internet y mejorando la seguridad. |
| **Zonas de disponibilidad** | Permiten distribuir las instancias en ubicaciones físicamente separadas dentro de la misma región. Si una zona falla, la instancia en la otra zona continúa operando. |

---

## Actividad 4: Propuesta de Mejora (Sección 26)

### ¿Cómo agregaría recuperación automática?
Implementando un **Auto Scaling Group** con un Launch Template que lance automáticamente una nueva instancia cuando una existente falle o sea marcada como Unhealthy. El ASG se integraría con el Target Group para monitorear el estado de las instancias y reemplazar las que no pasen los health checks.

### ¿Cómo protegería las instancias para que no sean públicas?
Colocando las instancias EC2 en **subnets privadas** en lugar de públicas. Solo el ALB estaría en subnets públicas. Las instancias se comunicarían con el ALB y tendrían salida a Internet mediante un **NAT Gateway** o **NAT Instance** en una subnet pública.

### ¿Cómo agregaría HTTPS?
Asociando un certificado SSL/TLS (de AWS Certificate Manager) al listener del ALB en el puerto 443 y redirigiendo el tráfico HTTP (80) hacia HTTPS. Esto cifraría la comunicación entre el usuario y el balanceador.

### ¿Cómo registraría logs y métricas?
Habilitando **Access Logs** en el ALB para almacenar los logs de todas las solicitudes en un bucket de S3. Además, usando **Amazon CloudWatch** para monitorear métricas como latencia, número de solicitudes, y estado de los targets, con alarmas configuradas para notificar sobre comportamientos anómalos.

### ¿Cómo manejaría despliegues sin caída?
Implementando **Blue/Green Deployment** creando un nuevo Target Group con las instancias actualizadas, y luego cambiando la regla del listener del ALB para apuntar al nuevo Target Group. También se podría usar **rolling updates** con Auto Scaling, actualizando instancias de a una sin afectar la disponibilidad.

### ¿Qué componentes agregaría para una base de datos altamente disponible?
Usaría **Amazon RDS Multi-AZ**, que replica automáticamente la base de datos en otra zona de disponibilidad con failover automático. Para mayor escalabilidad, podría agregar **Amazon ElastiCache** como caché en memoria para reducir la carga en la base de datos.

---

## Reto Final (Sección 27)

### 1. Diagrama de arquitectura implementada

```
┌─────────────────────────────┐
│  Usuario / Navegador / curl  │
└─────────────────────────────┘
               │
               ▼
┌─────────────────────────────┐
│  Application Load Balancer  │
│  alb-ha-web (Internet-facing)│
│  DNS público                 │
└─────────────────────────────┘
               │
       ┌───────┴───────┐
       ▼               ▼
┌─────────────┐ ┌─────────────┐
│ EC2 Web A   │ │ EC2 Web B   │
│ web-ha-a    │ │ web-ha-b    │
│ us-east-1c  │ │ us-east-1a  │
│ Apache HTTPD│ │ Apache HTTPD│
│ Private IP  │ │ Private IP  │
└─────────────┘ └─────────────┘
       │               │
       └───────┬───────┘
               ▼
      ┌─────────────────┐
      │ Target Group    │
      │ tg-ha-web       │
      │ Health checks   │
      │ GET /health     │
      └─────────────────┘
```

### 2-6. Capturas de evidencia
Ver sección **Evidencias del Laboratorio** al inicio de este README.

### 7. Evidencia de falla simulada
Ver imágenes **Simulación de Falla - Solo responde Instancia B** y **Target Group - Instancia A detenida (Unhealthy)**.

### 8. Explicación de cómo el balanceador mantiene la disponibilidad
El Application Load Balancer mantiene la disponibilidad mediante:
- **Health checks**: verifica periódicamente que cada instancia responda correctamente en la ruta `/health`
- **Redirección de tráfico**: cuando una instancia falla el health check, el ALB automáticamente deja de enviarle solicitudes y redirige todo el tráfico hacia las instancias saludables
- **Redundancia**: al tener instancias en dos zonas de disponibilidad diferentes, el sistema tolera la falla de una zona completa
- **Punto único de entrada**: el usuario solo interactúa con el DNS del ALB, sin necesidad de conocer ni gestionar las IPs de las instancias individuales

### 9. Limitaciones de la arquitectura
- **Recuperación manual**: si una instancia falla, no se crea una nueva automáticamente; requiere intervención manual para reiniciarla o reemplazarla
- **Sin Auto Scaling**: no escala automáticamente ante aumentos de carga
- **Sin HTTPS**: el tráfico viaja sin cifrado (solo HTTP)
- **Instancias públicas**: las instancias EC2 están en subnets públicas, expuestas a Internet (aunque protegidas por Security Groups)
- **Sin monitoreo avanzado**: no se registran logs ni métricas detalladas del tráfico
- **Sin base de datos redundante**: si la aplicación usara una base de datos, esta sería un punto único de falla

### 10. Propuesta de mejora hacia producción
Ver **Actividad 4: Propuesta de Mejora** más arriba.
