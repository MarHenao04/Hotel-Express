# Pruebas del workflow «Hotel - Manejador de errores»

## Qué es y qué hace

Es un **workflow de errores**: n8n lo ejecuta automáticamente cuando falla un workflow al que está asociado (en este proyecto, `Hotel - Principal reservas` y el workflow de laboratorio). No lo llama otro workflow ni necesita publicarse. Se asocia en *Settings → Error workflow* del workflow que se quiere vigilar.

Su responsabilidad es **registrar el incidente**, no repararlo. No reanuda la ejecución que falló y no responde al cliente del Webhook: esa respuesta (503) la da el workflow principal.

```text
Error Trigger → Preparar registro del incidente → Guardar incidente en Baserow
                                                      ├─ éxito: Incidente almacenado
                                                      └─ error: Registro no almacenado (Stop And Error)
```

Estado de construcción de este documento: están construidos los dos primeros nodos. Los demás se marcan como **pendiente** en las tablas hasta ejecutarlos.

## Nodos

| Nodo | Tipo | Responsabilidad | Estado |
|---|---|---|---|
| `Recibir fallo automatico` | Error Trigger | Recibe los metadatos del fallo: workflow, ejecución, nodo y mensaje. | Construido |
| `Preparar registro del incidente` | Edit Fields (Set) | Arma los campos del incidente. **Preparar un objeto no equivale a guardarlo.** | Construido |
| `Guardar incidente en Baserow` | HTTP Request (POST) | Escribe una fila en la tabla `Incidencias` (ID 1284431). | Pendiente |
| `Incidente almacenado` | Edit Fields (Set) | Marca que la escritura terminó bien. | Pendiente |
| `Registro no almacenado` | Stop And Error | Deja esta ejecución en rojo si no se pudo guardar el incidente. | Pendiente |

## Contrato

**Entrada (la entrega n8n al Error Trigger):** `workflow.name`, `execution.id`, `execution.url`, `execution.lastNodeExecuted`, `execution.error.message`. Los campos disponibles cambian según dónde ocurrió el fallo. **No** llegan los datos del negocio (nombre, email, fechas).

**Salida de `Preparar registro del incidente`:**

| Campo | Origen | Para qué sirve |
|---|---|---|
| `fecha` | Reloj de n8n (`$now.toISO()`) | Ubicar cuándo ocurrió |
| `proyecto` | Texto fijo `Reservas Hotel` | Identificar el proceso |
| `workflow` | `workflow.name` | Saber qué workflow falló |
| `ejecucion_id` | `execution.id` | Abrir la ejecución con el detalle técnico |
| `ejecucion_url` | `execution.url` | Enlace directo a esa ejecución |
| `nodo` | `execution.lastNodeExecuted` | Localizar la operación que falló |
| `solicitud_ref` | Texto `[ref=...]` dentro del mensaje | Relacionar el incidente con una solicitud |
| `mensaje` | `execution.error.message`, recortado a 500 caracteres | Describir el problema |
| `estado` | Texto fijo `pendiente_revision` | Marcar el incidente como pendiente de atención |

**Por qué la referencia viaja dentro del mensaje:** el Error Trigger no recibe los campos del negocio. El workflow principal escribe `[ref=Tipo Entrada..Salida]` dentro del texto del error, y el manejador lo extrae con una expresión regular. La referencia contiene tipo de habitación y fechas, **no** datos personales.

## Expresión importante: extraer la referencia

```
{{ (String($json.execution?.error?.message || '').match(/\[ref=([^\]]*)\]/) || [])[1] || 'no_disponible' }}
```

| Parte | Qué hace |
|---|---|
| `String(... \|\| '')` | Convierte el mensaje a texto; si no existe, usa texto vacío. |
| `.match(/\[ref=([^\]]*)\]/)` | Busca `[ref=` seguido de cualquier texto hasta el primer `]` y captura ese texto. |
| `\|\| []` y `[1]` | Si no hubo coincidencia usa una lista vacía; si la hubo, toma lo capturado. |
| `\|\| 'no_disponible'` | Si no hay referencia, escribe `no_disponible`. |

| Entrada (mensaje de error) | Salida (`solicitud_ref`) |
|---|---|
| `ERROR_TECNICO en "Llamar a Consultar disponibilidad": timeout [ref=Doble 2026-12-10..2026-12-12]` | `Doble 2026-12-10..2026-12-12` |
| `Error cualquiera` | `no_disponible` |

## Pruebas

