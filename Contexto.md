# Guía de Laboratorio

## Alta Disponibilidad con Application Load Balancer en AWS

**Asignatura:** Arquitecturas de Software
**Duración sugerida:** 2 a 3 horas
**Modalidad:** Individual o parejas
**Plataforma:** AWS Academy Learner Lab
**Créditos disponibles:** 50 USD
**Restricción:** Sin creación ni uso de perfiles IAM personalizados

---

## 1. Propósito del laboratorio

En los sistemas modernos no basta con desplegar una aplicación en un único servidor. Si ese servidor falla, el sistema completo queda fuera de servicio. La alta disponibilidad busca reducir ese riesgo mediante redundancia, balanceo de carga, detección de fallos y distribución de componentes en varias zonas de disponibilidad.

En este laboratorio, el estudiante implementará una arquitectura básica de alta disponibilidad en AWS utilizando:

- Amazon EC2
- Application Load Balancer
- Target Group
- Security Groups
- Health Checks
- Dos zonas de disponibilidad

---

## 2. Resultados de aprendizaje

Al finalizar el laboratorio, el estudiante estará en capacidad de:

- Explicar qué es alta disponibilidad.
- Diferenciar disponibilidad, tolerancia a fallos y escalabilidad.
- Crear una arquitectura redundante con dos instancias EC2.
- Configurar un Application Load Balancer.
- Crear un Target Group con health checks.
- Validar el balanceo de carga entre instancias.
- Simular una falla y observar el comportamiento del balanceador.
- Relacionar la solución con atributos de calidad como disponibilidad, resiliencia y mantenibilidad.

---

## 3. Conceptos base

### 3.1 ¿Qué es alta disponibilidad?

La alta disponibilidad es la capacidad de un sistema para mantenerse operativo incluso cuando alguno de sus componentes falla.

Un sistema con una sola instancia tiene un punto único de falla:

```
Usuario
  ↓
Servidor único
```

Si ese servidor falla, el sistema queda indisponible.

Una arquitectura de alta disponibilidad introduce redundancia:

```
Usuario
  ↓
Balanceador de carga
  ↓
Servidor A    Servidor B
```

Si el Servidor A falla, el balanceador puede enviar tráfico al Servidor B.

---

### 3.2 Diferencia entre disponibilidad y escalabilidad

La disponibilidad responde a la pregunta:

*¿El sistema sigue funcionando si algo falla?*

La escalabilidad responde a la pregunta:

*¿El sistema puede atender más carga si aumenta la demanda?*

Una arquitectura puede ser escalable, pero no altamente disponible si todos sus componentes están en una sola zona de disponibilidad. También puede ser altamente disponible, pero no escalar automáticamente si no se configura Auto Scaling.

En este laboratorio nos enfocaremos principalmente en **alta disponibilidad con balanceo de carga**.

---

### 3.3 Balanceador de carga

Un balanceador de carga recibe solicitudes de los usuarios y las distribuye entre varios servidores.

En AWS usaremos un **Application Load Balancer**, que opera principalmente a nivel HTTP/HTTPS. Los Application Load Balancers permiten configurar health checks para monitorear los targets registrados y enviar tráfico únicamente a los que estén saludables.

---

### 3.4 Target Group

Un Target Group es un grupo de destinos a los cuales el balanceador enviará tráfico.

En este laboratorio, los targets serán dos instancias EC2.

```
Target Group: tg-ha-web
  ├── EC2 instancia A
  └── EC2 instancia B
```

Los Target Groups permiten registrar targets, como instancias EC2, usando un protocolo y puerto específico. También permiten configurar health checks para validar el estado de los targets.

---

### 3.5 Health Check

Un health check es una verificación periódica que realiza el balanceador para saber si una instancia puede recibir tráfico.

Por ejemplo:

```
GET /health
```

Si la instancia responde correctamente, se considera saludable.

Si no responde, se considera no saludable y el balanceador deja de enviarle tráfico.

