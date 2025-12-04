# Datadog-Daemon-EC2-and-App-Instrumentation
Ejemplo y guía para desplegar el agente de Datadog como un servicio Daemon en un clúster de AWS ECS con   instancias EC2.

# Guía: Desplegar el Agente de Datadog como Daemon en ECS con EC2

Esta guía explica paso a paso cómo desplegar el agente de Datadog como un servicio de tipo `DAEMON` en un clúster de Amazon ECS que utiliza instancias EC2 gestionadas por un Auto Scaling Group.

El modo `DAEMON` asegura que una (y solo una) instancia del agente de Datadog se ejecute en cada instancia EC2 del clúster. Esto es eficiente y permite que todas las aplicaciones en la misma instancia envíen métricas, trazas y logs al agente local.

## Prerrequisitos

Antes de empezar, asegúrate de tener lo siguiente:

1.  **Un Clúster de ECS con instancias EC2:** Debes tener un clúster de ECS funcional que utilice un "Capacity Provider" de tipo Auto Scaling para lanzar instancias EC2.
2.  **Datadog API Key:** Tu clave de API de Datadog.
3.  **AWS Secrets Manager:** La clave de API de Datadog debe estar almacenada de forma segura en AWS Secrets Manager.
4.  **AWS CLI:** La Interfaz de Línea de Comandos de AWS configurada con los permisos necesarios para modificar ECS y IAM.

---

## Paso 1: Configurar los Permisos de IAM

El agente de Datadog necesita permisos para leer la clave de API desde Secrets Manager. Estos permisos se deben añadir al rol que utilizan las tareas de ECS para su ejecución.

**Rol a modificar:** `ecsTaskExecutionRole` (o el nombre que le hayas dado a tu rol de ejecución de tareas).

Añade la siguiente política de permisos a dicho rol:

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": [
                "secretsmanager:GetSecretValue"
            ],
            "Resource": [
                "arn:aws:secretsmanager:REGION:ACCOUNT_ID:secret:SECRET_NAME"
            ]
        }
    ]
}
```
**Importante:** Reemplaza `REGION`, `ACCOUNT_ID` y `SECRET_NAME` con los valores correspondientes al ARN de tu secreto donde guardas la API Key.

---

## Paso 2: Crear la Definición de Tarea (Task Definition) para el Agente

Este es el corazón de la configuración. Define cómo se debe ejecutar el contenedor del agente de Datadog. A continuación se muestra un ejemplo completo en formato JSON. Puedes guardarlo como `datadog-agent-daemon-td.json`.

```json
{
    "family": "DataDog-Daemon-EC2",
    "networkMode": "awsvpc",
    "requiresCompatibilities": [
        "EC2"
    ],
    "executionRoleArn": "arn:aws:iam::ACCOUNT_ID:role/ecsTaskExecutionRole",
    "cpu": "100",
    "memory": "512",
    "containerDefinitions": [
        {
            "name": "datadog-agent",
            "image": "public.ecr.aws/datadog/agent:latest",
            "essential": true,
            "secrets": [
                {
                    "name": "DD_API_KEY",
                    "valueFrom": "arn:aws:secretsmanager:REGION:ACCOUNT_ID:secret:SECRET_NAME"
                }
            ],
            "environment": [
                {
                    "name": "DD_SITE",
                    "value": "datadoghq.com"
                },
                {
                    "name": "DD_LOGS_ENABLED",
                    "value": "true"
                },
                {
                    "name": "DD_LOGS_CONFIG_CONTAINER_COLLECT_ALL",
                    "value": "true"
                },
                {
                    "name": "DD_APM_ENABLED",
                    "value": "true"
                },
                {
                    "name": "DD_PROCESS_AGENT_ENABLED",
                    "value": "true"
                }
            ],
            "mountPoints": [
                {
                    "sourceVolume": "docker_sock",
                    "containerPath": "/var/run/docker.sock",
                    "readOnly": true
                },
                {
                    "sourceVolume": "proc",
                    "containerPath": "/host/proc",
                    "readOnly": true
                },
                {
                    "sourceVolume": "cgroup",
                    "containerPath": "/host/sys/fs/cgroup",
                    "readOnly": true
                }
            ],
            "portMappings": [
                {
                    "containerPort": 8126,
                    "hostPort": 8126,
                    "protocol": "tcp"
                }
            ]
        }
    ],
    "volumes": [
        {
            "name": "docker_sock",
            "host": {
                "sourcePath": "/var/run/docker.sock"
            }
        },
        {
            "name": "proc",
            "host": {
                "sourcePath": "/proc/"
            }
        },
        {
            "name": "cgroup",
            "host": {
                "sourcePath": "/sys/fs/cgroup/"
            }
        }
    ]
}
```

**Puntos Clave de la Definición de Tarea:**

*   **`secrets`**: Esta es la forma **correcta** de inyectar la `DD_API_KEY`. El agente de ECS obtiene el valor del secreto y lo establece como una variable de entorno para el contenedor.
*   **`environment`**:
    *   `DD_SITE`: Asegúrate de que coincida con tu región de Datadog (ej. `datadoghq.eu` para Europa).
    *   `DD_LOGS_ENABLED` y `DD_APM_ENABLED`: Habilitan la recolección de logs y trazas de APM.
*   **`mountPoints` y `volumes`**: Son cruciales. Permiten al agente acceder al socket de Docker y a los sistemas de archivos del host para recopilar métricas de todos los contenedores y de la propia instancia EC2.

Puedes registrar esta definición de tarea usando la AWS CLI:
```bash
aws ecs register-task-definition --cli-input-json file://datadog-agent-daemon-td.json
```

---

## Paso 3: Crear el Servicio Daemon

Una vez registrada la Task Definition, crea el servicio en tu clúster, asegurándote de que el tipo de despliegue sea `DAEMON`.

```bash
aws ecs create-service \
  --cluster ecs-ec2 \
  --service-name "datadog-daemon-agent" \
  --task-definition DataDog-Daemon-EC2 \
  --scheduling-strategy DAEMON
