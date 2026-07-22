---
name: sales_analysis
description: >-
  Ventas por segmento, canal, marca, evolución temporal.
metadata:
  category: data
  agent: sap_analyst
  display-name: "Análisis de Ventas"
---

# Análisis de Ventas KH Lloreda

## Selección de query según la pregunta

| Tipo de pregunta | Query | Notas |
|---|---|---|
| Ventas del mes/año actual | `PBI_SEG_CLI_VNE_Q002` | Tiene 2026, VN estimada |
| Ventas históricas con detalle | `PBI_SEG_CLI_VNE_Q004` | Sin 2026, pero más medidas detalladas |
| Cierre de ventas con márgenes | `PBI_CIERRE_VN` | VN real/obj/prev + márgenes. Variable mes obligatoria |
| Evolución anual rápida | `CO_EVOL_VENTAS_ANUAL_OPT` | 5 medidas, multi-año |
| Histórico mensual (unidades) | `VTAS_HIST_MENS_OPT` | 3 medidas: unidades, facturado, PM |
| Año cerrado (post-cierre) | `PBI_SEG_CLI_VNE_Q005` | Solo disponible tras cierre fiscal |

## Medidas clave de ventas (Q002)

Las 21 medidas incluyen:
- `00O2TOVQWROIO6HAL7LO7NP33` — Venta Neta estimada
- `PBI_VNETA_ACUM_MC` — Venta Neta acumulada **meses cerrados**
- `PBI_VNETA_ACUM_MA` — Venta neta estimada acumulada **meses abiertos**
- `PBI_PREV_MESACT` — Previsión mes actual
- `PBI_OBJ_MESACT` — Objetivo mes actual
- `00O2TOVQWROHDPXYP1VSGSW4G` — **Unidades totales meses cerrados** (usar en meses cerrados)
- `00O2TOVQWROHDPXYP1VSGT2G0` — **Unidades estimadas meses abiertos** (usar SOLO en mes abierto)
- `00O2TOVQWROIZJ0DRA4PJ5KGL` — Unidades Facturadas Año Actual (parcial en mes abierto; NO sumar con estimadas)
- `00O2TOVQWROIZJ09GWT0Y51Y5` — Un Estimadas

### Regla mes abierto vs cerrado (CRÍTICO)

| Estado del mes | Unidades a reportar | VN a reportar |
|---|---|---|
| **Cerrado** | `Unidades totales meses cerrados` | `PBI_VNETA_ACUM_MC` |
| **Abierto** (mes en curso) | SOLO `Unidades estimadas meses abiertos` | SOLO `PBI_VNETA_ACUM_MA` / VN estimada |

**NUNCA** sumar facturadas + estimadas en el mes abierto (ej. 128.700 + 154.440 → 283.140 es incorrecto; lo correcto es 154.440).

### Unidades + VN + Margen Distribución

No mezclar VN del P&L (`MKT_CUENTA_RES_*`) con unidades de Q002: el P&L puede devolver VN multi-año si el filtro de año no aplica bien.
- Unidades + VN → `PBI_SEG_CLI_VNE_Q002` (regla de arriba)
- Margen Distribución → `PBI_CIERRE_VN` / `00O2TOVQWROINRRJ8VNIKAU24` (Margen Distribución OLD); la VN Real de CIERRE debe alinearse con Q002
- Meses futuros del año en curso → "—" (sin inventar datos)

## Dimensiones de análisis de ventas

### Por segmento de mercado
`dimension="ZSEGMEN"` → NACIONAL, EXPORTACION, DISTRIBUCION, USA

### Por marca
`dimension="0MATERIAL__YCOPAPH1"` → KH-7, CIF, DOMESTOS

### Por sub-marca
`dimension="0MATERIAL__YCOPAPH2"` → Quitagrasas, Sin Manchas, Antical, etc.

### Por grupo de cliente
`dimension="0CUST_SALES__0CUST_GROUP"` → SUPER, HIPER, DISCOUNT, MERCADONA, etc.

### Por mes (dentro de un año)
`dimension="0CALMONTH2"` → 01, 02, ..., 12

## Patrones de análisis