AWS documenta que los Application Load Balancers envían solicitudes periódicas a los targets registrados para probar su estado; estas pruebas son los health checks.

---

## 4. Arquitectura del laboratorio

La arquitectura final será:

```
┌─────────────────────────────┐
│  Usuario / Navegador / curl  │
└─────────────────────────────┘
               │
               ▼
┌─────────────────────────────┐
│  Application Load Balancer  │
│  DNS público                │
└─────────────────────────────┘
               │
       ┌───────┴───────┐
       ▼               ▼
┌─────────────┐ ┌─────────────┐
│ EC2 Web A   │ │ EC2 Web B   │
│ AZ 1        │ │ AZ 2        │
│ Apache HTTPD│ │ Apache HTTPD│
└─────────────┘ └─────────────┘
```

Cada instancia mostrará una página diferente indicando:

- Nombre de la instancia
- ID de instancia
- Zona de disponibilidad
- Estado del servicio

Esto permitirá comprobar visualmente cuándo el balanceador envía tráfico a una u otra instancia.

---

## 5. Restricciones del laboratorio

Como los estudiantes trabajan en AWS Academy y no cuentan con perfiles IAM personalizados, se evitará:

- Crear roles IAM.
- Asignar instance profiles.
- Usar CloudWatch Agent.
- Usar SSM Session Manager.
- Usar Auto Scaling obligatorio.
- Usar infraestructura como código que requiera permisos avanzados.

El laboratorio se realizará desde la consola web de AWS y con scripts de inicialización en EC2.

---

## 6. Servicios AWS utilizados

Se usarán únicamente:

- EC2
- Application Load Balancer
- Target Groups
- Security Groups
- VPC predeterminada
- Subnets públicas existentes

No se requiere crear una VPC desde cero.

Esto reduce el riesgo de errores y consumo innecesario de créditos.

---

## 7. Consideraciones de costo

Aunque AWS Academy entrega créditos para laboratorio, los recursos consumen saldo mientras estén activos.

Para controlar costos:

- Usar instancias pequeñas, como t2.micro o t3.micro si están disponibles.
- Eliminar el Load Balancer al finalizar.
- Terminar las instancias EC2 al finalizar.
- Eliminar Target Groups innecesarios.
- No dejar recursos ejecutándose después de clase.

El balanceador de carga también consume créditos, por lo que debe eliminarse al terminar el laboratorio.

---

## 8. Preparación inicial

Ingrese a AWS Academy Learner Lab y abra la consola de AWS.

Verifique la región asignada por el laboratorio. Use una sola región durante toda la práctica.

Ejemplo:
```
us-east-1
us-west-2
```

No cambie de región durante el laboratorio para evitar confusión con recursos creados.

---

## 9. Crear Security Groups

Se crearán dos Security Groups:

- sg-alb-ha
- sg-ec2-ha

### 9.1 Security Group para el Load Balancer

**Nombre:** sg-alb-ha

**Descripción:** Permite tráfico HTTP desde Internet hacia el Application Load Balancer.

**Reglas de entrada:**

| Tipo | Protocolo | Puerto | Origen |
|------|-----------|--------|--------|
| HTTP | TCP | 80 | 0.0.0.0/0 |

**Reglas de salida:**

| Tipo | Protocolo | Puerto | Destino |
|------|-----------|--------|---------|
| All traffic | All | All | 0.0.0.0/0 |

---

### 9.2 Security Group para las instancias EC2

**Nombre:** sg-ec2-ha

**Descripción:** Permite tráfico HTTP únicamente desde el Load Balancer.

**Reglas de entrada:**

| Tipo | Protocolo | Puerto | Origen |
|------|-----------|--------|--------|
| HTTP | TCP | 80 | sg-alb-ha |

Opcionalmente, si el docente desea permitir conexión SSH:

| Tipo | Protocolo | Puerto | Origen |
|------|-----------|--------|--------|
| SSH | TCP | 22 | My IP |

Para el laboratorio no es obligatorio usar SSH, porque la configuración se hará con User Data.

---

