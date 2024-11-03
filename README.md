DIVIDENDOS DIAGRAMA SENCILLO:




DIVIDENDOS COBBRADOS EN DAX que permite filtrar por Año/Trimestre, Fecha seleccionada concreta y Rango de Fechas, mejorando así la granularidad de la medida:

```
DIVIDENDOS COBRADOS MEJORA DE GRANULARIDAD DEFINITIVO = 

--VAR FechaSeleccionada = SELECTEDVALUE('CALENDARIO'[Date])
--VAR TrimestreSeleccionado = SELECTEDVALUE('CALENDARIO'[AñoTrimestre])

-- Definir los rangos de fechas tipo XX-XX-XXXX a YY-YY-YYYY:
VAR FechaInicioRango = MIN('CALENDARIO'[Date])  -- La fecha mínima seleccionada en el calendario
VAR FechaFinRango = MAX('CALENDARIO'[Date])    -- La fecha máxima seleccionada en el calendario

VAR TrimestreSeleccionado = SELECTEDVALUE('CALENDARIO'[AñoTrimestre])

-- Extraemos el año y el trimestre de 'TrimestreSeleccionado'
VAR AnioSeleccionado = VALUE(LEFT(TrimestreSeleccionado, 4))
VAR Trimestre = VALUE(RIGHT(TrimestreSeleccionado, 1))

-- Calculamos las fechas de inicio y fin del trimestre
VAR FechaInicioTrimestre = DATE(AnioSeleccionado, (Trimestre - 1) * 3 + 1, 1)
VAR FechaFinTrimestre = EOMONTH(FechaInicioTrimestre, 2)


                                                                                -- FECHA EXACTA:

-- IMPORTANTE ESTE CASO DEBIDO AL ALL(). Este caso va a mostrar TODOS los dividendos potenciales a cobrar desde una fecha seleccionada en adelante, siempre
-- y cuando no se haya sobrepasado la fecha_cobro. Por ejemplo, serviría para tener un histórico de dividendos cobrados y poder visualizar cómo estos han 
-- evolucionado o no.
-- Filtramos los dividendos DESDE la fecha seleccionada. Esto calculará los dividendos COBRADOS HASTA la Fecha seleccionada:
VAR DividendosFiltradosCobradosFechaExacta =
    FILTER(
            ALL('FINANZAS DIVIDENDOS'), -- ALL() soluciona el problema de cuando seleccionamos una fecha_cobro que no está en la tabla 'FINANZAS DIVIDENDOS'
            FechaInicioRango = FechaFinRango &&
            'FINANZAS DIVIDENDOS'[fecha_cobro] <= FechaInicioRango
    )

/* Antes teníamos esto, pero el SUMMARIZE() que funcionaba bien para todas las fechas fecha_cobro. El problema es que a veces se obtienen resultados inesperados
dado que el motor de DAX utiliza clustering con SUMMARIZE(); por lo que funciona distinto al GROUP BY() de SQL:
VAR DividendosAgrupados =
    SUMMARIZE( 
        FILTER(
            'FINANZAS DIVIDENDOS', 
            'FINANZAS DIVIDENDOS'[fecha_cobro] >= FechaInicioTrimestre &&
            'FINANZAS DIVIDENDOS'[fecha_cobro] <= FechaInicioTrimestre
        ),
        'FINANZAS DIVIDENDOS'[isin],
        'FINANZAS DIVIDENDOS'[fecha_ex_div],
        "DividendoCobradoPorAccion", SUM('FINANZAS DIVIDENDOS'[dividendo_eur])
    )
*/

-- Sumamos los dividendos cobrados considerando la cantidad en la cartera
VAR CalculoDividendosCobradosFechaExacta =
    SUMX(
        DividendosFiltradosCobradosFechaExacta,
        VAR ISINActual = 'FINANZAS DIVIDENDOS'[isin]
        VAR FechaExDivActual = 'FINANZAS DIVIDENDOS'[fecha_ex_div]
        VAR DividendoPorAccion = 'FINANZAS DIVIDENDOS'[dividendo_eur]
        
        VAR CantidadComprada =
            CALCULATE(
                SUM('FINANZAS CARTERA_REAL_NO_FIFO'[cantidad]),
                'FINANZAS CARTERA_REAL_NO_FIFO'[isin] = ISINActual &&
                'FINANZAS CARTERA_REAL_NO_FIFO'[fecha] < FechaExDivActual
            )

        RETURN
            CantidadComprada * DividendoPorAccion
    )

                                                                                -- RANGO DE FECHAS:

-- Filtramos los dividendos dentro del rango de fechas seleccionado:
VAR DividendosFiltradosCobradosRango =
    FILTER(
        'FINANZAS DIVIDENDOS',
        'FINANZAS DIVIDENDOS'[fecha_cobro] >= FechaInicioRango &&
        'FINANZAS DIVIDENDOS'[fecha_cobro] <= FechaFinRango
    )

/* Antes teníamos esto, pero el SUMMARIZE() me devolvía una tabla vacía cuando no debería ser así para T2 y T3 de 2024:
VAR DividendosAgrupados =
    SUMMARIZE( 
        FILTER(
            'FINANZAS DIVIDENDOS', 
            'FINANZAS DIVIDENDOS'[fecha_cobro] >= FechaInicioTrimestre &&
            'FINANZAS DIVIDENDOS'[fecha_cobro] <= FechaInicioTrimestre
        ),
        'FINANZAS DIVIDENDOS'[isin],
        'FINANZAS DIVIDENDOS'[fecha_ex_div],
        "DividendoCobradoPorAccion", SUM('FINANZAS DIVIDENDOS'[dividendo_eur])
    )
*/

-- Sumamos los dividendos cobrados considerando la cantidad en la cartera
VAR CalculoDividendosCobradosRango =
    SUMX(
        DividendosFiltradosCobradosRango,
        VAR ISINActual = 'FINANZAS DIVIDENDOS'[isin]
        VAR FechaExDivActual = 'FINANZAS DIVIDENDOS'[fecha_ex_div]
        VAR DividendoPorAccion = 'FINANZAS DIVIDENDOS'[dividendo_eur]
        
        VAR CantidadComprada =
            CALCULATE(
                SUM('FINANZAS CARTERA_REAL_NO_FIFO'[cantidad]),
                'FINANZAS CARTERA_REAL_NO_FIFO'[isin] = ISINActual &&
                'FINANZAS CARTERA_REAL_NO_FIFO'[fecha] < FechaExDivActual
            )

        RETURN
            CantidadComprada * DividendoPorAccion
    )


                                                                                -- AÑO/TRIMESTRE:

-- Filtramos los dividendos dentro del trimestre seleccionado
VAR DividendosFiltradosTrimestre =
    FILTER(
        'FINANZAS DIVIDENDOS',
        'FINANZAS DIVIDENDOS'[fecha_cobro] >= FechaInicioTrimestre &&
        'FINANZAS DIVIDENDOS'[fecha_cobro] <= FechaFinTrimestre
    )

/* Antes teníamos esto, pero el SUMMARIZE() me devolvía una tabla vacía cuando no debería ser así para T2 y T3 de 2024:
VAR DividendosAgrupados =
    SUMMARIZE( 
        FILTER(
            'FINANZAS DIVIDENDOS', 
            'FINANZAS DIVIDENDOS'[fecha_cobro] >= FechaInicioTrimestre &&
            'FINANZAS DIVIDENDOS'[fecha_cobro] <= FechaInicioTrimestre
        ),
        'FINANZAS DIVIDENDOS'[isin],
        'FINANZAS DIVIDENDOS'[fecha_ex_div],
        "DividendoCobradoPorAccion", SUM('FINANZAS DIVIDENDOS'[dividendo_eur])
    )
*/

-- Sumamos los dividendos cobrados considerando la cantidad en la cartera
VAR CalculoDividendosCobradosTrimestre =
    SUMX(
        DividendosFiltradosTrimestre,
        VAR ISINActual = 'FINANZAS DIVIDENDOS'[isin]
        VAR FechaExDivActual = 'FINANZAS DIVIDENDOS'[fecha_ex_div]
        VAR DividendoPorAccion = 'FINANZAS DIVIDENDOS'[dividendo_eur]
        
        VAR CantidadComprada =
            CALCULATE(
                SUM('FINANZAS CARTERA_REAL_NO_FIFO'[cantidad]),
                'FINANZAS CARTERA_REAL_NO_FIFO'[isin] = ISINActual &&
                'FINANZAS CARTERA_REAL_NO_FIFO'[fecha] < FechaExDivActual
            )

        RETURN
            CantidadComprada * DividendoPorAccion
    )



-- Resultado final con SWITCH basado en las selecciones del usuario
RETURN
    SWITCH(
        TRUE(),
        FechaInicioRango = FechaFinRango, CalculoDividendosCobradosFechaExacta,
        FechaInicioRango <> FechaFinRango, CalculoDividendosCobradosRango,
        NOT(ISBLANK(TrimestreSeleccionado)), CalculoDividendosCobradosTrimestre,
        0
    )
```