| ID   | Caso                                    | Datos de entrada                                                            | Resultado esperado                                                                                       | Resultado observado                                                                         | Evidencia                                   | Estado        |
| ---- | --------------------------------------- | --------------------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------- | ------------------------------------------- | ------------- |
| M-01 | Mapeo del incidente con datos simulados | Fallo ficticio cargado con *Set mock data* en el Error Trigger (JSON abajo) | `Preparar registro del incidente` entrega los 9 campos; `solicitud_ref` = `Doble 2026-12-10..2026-12-12` | Los campos coinciden con lo esperado; `solicitud_ref` salió limpio, sin corchetes ni `ref=` | `evidencias/M01_Mapeo_Incidente.png`        | Realizada     |
| M-02 | Escritura persistente en `Incidencias`  | Salida de M-01                                                              | Aparece una fila nueva en la tabla `Incidencias` de Baserow con los mismos valores                       | —                                                                                           | `evidencias/M02_Fila_Incidencias.png`       | **Pendiente** |
| M-03 | Falla el propio registro                | Tabla o credencial inválida a propósito, solo en una copia                  | La ejecución del manejador queda en rojo en *Executions*                                                 | —                                                                                           | `evidencias/M04_Registro_No_Almacenado.png` | **Pendiente** |

**Tipo de ejecución de M-01:** manual, con datos simulados. **No demuestra** que el Error Trigger se active ante un fallo real: eso lo prueba M-03. Tampoco demuestra almacenamiento: eso lo prueba M-02, con la evidencia en Baserow y no solo en el OUTPUT de n8n.

### Datos de entrada de M-01 (ficticios)

```json
[
  {
    "execution": {
      "id": "231",
      "url": "https://n8n.example.com/execution/231",
      "error": {
        "message": "ERROR_TECNICO en \"Llamar a Consultar disponibilidad\": timeout [ref=Doble 2026-12-10..2026-12-12]"
      },
      "lastNodeExecuted": "Llamar a Consultar disponibilidad",
      "mode": "webhook"
    },
    "workflow": {
      "id": "1",
      "name": "Hotel - Principal reservas"
    }
  }
]
```

### Salida observada de M-01

```json
{
  "fecha": "(fecha y hora de la ejecución)",
  "proyecto": "Reservas Hotel",
  "workflow": "Hotel - Principal reservas",
  "ejecucion_id": "231",
  "ejecucion_url": "https://n8n.example.com/execution/231",
  "nodo": "Llamar a Consultar disponibilidad",
  "solicitud_ref": "Doble 2026-12-10..2026-12-12",
  "mensaje": "ERROR_TECNICO en \"Llamar a Consultar disponibilidad\": timeout [ref=Doble 2026-12-10..2026-12-12]",
  "estado": "pendiente_revision"
}
```

### Pasos para repetir M-01

1. Abrir `Hotel - Manejador de errores`.
2. En `Recibir fallo automatico`, usar **Edit output** y pegar el JSON de entrada.
3. Abrir `Preparar registro del incidente` y pulsar **Execute step**.
4. Comparar el Output con la salida esperada.

## Tabla de destino en Baserow

Tabla `Incidencias` (ID 1284431). Todas las columnas son de texto.

| Columna | Tipo | Campo del registro que recibe |
|---|---|---|
| `Fecha` (primaria) | Texto de una línea | `fecha` |
| `Proyecto` | Texto de una línea | `proyecto` |
| `Workflow` | Texto de una línea | `workflow` |
| `Ejecucion_ID` | Texto de una línea | `ejecucion_id` |
| `Ejecucion_URL` | Texto de una línea | `ejecucion_url` |
| `Nodo` | Texto de una línea | `nodo` |
| `Solicitud_Ref` | Texto de una línea | `solicitud_ref` |
| `Mensaje` | Texto largo | `mensaje` |
| `Estado` | Texto de una línea | `estado` |

## Limitaciones conocidas

- El manejador **registra**, no repara ni reanuda la ejecución interrumpida.
- Depende de Baserow para guardar. Si Baserow es la causa del fallo original, el registro puede fallar también; por eso `Registro no almacenado` deja la ejecución del manejador en rojo, localizable en *Executions*.
- Una fila de incidente puede repetirse si se reintenta la escritura. Se acepta: es preferible una fila repetida a un incidente perdido. Se distinguen por `Ejecucion_ID`.
- No envía avisos por correo ni mensajería. **Propuesto**, no implementado.
- El historial de *Executions* depende de la retención configurada en n8n; no es un archivo permanente.
