# Automatizacion-N8N
# Procesador de facturas y nóminas con n8n + Gemini

Esta guía explica qué hace el workflow, qué tienes que preparar, qué cambiar para que funcione con tu cuenta y cómo resolver los problemas más habituales.

---

## 1. ¿Qué hace?

Coges un PDF (una factura, una nómina o cualquier otro documento), y el sistema:

1. Extrae el texto del PDF.
2. Le pide a Gemini (una IA de Google) que identifique qué tipo de documento es y que saque los datos importantes.
3. Guarda esos datos en una hoja de cálculo de Google Sheets, en la pestaña que corresponde.
4. Si no sabe qué es o algo falla, lo apunta en una pestaña de errores para que lo revises tú.

Hay **dos formas de entrar** un documento:

- **Formulario web:** subes uno o varios PDF desde una página web.
- **Correo de Gmail:** cuando llega un correo a tu bandeja de entrada con un PDF adjunto.

---

## 2. Qué necesitas antes de empezar

- Una cuenta de **n8n** (en la nube o instalado en tu equipo).
- Una cuenta de **Google** con:
  - Acceso a **Google Sheets**.
  - Acceso a **Gmail** (solo si quieres usar el camino del correo).
  - Una **clave de API de Gemini** (se obtiene en Google AI Studio).
- El archivo `workflow_formulario_config_publico.json` (este workflow).
- El archivo `configuracion.csv` (la configuración de tipos de documento).

---

## 3. Preparar la hoja de cálculo

Crea una hoja nueva de Google Sheets (por ejemplo, "Registro Documental"). Dentro tendrás que crear **estas pestañas**, con **exactamente estos nombres** (con tildes incluidas):

| Pestaña | Obligatoria | Para qué sirve |
|---|---|---|
| `Configuracion` | Sí | Qué tipos de documento existen y qué campos extraer |
| `Factura` | Sí | Facturas |
| `Nóminas` | Sí | Nóminas |
| `Log` | Sí | Registro de cada documento procesado |
| `Errores` | Sí | Documentos no reconocidos o con fallos |
| `Presupuesto` | No | Todavía no se usa |

### 3.1 Pestaña `Configuracion`

Importa el archivo `configuracion.csv` dentro de esta pestaña:

1. Abre la pestaña `Configuracion` (vacía).
2. Ve a **Archivo > Importar > Subir**, elige `configuracion.csv`.
3. Elige **Reemplazar hoja actual**.
4. Pulsa **Importar datos**.

Debe quedar así (cabeceras en la fila 1 y cuatro filas de datos):

| tipo_documento | activo | campos | instrucciones |
|---|---|---|---|
| _General | SI | (vacío) | Reglas generales del prompt |
| Factura | SI | Campos de factura | (vacío) |
| Nomina | SI | Campos de nómina | Regla de importes negativos |
| Presupuesto | NO | Campos de presupuesto | Nota |

### 3.2 Pestaña `Factura`

Fila 1, en este orden:

```
Tipo Documento | Fecha Documento | Confianza | Archivo | Nº Factura | Fecha Vencimiento | Emisor (Nombre) | Emisor (CIF) | Receptor (Nombre) | Receptor (CIF) | Desglose IVA (JSON) | IVA | IRPF | Base Imponible | Total | Moneda | Forma de Pago | IBAN
```

### 3.3 Pestaña `Nóminas`

Fila 1, en este orden:

```
id_documento | archivo | fecha_documento | periodo | fecha_pago | dias | empresa_nombre | empresa_cif | centro_trabajo | ccc_ss | empleado_nombre | empleado_codigo | empleado_nif | num_afiliacion_ss | categoria | jornada_pct | antiguedad | total_devengos | total_descuentos | liquido | irpf_porcentaje | irpf_base | irpf_importe | base_cotizacion_ss | confianza | fecha_procesado
```

### 3.4 Pestaña `Log`

Fila 1:

```
fecha | origen | archivo | tipo_documento | confianza | caracteres | tokens_estimados
```

### 3.5 Pestaña `Errores`

Fila 1:

```
fecha | origen | archivo | tipo_documento | motivo | texto_original
```

