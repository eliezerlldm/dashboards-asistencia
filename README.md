---
name: attendance-dashboard
description: >
  Genera un dashboard interactivo y compartible de asistencia a partir de un archivo CSV.
  Úsalo siempre que el usuario suba un CSV de asistencia, mencione lista de asistentes,
  registro de entrada, control de asistencia a reuniones o estudios, o pida visualizar
  quiénes asistieron a un evento con datos de hora, iglesia, zona o grado.
  También aplica cuando el usuario quiera reportes de puntualidad, retardos, distribución
  por iglesia/congregación, o un archivo HTML compartible con esos datos.
---

# Attendance Dashboard Skill

Genera un dashboard HTML completo, interactivo y compartible a partir de un CSV de asistencia.
El output es un único archivo `.html` autocontenido, listo para abrir en cualquier navegador
o dispositivo (móvil o escritorio), sin dependencias externas salvo Chart.js desde CDN.

---

## Paso 1 — Leer y procesar el archivo (CSV o Excel)

El usuario puede subir `.csv` o `.xlsx` — ambos son válidos.
Detectar la extensión y leer con el método correspondiente:

```python
import json, os
from collections import defaultdict

filepath = '/mnt/user-data/uploads/ARCHIVO'  # ajustar al nombre real del archivo

records = []
ext = os.path.splitext(filepath)[1].lower()

if ext == '.csv':
    import csv
    with open(filepath, encoding='utf-8-sig') as f:
        reader = csv.DictReader(f)
        for row in reader:
            records.append({k.strip(): v.strip() for k, v in row.items()})

elif ext in ('.xlsx', '.xls'):
    # instalar si no está disponible:
    # pip install openpyxl --break-system-packages --quiet
    import openpyxl
    wb = openpyxl.load_workbook(filepath, data_only=True)
    ws = wb.active
    headers = [str(cell.value).strip() for cell in next(ws.iter_rows(min_row=1, max_row=1))]
    for row in ws.iter_rows(min_row=2, values_only=True):
        records.append({headers[i]: (str(v).strip() if v is not None else '') for i, v in enumerate(row)})

# 1. DEDUPLICAR: si hay nombre repetido, conservar el registro más temprano
seen = {}
for r in records:
    nombre = r['Nombre'].strip()
    h, m, s = r['Hora'].split(':')
    t = int(h)*3600 + int(m)*60 + int(s)
    if nombre not in seen or t < seen[nombre]['_t']:
        seen[nombre] = {**r, '_t': t}
deduped = list(seen.values())

# 2. SEPARAR registros sin clave de iglesia válida → Credencial de Staff
staff, iglesias_data = [], defaultdict(lambda: {'clave':'', 'personas':[]})
for r in deduped:
    clave = r.get('Clave','').strip()
    iglesia = r.get('Iglesia','').strip()
    if 'no encontrada' in clave.lower() and 'ACREDITACIONES' in iglesia.upper():
        staff.append({'nombre': r['Nombre'], 'hora': r['Hora'][:5]})
    else:
        iglesias_data[iglesia]['clave'] = clave
        iglesias_data[iglesia]['personas'].append({
            'nombre': r['Nombre'],
            'hora': r['Hora'][:5],
            'grado': r['Grado']
        })

# 3. Ordenar personas dentro de cada iglesia por hora
for ig in iglesias_data:
    iglesias_data[ig]['personas'].sort(key=lambda x: x['hora'])

# 4. Construir lista ordenada por count descendente
data = sorted([
    {'iglesia': k, 'clave': v['clave'],
     'count': len(v['personas']), 'personas': v['personas']}
    for k, v in iglesias_data.items()
], key=lambda x: -x['count'])

# 5. Bloques de 10 minutos
blocks = defaultdict(int)
for r in deduped:
    h, m = int(r['Hora'][:2]), int(r['Hora'][3:5])
    key = f"{h:02d}:{(m//10*10):02d}"
    blocks[key] += 1
bl = sorted(blocks.keys())
bv = [blocks[k] for k in bl]

# 6. Franjas de 30 minutos
slots = defaultdict(int)
slot_names = []  # generar dinámicamente según rango de horas
for r in deduped:
    h, m = int(r['Hora'][:2]), int(r['Hora'][3:5])
    mins = h*60+m
    # bucket en 30 min
    bucket = (mins // 30) * 30
    bh, bm = bucket//60, bucket%60
    slots[f"{bh:02d}:{bm:02d}"] += 1
slot_labels = sorted(slots.keys())
slot_vals = [slots[k] for k in slot_labels]

# 7. Promedio de entrada
tiempos = []
for r in deduped:
    h, m, s = r['Hora'].split(':')
    tiempos.append(int(h)*3600+int(m)*60+int(s))
prom = sum(tiempos)//len(tiempos)
prom_str = f"{prom//3600:02d}:{(prom%3600)//60:02d}"

# 8. Distribución por grado
roles = defaultdict(int)
for r in deduped:
    roles[r['Grado']] += 1

print(json.dumps({
    'total': len(deduped),
    'n_iglesias': len(data),
    'promedio': prom_str,
    'bloque_pico': bl[bv.index(max(bv))] if bv else '',
    'pico_val': max(bv) if bv else 0,
    'data': data,
    'staff': staff,
    'bl': bl, 'bv': bv,
    'slot_labels': slot_labels, 'slot_vals': slot_vals,
    'roles': dict(roles)
}, ensure_ascii=False))
```

