# Checklist de revisión

## 1. Alucionaciones de liberías

### Preguntas clave

* ¿Ese import existe realmente?
* ¿Cual es la tasa de bugs que tiene?
* ¿Quién lo usa actualmente?
* ¿Cual es el velocity con que cambia el major?
* ¿Necesitamos toda la libería o una parte de ella.?
* ¿Es realmente necesaria?

**Verificar:**
- Documentación oficial
- Existencia real del paquete
- Compatibilidad con la versión usada
- Revisión del API *si aplica*
- Tipo de Bugs que tiene o han solventado.

### Checklist
- [ ] Los import existen
- [ ] Las funciones usadas son reales
- [ ] La API corresponde a la versión actual
- [ ] Estamos usando mas del 50% de la librería o sólo 1 función.

## 2. Lógica de negocio sútil

### Preguntas clave

* ¿La lógica es realmente correcta?
* ¿Los cálculos cumplen la formulación?
* ¿El formato de número es el indicado?
* ¿Que redondeo está usando?

**Errores Comunes generados por la IA:**
- Cálculos Incorrectos
- Redondeos Incorrectos
- Manejo Incorrecto de Fechas
- Uso de float para dinero


### Checklist ###

- [ ] Los calculos son correctos
- [ ] El manejo de fechas es consistente
- [ ] No se usa float para dinero
- [ ] Edge cases están contemplados
- [ ] Las formulas están bien implementas


## 3. Seguridad

### Preguntas clave

* ¿El código introduce riesgos de seguridad?
* ¿El dueño de la solicitud es quien dice ser?
* ¿No exponemos información sensible a quien no es su dueño?
* ¿Tiene rate limit?
* ¿Puede haber inyección a la BD?

**Verificar:**
- Validación de inputs
- inyeción (SQL, Comandos)
- liabilidad de la información
- Soportamos un DoS
- Exposición de secretos
- Manejo de datos sensibles

### Checklist
- [ ] Inputs están validados
- [ ] No hay riesgo de inyección
- [ ] Se retorna información al dueño real
- [ ] No se hacen logs de data sensible
- [ ] Se verifica el rate limit

## 4. Context Window

### Preguntas Claves

- Tiene toda la información del brief

**Verificar:**
- Definition of done
- Desiciones arquitectónicas
- Requisitos técnicos
- Constaints

### Checklist

- [ ] Se omite alguna restricción
- [ ] Usa los patrones de arquitectura definida
- [ ] Los test validan bien los casos borde
- [ ] Los test validan bien la lógica de negocio
- [ ] Si se modifica código de forma maliciosa, los Test lo detectan
