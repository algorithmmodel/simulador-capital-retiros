# Simulador de capital y retiros — contexto del proyecto

Documento de traspaso. Léalo antes de tocar `index.html`.

---

## 1. Reglas de trabajo (no negociables)

- **Tratar de "usted".** Nunca "vos" ni "tú".
- **No escribir ni modificar código sin un "OK" explícito.** Proponer primero, esperar confirmación, después ejecutar.
- **Accesibilidad:** requisito de baja visión. Prohibido gris oscuro sobre fondo negro. El diseño es de fondo claro, alto contraste, tipografía grande. No proponer tema oscuro. El color nunca es la única señal: las líneas de los gráficos se distinguen además por el trazo (continuo, rayado, punteado).
- **Sin dependencias externas.** Ni CDN, ni npm, ni build. Un archivo HTML autocontenido que se abre en Safari iOS y se publica tal cual en GitHub Pages.
- **Objetivo de uso:** iPhone (Safari) y Mac. La UI se diseña primero para pantalla angosta.

---

## 2. Qué es esto

Un simulador de horizonte financiero: capital inicial, aportes posteriores, retiros semestrales indexados por inflación, y un objetivo de capital final. Responde dos preguntas:

1. ¿Cuánto puedo retirar por semestre sin bajar del capital final que quiero dejar?
2. ¿Cuál es la probabilidad de que ese plan sobreviva?

Nació como reemplazo de un script de Scriptable para iOS que tenía varios bugs: el IIFE sin cerrar, aportes hardcodeados a los semestres 3/5/7 (se sumaban al capital invertido aunque el horizonte fuera menor a 4 años), y un criterio de sostenibilidad que comparaba capital real final contra capital nominal invertido.

---

## 3. Archivos

| Archivo | Qué es |
|---|---|
| `index.html` | La aplicación completa. Corre tres modelos con los mismos datos: CAGR fijo, backtest histórico y Monte Carlo. |
| `CONTEXTO.md` | Este documento. |

**Hay un solo archivo, y se llama `index.html` porque es lo que GitHub Pages sirve por defecto.** No renombrarlo.

Hasta el 1 de agosto de 2026 hubo dos archivos: `index.html` era la v1 (solo el modelo determinístico) e `indexV2.html` agregaba el backtest y el Monte Carlo. Se unificaron en uno. La v1 aportó, y hay que conservar: el KPI grande de retiro sostenible, el gráfico de evolución del capital (nominal vs real), la lectura del estado en dólares ("sobran / faltan / capital agotado en"), las filas de total retirado y los dos multiplicadores, y el aviso al pasar de 30 años. Todo eso está marcado como "CAGR fijo" en la UI, porque son lecturas del modelo determinístico.

---

## 4. El motor

Es el contrato central del proyecto. Antes estaba duplicado en los dos archivos y había que mantener las dos copias sincronizadas a mano; ahora hay una sola. Los tres modelos lo llaman con los mismos datos: lo único que cambia es quién arma el array de retornos y cuántas veces se llama.

```js
simular(cfg, retornos, escala) -> {
  serie, finalNominal, finalReal, totalRetirado, totalAportado, agotado, pAgot
}
```

- `retornos` es un **array de un retorno por semestre**, no un escalar. Esto es lo que permitió agregar el backtest y el Monte Carlo sin reescribir el motor: solo cambia quién arma el array y cuántas veces se llama.
- `escala` multiplica todos los montos de retiro (el base y los ajustes puntuales). Sirve para la búsqueda del retiro sostenible: escala la forma del plan sin deformarla.
- **Las funciones de sostenibilidad devuelven el factor de escala, no un monto.** Hay que multiplicar por `cfg.retBase` (o por 1 si el base es 0). Ya fue un bug una vez; no repetirlo.
- `orden` es una variable global: `"A"` = aporte → crecimiento → retiro (más favorable), `"B"` = aporte → retiro → crecimiento.

### Paso temporal

Todo es **semestral**. Las tasas anuales se parten en mitades geométricas: `(1+r)^0.5 - 1`. Esto vale tanto para el retorno como para la inflación.

### Inflación

