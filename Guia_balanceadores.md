# Guía Detallada: Alta Disponibilidad con Application Load Balancer en AWS

## Versión basada en experiencia práctica — incluye errores comunes y solución

---

## Índice

1. [Preparación inicial](#1-preparación-inicial)
2. [Crear Security Groups](#2-crear-security-groups)
3. [Lanzar instancia web-ha-a](#3-lanzar-instancia-web-ha-a)
4. [Lanzar instancia web-ha-b](#4-lanzar-instancia-web-ha-b)
5. [Verificar las instancias](#5-verificar-las-instancias)
6. [Solucionar problemas de conectividad](#6-solucionar-problemas-de-conectividad)
7. [Crear el Target Group](#7-crear-el-target-group)
8. [Crear el Application Load Balancer](#8-crear-el-application-load-balancer)
9. [Verificar targets Healthy](#9-verificar-targets-healthy)
10. [Probar el balanceo de carga](#10-probar-el-balanceo-de-carga)
11. [Simulación de falla](#11-simulación-de-falla)
12. [Recuperación de la instancia](#12-recuperación-de-la-instancia)
13. [Actividades de análisis](#13-actividades-de-análisis)
14. [Validación arquitectónica](#14-validación-arquitectónica)
15. [Propuesta de mejora](#15-propuesta-de-mejora)
16. [Limpieza de recursos](#16-limpieza-de-recursos)
17. [Errores comunes y soluciones](#17-errores-comunes-y-soluciones)

---

## 1. Preparación inicial

1. Ingresa a **AWS Academy Learner Lab** y abre la consola de AWS
2. Verifica la región asignada (ej. `us-east-1`, `us-west-2`)
3. **Importante:** usa una sola región durante todo el laboratorio
4. Recursos recomendados: instancias `t2.micro` o `t3.micro`

---

## 2. Crear Security Groups

### 2.1 Security Group para el ALB (`sg-alb-ha`)

1. Ve a **EC2 → Security Groups → Create security group**
2. Configura:

| Campo | Valor |
|---|---|
| **Name** | `sg-alb-ha` |
| **Description** | Permite HTTP desde Internet hacia el ALB |
| **VPC** | VPC predeterminada |

#### Reglas de entrada (Inbound rules)

| Tipo | Protocolo | Puerto | Origen | Descripción |
|------|-----------|--------|--------|-------------|
| HTTP | TCP | 80 | `0.0.0.0/0` | HTTP desde cualquier lugar IPv4 |
| HTTP | TCP | 80 | `::/0` | HTTP desde cualquier lugar IPv6 |

#### Reglas de salida (Outbound rules)

| Tipo | Protocolo | Puerto | Destino |
|------|-----------|--------|---------|
| All traffic | All | All | `0.0.0.0/0` |

---

### 2.2 Security Group para las instancias EC2 (`sg-ec2-ha`)

1. **Create security group**
2. Configura:

| Campo | Valor |
|---|---|
| **Name** | `sg-ec2-ha` |
| **Description** | Permite HTTP solo desde el ALB |
| **VPC** | VPC predeterminada |

#### Reglas de entrada (Inbound rules)

| Tipo | Protocolo | Puerto | Origen | Descripción |
|------|-----------|--------|--------|-------------|
| HTTP | TCP | 80 | **ID de sg-alb-ha** (ej. `sg-0ea21...`) | HTTP solo desde el ALB |
| HTTP | TCP | 80 | `0.0.0.0/0` | HTTP desde Internet (solo para pruebas iniciales) |

> **Nota importante:** La regla `0.0.0.0/0` es **temporal** para verificar que Apache funciona al acceder directamente por IP pública. Una vez confirmado, se puede eliminar. El ALB usará la regla con origen `sg-alb-ha`.

**Opcional — SSH:**

| Tipo | Protocolo | Puerto | Origen |
|------|-----------|--------|--------|
| SSH | TCP | 22 | Tu IP |

#### Reglas de salida (Outbound rules)

| Tipo | Protocolo | Puerto | Destino |
|------|-----------|--------|---------|
| All traffic | All | All | `0.0.0.0/0` |

---

## 3. Lanzar instancia web-ha-a

1. Ve a **EC2 → Instances → Launch instances**
2. Configura:

| Campo | Valor |
|---|---|
| **Name** | `web-ha-a` |
| **AMI** | Amazon Linux 2023 |
| **Instance type** | `t2.micro` (o `t3.micro`) |
| **Key pair** | Seleccionar existente o "Proceed without a key pair" |
| **Network** | VPC predeterminada |
| **Subnet** | Subnet pública (ej. `us-east-1c`) |
| **Auto-assign public IP** | **Enable** |
| **Security Group** | Seleccionar existente → **`sg-ec2-ha`** |

### User Data (Advanced details → User data)

Pega el siguiente script:

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

3. **Launch instance**

---

## 4. Lanzar instancia web-ha-b

Repite el proceso anterior con estos cambios:

| Campo | Valor |
|---|---|
| **Name** | `web-ha-b` |
| **Subnet** | Subnet pública en **zona de disponibilidad diferente** (ej. `us-east-1a` o `us-east-1d`) |
| **Security Group** | **`sg-ec2-ha`** |

### User Data para Instancia B

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

---

## 5. Verificar las instancias

1. Ve a **EC2 → Instances**
2. Espera a que ambas tengan **InstanceState:** `Running` y **Status check:** `2/2 checks passed`
3. Copia la **Public IPv4 address** de cada instancia
4. Prueba en navegador o terminal:

```bash
curl http://IP_PUBLICA_A
curl http://IP_PUBLICA_B
```

5. Prueba el health check:

```bash
curl http://IP_PUBLICA_A/health
curl http://IP_PUBLICA_B/health
```

Debe responder `OK`.

---

## 6. Solucionar problemas de conectividad

### Error común: Timeout al conectar por IP pública

Si obtienes `Failed to connect — Timed out`, revisa en este orden:

#### 6.1 Verificar Security Group
- **EC2 → Security Groups → sg-ec2-ha → Inbound rules**
- Debe tener HTTP (80) desde `0.0.0.0/0` para pruebas directas

#### 6.2 Verificar que la subnet es pública
1. **EC2 → Instances → selecciona la instancia → pestaña Networking**
2. Copia el **Subnet ID**
3. Ve a **VPC → Subnets** y busca esa subnet
4. Ve a la pestaña **Route Table**
5. Debe tener una ruta:
   ```
   0.0.0.0/0 → igw-xxxxxxxx (Internet Gateway)
   ```

**Solución si falta la ruta:**
1. En la tabla de rutas, haz clic en **Edit routes**
2. **Add route:**
   - **Destination:** `0.0.0.0/0`
   - **Target:** Internet Gateway (selecciona el existente, ej. `igw-xxxxx`)
3. **Save changes**

**Si no existe Internet Gateway:**
1. Ve a **VPC → Internet Gateways → Create internet gateway**
2. **Name:** `igw-ha` → **Create**
3. Selecciona el IGW → **Actions → Attach to VPC** → elige la VPC predeterminada
4. Luego vuelve a la tabla de rutas y agrega la ruta como se indicó arriba

#### 6.3 Verificar que la instancia tiene el Security Group correcto

**Error común:** Al crear la instancia, a veces se asigna el grupo `launch-wizard-1` en lugar de `sg-ec2-ha`.

**Solución:**
1. **EC2 → Instances → selecciona la instancia**
2. **Actions → Security → Change security groups**
3. Desmarca `launch-wizard-xxx`
4. Marca **`sg-ec2-ha`**
5. **Save**

#### 6.4 Verificar que Apache está corriendo
1. **EC2 → Instances → selecciona la instancia**
2. **Actions → Monitor and troubleshoot → Get instance screenshot**
3. Si la pantalla está en negro, Apache no arrancó. Revisa el user data.

---

## 7. Crear el Target Group

1. Ve a **EC2 → Target Groups → Create target group**
2. Configura:

| Campo | Valor |
|---|---|
| **Target type** | **Instances** |
| **Target group name** | `tg-ha-web` |
| **Protocol** | HTTP |
| **Port** | 80 |
| **VPC** | VPC predeterminada |
| **Protocol version** | HTTP1 |

### Health checks

| Campo | Valor |
|---|---|
| **Protocol** | HTTP |
| **Path** | `/health` |
| **Port** | traffic port |
| **Healthy threshold** | 2 |
| **Unhealthy threshold** | 2 |
| **Timeout** | 5 seconds |
| **Interval** | 15 seconds |
| **Success codes** | 200 |

3. **Next**
4. En **Register targets**, selecciona `web-ha-a` y `web-ha-b`
5. **Include as pending below → Create target group**

---

## 8. Crear el Application Load Balancer

1. Ve a **EC2 → Load Balancers → Create load balancer**
2. Haz clic en **Create** bajo **Application Load Balancer**
3. Configura:

| Campo | Valor |
|---|---|
| **Name** | `alb-ha-web` |
| **Scheme** | **Exposed to the internet** (Internet-facing) |
| **IP address type** | **IPv4** |

### Network mapping
- **VPC:** VPC predeterminada
- **Subnets:** marca **dos subnets públicas** (las mismas donde están tus instancias)

### Security groups
- Desmarca el grupo por defecto
- **Marca solo `sg-alb-ha`**

### Listeners and routing
- **Protocol:** HTTP
- **Port:** 80
- **Default action:** Forward to → **`tg-ha-web`**

4. **Create load balancer**
5. Espera a que el estado pase a **Active**

---

## 9. Verificar targets Healthy

### Error común: "Target is in an Availability Zone that is not enabled for the load balancer"

**Causa:** El ALB solo tiene subnets de ciertas AZs, pero la instancia está en una AZ no cubierta.

**Solución:**
1. Ve a **EC2 → Load Balancers → `alb-ha-web`**
2. Pestaña **Details → Network mapping → Edit subnets**
3. Marca también la subnet pública de la AZ donde está la instancia faltante
4. **Save**
5. Ve a **Target Groups → `tg-ha-web` → Targets → Refresh**

Ambas deben aparecer como **Healthy**.

### Error común: Target aparece como "Unused"
- Revisa que el ALB esté en estado **Active**
- Si hay instancias viejas (terminadas) listadas, selecciónalas y haz **Deregister**
- Si es una instancia nueva, regístrala con **Register targets**

---

## 10. Probar el balanceo de carga

1. Copia el **DNS name** del ALB: `alb-ha-web-xxxxx.us-east-1.elb.amazonaws.com`
2. Prueba en el navegador o terminal:

```bash
curl http://DNS_DEL_ALB
```

3. Para ver el balanceo claramente:

```bash
for i in {1..10}; do curl -s http://DNS_DEL_ALB | grep "Instancia"; done
```

Debe alternar entre "Instancia A" e "Instancia B".

---

## 11. Simulación de falla

1. Ve a **EC2 → Instances → selecciona `web-ha-a`**
2. **Actions → Instance state → Stop instance** → Confirma
3. Ve a **Target Groups → `tg-ha-web` → Targets**
4. Espera 1-2 minutos y haz **Refresh**
   - `web-ha-a`: pasa a **Unhealthy** o "Target is in the stopped state"
5. Prueba el DNS del ALB:

```bash
for i in {1..10}; do curl -s http://DNS_DEL_ALB | grep "Instancia"; done
```

Solo debe responder "Instancia B".

---

## 12. Recuperación de la instancia

1. Ve a **EC2 → Instances → selecciona `web-ha-a`**
2. **Actions → Instance state → Start instance** → Confirma
3. Espera a que esté **Running** con **2/2 checks passed**
4. Ve a **Target Groups → `tg-ha-web` → Targets → Refresh**
   - `web-ha-a` debe volver a **Healthy**
5. Prueba el ALB — debe alternar entre A y B de nuevo

---

## 13. Actividades de análisis

### Actividad 1: Análisis del balanceo

1. **¿Qué instancia respondió primero?** — Depende del orden del ALB, puede ser A o B.
2. **¿El balanceador alternó entre ambas instancias?** — Sí, al recargar el DNS del ALB las respuestas alternan entre A y B.
3. **¿Qué información permite confirmar que hay más de una instancia activa?** — Cada página muestra el Instance ID y la Availability Zone diferentes.
4. **¿Qué papel cumple el Target Group?** — Registra las instancias, define puerto/protocolo y ejecuta health checks.
5. **¿Qué papel cumplen los health checks?** — Verifican periódicamente que las instancias respondan correctamente; si fallan, el ALB deja de enviarles tráfico.
6. **¿Por qué el usuario no necesita conocer las IP públicas?** — El ALB es el punto único de entrada (DNS), abstrae los cambios de instancias y mantiene la disponibilidad.

### Actividad 2: Análisis de falla

1. **¿Qué ocurrió cuando se detuvo la instancia A?** — El health check falló, el ALB marcó A como Unhealthy y dejó de enviarle tráfico.
2. **¿El sistema completo dejó de estar disponible?** — No, la instancia B continuó atendiendo las solicitudes.
3. **¿Qué hizo el Load Balancer?** — Detectó la falla via health check, eliminó A del pool y redirigió todo el tráfico a B.
4. **¿Diferencia si solo existiera una instancia?** — El sistema habría quedado completamente fuera de servicio.
5. **¿Qué atributo de calidad mejora?** — Disponibilidad, resiliencia y tolerancia a fallos.

### Actividad 3: Análisis de recuperación

1. **¿Qué ocurrió cuando la instancia A volvió a estar saludable?** — El health check la verificó, el ALB la reincorporó al pool.
2. **¿El balanceador volvió a enviarle tráfico?** — Sí, alternando entre A y B nuevamente.
3. **¿Por qué es importante la recuperación automática?** — Para que el usuario no perciba interrupciones y el servicio se mantenga disponible.
4. **¿Limitaciones?** — Sin Auto Scaling, si la instancia no se reinicia manualmente, solo queda una instancia activa, perdiendo redundancia.

---

## 14. Validación arquitectónica

| Elemento | Función |
|----------|---------|
| **EC2 instancia A** | Servidor web Apache en AZ 1 (us-east-1c) |
| **EC2 instancia B** | Servidor web Apache en AZ 2 (us-east-1a) |
| **Application Load Balancer** | Punto de entrada único, distribuye tráfico entre instancias |
| **Target Group** | Agrupa instancias, configura health checks |
| **Health Check** | Verifica disponibilidad de cada instancia (GET /health) |
| **Security Group del ALB** | Permite HTTP (80) desde Internet |
| **Security Group de EC2** | Permite HTTP solo desde el ALB |
| **Zonas de disponibilidad** | Redundancia geográfica dentro de la región |

---

## 15. Propuesta de mejora

- **Recuperación automática:** Auto Scaling Group con Launch Template
- **Seguridad:** Instancias en subnets privadas con NAT Gateway
- **HTTPS:** Certificado SSL en ACM + listener 443 con redirect 80→443
- **Monitoreo:** Access Logs del ALB en S3 + CloudWatch metrics y alarms
- **Despliegues sin caída:** Blue/Green deployment con múltiples Target Groups
- **Base de datos HA:** RDS Multi-AZ con failover automático

---

## 16. Limpieza de recursos

> **Importante:** Los recursos consumen créditos mientras estén activos. Elimínalos en este orden:

1. **Load Balancer:** EC2 → Load Balancers → `alb-ha-web` → Actions → Delete
2. **Target Group:** EC2 → Target Groups → `tg-ha-web` → Actions → Delete
3. **Instancias EC2:** EC2 → Instances → seleccionar ambas → Instance state → Terminate
4. **Security Groups:** EC2 → Security Groups → `sg-alb-ha`, `sg-ec2-ha` → Actions → Delete

---

## 17. Errores comunes y soluciones

| Error | Causa | Solución |
|-------|-------|----------|
| Timeout al conectar por IP pública | Subnet sin ruta a Internet Gateway | Agregar ruta `0.0.0.0/0` → IGW en la tabla de rutas |
| No responde el health check | El archivo `/health` no se creó | Revisar user data, recrear instancia |
| Target "Unused" en Target Group | ALB no está en AZ de la instancia | Agregar subnet de esa AZ al ALB |
| Instancia con "launch-wizard-1" en vez de sg-ec2-ha | Error al seleccionar SG al crear instancia | Actions → Security → Change security groups |
| ALB no se crea por permisos | Política de AWS Academy restringe | Verificar con el docente si el lab permite ALB |
| Solo responde una instancia | Stickiness habilitado o la otra está Unhealthy | Deshabilitar stickiness en Target Group |
