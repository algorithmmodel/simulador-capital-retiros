# Simulador de capital y retiros

Calculadora de escenarios de retiro. Responde dos preguntas sobre un plan de inversión con aportes y retiros semestrales:

1. ¿Cuánto se puede retirar por semestre sin bajar del capital final que se quiere dejar?
2. ¿Qué probabilidad hay de que ese plan sobreviva?

Corre **tres modelos con los mismos datos** y muestra los resultados juntos:

| Modelo | Qué supone |
|---|---|
| **CAGR fijo** | El mercado rinde exactamente lo mismo todos los años. |
| **Backtest histórico** | Cada ventana consecutiva del S&P 500 entre 1928 y 2024 (78 ventanas de 20 años). |
| **Monte Carlo** | Bootstrap por bloques: tramos consecutivos de la historia real, sorteados. |

La diferencia entre el primero y los otros dos es el punto del proyecto. Con la configuración por defecto, el modelo de CAGR fijo permite retirar un 40% más que el histórico exigiendo 90% de éxito. Esa diferencia es el costo de planificar contra la peor secuencia posible en vez de contra el promedio.

## Uso

Abrir `index.html`. No hay nada que instalar.

Es un archivo HTML autocontenido, sin dependencias, sin CDN y sin compilación. Funciona abierto desde el disco o publicado en cualquier servidor estático. En iPhone se puede agregar a la pantalla de inicio y queda como una app.

## Qué hace

- Capital inicial, aportes de capital nuevo en cualquier semestre, retiros indexados por inflación y ajustes puntuales del retiro.
- Objetivo de capital final como porcentaje de lo invertido, con la lectura en poder de compra al lado (preservar el 100% nominal a 20 años con 3% de inflación equivale a conservar ~55% del poder adquisitivo).
- Inflación constante o **el IPC real de cada año** en el backtest y el Monte Carlo.
- Gráfico de evolución del capital y abanico de resultados históricos.
- Comparación de hasta 6 escenarios guardados, lado a lado.
- Exportación a PDF generada a mano, sin librerías.
- Tabla semestre a semestre y detalle de las peores ventanas históricas.

## Datos

- **Retornos:** S&P 500 con dividendos reinvertidos, retornos totales anuales 1928–2024. Los años 1928–2015 coinciden exactamente con la serie de NYU Stern (Damodaran); 2016–2024 son los retornos totales publicados del índice, que difieren de la reconstrucción de Damodaran en menos de 0,4 puntos por año.
- **Inflación:** IPC de EE.UU., variación diciembre a diciembre, mismo período. Calculada sobre la serie `CPIAUCNS` de FRED (Reserva Federal de St. Louis).

## Limitaciones

Están declaradas dentro de la aplicación, y conviene leerlas:

1. Las ventanas históricas **se solapan**. Con 97 años de datos, las ventanas de 20 años realmente independientes son unas 5, no 78. El porcentaje describe la historia disponible; no es una probabilidad.
2. Las bandas del abanico son percentiles semestre a semestre, no el recorrido de ninguna ventana en particular.
3. Un retorno anual se reparte en dos semestres iguales. En la realidad el año no se reparte parejo.
4. No hay costos, impuestos ni comisiones.

## Accesibilidad

Diseñado para baja visión: fondo claro, alto contraste, tipografía grande, controles de 48 px o más. El color nunca es la única señal — las líneas de los gráficos se distinguen además por el trazo.

## Documentación técnica

`CONTEXTO.md` documenta el motor, los criterios de éxito, las decisiones tomadas y los valores de referencia para verificar cambios.

## Aviso

Datos históricos y reglas de planificación de uso general. **No es asesoramiento financiero.** El rendimiento pasado no predice el futuro.

## Licencia

MIT.