> **Importante:** los nombres de las cabeceras deben escribirse exactamente igual, en minúsculas o mayúsculas según la lista. Si una cabecera no coincide, el dato no se guarda.

---

## 4. Conseguir las credenciales

Necesitas tres conexiones. Se crean una vez y luego se reutilizan.

1. **Google Sheets:** en n8n, *Credentials > New > Google Sheets OAuth2 API*. Sigue las instrucciones para autorizar tu cuenta de Google.
2. **Gmail:** *Credentials > New > Gmail OAuth2 API*. Solo necesario si usas el camino del correo.
3. **Gemini:** *Credentials > New > Google Gemini(PaLM) API*. Pega tu clave de API de Google AI Studio.

---

## 5. Importar el workflow

1. En n8n, pulsa **Add workflow > Import from file** (o los tres puntos > **Import from file**).
2. Selecciona `workflow_formulario_config_publico.json`.
3. Aparecerán todos los nodos. Algunos mostrarán un aviso rojo porque les falta la credencial o el ID de la hoja. Es normal; sigue los pasos siguientes.

---

## 6. Variables que tienes que cambiar

Esta es la lista completa de lo que hay que tocar para que el workflow funcione con tu cuenta.

### 6.1 Tu hoja de cálculo (obligatorio)

En **cada** nodo de Google Sheets, el campo *Document* contiene el texto `PON_AQUI_EL_ID_DE_TU_HOJA`. Hay 6 nodos: `Leer configuracion Gmail`, `Leer configuracion formulario`, `Facturas`, `Nóminas`, `Errores` y `Log`.

Para cambiarlo:

1. Abre tu hoja en Google Sheets.
2. Copia el ID de la URL: es la parte larga entre `/d/` y `/edit`.
   `https://docs.google.com/spreadsheets/d/ESTE_ES_EL_ID/edit`
3. En cada nodo de Sheets, en *Document*, cambia el modo a **From list** y elige tu hoja. Si prefieres pegar el ID, déjalo en **By ID** y pega el ID.

### 6.2 Credenciales (obligatorio)

En cada nodo que tenga un aviso de credencial:

- Nodos de Sheets: elige tu credencial de **Google Sheets**.
- Nodos de Gemini (`Gemini Gmail` y `Gemini formulario`): elige tu credencial de **Gemini**.
- Nodos de Gmail (`Gmail Trigger` y `Get a message`): elige tu credencial de **Gmail**. Si no vas a usar el correo, puedes borrar estos dos nodos y su conexión.

### 6.3 Columnas de cada pestaña (obligatorio)

En los nodos `Facturas`, `Nóminas`, `Errores` y `Log`:

1. Comprueba que en *Sheet* aparece el nombre de la pestaña correcta (`Factura`, `Nóminas`, `Errores` o `Log`).
2. Si aparece el aviso "No columns found" o "columns.schema is required", pulsa **Retry** o cambia el modo de la hoja a **From list** y vuelve a elegirla.
3. Comprueba que aparecen todos los campos en *Values to Send*.
4. Si alguno no aparece, vuelve a pegar su expresión (ver sección 9).

### 6.4 Parámetros opcionales (ajustables)

| Qué | Dónde | Valor por defecto | Para qué |
|---|---|---|---|
| Máximo de documentos por envío | Nodo `Limitar a 20 documentos` | 20 | Controla el coste y el tiempo |
| Documentos por lote | Nodo `Lotes` | 2 | Cuántos PDF se envían a Gemini a la vez |
| Pausa entre lotes | Nodo `Pausa entre lotes` | 15 segundos | Evita superar el límite de peticiones de Gemini |
| Reintentos si Gemini falla | Nodos `Gemini Gmail` y `Gemini formulario` | 3 intentos, 5 s | Recupera fallos temporales |
| Modelo de IA | Nodos `Gemini Gmail` y `Gemini formulario` | `gemini-3-flash-preview` | Cambiar si tu cuenta tiene otro modelo disponible |
| Frecuencia de revisión del correo | Nodo `Gmail Trigger` | Cada minuto | Cada cuánto busca correos nuevos |
| Etiqueta de Gmail | Nodo `Gmail Trigger` | `INBOX` (bandeja de entrada) | Puedes filtrar por otra etiqueta si lo prefieres |
| Título y texto del formulario | Nodo `Form Trigger` | "Subir documentos" | Lo que ve quien sube los archivos |