## 10. Crear la primera instancia EC2

Ingrese a:

```
EC2 → Instances → Launch instances
```

Configure:

- **Name:** web-ha-a
- **AMI:** Amazon Linux 2023
- **Instance type:** t2.micro o t3.micro
- **Key pair:** seleccionar una existente o crear una nueva
- **Network:** VPC predeterminada
- **Subnet:** seleccionar una subnet pública en la primera zona de disponibilidad
- **Auto-assign public IP:** Enable
- **Security Group:** sg-ec2-ha
- **IAM instance profile:** None

En **Advanced details → User data**, agregue:

```bash
#!/bin/bash
dnf update -y
dnf install -y httpd

systemctl enable httpd
systemctl start httpd

TOKEN=$(curl -X PUT "http://169.254.169.254/latest/api/token" \
-H "X-aws-ec2-metadata-token-ttl-seconds: 21600")

INSTANCE_ID=$(curl -H "X-aws-ec2-metadata-token: $TOKEN" \
http://169.254.169.254/latest/meta-data/instance-id)

AZ=$(curl -H "X-aws-ec2-metadata-token: $TOKEN" \
http://169.254.169.254/latest/meta-data/placement/availability-zone)

cat <<EOF > /var/www/html/index.html
<!DOCTYPE html>
<html>
<head>
  <title>Alta Disponibilidad - Instancia A</title>
  <style>
    body {
      font-family: Arial, sans-serif;
      background: #f4f7fb;
      color: #1f2937;
      padding: 40px;
    }
    .card {
      background: white;
      border-radius: 12px;
      padding: 30px;
      max-width: 700px;
      margin: auto;
      box-shadow: 0 10px 25px rgba(0,0,0,0.08);
    }
    h1 {
      color: #2563eb;
    }
    .badge {
      display: inline-block;
      background: #dbeafe;
      color: #1d4ed8;
      padding: 6px 12px;
      border-radius: 20px;
      font-weight: bold;
    }
    code {
      background: #f3f4f6;
      padding: 4px 8px;
      border-radius: 6px;
    }
  </style>
</head>
<body>
  <div class="card">
    <span class="badge">Instancia A</span>
    <h1>Aplicación Web en Alta Disponibilidad</h1>
    <p>Esta respuesta fue generada por la instancia A.</p>
    <p><strong>Instance ID:</strong> <code>$INSTANCE_ID</code></p>
    <p><strong>Availability Zone:</strong> <code>$AZ</code></p>
    <p><strong>Estado:</strong> Servicio disponible</p>
  </div>
</body>
</html>
EOF

echo "OK" > /var/www/html/health
```

Cree la instancia.

---

## 11. Crear la segunda instancia EC2

Repita el proceso anterior, pero con estos cambios:

- **Name:** web-ha-b
- **Subnet:** seleccionar una subnet pública en una zona de disponibilidad diferente
- **Security Group:** sg-ec2-ha
- **IAM instance profile:** None

En User Data use este script:

```bash
#!/bin/bash
dnf update -y
dnf install -y httpd

systemctl enable httpd
systemctl start httpd

TOKEN=$(curl -X PUT "http://169.254.169.254/latest/api/token" \
-H "X-aws-ec2-metadata-token-ttl-seconds: 21600")

INSTANCE_ID=$(curl -H "X-aws-ec2-metadata-token: $TOKEN" \
http://169.254.169.254/latest/meta-data/instance-id)

AZ=$(curl -H "X-aws-ec2-metadata-token: $TOKEN" \
http://169.254.169.254/latest/meta-data/placement/availability-zone)

cat <<EOF > /var/www/html/index.html
<!DOCTYPE html>
<html>
<head>
  <title>Alta Disponibilidad - Instancia B</title>
  <style>
    body {
      font-family: Arial, sans-serif;
      background: #f4f7fb;
      color: #1f2937;
      padding: 40px;
    }
    .card {
      background: white;
      border-radius: 12px;
      padding: 30px;
      max-width: 700px;
      margin: auto;
      box-shadow: 0 10px 25px rgba(0,0,0,0.08);
    }
    h1 {
      color: #059669;
    }
    .badge {
      display: inline-block;
      background: #d1fae5;
      color: #047857;
      padding: 6px 12px;
      border-radius: 20px;
      font-weight: bold;
    }
    code {
      background: #f3f4f6;
      padding: 4px 8px;
      border-radius: 6px;
    }
  </style>
</head>
<body>
  <div class="card">
    <span class="badge">Instancia B</span>
    <h1>Aplicación Web en Alta Disponibilidad</h1>
    <p>Esta respuesta fue generada por la instancia B.</p>
    <p><strong>Instance ID:</strong> <code>$INSTANCE_ID</code></p>
    <p><strong>Availability Zone:</strong> <code>$AZ</code></p>
    <p><strong>Estado:</strong> Servicio disponible</p>
  </div>
</body>
</html>
EOF

echo "OK" > /var/www/html/health
```