`acum` es un array de `n+1` posiciones con el nivel de precios semestre a semestre: `acum[0] = 1` y `acum[i]` son los precios en el semestre `i`. El motor lo usa para dos cosas: indexar el retiro (`base * acum[i]/acum[baseP]`) y deflactar el capital (`cap/acum[i]`). Antes eran dos `Math.pow(1+is, ...)`, que solo funcionaban con inflación fija; con inflación constante el array da exactamente el mismo número, verificado contra los siete valores de referencia.

Esto es lo que permite elegir, con un selector en la pantalla de carga, si el backtest y el Monte Carlo usan **la inflación constante que cargó el usuario** (opción por defecto) o **el IPC real de cada año**. El modelo de CAGR fijo usa siempre la constante. En Monte Carlo el bootstrap sortea **índices de año**, no retornos sueltos, así que el retorno y la inflación que entran a una corrida son los del mismo año: eso conserva la correlación entre inflación alta y retorno malo, que es justamente lo que hace peligrosas a las ventanas de los años setenta.

Un retorno anual histórico se convierte en dos semestres iguales (`aSemestral`). Es un supuesto: en la realidad el año no se reparte parejo. Está documentado en la app y hay que mantenerlo visible si se cambia algo.

---

## 5. Los tres modelos

| Modelo | Cómo arma `retornos` |
|---|---|
| CAGR fijo | `setConstante(cagr, anios)` — el mismo número en los 2n semestres. |
| Backtest histórico | `ventanasHistoricas(anios)` — cada ventana consecutiva de la serie real. Con 97 años y horizonte 20 salen 78 ventanas. |
| Monte Carlo | `setsMonteCarlo(anios, corridas, bloque, seed)` — bootstrap por bloques: toma tramos consecutivos al azar (con wrap-around) para conservar la autocorrelación. |

### Serie de datos

`SERIE` = retornos totales anuales del S&P 500 con dividendos reinvertidos, 1928–2024, en porcentaje. `ANIO0 = 1928`. CAGR implícito de la serie: 9,95%.

**La serie tiene dos tramos y conviene saberlo.** Verificada contra NYU Stern (Damodaran) el 2 de agosto de 2026: los 88 años de 1928 a 2015 coinciden con esa fuente al segundo decimal. Los 9 años de 2016 a 2024 **no** coinciden: son los retornos totales publicados del índice (18,40% en 2020, 28,71% en 2021, −18,11% en 2022, 26,29% en 2023, 25,02% en 2024), y Damodaran, que reconstruye la serie con su propio método, da entre 0,07 y 0,38 puntos menos en cada uno de esos años.

Los valores que están en el archivo son los correctos para este proyecto, porque son los que replica un ETF como SPY. Se midió el efecto de la diferencia: mueve el capital final de las ventanas que llegan hasta 2016–2024 entre 0,2% y 1,5%, y no cambia ninguna conclusión —ni la tasa de éxito, ni el retiro sostenible, ni la mediana, ni cuáles son las peores ventanas—, porque las ventanas críticas (1928 y 1929) están en el tramo que coincide. **No "corregir" esos nueve años hacia los de Damodaran creyendo que están mal.**

`SERIE_CPI` = inflación anual de EE.UU. en el mismo período y en el mismo orden, calculada como variación diciembre a diciembre sobre la serie CPIAUCNS de FRED (Reserva Federal de St. Louis). `SERIE_CPI[k]` es la inflación del año en que el mercado rindió `SERIE[k]`; **el orden tiene que seguir emparejado**. Acumula 18,2x en los 97 años: 3,04% anual.

**Ojo:** son retornos *totales*, no de precio. Circulan tablas de retornos anuales que son de precio (2024 +23,31% en vez de +25,02%). No mezclar las dos.

### Búsqueda del retiro sostenible

`sostenibleNivel(cfg, sets, objetivo, nivel)` hace bisección sobre `escala` buscando el mayor retiro que sobrevive en al menos `nivel` de los casos. La tasa de éxito es monótona decreciente en la escala, así que la bisección es válida.

**Crítico:** los paths (`sets`) se generan **una sola vez** antes de bisecar. Si se regeneraran en cada iteración, el ruido rompería la monotonía y la búsqueda no convergería. Monte Carlo usa semilla fija (`20260801`) con un LCG para que los resultados sean reproducibles.

