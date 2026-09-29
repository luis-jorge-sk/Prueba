# API Management aplicado al Módulo de Despachos y Reportes
### Tópicos en Diseño de Software — Componente Individual

**Tema grupal:** Sistema ERP para gestión de ventas mediante WhatsApp
**Módulo de enfoque:** Despachos y Reportes
**Tema individual:** API Management

---
![](https://github.com/luis-jorge-sk/Prueba/blob/b2da6b09b9e70c5be601e8b45d5914d62b27fa81/pc1/imagenes/API-management-DataScientest.webp)

# Desarrollo conceptual

## API Management

**API Management** (Gestión de APIs) es el conjunto de prácticas, herramientas y procesos que permiten diseñar, publicar, documentar, proteger, monitorear y analizar el uso de las APIs de una organización. Su objetivo es actuar como una capa intermedia entre los consumidores de una API (aplicaciones, bots, usuarios externos) y los servicios backend que realmente procesan la información.

Este concepto es independiente de cualquier proveedor de nube o tecnología específica: puede implementarse con herramientas open-source instaladas localmente, como servicio administrado en la nube, o como una combinación de ambos (modelo híbrido).

## Componentes principales

| Componente | Función |
|---|---|
| **API Gateway** | Punto único de entrada que enruta las peticiones hacia los servicios backend correspondientes |
| **Portal de desarrolladores** | Espacio donde se documentan y prueban las APIs disponibles |
| **Motor de políticas** | Aplica reglas de seguridad, autenticación, transformación de datos y control de tráfico |
| **Monitoreo y analítica** | Registra métricas de uso: número de llamadas, latencia, errores, patrones de consumo |
| **Gestión de versiones** | Permite mantener múltiples versiones de una misma API sin romper integraciones existentes |

##  Componentes principales para un ERP de ventas por WhatsApp

En el escenario grupal, el módulo de **Despachos y Reportes** no es consumido por un solo canal: puede recibir solicitudes desde el bot de WhatsApp, una aplicación móvil de repartidores, un dashboard web administrativo, o incluso sistemas de terceros (transportistas, facturación electrónica). Sin una capa de gestión de APIs, cada uno de estos consumidores tendría que conectarse directamente al ERP, generando:

- **Riesgo de seguridad**: el ERP quedaría expuesto directamente a internet.
- **Falta de control**: no habría forma centralizada de limitar cuántas peticiones puede hacer cada canal.
- **Ausencia de visibilidad**: sería difícil saber cuántas consultas de despacho se originan desde WhatsApp versus otros canales.

API Management resuelve esto introduciendo un **Gateway** como intermediario único, que autentica, controla y monitorea todo el tráfico antes de que llegue al ERP.

## Patrones conceptuales relacionados

- **Backend for Frontend (BFF):** el gateway puede adaptar la respuesta del ERP al formato que espera cada canal (ej. un mensaje de texto simple para WhatsApp vs. un JSON estructurado para el dashboard).
- **Rate Limiting / Throttling:** evita que un canal (o un uso malicioso del bot) sature el ERP con peticiones excesivas.
- **Circuit Breaker (complementario):** protege al ERP si empieza a fallar, evitando que seguir enviándole tráfico agrave el problema.

---

# Consideraciones técnicas

Para la demo se utilizó **Kong**, una de las herramientas de API Management más adoptadas en la industria, en su variante **Kong Konnect** (SaaS / servicio en la nube), dado que la virtualización local (Docker) no estaba disponible en el entorno de prueba.

## Kong Konnect

Kong Konnect es la plataforma en la nube de Kong que permite gestionar Gateways sin necesidad de infraestructura propia. Ofrece un plan gratuito (tier *Serverless*) suficiente para fines académicos y de demostración.

## Paso a paso: creación de cuenta y configuración base

**Paso 1 — Crear la cuenta en Kong Konnect**
1. Ingresar a `https://konghq.com/products/kong-konnect`
2. Seleccionar **"Start for free"**
3. Registrarse con correo electrónico o cuenta de Google/GitHub
4. Confirmar el correo electrónico y acceder al dashboard (no requiere tarjeta de crédito)

**Paso 2 — Crear el Gateway (Control Plane)**
1. En el dashboard, ir a **Gateway Manager**
2. Seleccionar **"New Gateway"** → tipo **Serverless** (plan gratuito)
3. Asignar un nombre descriptivo (ej. `erp-despachos-gw`)

Con esto queda disponible un Gateway en la nube, listo para configurarse con los servicios, rutas y políticas específicas de cualquier proyecto (la configuración particular del escenario de despachos se detalla en la sección de Demo).

## Postman

Para probar cualquier API gestionada por el Gateway es necesaria una herramienta de cliente HTTP. Se utilizó **Postman** por ser gratuita y ampliamente usada en la industria.

**Instalación:**
1. Descargar desde `https://www.postman.com/downloads/`
2. Instalar y abrir la aplicación (puede usarse sin crear cuenta, como "Lightweight API Client")

**Uso básico relevante para este proyecto:**
- Crear una petición `GET` indicando la URL del Gateway
- Añadir headers personalizados (ej. `apikey`) en la pestaña **Headers**
- Enviar la petición y revisar el código de respuesta HTTP (200, 401, 404, 429, etc.) en el panel inferior

Postman permite validar de forma controlada cada política aplicada en el Gateway (autenticación, límite de tráfico) antes de integrar el flujo con un canal real como WhatsApp.

## ngrok

Como Kong Konnect es un servicio en la nube, no puede acceder directamente a un backend corriendo en `localhost`. **ngrok** es una herramienta que crea un túnel público temporal hacia un puerto local, permitiendo que un servicio en la nube alcance una aplicación que corre en la propia máquina.

**Instalación y registro:**
1. Descargar desde [https://ngrok.com/download] (https://ngrok.com/download)
2. Crear una cuenta gratuita en `https://ngrok.com`
3. Copiar el **authtoken** personal desde `https://dashboard.ngrok.com/get-started/your-authtoken`
4. Configurar el token localmente:

   ```
   ngrok config add-authtoken TU_TOKEN
   ```

Con esto, ngrok queda listo para exponer cualquier puerto local a internet cuando se necesite .


---

# Demo 

## Escenario de aplicación

Se simula el flujo en el que un cliente escribe por WhatsApp "¿Cuál es el estado de mi pedido#123?", y esa solicitud, en lugar de llegar directo al ERP, pasa primero por el API Gateway (Kong Konnect), que valida la autenticación, aplica límite de tráfico y registra la petición antes de reenviarla al módulo de despachos del ERP.

```
[Cliente WhatsApp] → [Kong Konnect Gateway] → [ngrok] → [ERP Mock (Flask/Python)]
```

## Backend del ERP (módulo Despachos y Reportes)

Se implementó un servicio mínimo en **Python (Flask)** con dos endpoints relevantes al módulo:

```python
from flask import Flask, jsonify
from datetime import date

app = Flask(__name__)

despachos = {
    '123': {'id': '123', 'cliente': 'Juan Perez', 'estado': 'En camino', 'fechaEstim': '2026-09-29'},
    '124': {'id': '124', 'cliente': 'Maria Lopez', 'estado': 'Entregado', 'fechaEstim': '2026-09-27'},
    '125': {'id': '125', 'cliente': 'Carlos Ruiz', 'estado': 'Preparando pedido', 'fechaEstim': '2026-09-30'},
}

@app.route('/despachos/<id>', methods=['GET'])
def get_despacho(id):
    despacho = despachos.get(id)
    if not despacho:
        return jsonify({'error': 'Despacho no encontrado'}), 404
    return jsonify(despacho)

@app.route('/reportes/despachos-diarios', methods=['GET'])
def reporte_diario():
    lista = list(despachos.values())
    reporte = {
        'fecha': date.today().isoformat(),
        'totalDespachos': len(lista),
        'porEstado': {
            'enCamino': sum(1 for d in lista if d['estado'] == 'En camino'),
            'entregado': sum(1 for d in lista if d['estado'] == 'Entregado'),
            'preparando': sum(1 for d in lista if d['estado'] == 'Preparando pedido'),
        },
        'detalle': lista,
    }
    return jsonify(reporte)

if __name__ == '__main__':
    app.run(host='0.0.0.0', port=4000)
```

## Exponer el backend local a internet (uso de ngrok)

Con ngrok ya instalado y configurado (sección 2.4), se levantó el túnel apuntando al puerto donde corre el ERP mock:

```
ngrok http 4000
```

Esto generó una URL pública (ej. `https://abc123.ngrok-free.app`) que redirige el tráfico entrante hacia `localhost:4000`, donde corre el Flask del ERP.


## Configuración específica del Gateway para el escenario

Dentro del Control Plane creado en la sección 2.2, se configuró:

**Service**
- Nombre: `erp-despachos-service`
- URL: la URL pública generada por ngrok (apunta al Flask local)

**Routes**
- `ruta-despachos` → path `/despachos` (Strip Path: `false`)
- `ruta-reporte` → path `/reportes/despachos-diarios` (Strip Path: `false`)

**Plugins aplicados al Service**
- **Key Authentication**: header `apikey`, exige credencial válida en cada request
- **Rate Limiting**: `5` peticiones por `minute`, política `local`

**Consumer**
- Username: `whatsapp-bot` (representa al canal de WhatsApp como cliente autorizado)
- Credencial generada en **Credentials → Key Auth**, usada como valor del header `apikey`

## Pruebas realizadas (Postman)

| Prueba | Petición | Resultado esperado | Resultado obtenido |
|---|---|---|---|
| Sin API key | `GET /despachos/123` | `401 Unauthorized` | ✅ Correcto |
| Con API key válida | `GET /despachos/123` | `200 OK` + JSON del despacho | ✅ Correcto |
| Reporte diario | `GET /reportes/despachos-diarios` | `200 OK` + JSON del reporte | ✅ Correcto |
| Más de 5 peticiones/min | `GET /despachos/123` (repetido) | `429 Too Many Requests` | ✅ Correcto |

Estas pruebas confirman que el Gateway cumple las tres funciones clave de API Management evaluadas: **autenticación**, **control de tráfico** y **enrutamiento hacia el backend correcto**, sin que WhatsApp o cualquier otro consumidor necesite conocer la ubicación real del ERP.

---


