# Technical Brief: PDF Payment Plan Table Extractor

## Título
Extractor de tablas de planes de pago desde archivos PDF a formato CSV.

---

## Contexto

Los archivos `PlanDePag*.pdf` representan planes de pago de un crédito personal a **hasta 72 cuotas**, generados en distintas fechas de corte. Cada archivo contiene una tabla estructurada donde cada fila representa una cuota con su estado de pago.

El objetivo es consolidar la información de múltiples archivos PDF en un único CSV estructurado, con trazabilidad de origen y registro de fallos por fila.

---

## Alcance

- Se procesarán **todos los archivos** que coincidan con el patrón `PlanDePag*.pdf` dentro de un **directorio configurable** (por defecto: directorio actual).
- La búsqueda de archivos **no es recursiva** salvo configuración explícita.
- La salida se consolida en **un único CSV** para todos los archivos procesados.
- El orden de filas en el CSV respeta: primero orden de archivos (alfabético por filename), luego orden de filas dentro de cada archivo.

---

## Requerimiento Técnico

### Entrada

| Parámetro | Flag CLI | Valor por defecto | Descripción |
|-----------|----------|-------------------|-------------|
| Directorio de PDFs | `--input-dir` | `.` | Directorio donde se buscan los archivos `PlanDePag*.pdf` |
| Archivo de salida CSV | `--output` | `output.csv` | Ruta del archivo CSV generado |
| Archivo de fallos | `--failures` | `failures.log` | Ruta del archivo de registro de fallos |
| Comportamiento si output existe | `--overwrite` | `false` | Si `true`, sobreescribe; si `false`, aborta con error |

### Columnas del CSV de salida

| Columna | Tipo | Restricciones |
|---------|------|---------------|
| `Filename` | string | Nombre del archivo PDF de origen (solo nombre, sin ruta) |
| `Row Number` | integer | Número de fila **dentro de la tabla del PDF** (1-indexed, sin contar header) |
| `Fecha Fin` | string | Formato `YYYY-MM-DD` normalizado desde el PDF |
| `Capital` | decimal | Sin símbolo monetario, sin separador de miles, decimal con `.` |
| `Interes Corriente` | decimal | Ídem |
| `Interes Mora` | decimal | Ídem. Si está vacío o ausente en el PDF: `0.00` |
| `Seguros/FNG` | decimal | Ídem |
| `Otros Conceptos` | decimal | Ídem. Si está vacío o ausente en el PDF: `0.00` |
| `Total Exigible` | decimal | Ídem |
| `Total Pagado` | decimal | Ídem |
| `Estado` | string | Valor tal como aparece en el PDF. Valores esperados: `Pagada`, `Pendiente`, `Vencida`, `Condonada` (no excluyente; documentar si se encuentran otros) |
| `Pendiente por Pagar` | decimal | Ídem formato numérico |
| `Condonado` | decimal | Ídem |
| `Saldo de Capital` | decimal | Ídem |

### Formato numérico
- Separador de miles: **omitir** (ej: `1.234.567` → `1234567`)
- Separador decimal: **punto** `.` (ej: `1.234,56` → `1234.56`)
- Símbolo monetario: **omitir** (ej: `$ 1.234,56` → `1234.56`)
- Precisión decimal: **2 decimales** en el CSV (usar `%.2f`)
- Valores negativos: representar con signo `-` (ej: `-150.00`)

### Formato de fecha
- El PDF puede contener fechas en formato `DD/MM/YYYY` o `DD-MM-YYYY`.
- El CSV debe normalizarlas a `YYYY-MM-DD`.
- Si el formato encontrado no coincide con ningún patrón conocido, la fila se considera fallo.

### Encoding
- El CSV de salida debe estar codificado en **UTF-8 sin BOM**.
- El delimitador del CSV es la **coma** `,`.
- Los campos string que contengan comas o comillas deben estar **entre comillas dobles** (`"`) siguiendo RFC 4180.

---

## Manejo de Errores y Archivo de Fallos

