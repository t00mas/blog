---
layout: post
title: "Cuando el BOJA no es parseable"
date: 2026-04-30
lang: es
permalink: /boja-parser/
---

El 27 de abril de 2026, el Boletín Oficial de la Junta de Andalucía publicó el número 79 C1: 402 páginas con las declaraciones de actividades, bienes e intereses de los 1.129 candidatos a las elecciones del 17 de mayo. Datos públicos, en teoría accesibles para cualquier ciudadano. En la práctica, un PDF.

No cualquier PDF. Un PDF generado a partir de formularios escaneados y recompuestos, con columnas que `pdftotext` extrae en orden incorrecto, cabeceras del boletín que se intercalan con los datos, y secciones enteras que aparecen desplazadas varias páginas respecto a donde deberían estar.

## El problema con `pdftotext`

`pdftotext` convierte el PDF a texto plano siguiendo el orden visual de las columnas. Para las tablas del BOJA eso es un desastre: algunas secciones salen en orden de filas (primero todos los datos de la fila 1, luego de la fila 2...), pero otras salen en orden de columnas (primero todos los valores de la columna 1 para todos los candidatos, luego todos los de la columna 2). No hay manera de saber cuál es cuál sin inspeccionar el layout original del PDF.

La solución fue detectar el patrón: si el texto de una sección empieza con una cabecera de columna seguida de todos sus valores antes de que aparezca la siguiente cabecera, es orden de columnas. Si los valores de columnas distintas se intercalan, es orden de filas. El parser aplica una estrategia diferente según el caso.

## Ruido del boletín

Entre cada página, el PDF tiene fragmentos de pie de página del BOJA: número de depósito legal, URL de la Junta, número de página, la palabra "BOJA" sola en una línea. Todo eso aparece intercalado con los datos de los candidatos. Hay que eliminar esos fragmentos sin borrar datos reales que puedan parecerse.

```python
def clean_headers(text):
    text = re.sub(r"Depósito Legal:[^\n]+\n\s*\n?\s*https?://[^\n]+\n?", "\n", text)
    text = re.sub(r"(?:BOJA\s*\n\s*)?Boletín Oficial de la Junta de Andalucía\s*\n[^\n]+\n\s*página \d+/\d+", "", text)
    text = re.sub(r"^\s*BOJA\s*$", "", text, flags=re.MULTILINE)
    # ...
    return text
```

## Filas desplazadas y secciones fuera de lugar

El caso más frecuente: `pdftotext` mete el contenido de una sección dentro del texto de otra. La sección 2.3 (acciones y valores) puede aparecer con filas que en realidad pertenecen a la 2.4 (vehículos) o 2.6 (deudas), porque el layout del PDF original los tenía cerca y la extracción los mezcló.

La heurística para detectarlo: si en el texto de la sección 2.3 aparece una cabecera "Descripción" (que pertenece a 2.4), hay contenido desplazado. Se re-parsea con el formato de columnas correcto y se reclasifica cada fila por palabras clave: "hipoteca" → deuda, "seguro de vida" → seguro, el resto → vehículos.

```python
_seg_kw = re.compile(r"(?i)(segur|póliza)")
_deu_kw = re.compile(r"(?i)(hipoteca|préstamo|crédito|financiaci)")

if re.search(r"(?m)^Descripción\s*$", sec23_text):
    displaced_rows = parse_table("2.4", sec23_text)
    for row in displaced_rows:
        desc = str(row.get("Descripción") or "")
        if _seg_kw.search(desc):
            seguros.append(row)
        elif _deu_kw.search(desc):
            deudas.append(row)
        else:
            vehiculos.append(row)
```

## Overrides manuales

Para algunos candidatos el layout es tan irregular que no hay heurística que lo resuelva bien. El presidente en funciones de la Junta, Juan Manuel Moreno Bonilla, tiene dos fincas rústicas en Cártama cuyo valor catastral aparece fundido con la descripción de la situación en un único bloque de texto. El parser lo detecta pero no lo separa correctamente. Solución: override manual para ese candidato.

## El resultado

Un fichero `candidates.json` con 1.129 objetos estructurados: datos financieros calculados (patrimonio neto, total activos, total deudas) y las tablas de detalle completas.

Sobre ese JSON, una aplicación web de una sola página sin dependencias de servidor ni paso de compilación: tabla con búsqueda y filtros, comparativa por partido, señales de alerta para patrones financieros llamativos, y diagrama de dispersión con todos los candidatos coloreados por partido.

**[→ Explorar las declaraciones](https://tomaspica.com/declaraciones-andalucia-2026/)**

El código del parser y la web están en el repositorio, bajo licencia MIT. Los datos son públicos (BOJA).