```

*   `--scheduling-strategy DAEMON`: Este es el parámetro clave que le dice a ECS que coloque una instancia de esta tarea en cada instancia EC2 del clúster.

---

## Paso 4: Configurar tus Aplicaciones

Ahora, tus contenedores de aplicación necesitan saber dónde enviar sus datos. Deben apuntar a la IP de la instancia EC2 en la que se están ejecutando, ya que el agente de Datadog está corriendo allí.

La mejor manera de hacer esto es dinámicamente al iniciar el contenedor de la aplicación.

**Modifica la Task Definition de tu aplicación:**

En la definición del contenedor de tu aplicación, debes usar un `entrypoint` y `command` para obtener la IP del host y exportarla como una variable de entorno.

**Si tus instancias requieren IMDSv2 (recomendado):**

```json
"entryPoint": ["sh", "-c"],
"command": [
    "TOKEN=$(curl -X PUT 'http://169.254.169.254/latest/api/token' -H 'X-aws-ec2-metadata-token-ttl-seconds: 21600') && export DD_AGENT_HOST=$(curl -H 'X-aws-ec2-metadata-token: $TOKEN' -s http://169.254.169.254/latest/meta-data/local-ipv4) && /tu-comando-de-inicio"
]
```

**Si tus instancias usan IMDSv1:**

```json
"entryPoint": ["sh", "-c"],
"command": [
    "export DD_AGENT_HOST=$(curl -s http://169.254.169.254/latest/meta-data/local-ipv4) && /tu-comando-de-inicio"
]
```

**Importante:** Reemplaza `/tu-comando-de-inicio` con el comando real que arranca tu aplicación (por ejemplo, `gunicorn --workers 4 --bind 0.0.0.0:8080 app:app` para una aplicación Python/Flask).

**Otras variables de entorno para tu aplicación:**

Asegúrate de que tu aplicación también tenga estas variables para una mejor trazabilidad:
```json
"environment": [
    {
        "name": "DD_SERVICE",
        "value": "nombre-de-tu-servicio"
    },
    {
        "name": "DD_ENV",
        "value": "production"
    },
    {
        "name": "DD_VERSION",
        "value": "1.0.0"
    }
]
```

---

## Paso 5: Verificación

Una vez que hayas actualizado tus servicios de aplicación para que usen la nueva configuración, deberías poder ver los datos fluyendo hacia Datadog:

1.  **Infraestructura > Host Map:** Tus instancias EC2 deberían aparecer aquí.
2.  **APM > Traces:** Las trazas de tu aplicación deberían empezar a aparecer a medida que recibe tráfico.
3.  **Logs > Search:** Los logs de tus contenedores deberían ser recolectados automáticamente.