### Definición de "fila fallida"
Una fila se considera inválida y debe omitirse del CSV si:
1. Algún campo numérico obligatorio (`Capital`, `Interes Corriente`, `Seguros/FNG`, `Total Exigible`, `Total Pagado`, `Pendiente por Pagar`, `Saldo de Capital`) **no es parseable** como número.
2. `Fecha Fin` **no coincide** con ningún patrón de fecha configurado.
3. `Estado` está **vacío**.
4. El número de columnas extraídas de la fila **no coincide** con el esperado.

### Definición de "archivo fallido" (nivel PDF)
Un archivo se considera fallido completamente si:
- No se puede abrir o leer.
- No se encuentra ninguna tabla con el patrón esperado.
- Todas las filas del archivo fallan.

### Formato del archivo de fallos (`failures.log`)
Cada entrada en el log de fallos debe contener:
```
[TIMESTAMP] [LEVEL] Filename=<nombre> Row=<número|"N/A"> Reason=<descripción> RawContent=<contenido original de la fila>
```
- `TIMESTAMP`: formato RFC3339 (`2006-01-02T15:04:05Z07:00`)
- `LEVEL`: `ROW_ERROR` para filas individuales, `FILE_ERROR` para archivos completos.
- `RawContent`: el texto crudo extraído de la fila antes de parsear, truncado a 200 caracteres.

---

## Expresiones Regulares

Todas las expresiones regulares **deben estar agrupadas en un único archivo de configuración** (`config.go` o `regex.go`) como constantes nombradas con comentario explicativo. No deben estar dispersas en la lógica de negocio. Ejemplo:

```go
// ReNumeric matches currency values with optional symbol, thousands sep and comma decimal
const ReNumeric = `\$?\s*([\d\.]+),([\d]{2})`

// ReDate matches dates in DD/MM/YYYY or DD-MM-YYYY format
const ReDate = `(\d{2})[/\-](\d{2})[/\-](\d{4})`
```

---

## Requerimientos Técnicos

| Aspecto | Especificación |
|---------|----------------|
| Lenguaje | Go 1.24 |
| Target | `darwin/arm64` (Apple Silicon). El código debe compilar también en `linux/amd64` para CI. |
| Librería PDF | `pdfcpu` o `ledongthuc/pdfreader`. Justificar elección en README. |
| Concurrencia | Procesamiento de múltiples PDFs en paralelo con un pool de **N goroutines** (configurable via `--workers`, default: número de CPUs). |
| Logging | Usar `log/slog` (stdlib). Nivel configurable via `--log-level` (debug/info/warn/error). |
| Configuración | Flags CLI usando `flag` stdlib o `cobra`. Sin archivos de configuración externos. |
| Exit codes | `0` éxito, `1` error de argumentos, `2` ningún PDF encontrado, `3` todos los archivos fallaron |

---

## Definition of Done

1. **CSV mínimo**: El archivo CSV generado contiene **al menos 72 filas de datos** (más el header) al procesar el conjunto de PDFs de referencia provistos.
2. **Trazabilidad de fallos**: Cada fila omitida queda registrada en el archivo de fallos con su causa específica (no genérica).
3. **Cobertura de tests**: La cobertura de código (medida con `go test -coverprofile`) es **≥ 95%** en los paquetes de parsing y transformación. Los paquetes de I/O de archivos y CLI están excluidos del cálculo de cobertura mínima.
4. **Tests con fixtures**: Los tests de parsing usan **archivos PDF de fixture reales** (anonimizados si aplica) o mocks de la interfaz de extracción de texto, **no strings hardcodeados del PDF de producción**.
5. **Sin datos de producción en el repo**: Los PDFs de ejemplo en el repositorio deben ser sintéticos o anonimizados.
6. **Compilación limpia**: `go build ./...` y `go vet ./...` sin errores ni warnings.
7. **Race condition free**: `go test -race ./...` pasa sin errores.
8. **Documentación**: README con instrucciones de compilación, uso con ejemplos de CLI y descripción de cómo modificar las expresiones regulares.

---

## Fuera de Alcance

- Soporte para PDFs protegidos con contraseña.
- Extracción de datos fuera de la tabla de cuotas (ej: encabezado del contrato, datos del deudor).
- Interfaz gráfica o API REST.
- Exportación a formatos distintos de CSV.
- Procesamiento de archivos que no sigan el patrón `PlanDePag*.pdf`.