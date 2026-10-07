# Tarea 1 - INF326 Arquitectura de Software

## Sistema de notificación de sismos

Este proyecto implementa un sistema de notificación de sismos utilizando una arquitectura **publish-subscribe** con una adaptación del modelo **pull** para obtener información detallada mediante HTTP.

La solución utiliza:

- **RabbitMQ** como sistema de mensajería.
- **FastAPI** para el servicio HTTP de datos sísmicos.
- **Python** para el publisher y los subscribers.

## Integrantes

| Nombre | Rol |
|---|---|
| Matías Acuña | 202210097-4 |
| Catalina M. Rosales | 202073030-K |

## Arquitectura

El sistema está compuesto por:

- Un **publisher** que notifica la ocurrencia de un sismo.
- Un **exchange fanout** en RabbitMQ.
- Cinco **subscribers**, correspondientes a:
  - Arica
  - Coquimbo
  - Valparaíso
  - Concepción
  - Punta Arenas
- Un servicio HTTP implementado con FastAPI.

El publisher envía un mensaje mínimo con la siguiente estructura:

```json
{
  "id": "sismo-001",
  "latitud": -33.036,
  "longitud": -71.629
}
```

Cada subscriber recibe el evento y calcula la distancia entre su ubicación y el epicentro del sismo.

Si la distancia es menor a **500 km**, el subscriber consulta el servicio HTTP:

```text
GET /sismos/{sismo_id}
```

De esta forma, la mensajería se utiliza para notificar la ocurrencia del evento y el servicio HTTP se utiliza para obtener los datos detallados solo cuando son necesarios.

## Estructura del proyecto

```text
.
├── api.py
├── publisher.py
├── subscriber_common.py
├── subscriber_arica.py
├── subscriber_coquimbo.py
├── subscriber_valparaiso.py
├── subscriber_concepcion.py
├── subscriber_punta_arenas.py
├── docker-compose.yml
├── requirements.txt
├── README.md
└── DISCUSSION.md
```

## Requisitos

Para ejecutar el proyecto se requiere:

- Python 3.10 o superior
- Docker
- Docker Compose

Dependencias de Python:

```text
fastapi
uvicorn
pika
requests
```

## Instalación

Se recomienda crear un entorno virtual:

### Linux / macOS

```bash
python3 -m venv .venv
source venv/bin/activate
```

### Windows

```bash
python -m venv .venv
.venv\Scripts\activate
```

Instalar las dependencias:

```bash
pip install -r requirements.txt
```

## Ejecución

### 1. Levantar RabbitMQ

Si el contenedor todavía no existe:

```bash
docker compose up -d
```

Si ya existe pero está detenido:

```bash
docker start rabbitmq
```

Para verificar que RabbitMQ esté ejecutándose:

```bash
docker ps
```

La interfaz de administración queda disponible en:

```text
http://localhost:15672
```

Credenciales por defecto:

```text
usuario: guest
contraseña: guest
```

### 2. Levantar el servicio HTTP

En una terminal:

```bash
uvicorn api:app --reload
```

El servicio queda disponible en:

```text
http://localhost:8000
```

Ejemplo:

```text
http://localhost:8000/sismos/sismo-001
```

### 3. Levantar los cinco subscribers

Cada subscriber debe ejecutarse en una terminal distinta.

#### Arica

```bash
python subscriber_arica.py
```

#### Coquimbo

```bash
python subscriber_coquimbo.py
```

#### Valparaíso

```bash
python subscriber_valparaiso.py
```

#### Concepción

```bash
python subscriber_concepcion.py
```

#### Punta Arenas

```bash
python subscriber_punta_arenas.py
```

Cada subscriber posee una cola propia enlazada al exchange `sismos`, de modo que todos reciben una copia del mismo evento.

### 4. Ejecutar el publisher

Una vez iniciados los cinco subscribers:

```bash
python publisher.py
```

El publisher enviará el evento al exchange `sismos`.

Cada subscriber:

1. recibe el mensaje;
2. calcula su distancia al sismo;
3. determina si está a menos de 500 km;
4. si está interesado, consulta el servicio HTTP para obtener los detalles;
5. si no está interesado, no realiza ninguna consulta HTTP.

## Prueba del sistema

La implementación incluye sismos de prueba almacenados en memoria dentro de `api.py`.

Por ejemplo, `sismo-001` corresponde a un evento cercano a Valparaíso.

Al publicarlo:

```json
{
  "id": "sismo-001",
  "latitud": -33.036,
  "longitud": -71.629
}
```

los cinco subscribers reciben la notificación, pero solo aquellos ubicados a menos de 500 km realizan una consulta al servicio HTTP.

También puede utilizarse `sismo-002`, ubicado cerca de Arica, para verificar el comportamiento con una ubicación diferente.

## Diseño de mensajes

El evento publicado mediante RabbitMQ contiene únicamente:

- `id`
- `latitud`
- `longitud`

No se incluyen datos como magnitud, profundidad, escala o referencia debido a que estos no son necesarios para decidir si el evento se encuentra dentro del radio de interés.

La información detallada se obtiene posteriormente mediante HTTP.

## Consideraciones de implementación

- Se utiliza un exchange RabbitMQ de tipo `fanout`.
- Cada subscriber utiliza una cola independiente.
- El cálculo de distancia se realiza mediante la fórmula de Haversine.
- El radio de interés es de 500 km.
- Los datos sísmicos se mantienen en memoria.
- Los subscribers deben estar iniciados antes de publicar un evento para garantizar que sus colas estén declaradas y enlazadas al exchange.
- La API y RabbitMQ deben estar disponibles antes de ejecutar el flujo completo.

## Detener el sistema

Los procesos de FastAPI y de los subscribers pueden detenerse con:

```text
Ctrl + C
```

Para detener el contenedor de RabbitMQ sin eliminarlo:

```bash
docker stop rabbitmq
```

Para detener y eliminar los recursos creados por Docker Compose:

```bash
docker compose down
```

## Discusión

La discusión sobre los trade-offs de la arquitectura y el posible uso de una estimación tipo Back of the Envelope se encuentra en:

```text
DISCUSSION.md
```