### Análisis de mes actual
```
biw_executeQuery(
  query="ZBOKCOPA/PBI_SEG_CLI_VNE_Q002",
  measures=["PBI_VNETA_ACUM_MC", "PBI_PREV_ACUM", "PBI_OBJ_ACUM"],
  filters={"0CALMONTH": "YYYYMM"}
)
```
Compara VN real vs previsión y objetivo. Calcula % cumplimiento.

### Comparación año-sobre-año (YoY)
**RECOMENDADO**: Usa `PBI_CIERRE_VN` que devuelve VN Real + VN Anterior + % Crec directamente:
```
biw_executeQuery(
  query="ZBOKCOPA/PBI_CIERRE_VN",
  dimension="ZSEGMEN",
  filters={"0CALMONTH2": "01"}
)
```
Devuelve YoY desglosado con real, anterior y % crecimiento por segmento en una sola llamada.

Para histórico multi-año, usa `VTAS_HIST_MENS_OPT` con `dimension="0CALYEAR"`.

**Nota**: Las medidas "AA"/"Anterior" de Q002/Q004 devuelven 0 (no tienen customer exits). Para YoY siempre prefiere `PBI_CIERRE_VN`.

### Evolución mensual del año
```
biw_executeQuery(
  query="ZBOKCOPA/PBI_SEG_CLI_VNE_Q002",
  dimension="0CALMONTH",
  filters={"0CALYEAR": "2026"}
)
```

### Top N clientes por VN (YTD / ene–jun / meses cerrados del año)
```
biw_executeQuery(
  query="ZBOKCOPA/PBI_SEG_CLI_VNE_Q002",
  measures=["PBI_VNETA_ACUM_MC"],
  dimension="0CUSTOMER",
  filters={"0CALYEAR": "2026", "ZSEGMEN": "NAC"}  # DIS = Distribución
)
```
**CRÍTICO**: para un rango de meses cerrados del año **no filtres mes**. Con `0CALMONTH`/`0CALMONTH2`, `PBI_VNETA_ACUM_MC` devuelve **solo ese mes** (p.ej. enero → Mercadona 435k; YTD correcto → ~2,6M). Ordena desc en la respuesta.

### Top-down analysis
1. **Totales** → ¿cómo va el mes?
2. **Por segmento** → ¿quién contribuye más/menos?
3. **Por marca** (filtrado al segmento destacado) → ¿qué marca impulsa/frena?
4. **Por cliente** → ¿quién compra más/menos?

### Mes abierto vs mes cerrado
- **Mes cerrado**: Datos definitivos (`Unidades totales meses cerrados` + `PBI_VNETA_ACUM_MC`), desglosables por dimensiones
- **Mes abierto (actual)**: Usar SOLO estimadas (`Unidades estimadas meses abiertos` + `PBI_VNETA_ACUM_MA`). Las facturadas parciales existen pero **no se suman** a las estimadas. Meses posteriores al abierto → sin dato real.

## Cierre de ventas (PBI_CIERRE_VN)

Query recomendada para YoY y cierre con márgenes. El proxy auto-gestiona las variables SAP:
- `0CALMONTH2` se enruta a la variable `MESOBLI` (customer exit)
- `0CALYEAR` se elimina del WHERE automáticamente (el customer exit usa fecha del sistema)
- `0CALMONTH` compuesto ("202601") se descompone automáticamente

```
biw_executeQuery(
  query="ZBOKCOPA/PBI_CIERRE_VN",
  dimension="ZSEGMEN",
  filters={"0CALMONTH2": "06"}
)
```
Devuelve por segmento: VN Real (2026 hasta junio), VN Anterior (2025 hasta junio), % Crec, márgenes, etc.

## Stock e Inventario

Query `0IC_C03/PBI_ST_001` (4 medidas): unidades y valor en stock.
Útil para contrastar con datos de ventas (rotación, cobertura).

## Contexto de negocio para interpretar datos

- KH-7 domina con 88% de ingresos → cualquier variación en KH-7 impacta fuertemente al total
- NACIONAL es el segmento principal → EXPORTACION puede tener mayor crecimiento %
- Estacionalidad: Q1 y Q3 suelen ser más fuertes en limpieza doméstica
- MERCADONA (ZD) y SUPER (Z2) son los canales de mayor volumen