Cree la instancia.

---

## 12. Verificar las instancias

Espere a que ambas instancias estén en estado:

```
Running
```

Y con status checks:

```
2/2 checks passed
```

Copie la IP pública de cada instancia y pruebe en navegador:

```
http://IP_PUBLICA_INSTANCIA_A
http://IP_PUBLICA_INSTANCIA_B
```

Debe visualizar una página diferente para cada instancia.

También pruebe el health check:

```
http://IP_PUBLICA_INSTANCIA_A/health
http://IP_PUBLICA_INSTANCIA_B/health
```

Debe responder:

```
OK
```

---

## 13. Crear el Target Group

Ingrese a:

```
EC2 → Target Groups → Create target group
```

Configure:

- **Target type:** Instances
- **Target group name:** tg-ha-web
- **Protocol:** HTTP
- **Port:** 80
- **VPC:** VPC predeterminada
- **Protocol version:** HTTP1

**Health checks:**

- **Protocol:** HTTP
- **Path:** /health
- **Port:** traffic port
- **Healthy threshold:** 2
- **Unhealthy threshold:** 2
- **Timeout:** 5 seconds
- **Interval:** 15 seconds
- **Success codes:** 200

Registre las dos instancias:

- web-ha-a
- web-ha-b

Cree el Target Group.

---

## 14. Crear el Application Load Balancer

Ingrese a:

```
EC2 → Load Balancers → Create load balancer
```

Seleccione:

```
Application Load Balancer
```

Configure:

- **Name:** alb-ha-web
- **Scheme:** Internet-facing
- **IP address type:** IPv4

**Network mapping:**

- **VPC:** VPC predeterminada
- Seleccionar al menos dos zonas de disponibilidad
- Seleccionar las subnets públicas donde están las instancias

**Security Group:**

- sg-alb-ha

**Listener:**

- **Protocol:** HTTP
- **Port:** 80
- **Default action:** forward to tg-ha-web

Cree el Load Balancer.

---

## 15. Verificar el estado de los targets

Ingrese a:

```
EC2 → Target Groups → tg-ha-web → Targets
```

Espere a que ambas instancias aparezcan como:

```
Healthy
```

AWS permite revisar la salud de los targets desde el Target Group; allí se muestra el estado de cada target registrado.

Si alguna aparece como Unhealthy, revise:

- La instancia está encendida.
- Apache está instalado y ejecutándose.
- La ruta /health responde OK.
- El Security Group de EC2 permite HTTP desde sg-alb-ha.
- El Target Group usa el puerto 80.

---

## 16. Probar el Load Balancer

Ingrese a:

```
EC2 → Load Balancers → alb-ha-web
```

Copie el DNS del Load Balancer.

Ejemplo:

```
alb-ha-web-123456.us-east-1.elb.amazonaws.com
```

Abra en el navegador:

```
http://DNS_DEL_LOAD_BALANCER
```

Actualice varias veces.

Debe observar que en algunas solicitudes responde la instancia A y en otras responde la instancia B.

También puede probar desde terminal:

```bash
for i in {1..10}; do
  curl -s http://DNS_DEL_LOAD_BALANCER | grep "Instancia"
done
```

---

## 17. Actividad 1: análisis del balanceo

Responda:

- ¿Qué instancia respondió primero?
- ¿El balanceador alternó entre ambas instancias?
- ¿Qué información permite confirmar que hay más de una instancia activa?
- ¿Qué papel cumple el Target Group?
- ¿Qué papel cumplen los health checks?
- ¿Por qué el usuario no necesita conocer las IP públicas de las instancias?

---

## 18. Simulación de falla

Ahora se simulará la caída de una instancia.

Seleccione la instancia web-ha-a.

Ingrese a:

```
EC2 → Instances → web-ha-a → Instance state → Stop instance
```

Confirme la acción.

Espere unos minutos.

Luego vaya a:

```
Target Groups → tg-ha-web → Targets
```

La instancia A debe pasar a estado:

```
Unhealthy
```

o dejar de recibir tráfico.

Pruebe nuevamente el DNS del Load Balancer:

```
http://DNS_DEL_LOAD_BALANCER
```

El sistema debe seguir respondiendo desde la instancia B.

---

## 19. Actividad 2: análisis de falla

Responda:

- ¿Qué ocurrió cuando se detuvo la instancia A?
- ¿El sistema completo dejó de estar disponible?
- ¿Qué hizo el Load Balancer cuando detectó la falla?
- ¿Qué diferencia habría si solo existiera una instancia?
- ¿Qué atributo de calidad mejora esta arquitectura?

---

## 20. Recuperación de la instancia

Inicie nuevamente la instancia A:

```
EC2 → Instances → web-ha-a → Instance state → Start instance
```

Espere a que pase los status checks.

Luego revise el Target Group.

La instancia debe volver a estado:

```
Healthy
```

Pruebe nuevamente el Load Balancer y verifique que ambas instancias vuelven a recibir tráfico.

---

## 21. Actividad 3: análisis de recuperación

Responda:

- ¿Qué ocurrió cuando la instancia A volvió a estar saludable?
- ¿El balanceador volvió a enviarle tráfico?
- ¿Por qué es importante que la recuperación sea automática desde el punto de vista del usuario?
- ¿Qué limitaciones tiene esta arquitectura si la instancia no se reinicia manualmente?

---

## 22. Discusión: alta disponibilidad vs recuperación automática

La arquitectura implementada mejora la disponibilidad porque el sistema puede seguir respondiendo aunque una instancia falle.

Sin embargo, todavía tiene una limitación:

Si una instancia falla, el balanceador deja de enviarle tráfico, pero no crea una nueva instancia automáticamente.

Para lograr recuperación automática se necesitaría agregar:

- Launch Template
- Auto Scaling Group
- Políticas de reemplazo
- Health checks integrados con ELB

Amazon EC2 Auto Scaling puede integrarse con Elastic Load Balancing para que el Auto Scaling Group use health checks del balanceador y reemplace instancias que se reporten como no saludables.

En AWS Academy, esta parte puede depender de los permisos disponibles. Por eso, en este laboratorio se deja como extensión opcional.

---

## 23. Extensión opcional: Auto Scaling Group

Esta sección solo debe realizarse si el entorno de AWS Academy lo permite.

La idea sería reemplazar la creación manual de instancias por:

- Launch Template
- Auto Scaling Group
- Desired capacity: 2
- Minimum capacity: 2
- Maximum capacity: 3
- Target Group: tg-ha-web

Con esto, si una instancia falla, Auto Scaling podría lanzar otra.

AWS recomienda usar un Application Load Balancer en tutoriales de aplicaciones escaladas y balanceadas con Auto Scaling.

Si el laboratorio no permite crear o usar algunas configuraciones, se mantiene la versión manual con dos EC2.

---

## 24. Validación arquitectónica

Complete la siguiente tabla:

| Elemento | Función en la arquitectura |
|----------|---------------------------|
| EC2 instancia A | |
| EC2 instancia B | |
| Application Load Balancer | |
| Target Group | |
| Health Check | |
| Security Group del ALB | |
| Security Group de EC2 | |
| Zonas de disponibilidad | |

---

## 25. Diagnóstico de errores comunes

### Caso 1: las instancias aparecen Unhealthy

Verifique:

- La ruta /health existe.
- Apache está activo.
- El Target Group usa HTTP puerto 80.
- El Security Group de EC2 permite tráfico desde sg-alb-ha.
- Las instancias están en la misma VPC del Target Group.

### Caso 2: el Load Balancer no carga en navegador

Verifique:

- El ALB es Internet-facing.
- El Security Group del ALB permite HTTP desde 0.0.0.0/0.
- Las subnets seleccionadas son públicas.
- El DNS se copió correctamente.
- El Target Group tiene al menos una instancia Healthy.

### Caso 3: solo responde una instancia

Puede ocurrir por:

- Una instancia está Unhealthy.
- El navegador conserva caché.
- El balanceador mantiene afinidad temporal.
- Una instancia no tiene Apache activo.
- El health check no está configurado correctamente.

Pruebe con curl varias veces para observar mejor el comportamiento.

---

## 26. Actividad 4: propuesta de mejora

A partir de la arquitectura construida, proponga una versión mejorada para producción.

Debe incluir:

- ¿Cómo agregaría recuperación automática?
- ¿Cómo protegería las instancias para que no sean públicas?
- ¿Cómo agregaría HTTPS?
- ¿Cómo registraría logs y métricas?
- ¿Cómo manejaría despliegues sin caída?
- ¿Qué componentes agregaría para una base de datos altamente disponible?

No se espera que implemente la solución completa, sino que justifique la evolución arquitectónica.

---

## 27. Reto final

El estudiante debe entregar un documento breve con:

1. Diagrama de arquitectura implementada.
2. Captura de las dos instancias EC2.
3. Captura del Target Group con targets Healthy.
4. Captura del Application Load Balancer.
5. Evidencia de respuesta desde instancia A.
6. Evidencia de respuesta desde instancia B.
7. Evidencia de falla simulada.
8. Explicación de cómo el balanceador mantiene la disponibilidad.
9. Limitaciones de la arquitectura.
10. Propuesta de mejora hacia producción.

---

## 28. Rúbrica de evaluación

| Criterio | Descripción | Peso |
|----------|-------------|------|
| Arquitectura implementada | Crea dos instancias en diferentes zonas y las registra en un Target Group. | 25% |
| Load Balancer funcional | El ALB recibe tráfico y lo distribuye correctamente. | 20% |
| Health checks | Configura y valida correctamente la salud de las instancias. | 15% |
| Simulación de falla | Demuestra que el sistema sigue funcionando cuando una instancia falla. | 15% |
| Análisis arquitectónico | Relaciona la solución con alta disponibilidad, redundancia y tolerancia a fallos. | 15% |
| Limpieza de recursos | Elimina los recursos creados para evitar consumo innecesario de créditos. | 10% |

---

## 29. Limpieza de recursos

Al finalizar el laboratorio, elimine los recursos en este orden:

1. Application Load Balancer.
2. Target Group.
3. Instancias EC2.
4. Security Groups creados.

**Importante:**

- No deje el Load Balancer activo.
- No deje instancias EC2 ejecutándose.
- Verifique que no existan recursos asociados consumiendo créditos.

---

## 30. Cierre del laboratorio

Este laboratorio permitió implementar una arquitectura básica de alta disponibilidad en AWS utilizando un Application Load Balancer y dos instancias EC2 distribuidas en diferentes zonas de disponibilidad.

El aprendizaje central es que la alta disponibilidad no depende únicamente de "tener más servidores", sino de diseñar correctamente:

- Redundancia
- Balanceo de carga
- Health checks
- Distribución por zonas
- Seguridad de red
- Recuperación ante fallos
- Monitoreo
- Limpieza operativa