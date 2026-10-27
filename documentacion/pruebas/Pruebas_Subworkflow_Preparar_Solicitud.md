# Pruebas del subworkflow «Hotel - Sub - Preparar solicitud»

## Qué se probó

La limpieza y validación de una solicitud de reserva antes de consultar cualquier servicio externo. El subworkflow no lee ni escribe en Baserow.

**Contrato de entrada** (campos en la raíz del item): `nombre`, `email`, `fecha_entrada` (AAAA-MM-DD), `fecha_salida` (AAAA-MM-DD), `huespedes`, `tipo_habitacion`

**Contrato de salida:** `valido`, `errores`, `Cliente`, `Email`, `Entrada`, `Salida`, `Huespedes`, `Tipo`, `Noches`

**Cómo se llama:** el workflow `Hotel - Principal reservas` envía el cuerpo de la petición mediante el nodo `Extraer solicitud` y llama al subworkflow con `Llamar a Preparar solicitud`, esperando su finalización. El IF `¿Datos completos?` lee el campo `valido` y decide entre responder 400 o continuar.

**Tipo de ejecución:** manual, con datos fijos cargados en el nodo `Webhook - Reserva` (Edit output). No hubo petición HTTP real ni escrituras en Baserow: la prueba se ejecutó solo hasta el IF.

## Tabla de pruebas

| ID | Caso | Datos de entrada (ficticios) | Resultado esperado | Resultado observado | Evidencia | Conclusión |
|---|---|---|---|---|---|---|
| P-01 | Solicitud válida con datos sucios | `nombre`: `" Ana Prueba "`, `email`: `"ANA.PRUEBA@Example.com"`, entrada 2026-12-10, salida 2026-12-12, `huespedes`: `"2"` (texto), tipo `Doble` | `valido: true`, `errores: []`, nombre sin espacios, email en minúsculas, `Huespedes: 2` (número), `Noches: 2`. IF por la rama **true**. | `valido: true`, `errores: []`, `Cliente: "Ana Prueba"`, `Email: "ana.prueba@example.com"`, `Huespedes: 2`, `Noches: 2`. IF por la rama **true**. | `evidencias/P01_Caso_Normal.png` | Correcto. Limpia espacios, pasa el email a minúsculas y convierte los huéspedes a número. |
| P-02 | Entrada no válida (cuatro errores a la vez) | `nombre`: `"   "`, `email`: `"no-es-email"`, entrada 2026-12-12, salida 2026-12-10, `huespedes`: `0`, tipo `Doble` | `valido: false` y cuatro mensajes en `errores`. IF por la rama **false**. | `valido: false`. `errores`: «nombre es obligatorio», «email no es valido», «fecha_salida debe ser posterior a fecha_entrada», «huespedes debe ser un entero mayor que 0». IF por la rama **false**. | `evidencias/P02_Entrada_No_Valida.png` | Correcto. Informa todos los problemas juntos. No es un error técnico: el subworkflow termina bien y devuelve `valido: false`. |

## Pasos para repetir las pruebas

1. Abrir el workflow `Hotel - Principal reservas`.
2. Abrir el nodo `Webhook - Reserva`, pulsar **Edit output** y pegar el JSON de la prueba.
3. Guardar y cerrar el nodo.
4. Abrir `¿Datos completos?` y pulsar **Execute step**. No usar *Execute workflow*, porque la rama true continúa hacia nodos que escriben en Baserow.
5. Abrir `Llamar a Preparar solicitud` y comparar su Output con el resultado esperado. Revisar en qué pestaña (true o false) quedó el item del IF.

### Datos de entrada en JSON

**P-01**
```json
[
  {
    "headers": {},
    "params": {},
    "query": {},
    "body": {
      "nombre": " Ana Prueba ",
      "email": "ANA.PRUEBA@Example.com",
      "fecha_entrada": "2026-12-10",
      "fecha_salida": "2026-12-12",
      "huespedes": "2",
      "tipo_habitacion": "Doble"
    }
  }
]
```

**P-02**
```json
[
  {
    "headers": {},
    "params": {},
    "query": {},
    "body": {
      "nombre": "   ",
      "email": "no-es-email",
      "fecha_entrada": "2026-12-12",
      "fecha_salida": "2026-12-10",
      "huespedes": 0,
      "tipo_habitacion": "Doble"
    }
  }
]
```

## Salidas completas

**P-01**
```json
{
  "valido": true,
  "errores": [],
  "Cliente": "Ana Prueba",
  "Email": "ana.prueba@example.com",
  "Entrada": "2026-12-10",
  "Salida": "2026-12-12",
  "Huespedes": 2,
  "Tipo": "Doble",
  "Noches": 2
}
```

**P-02**
```json
{
  "valido": false,
  "errores": [
    "nombre es obligatorio",
    "email no es valido",
    "fecha_salida debe ser posterior a fecha_entrada",
    "huespedes debe ser un entero mayor que 0"
  ],
  "Cliente": "",
  "Email": "no-es-email",
  "Entrada": "2026-12-12",
  "Salida": "2026-12-10",
  "Huespedes": 0,
  "Tipo": "Doble",
  "Noches": 0
}
```

## Reglas de validación aplicadas

| Campo | Regla | Mensaje cuando falla |
|---|---|---|
| `nombre` | Texto no vacío tras quitar espacios | nombre es obligatorio |
| `email` | Formato `algo@dominio.ext`; se guarda en minúsculas | email no es valido |
| `fecha_entrada`, `fecha_salida` | Formato `AAAA-MM-DD` y fecha existente (se rechaza 2026-02-31) | fecha_entrada / fecha_salida debe tener formato AAAA-MM-DD |
| Fechas entre sí | La salida debe ser posterior a la entrada | fecha_salida debe ser posterior a fecha_entrada |
| `huespedes` | Entero mayor que 0; acepta número o texto numérico | huespedes debe ser un entero mayor que 0 |
| `tipo_habitacion` | Texto no vacío | tipo_habitacion es obligatorio |

`Noches` se calcula como la diferencia en días entre salida y entrada, y vale 0 cuando las fechas no son válidas.

## Qué demuestran estas pruebas y qué no

- **Demuestran:** la limpieza de datos, las reglas de validación y que una solicitud inválida se distingue de un error técnico, porque el subworkflow termina correctamente en ambos casos.
- **No demuestran:** una respuesta HTTP 400 recibida por un cliente real, ni que se haya creado o evitado crear una reserva en Baserow. La respuesta 400 se verificó solo como salida del nodo `Responder 400`. La prueba con petición real se documenta aparte, con el flujo publicado.
- **Alcance de la validación:** la entrada del segundo caso tiene cuatro errores simultáneos. Cada regla individual no se probó por separado; si se quiere evidencia por regla, se pueden agregar casos con un solo error.

## Nota para las capturas

Cada captura debe mostrar el Output de `Llamar a Preparar solicitud`, recortado, y la rama que tomó el IF. No usar capturas del lienzo completo.
