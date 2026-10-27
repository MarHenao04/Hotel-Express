# Pruebas del subworkflow «Hotel - Sub - Consultar disponibilidad»

## Qué se probó

La decisión de disponibilidad: dado un tipo de habitación y unas fechas, el subworkflow devuelve si hay una habitación libre, si la solicitud ya existe como reserva, o si no hay disponibilidad.

**Contrato de entrada:** `Tipo`, `Entrada`, `Salida`, `Huespedes`, `Email`

**Contrato de salida:** `disponible`, `duplicada`, `motivo`, `habitacion`, `reserva_existente_id`, `habitaciones_revisadas`, `reservas_revisadas`

**Contexto de los datos:** en todas las pruebas la consulta revisó 2 habitaciones tipo `Doble` (102 y 201) y 8 reservas de la tabla de reservas, salvo D-04, donde no hay habitaciones del tipo pedido.

**Tipo de ejecución:** manual, con datos fijos cargados en `Recibir del principal`. Son pruebas de solo lectura: las consultas a Baserow son GET, así que ninguna prueba creó ni modificó registros.

## Tabla de pruebas

| ID | Caso | Datos de entrada (ficticios) | Resultado esperado | Resultado observado | Evidencia | Conclusión |
|---|---|---|---|---|---|---|
| D-01 | Fechas libres | Tipo `Doble`, 2026-12-10 → 2026-12-12, 2 huéspedes, `prueba.nueva@example.com` | `disponible: true`, habitación `102` | `disponible: true`, `motivo: DISPONIBLE`, `habitacion: "102"` | `evidencias/D01_Fechas_Libres.png` | Correcto. Asigna la primera habitación libre por número. |
| D-02 | Cruce con reservas existentes | Tipo `Doble`, 2026-10-20 → 2026-10-23, 2 huéspedes, `prueba.nueva@example.com` | `habitacion: "201"`. La 102 está ocupada por las reservas #6 y #39 en esas fechas. | `disponible: true`, `motivo: DISPONIBLE`, `habitacion: "201"` | `evidencias/D02_Cruce_Fechas.png` | Correcto. Salta la habitación ocupada. El flujo original habría ofrecido la 102. |
| D-03 | Reserva repetida | Tipo `Doble`, 2026-10-20 → 2026-10-23, 2 huéspedes, `juan@correo.com` (igual a la reserva #39) | `duplicada: true`, `reserva_existente_id: 39` | `disponible: false`, `duplicada: true`, `motivo: RESERVA_DUPLICADA`, `habitacion: "102"`, `reserva_existente_id: 39` | `evidencias/D03_Reserva_Duplicada.png` | Correcto. Detecta la solicitud repetida y no propone crear otra reserva. |
| D-04 | Sin disponibilidad (tipo inexistente) | Tipo `Suite Imperial`, 2026-12-10 → 2026-12-12, 2 huéspedes, `prueba.nueva@example.com` | `disponible: false`, `motivo: SIN_DISPONIBILIDAD` | `disponible: false`, `motivo: SIN_DISPONIBILIDAD`, `habitacion: null`, `habitaciones_revisadas: 0` | `evidencias/D04_Sin_Disponibilidad.png` | Correcto. Es una respuesta del negocio: el nodo termina sin error. |
| D-05 | El día de salida no choca | Tipo `Doble`, 2026-10-23 → 2026-10-25, 2 huéspedes, `prueba.nueva@example.com` | `disponible: true`, habitación `102`. La reserva #6 sale el 23. | `disponible: true`, `motivo: DISPONIBLE`, `habitacion: "102"` | `evidencias/D05_Salida_Mismo_Dia.png` | Correcto. La regla de cruce permite entrar el mismo día en que otro huésped sale. |

## Pasos para repetir las pruebas

1. Abrir el workflow `Hotel - Sub - Consultar disponibilidad`.
2. Abrir el nodo `Recibir del principal`, pulsar **Edit output** y pegar los datos de entrada de la fila que se quiere repetir.
3. Guardar y pulsar **Execute workflow**.
4. Abrir `Determinar disponibilidad` y comparar su Output con el resultado esperado.

### Datos de entrada en JSON

**D-01**
```json
[{ "Tipo": "Doble", "Entrada": "2026-12-10", "Salida": "2026-12-12", "Huespedes": 2, "Email": "prueba.nueva@example.com" }]
```

**D-02**
```json
[{ "Tipo": "Doble", "Entrada": "2026-10-20", "Salida": "2026-10-23", "Huespedes": 2, "Email": "prueba.nueva@example.com" }]
```

**D-03**
```json
[{ "Tipo": "Doble", "Entrada": "2026-10-20", "Salida": "2026-10-23", "Huespedes": 2, "Email": "juan@correo.com" }]
```

**D-04**
```json
[{ "Tipo": "Suite Imperial", "Entrada": "2026-12-10", "Salida": "2026-12-12", "Huespedes": 2, "Email": "prueba.nueva@example.com" }]
```

**D-05**
```json
[{ "Tipo": "Doble", "Entrada": "2026-10-23", "Salida": "2026-10-25", "Huespedes": 2, "Email": "prueba.nueva@example.com" }]
```

## Regla de disponibilidad aplicada

Una habitación está ocupada si existe una reserva con `Estado = Confirmada` en esa habitación que cumpla las dos condiciones a la vez:

```
Entrada de la reserva < Salida solicitada   Y   Salida de la reserva > Entrada solicitada
```

Una solicitud es duplicada si ya existe una reserva confirmada con el mismo email, las mismas fechas y una habitación del tipo pedido.

## Qué demuestran estas pruebas y qué no

- **Demuestran:** la lógica de decisión del subworkflow frente a los datos reales de las tablas de habitaciones y reservas.
- **No demuestran:** una petición HTTP real al Webhook, ni la creación de una reserva en Baserow, ni el comportamiento ante fallos técnicos. Esas pruebas se documentan por separado.
- **Nota sobre D-02:** el cruce se comprobó con reservas que ya existían en la tabla (#6 y #39, ambas en la habitación 102 para las mismas fechas). Esas dos reservas son también evidencia del problema del flujo original, que no revisaba las reservas antes de asignar habitación.

## Nota para las capturas

Cada captura debe mostrar el Output de `Determinar disponibilidad`, recortado. Si aparece el Output de `Consultar reservas`, cubrir los correos que parezcan reales: la guía de entrega exige usar solo datos ficticios.