La bisección hace **22 pasos**, no 40. Sobre un rango que arranca en `[0, hi]` con `hi` de un dígito, 22 pasos dejan un error de menos de un centavo, y la tasa de éxito es una función escalonada cuyo escalón más chico con 1.000 corridas es de una décima de punto: más pasos no mueven el resultado. Verificado: 40 y 22 pasos dan el mismo número, y 22 tarda 45% menos. La búsqueda determinística hace 34 pasos por el mismo motivo.

Si se sube a 5.000 corridas conviene mover el cálculo a un Web Worker.

---

## 6. Criterio de éxito

Una corrida es exitosa si **no se agotó el capital y el capital final nominal llega al objetivo**. Está en la función `exitoso(s, objetivo)`, y **las tres búsquedas de retiro sostenible tienen que usarla**. No alcanza con mirar `finalNominal`: con el objetivo en 0% la comparación se vuelve siempre verdadera y un plan que funde el capital antes de tiempo pasa como sostenible. Era un bug real del modelo determinístico (reportaba $48.000 con los defaults, contra $43.258 que es el valor correcto).

El objetivo es un porcentaje del capital total invertido (nominal), configurable con slider de 0% a 100%. Preservar el 100% nominal es el techo del slider, y 0% significa que se puede consumir todo el capital dentro del horizonte.

**Advertencia que debe seguir visible en la UI:** el objetivo se define en términos nominales, pero preservar el 100% nominal a 20 años con 3% de inflación equivale a conservar solo ~55% del poder adquisitivo. La app muestra las dos lecturas al lado del slider. No sacar eso.

---

## 7. Cosas resueltas que no hay que volver a romper

- **`window.print()` no sirve.** Está bloqueado en vistas incrustadas. Hay un generador de PDF escrito a mano (`construirPDF`) que arma el archivo byte a byte: objetos, xref, offsets. Sin librerías. Validado con un lector real de PDF. Fuentes Helvetica/Helvetica-Bold con WinAnsiEncoding; el texto pasa por `latin1()` que reemplaza caracteres fuera de Latin-1.
- **Slider trabado en iOS.** El `range` necesita `touch-action: none`, si no el arrastre lo captura el scroll de la página. Además el thumb es de 34px y hay un campo numérico y botones −/+ como alternativa. No quitar ninguna de las tres vías.
- **Etiquetas de dos líneas desalineaban los campos.** Resuelto con `display:flex; flex-direction:column` en los items de grid y `margin-top:auto` en el input.
- **Compartir** usa `navigator.share()` con cadena de respaldo: clipboard API → `execCommand` → textarea visible para copiar a mano. Siempre da respuesta visible.
- **Nomenclatura de tasas.** Se muestran las cuatro por separado y con nombre completo: retorno nominal anual, inflación anual, retorno real anual (Fisher), retorno nominal semestral, inflación semestral. Antes decía solo "Tasa semestral" al lado del retorno real y se leía como contradicción. No volver a abreviar.
- **El gráfico de trade-off (retiro sostenible vs objetivo de legado) fue eliminado a pedido.** No reponerlo sin que lo pida.
- **El agotamiento no corta la simulación.** El motor recorre siempre los `n` semestres. Si el capital llega a cero, los períodos siguientes quedan en cero con retiro cero, pero un aporte posterior lo recupera y los retiros se reanudan. `agotado` se marca con el período del **primer** agotamiento y no se limpia nunca, así que el plan sigue siendo un fracaso para `exitoso()` aunque termine con capital. Antes había un `break`: los aportes posteriores se perdían y `totalAportado` los ignoraba, mientras que `invertido` —que define el objetivo de capital final— sí los contaba. Era una comparación contra un objetivo que incluía plata que el modelo nunca había sumado. La tabla pinta en rojo el semestre del agotamiento y los que quedan en cero, no todo lo que viene después.

---

## 8. Limitaciones conocidas (declaradas en la app)

1. **Las ventanas históricas se solapan.** Con 97 años de datos, las ventanas de 20 años independientes son ~5, no 78. El porcentaje describe la historia disponible, no es una probabilidad. La UI dice "76 de 78 ventanas", no "97,4% de probabilidad".

