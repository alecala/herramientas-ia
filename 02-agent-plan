
# Brief del Agente Didáctico

## 1. Título de la tarea

Extender el diseño actual para que se incluya una API para viajes.

---

## 2. Contexto

El proyecto ya tiene una base funcional y didactica en el construida en el @brief-agent.md.

Se requiere externder el modelo para que en esta versión pueda hacer uso de un api de agencia de viajes que pueda entregar tiquetes de avión más estadías por N noches. 

---

## 3. Requerimientos del proyecto

### Arquitectura y enfoque

La solución está dividida en capas claras para que cada parte tenga una responsabilidad concreta:

- Una capa de entrada recibe la pregunta del usuario.
- Una capa de ejecución coordina el proceso de respuesta.
- Una capa de composición arma el agente con su modelo, prompt y herramientas.
- Una capa de capacidades concentra las herramientas del dominio (cálculo, hora y plan de viaje).
- Una capa de request debe concentrar la conexión con el API
- Una capa de configuración centraliza y valida variables de entorno.

Este enfoque permite:

- Entender rápido qué hace cada módulo.
- Agregar nuevas herramientas sin reescribir todo.
- Probar piezas de forma aislada.
- Reducir riesgos al evolucionar el proyecto.

Política de resiliencia de conexión (obligatoria):
- timeout_ms: 1000 ms por request.
- retries_max: 2 reintentos.
- retryable_errors: timeout, HTTP 429, HTTP 5xx, errores de red.
- backoff: exponencial con jitter (ej. base 200 ms: 200 ms, 400 ms ± jitter).
- circuit_breaker_open_when: abrir circuito si en una ventana de 20 requests hay >= 5% de errores retryables.
- circuit_breaker_open_duration: 5 segundos (no se llama al proveedor durante ese tiempo).
- half_open_probe: al terminar los 5 s, permitir 1 request de prueba.
- close_condition: cerrar circuito si el request de prueba funciona.
- reopen_condition: reabrir circuito si el request de prueba falla.

#### Descripción del API a utilizar ####

**LIbrería:** serpapi ver 1.0.2

- Para el calculo de los tiquetes de avión

```
flight_search_params = z.object({
  engine: "google_flights",
  departure_id: z.string(),
  arrival_id: z.string(),
  currency: z.string(),
  type: z.enum(["1", "2"]), // 1: Round trip, 2: One way
  outbound_date: z.string(),
  return_date: z.string().optional(),
  adults: z.string().default("2"),
  max_flight_results: z.string().default("3"),
  api_key: z.string() // se obtiene de SERPAPI_API_KEY
})

getJson(flight_search_params, (json) => {
  console.log(json["best_flights"]);
});

Response (best_flights) = {
  "$schema": "https://json-schema.org/draft/2020-12/schema",
  "title": "FlightOffers",
  "type": "array",
  "items": {
    "type": "object",
    "additionalProperties": true,
    "properties": {
      "flights": {
        "type": "array",
        "minItems": 1,
        "items": {
          "type": "object",
          "properties": {
            "departure_airport": {
              "type": "object",
              "properties": {
                "name": {
                  "type": "string"
                },
                "id": {
                  "type": "string"
                },
                "time": {
                  "type": "string"
                }
              },
              "required": [
                "name",
                "id",
                "time"
              ]
            },
            "arrival_airport": {
              "type": "object",
              "properties": {
                "name": {
                  "type": "string"
                },
                "id": {
                  "type": "string"
                },
                "time": {
                  "type": "string"
                }
              },
              "required": [
                "name",
                "id",
                "time"
              ]
            },
            "duration": {
              "type": "integer"
            },
            "airplane": {
              "type": "string"
            },
            "airline": {
              "type": "string"
            },
            "airline_logo": {
              "type": "string"
            },
            "travel_class": {
              "type": "string"
            },
            "flight_number": {
              "type": "string"
            },
            "overnight": { "type": "boolean" }
          },
          "required": [
            "departure_airport",
            "arrival_airport",
            "duration",
            "airplane",
            "airline",
            "airline_logo",
            "travel_class",
            "flight_number"
          ]
        }
      },
      "layovers": {
        "type": "array",
        "items": {
          "type": "object",
          "properties": {
            "duration": {
              "type": "integer"
            },
            "name": {
              "type": "string"
            },
            "id": {
              "type": "string"
            },
            "overnight": { "type": "boolean" }
          },
          "required": [
            "duration",
            "name",
            "id"
          ]
        }
      },
      "total_duration": {
        "type": "integer"
      },
      "price": {
        "type": "number"
      },
      "type": {
        "type": "string"
      },
      "airline_logo": {
        "type": "string"
      },
      "extensions": {
        "type": "array",
        "items": {
          "type": "string"
        }
      },
      "carbon_emissions": {
        "type": "object",
        "properties": {
          "this_flight": { "type": "integer" },
          "typical_for_this_route": { "type": "integer" },
          "difference_percent": { "type": "integer" }
        }
      },
      "departure_token": {
        "type": "string"
      }
    },
    "required": [
      "flights",
      "layovers",
      "total_duration",
      "price",
      "type",
      "airline_logo",
      "extensions"
    ]
  }
}
```