---

## Paso 2 — Construir el HTML

El archivo final debe ser **un único `.html` autocontenido**. Estructura obligatoria:

### Header del dashboard
```html
<div class="hdr-eyebrow">Zona / Grupo del evento</div>
<div class="hdr-title">Asistencia <span>DD Mes AAAA</span></div>
<div class="hdr-pills">
  <span class="hdr-pill pill-g">N Asistentes</span>
  <span class="hdr-pill pill-o">N Iglesias</span>
</div>
<div class="hdr-meta">Inicio: HH:MM Hrs · Fin: HH:MM Hrs</div>
<!-- Botón imprimir -->
```

### Métricas (4 cards)
Total asistentes · Iglesias · Promedio entrada · Bloque pico

### Gráfica principal — Bloques de 10 min (COMBINADA)
- Tipo: `bar` + `line` en el mismo canvas (Chart.js mixed)
- Barras: color por zona horaria (verde=puntual, ámbar=retardo, rojo=tarde)
- Línea dorada: moving average de ventana 3
- Umbrales configurables (default: retardo ≥15:00, tarde ≥16:00)

### Gráfica secundaria — Bloques de 30 min
- Tipo: `bar`
- Colores: escala verde oscuro→verde claro para horas tempranas, ámbar/rojo para tardías

### Gráfica — Integrantes (dona)
- Coro / Miembro / Staff con colores verde, dorado, gris

### Lista de iglesias (interactiva)
- Cada fila: `[CLAVE] NOMBRE ████████ N`
- Al hacer clic → despliega tabla con nombre, hora, grado de cada persona
- Hora coloreada: normal / retardo (ámbar) / tarde (rojo)
- Grado centrado en columna

### Credencial de Staff
- Sección dorada separada debajo de la lista de iglesias
- Muestra nombre y hora de cada registro sin iglesia asignada

### Sección de impresión (oculta en pantalla)
- `@media print` muestra todas las iglesias expandidas en tablas
- Columnas Hora y Grado centradas
- Incluye sección Staff al final

---

## Paso 3 — Paleta de colores (verde + dorado)

```css
--gd: #1b5e35;   /* verde oscuro */
--gm: #2e7d52;   /* verde medio */
--gl: #e8f5ee;   /* verde claro bg */
--gb: #a8d5b8;   /* verde claro borde */
--go: #c9a227;   /* dorado */
--gol: #fdf3d7;  /* dorado claro bg */
--gob: #e8cc7a;  /* dorado borde */
--god: #8a6d0b;  /* dorado oscuro */
--am: #b7770d;   /* ámbar (retardo) */
--r:  #c0392b;   /* rojo (tarde) */
```

Barra superior: `linear-gradient(90deg, #1b5e35, #2e7d52, #c9a227)`

---

## Paso 4 — Mobile responsiveness

```css
/* Móvil: stack todo en columna */
@media (max-width: 640px) {
  .metrics { grid-template-columns: repeat(2, 1fr); }
  .row2    { grid-template-columns: 1fr; }
  .hdr     { flex-direction: column; }
  .hdr-pills { flex-wrap: wrap; }
  .ck      { min-width: 60px; font-size: 8px; }
  .cn      { font-size: 11px; }
  canvas   { max-height: 180px; }
  body     { padding: 1rem 0.75rem; }
}
```

Todas las gráficas usan `responsive: true, maintainAspectRatio: false` con wrappers de altura fija.

---

## Paso 5 — Seguridad: datos no manipulables

- Los datos van **hardcodeados** como `const DATA = [...]` y `const STAFF = [...]`
- No usar `localStorage`, `sessionStorage`, ni inputs editables
- No usar `contenteditable`
- El HTML se renderiza solo para lectura

---

## Paso 6 — Reglas de hora y alertas