### 6.5 Configuración de tipos (en Google Sheets)

Todo lo que defina qué extraer se cambia en la pestaña `Configuracion`, sin tocar el workflow:

- **activo:** `SI` para usar un tipo, `NO` para desactivarlo.
- **campos:** lista de datos que Gemini debe extraer para ese tipo.
- **instrucciones:** reglas extra para ese tipo.

> No cambies los nombres de los tipos (`Factura`, `Nomina`, `Presupuesto`) ni las cabeceras de la hoja: el workflow los usa para decidir a qué pestaña va cada documento.

---

## 7. Activar y probar

1. Pulsa **Save** en el workflow.
2. Activa el workflow con el interruptor de arriba a la derecha (**Active**).
3. Abre el nodo **Form Trigger** y copia la **Production URL** del formulario.
4. Ábrela en el navegador, sube un PDF y pulsa **Enviar**.
5. Espera 30 segundos y comprueba:
   - Que aparece una fila en `Factura` o `Nóminas`.
   - Que aparece una fila en `Log`.

Si todo está bien, prueba con un correo que tenga un PDF adjunto enviado a tu cuenta.

> En modo prueba (**Execute workflow**), el formulario solo funciona mientras tienes el editor abierto. Para uso real, el workflow tiene que estar activo.

---

## 8. Cómo funciona el flujo, paso a paso

### 8.1 Camino del formulario

| # | Nodo | Qué hace |
|---|---|---|
| 1 | **Form Trigger** | Muestra un formulario con un campo para subir PDF (varios a la vez). Cada envío arranca el flujo. |
| 2 | **Separar archivos formulario** | Coge cada PDF del envío y lo convierte en un elemento independiente. Descarta cualquier archivo que no sea PDF. |
| 3 | **Limitar a 20 documentos** | Si se suben más de 20, procesa solo los 20 primeros. Sirve para controlar el coste. |
| 4 | **Extraer texto formulario** | Lee el PDF y saca su texto. |
| 5 | **Leer configuracion formulario** | Lee la pestaña `Configuracion` **una sola vez** (aunque haya 20 documentos). |
| 6 | **Construir prompt formulario** | Con la configuración, monta las instrucciones para Gemini: qué tipos existen, qué campos hay que extraer para cada uno y las reglas generales. Asigna a cada documento un número (`id_interno`) para poder emparejar la respuesta después. |
| 7 | **Lotes** | Divide los documentos en grupos de 2 (ajustable). |
| 8 | **Gemini formulario** | Envía el prompt y el texto de cada documento a Gemini. Reintenta hasta 3 veces si falla. |
| 9 | **Procesar respuesta formulario** | Convierte la respuesta de Gemini en datos. Si la respuesta no es un JSON válido, la marca como error. Recupera el nombre del archivo usando el `id_interno`. |
| 10 | **Switch** | Decide a dónde va cada documento según su tipo (ver 8.3). |
| 11 | **Log** (en paralelo) | Anota una fila por documento: fecha, origen, archivo, tipo, confianza, caracteres y tokens estimados. |
| 12 | **Pausa entre lotes** | Espera 15 segundos antes del siguiente grupo y vuelve a **Lotes**. Así se repite hasta procesar todos. |

### 8.2 Camino del correo de Gmail