1bis. **Las bandas del abanico no son caminos.** Se calcula el percentil columna por columna, semestre a semestre, así que la línea de la mediana no es el recorrido de ninguna ventana real. Está aclarado abajo del gráfico; no sacarlo.
2. **La inflación es constante salvo que se pida lo contrario.** Por defecto los tres modelos usan la inflación que carga el usuario, lo que aísla el riesgo de secuencia de retornos. El selector "Inflación en el backtest y el Monte Carlo" permite cambiarla por el IPC real de cada año, que es el escenario más realista y más exigente: con la configuración de referencia, el retiro sostenible histórico al 90% baja de 21.165 a 19.227.
3. **El reparto de un retorno anual en dos semestres iguales** es una simplificación.
4. **No hay costos, impuestos ni comisiones.**

---

## 9. Cómo verificar cambios

```bash
# sintaxis del JS embebido
node -e "const s=require('fs').readFileSync('index.html','utf8');
         new Function(s.match(/<script>([\s\S]*)<\/script>/)[1]); console.log('OK')"
```

Para probar el motor sin DOM: extraer del archivo el bloque entre `var ANIO0` y `/* ---------- formato`, el bloque de `function percentil` hasta `/* ---------- filas dinámicas`, más el bloque entre `/* ===================== MOTOR` y `/* ---------- lectura de la UI`, declarar `var orden="A"` y evaluarlo.

Conviene además comprobar que todo `$("id")` usado en el JS tenga su elemento en el HTML: es el error más fácil de cometer al mover bloques de una vista a la otra.

### Validación externa: la regla del 4%

El test de referencia de abajo compara el motor consigo mismo, así que no detecta un error conceptual. Para eso está esta prueba, que lo compara contra un resultado publicado por terceros.

Caso canónico del Trinity Study: capital 1.000.000, retiro de 40.000 anuales (20.000 semestrales) ajustados por inflación desde el primer semestre, 30 años, sin aportes, objetivo 0% (éxito = no agotarse), cartera 100% acciones. Con 97 años de datos salen 68 ventanas.

| | Resultado |
|---|---|
| Orden A, IPC real de cada año | 66 de 68 = **97,1%** |
| Orden A, inflación constante 3% | 65 de 68 = **95,6%** |
| Orden B, IPC real | 63 de 68 = 92,6% |

La literatura reporta entre 95% y 98% de éxito para 100% acciones a 30 años con retiro del 4%. El motor cae dentro de ese rango con el orden A, que es el de la app. Barrido de tasas con orden B: hasta 3,5% sobreviven las 68 ventanas, en 4% empieza a fallar, en 5% cae a 76,5%.

**Si esta prueba se aleja del rango 95–98%, hay un error conceptual en el motor**, aunque el test de referencia siga dando bien.

### Contabilidad

3.000 escenarios al azar (horizontes de 1 a 40 años, inflación de 0 a 15%, ambos órdenes, con y sin IPC real, 0-3 aportes, 0-2 ajustes) contrastados contra una implementación independiente escrita desde esta especificación: cero discrepancias, diferencia máxima 4,4e-14. La identidad `final = inicial + aportes + ganancias − retiros` cierra con error de 1,3e-16 relativo al dinero movido, que es el límite de precisión de un número decimal en JavaScript.

Al medir ese descuadre hay que normalizar por **el dinero total movido**, no por el capital final: las corridas que se agotan terminan en cero y dividir por cero infla el error hasta hacerlo parecer un problema.

**Test de regresión obligatorio:** correr la configuración de referencia de acá abajo y comparar los siete números. Antes este test comparaba la v1 contra la v2; con un solo archivo, la referencia es esta tabla.

### Resultado de referencia

**Los valores por defecto de la app**, sin tocar nada: capital 100.000, 20 años, CAGR 9%, inflación 3%, aportes de 10.000 en años 2/3/4 (semestre 1), retiro 2.400 semestral desde año 5 semestre 1, objetivo 100% nominal, orden A, inflación constante.

| | Con los defaults | La misma configuración ×5 |
|---|---|---|
| Capital final CAGR fijo | 507.482 | 2.537.410 |
| Sostenible CAGR fijo | 7.050 | 35.251 |
| Sostenible histórico al 90% | 4.233 | 21.165 |
| Sostenible Monte Carlo al 90% | 3.031 | 15.154 |
| Éxito histórico | 76 de 78 (97,4%) | igual |
| Éxito Monte Carlo | 92,4% | igual |
| Peores arranques | 1929, 1928, 1930, 1999, 2000 | igual |