Explicación esperada de esta sección:
- El agente consulta `google_flights` y trabaja sobre `best_flights`.
- Para cada itinerario usa `price`, `total_duration`, `flights`, `layovers` y `carbon_emissions`.
- En viaje ida y vuelta (`type=1`), `return_date` es requerido a nivel de flujo de negocio.
- El resultado final al usuario debe resumir: precio total, duración, escalas y aerolínea principal.

- Para el calculo del hospedaje

```
hotel_search_params = z.object({
  engine: "google_hotels",
  q: z.string(), // destino o consulta, ej: "Bali Resorts"
  check_in_date: z.string(),
  check_out_date: z.string(),
  adults: z.string().default("2"),
  currency: z.string(),
  max_hotel_results: z.string().default("3"),
  api_key: z.string() // se obtiene de SERPAPI_API_KEY
})

getJson(hotel_search_params, (json) => {
  console.log(json["properties"]);
});

Response (properties) = {
  "$schema": "https://json-schema.org/draft/2020-12/schema",
  "title": "HotelProperties",
  "type": "array",
  "items": {
    "type": "object",
    "additionalProperties": true,
    "properties": {
      "name": { "type": "string" },
      "type": { "type": "string" },
      "rate_per_night": {
        "type": "object",
        "properties": {
          "lowest": { "type": "string" }
        }
      },
      "total_rate": {
        "type": "object",
        "properties": {
          "lowest": { "type": "string" }
        }
      },
      "overall_rating": { "type": "number" },
      "reviews": { "type": "integer" },
      "check_in_time": { "type": "string" },
      "check_out_time": { "type": "string" },
      "amenities": {
        "type": "array",
        "items": { "type": "string" }
      },
      "property_token": { "type": "string" },
      "serpapi_property_details_link": { "type": "string" }
    },
    "required": ["name", "type", "rate_per_night", "total_rate"]
  }
}

```

Explicación esperada de esta sección:
- El agente consulta `google_hotels` y trabaja sobre `properties`.
- Para selección base usa `total_rate.lowest`, `rate_per_night.lowest`, `overall_rating` y `reviews`.
- El filtro de fechas (`check_in_date` y `check_out_date`) debe alinearse con las noches solicitadas por el usuario.
- El resultado final al usuario debe resumir: costo total de hospedaje, costo por noche, calificación y tipo de propiedad.

### Input esperado

El sistema espera preguntas escritas en lenguaje natural por parte del usuario.

Tipos de solicitudes contempladas hoy:

- Preguntas que requieren planificación de un viaje con un budget limitado.

Datos necesarios en cada consulta
- Tipo de ruta: 1 Round trip (default) o 2 One way
- Budget 
- Ciudad de origen 
- Ciudad de destino
- Noches
- Adultos
- Moneda
- Fecha inicio
- Fecha fin

Resultado esperado para el usuario:

- Respuesta en español.
- Explicación breve de lo que hizo el agente.
- Uso de herramientas solo cuando realmente aporta valor.

---

## 4. Restricciones

- Mantener el enfoque pedagógico: primero claridad, luego complejidad.
- Evitar agregar componentes que no aporten al objetivo principal del aprendizaje.
- No introducir dependencias innecesarias sin justificación funcional.
- Cuidar que cualquier mejora preserve compatibilidad con la ejecución por consola y pruebas existentes.
- Documentar claramente cualquier cambio en configuración, comportamiento o límites del agente.
- Se conserva el Lenguaje y el stack definido en la versión previa.
- Se debe conservar la Arquitectura y enfoque
- Si no se obtienen vuelos no se deben realizar calculo de valores de hospedaje
- Si no logra obtener al menos una noche de hospedaje no debe retornar opciones
- Nunca recibir ni loggear api_key desde input de usuario
- Sólo se permiten viajes Round trip

---

## 5. Definition of Done (DoD)

El trabajo se considera terminado cuando:

- El brief refleja con precisión el estado actual del proyecto y su dirección inmediata.
- El lenguaje del documento es claro, humano y comprensible para perfiles no profundamente técnicos.
- Se explican propósito, funcionamiento general, límites y próximos pasos sin usar código.
- La información está alineada con la estructura real del repositorio, documentación y pruebas actuales.
- Queda claro cómo evaluar calidad mínima en cada cambio futuro (ejecución correcta, pruebas, consistencia y documentación).
- El calculo del viaje puede reducir noches hasta 1 noche para analizar un resultado válido.
- El calculo de la fecha de inicio de hospedaje debe estar asociado con la fecha de llegada a la ciudad destino
- El calculo de la fecha de finalización del hospedaje debe estar asociado con la fecha de retorno desde la ciudad destino
- El resultado se considera válido cuando costo_total_viaje (vuelos + hospedaje) <= budget_usuario.
- Se valida coherencia de fechas entre vuelos y hospedaje.
- Se retorna al menos una opción completa válida; no se exige ranking entre opciones válidas.