| # | Nodo | Qué hace |
|---|---|---|
| 1 | **Gmail Trigger** | Revisa cada minuto si hay correos nuevos en la bandeja de entrada. |
| 2 | **Get a message** | Descarga el correo completo, con sus adjuntos. |
| 3 | **If** | Comprueba que el correo tiene algún adjunto. Si no tiene, el flujo termina ahí. |
| 4 | **Separar adjuntos PDF** | Coge cada adjunto PDF y lo convierte en un elemento independiente. Ignora imágenes y otros formatos. |
| 5 | **Extraer texto Gmail** | Saca el texto de cada PDF. |
| 6 | **Leer configuracion Gmail** | Lee la pestaña `Configuracion`. |
| 7 | **Construir prompt Gmail** | Igual que en el formulario: monta las instrucciones para Gemini. |
| 8 | **Gemini Gmail** | Envía cada documento a Gemini. |
| 9 | **Procesar respuesta Gmail** | Convierte la respuesta en datos, igual que en el formulario. |
| 10 | **Switch** y **Log** | Enrutan el documento y lo registran (igual que en el formulario). |

Este camino no tiene límite de lotes ni pausa, porque normalmente llega un correo cada vez.

### 8.3 Enrutado (el Switch)

Cada documento sale del Switch hacia una de tres pestañas:

| Tipo que devuelve Gemini | Pestaña de destino |
|---|---|
| `Factura` | `Factura` |
| `Nomina` | `Nóminas` |
| Cualquier otro (`Otro`, `Presupuesto` desactivado, error de JSON) | `Errores` |

En `Errores` se guarda el motivo y los primeros 500 caracteres del texto que devolvió Gemini, para que puedas revisar qué pasó.

### 8.4 Nivel de confianza

Gemini indica en cada documento si la lectura ha sido `alta`, `media` o `baja`. Te ayuda a decidir qué datos revisar a mano. Los documentos de confianza baja no se separan automáticamente: se guardan igual, pero conviene mirarlos.

### 8.5 Coste y control

- La pestaña `Log` te permite ver cuántos documentos se han procesado y cuánto texto se ha enviado.
- Los tokens son una **estimación** (caracteres / 4), no el consumo real de Gemini. Para el consumo exacto, consulta tu panel de Google AI Studio.

---

## 9. Expresiones de las columnas (por si hay que volver a pegarlas)

Si al refrescar las columnas se borran las expresiones, pégalas de nuevo en cada nodo, en modo **Map Each Column Manually**.

**Facturas**

| Columna | Expresión |
|---|---|
| Tipo Documento | `{{ $json.tipo_documento }}` |
| Fecha Documento | `{{ $json.fecha_documento }}` |
| Confianza | `{{ $json.confianza }}` |
| Archivo | `{{ $json.archivo }}` |
| Nº Factura | `{{ $json.datos_relevantes.numero_factura }}` |
| Fecha Vencimiento | `{{ $json.datos_relevantes.fecha_vencimiento ?? '' }}` |
| Emisor (Nombre) | `{{ $json.datos_relevantes.emisor?.nombre }}` |
| Emisor (CIF) | `{{ $json.datos_relevantes.emisor?.cif }}` |
| Receptor (Nombre) | `{{ $json.datos_relevantes.receptor?.nombre }}` |
| Receptor (CIF) | `{{ $json.datos_relevantes.receptor?.cif }}` |
| Desglose IVA (JSON) | `{{ JSON.stringify($json.datos_relevantes.desglose_iva ?? []) }}` |
| IVA | `{{ $json.datos_relevantes.iva_total }}` |
| IRPF | `{{ $json.datos_relevantes.irpf_importe }}` |
| Base Imponible | `{{ $json.datos_relevantes.base_imponible }}` |
| Total | `{{ $json.datos_relevantes.total }}` |
| Moneda | `{{ $json.datos_relevantes.moneda }}` |
| Forma de Pago | `{{ $json.datos_relevantes.forma_pago }}` |
| IBAN | `{{ $json.datos_relevantes.iban }}` |

**Nóminas**