**El modelo es lineal en los montos**: multiplicar capital, aportes y retiro por la misma constante multiplica todos los resultados en dinero por esa constante, y deja las tasas de éxito idénticas. La columna de la derecha es la configuración histórica de referencia del proyecto (capital 500.000, aportes 50.000, retiro 12.000) y sirve de comprobación cruzada: si los defaults ×5 no dan esa columna, el motor dejó de ser lineal y algo se rompió.

Con los defaults pero el objetivo en **0%**, el sostenible con CAGR fijo tiene que dar **8.652** (43.258 en la escala ×5) y no agotar el capital. Ese caso es el que detecta si alguien volvió a sacar el chequeo de agotamiento de la búsqueda (ver sección 6).

Con los defaults pero el selector de inflación en **IPC real de cada año**: éxito histórico 76 de 78, éxito Monte Carlo 93,0%, sostenible histórico al 90% **3.845** (19.227 ×5), sostenible Monte Carlo al 90% **3.040** (15.201 ×5). El CAGR fijo no se mueve, porque ese modelo siempre usa la inflación constante.

Y para el agotamiento con recuperación (sección 7): capital 100.000, retiro 40.000 desde el año 1 semestre 1, un único aporte de 2.000.000 en el año 10 semestre 2, 20 años, CAGR 9%, inflación 3%, orden A. El capital tiene que agotarse en **Año 2 / S1**, quedar en cero hasta el Año 10 / S1, y terminar en **2.929.876** con `agotado` en `true`. Si el capital final da 0, alguien repuso el `break`.

Los 40% de diferencia entre el CAGR fijo y el histórico son el punto central del proyecto: es el costo de planificar contra la peor secuencia posible en vez de contra el promedio.

---

## 10. Comparación de escenarios

El botón "Guardar este escenario para comparar", en la pantalla de resultados, agrega el escenario a una tabla comparativa que se muestra en las dos vistas: columnas los escenarios, filas los conceptos (capital, horizonte, retiro planteado, los tres retiros sostenibles, las dos tasas de éxito y el capital final). Tope de 6.

Se guarda **solo el resumen**, no la simulación entera: alcanza para comparar y entra en `localStorage` sin problema. Si el navegador tiene `localStorage` bloqueado (Safari en navegación privada), cae a una variable en memoria y la comparación sigue funcionando dentro de la sesión, aunque se pierda al cerrar. Clave: `carg_escenarios_v1`. Si se cambian los campos del resumen, subir la versión de la clave para no leer datos viejos con forma distinta.

## 11. Ideas pendientes (ninguna aprobada)

- Pasar el escenario por la querystring, para compartir un caso por link.
- Web Worker para el cálculo, si crece la cantidad de corridas.
- Exportar CSV de la tabla semestral.
- Incluir los gráficos en el PDF (hoy no van; habría que rasterizar el canvas y el archivo se triplicaría).

---

## 12. Marco conceptual

Es el eje del proyecto:

- El S&P 500 rindió entre 10% y 12% en casi cualquier ventana larga. La última década (15,5%) es la excepción, no la línea de base.
- Retorno real por Fisher: `(1+nominal)/(1+inflación) - 1`. No es resta simple.
- Gastar el rendimiento nominal completo mantiene el capital intacto en papel y lo funde en términos reales: 2.000.000 al 9% gastando 180.000 al año conserva 824.000 de poder de compra a 30 años.
- **El promedio histórico no sirve para dimensionar un retiro.** La regla del 4% no salió de un promedio: salió de probar todas las ventanas posibles y quedarse con la tasa que sobrevivió incluso a las peores. Quien se retiró en 1929, 1966 o 2000 vio caídas del 40-50% en los primeros años.
- Calcular con el promedio funciona si el futuro se parece al pasado reciente; la regla del 4% funciona si el futuro puede parecerse al peor pasado. **La diferencia entre ambos números es el precio del seguro** — y es exactamente lo que la app cuantifica.

Todo esto son datos históricos y reglas de planificación de uso general. No es asesoramiento financiero.