| Hora de entrada | Etiqueta | Color |
|---|---|---|
| Antes del umbral_retardo | (normal) | texto muted |
| ≥ umbral_retardo (default 15:00) | Retardo | ámbar |
| ≥ umbral_tarde (default 16:00) | Tarde | rojo |

Los umbrales los define el usuario al inicio ("el estudio empezó a las X").
Si no los da, usar: retardo = inicio + 60 min, tarde = inicio + 120 min.

---

## Paso 7 — Output

```bash
# Guardar en outputs
cp /home/claude/dashboard_asistencia.html /mnt/user-data/outputs/dashboard_asistencia_FECHA.html
```

Usar `present_files` para entregar el archivo al usuario.
Confirmar: "¿Deseas ajustar umbrales, colores o agregar más secciones antes de cerrar?"

---

## Notas importantes

- **Deduplicar siempre** antes de procesar — un nombre = un registro (el más temprano)
- **Columnas Hora y Grado** siempre centradas (`text-align: center`) tanto en vista interactiva como en impresión
- **Botón Imprimir/PDF** siempre presente en header (color dorado, con icono de impresora SVG)
- **CDN permitido**: solo `cdnjs.cloudflare.com` para Chart.js 4.4.1
- El archivo debe funcionar offline una vez descargado (solo Chart.js necesita internet la primera vez)

---

## Regla de unidad "Hrs"

Mostrar `Hrs` **solo donde el número necesita contexto** para entenderse como hora:

| Dónde | Ejemplo | ¿Lleva Hrs? |
|---|---|---|
| Cards de métricas (promedio, bloque pico) | `15:12 Hrs` | ✅ Sí, en dorado pequeño |
| Header (inicio / fin del evento) | `Inicio: 14:00 Hrs` | ✅ Sí |
| Etiquetas de zona en leyenda de gráfica | `< 15:00 Hrs` | ✅ Sí |
| Leyenda de retardo/tarde | `después de las 15:00 Hrs` | ✅ Sí |
| Tooltip de gráfica al hacer hover | `Bloque 14:30 Hrs` | ✅ Sí |
| Filas de personas (columna Hora) | `14:26` | ❌ No — la columna ya dice "Hora" |
| Sección Staff | `15:02` | ❌ No |

La clase `.mc-unit` en dorado (`color: var(--go)`) se usa para el "Hrs" de las métricas:
```html
<div class="mc-val">15:12 <span class="mc-unit">Hrs</span></div>
```

---

## Gráfica de bloques de 10 min — tipo combinado (default)

La gráfica principal **siempre** debe ser de tipo **barras + línea combinadas** (Chart.js mixed type). No usar solo barras. Este es el default definitivo:

- **Barras**: volumen de entradas por bloque, coloreadas por zona horaria
- **Línea dorada**: moving average de ventana 3 para suavizar tendencia
- `interaction: { mode: 'index', intersect: false }` para tooltip conjunto
- `order: 2` en barras, `order: 1` en línea (línea encima)

```js
function movAvg(d, w) {
  return d.map((_, i) => {
    const s = Math.max(0, i - Math.floor(w/2));
    const e = Math.min(d.length, s + w);
    const sl = d.slice(s, e);
    return Math.round(sl.reduce((a,b) => a+b, 0) / sl.length * 10) / 10;
  });
}
```

---

## Capitalización y redacción de la interfaz

Usar **sentence case** en toda la interfaz — solo la primera letra de la oración en mayúscula, igual que se escribe en español natural. No usar Title Case en hints, subtítulos ni instrucciones al usuario.

| ❌ Evitar | ✅ Usar |
|---|---|
| `Clic En Una Iglesia Para Ver Sus Asistentes` | `Clic en una iglesia para ver sus asistentes` |
| `Barras = Cantidad Por Bloque` | `Barras = cantidad por bloque` |
| `Total De Personas Por Período` | `Total de personas por período` |

Las excepciones son nombres propios, siglas (STAFF, PDF) y labels de columnas en tablas.

---

## Hosting gratuito al entregar el archivo

Al entregar el archivo final, mencionar brevemente las opciones de publicación gratuita:

**Netlify Drop** — la más rápida, sin registro requerido:
- Ir a `netlify.com/drop`
- Arrastrar el archivo `.html`
- URL disponible al instante tipo `nombre-random.netlify.app`
- Sin cuenta: caduca en 24 hrs. Con cuenta gratuita: permanente.

**GitHub Pages** — recomendado para uso recurrente (múltiples zonas o eventos):
- Crear repositorio en `github.com`
- Subir el archivo como `index.html`
- Activar Pages en Settings → Pages → Branch: main
- URL permanente tipo `usuario.github.io/nombre-repo`

Sugerir GitHub Pages cuando el usuario mencione que habrá dashboards futuros o por zona.