| Columna | Expresión |
|---|---|
| id_documento | `{{ $json.datos_relevantes.periodo + '_' + $json.datos_relevantes.empleado.codigo }}` |
| archivo | `{{ $json.archivo }}` |
| fecha_documento | `{{ $json.fecha_documento }}` |
| periodo | `{{ $json.datos_relevantes.periodo }}` |
| fecha_pago | `{{ $json.datos_relevantes.fecha_pago }}` |
| dias | `{{ $json.datos_relevantes.dias }}` |
| empresa_nombre | `{{ $json.datos_relevantes.empresa.nombre }}` |
| empresa_cif | `{{ $json.datos_relevantes.empresa.cif }}` |
| centro_trabajo | `{{ $json.datos_relevantes.empresa.centro_trabajo }}` |
| ccc_ss | `{{ $json.datos_relevantes.empresa.ccc_ss }}` |
| empleado_nombre | `{{ $json.datos_relevantes.empleado.nombre }}` |
| empleado_codigo | `{{ $json.datos_relevantes.empleado.codigo }}` |
| empleado_nif | `{{ $json.datos_relevantes.empleado.nif }}` |
| num_afiliacion_ss | `{{ $json.datos_relevantes.empleado.num_afiliacion_ss }}` |
| categoria | `{{ $json.datos_relevantes.empleado.categoria }}` |
| jornada_pct | `{{ $json.datos_relevantes.empleado.jornada_pct }}` |
| antiguedad | `{{ $json.datos_relevantes.empleado.antiguedad }}` |
| total_devengos | `{{ $json.datos_relevantes.total_devengos }}` |
| total_descuentos | `{{ $json.datos_relevantes.total_descuentos }}` |
| liquido | `{{ $json.datos_relevantes.liquido }}` |
| irpf_porcentaje | `{{ $json.datos_relevantes.irpf.porcentaje }}` |
| irpf_base | `{{ $json.datos_relevantes.irpf.base }}` |
| irpf_importe | `{{ $json.datos_relevantes.irpf.importe }}` |
| base_cotizacion_ss | `{{ $json.datos_relevantes.base_cotizacion_ss }}` |
| confianza | `{{ $json.confianza }}` |
| fecha_procesado | `{{ $now.toISO() }}` |

**Log**

| Columna | Expresión |
|---|---|
| fecha | `{{ $now.toISO() }}` |
| origen | `{{ $json.origen }}` |
| archivo | `{{ $json.archivo }}` |
| tipo_documento | `{{ $json.tipo_documento }}` |
| confianza | `{{ $json.confianza }}` |
| caracteres | `{{ $json.caracteres }}` |
| tokens_estimados | `{{ $json.tokens_estimados }}` |

**Errores**

| Columna | Expresión |
|---|---|
| fecha | `{{ $now.toISO() }}` |
| origen | `{{ $json.origen }}` |
| archivo | `{{ $json.archivo }}` |
| tipo_documento | `{{ $json.tipo_documento }}` |
| motivo | `{{ $json.motivo ?? 'Tipo no soportado' }}` |
| texto_original | `{{ ($json.texto_original ?? '').slice(0, 500) }}` |

---

## 10. Problemas habituales

| Síntoma | Causa probable | Solución |
|---|---|---|
| "Sheet with name ... not found" | La pestaña no existe o el nombre no coincide | Crea la pestaña con ese nombre exacto (sección 3). |
| "columns.schema is required" o "No columns found" | El nodo no ha cargado las columnas | Elige la hoja **From list** y pulsa **Retry**. |
| Las celdas quedan vacías | Las expresiones se han borrado o los nombres de cabecera no coinciden | Revisa las cabeceras (sección 3) y pega las expresiones (sección 9). |
| El formulario no responde | El workflow no está activo | Activa el workflow y usa la **Production URL**. |
| Documentos que van a `Errores` | El tipo no está activo, o Gemini no devolvió un JSON válido | Mira el campo `motivo` y `texto_original` en `Errores`. |
| Error de credencial o de permisos | La credencial no está autorizada | Vuelve a conectar la credencial en el nodo. |
| Se procesa la misma factura dos veces | No hay control de duplicados | Revisa la pestaña antes de volver a subir el documento. |
| Gemini devuelve error 429 o "quota" | Has superado el límite de peticiones | Baja el tamaño de lote, aumenta la pausa o espera a que se renueve tu cuota. |

---

## 11. Aviso sobre datos personales

El workflow está preparado sin datos personales ni credenciales. Al importarlo, sus datos (hoja, cuentas, claves) son los tuyos. No compartas tu clave de API ni los archivos de tu hoja con terceros.
