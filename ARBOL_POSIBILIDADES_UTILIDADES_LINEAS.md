# Árbol de posibilidades (Pasarela_ARTEZ) para invocar selectivamente `UtilidadesCreacionLineasDetalleFlujo`

Este documento define un **árbol de decisión en .NET (VB)** para traducir XML de Procedimiento en llamadas selectivas a `UtilidadesCreacionLineasDetalleFlujo`.

## 1) Esqueleto de orquestación

```vbnet
Public Function GenerarLineasDetalleDesdeXml(proc As Procedimiento,
                                             catalogo As CatalogoAcciones,
                                             baseDatosInfra As String,
                                             baseDatosDestino As String) As List(Of LineaDetalle)

    Dim lineas As New List(Of LineaDetalle)
    Dim flowOrder As Integer = 0
    Dim contadorReferencias As Integer = 0
    Dim contadorJumps As Integer = 0
    Dim contadorUnions As Integer = 0

    ' 1) Inicio de flujo
    lineas.AddRange(UtilidadesCreacionLineasDetalleFlujo.ObtenerLineasDetalleInicioFlujo())

    ' 2) Inicio tramitación
    flowOrder += 100
    lineas.AddRange(UtilidadesCreacionLineasDetalleFlujo.ObtenerLineasDetalle_Accion_LABEL(
        flowOrder:=flowOrder,
        path:="INICIO_TRAMITACION",
        tieneParametro_STATE:=True,
        valorParametro_STATE:="INICIO_TRAMITACION",
        valorParametro_SUBMIT:="0",
        tipoAgrupacion:=String.Empty,
        level:=1,
        comentario:=String.Empty))

    flowOrder += 100
    contadorReferencias += 1
    lineas.AddRange(UtilidadesCreacionLineasDetalleFlujo.ObtenerLineasDetalle_Accion_APIINIENDEXT(
        identificadorFlujo:=proc.DatosGenerales.IdentificadorFlujo,
        flowOrder:=flowOrder,
        path:="INICIAR_TRAMITACION",
        baseDatosAccion:=baseDatosInfra,
        accion:="INICIAR_TRAMITACION",
        aplicacion:=Constantes.APLICACION,
        contadorReferencia:=contadorReferencias,
        level:=2,
        diferido:=False,
        idnt_Elemento:=String.Empty,
        sistemaFuncional:=String.Empty,
        grupoUsuarios:=String.Empty,
        usuario:=String.Empty,
        baseDatosRetorno:=baseDatosDestino,
        pathRetorno:=ObtenerPathRetornoInicio(proc, catalogo),
        comentario:=String.Empty,
        parametrosEspecificos:=Nothing,
        habilitado:=False))

    If proc.PasosIniciarTramitacion IsNot Nothing AndAlso proc.PasosIniciarTramitacion.Count > 0 Then
        For Each paso In proc.PasosIniciarTramitacion
            flowOrder += 100
            lineas.AddRange(ResolverPaso(paso, TipoTratamientoPasos.InicioTramitacion, proc, Nothing, catalogo,
                                         flowOrder, contadorReferencias, contadorJumps, baseDatosInfra, baseDatosDestino))
        Next
    End If

    ' 3) Trámites
    For Each tr In proc.Tramites
        flowOrder += 100
        lineas.AddRange(UtilidadesCreacionLineasDetalleFlujo.ObtenerLineasDetalle_Accion_LABEL(
            flowOrder:=flowOrder,
            path:=tr.DatosGenerales.ObtenerNombrePathTramite,
            tieneParametro_STATE:=True,
            valorParametro_STATE:=tr.DatosGenerales.NombreEnFlujo,
            valorParametro_SUBMIT:="0",
            tipoAgrupacion:=If(tr.DatosGenerales.EsBloqueTramitacionComun, Constantes.TIPO_AGRUPACION_BLOQUE_TRAMITACION_COMUN, String.Empty),
            level:=1,
            comentario:=tr.DatosGenerales.Nombre))

        lineas.AddRange(ResolverTramite(tr, proc, catalogo, flowOrder, contadorReferencias, contadorJumps, contadorUnions, baseDatosInfra, baseDatosDestino))
    Next

    ' 4) Fin
    lineas.AddRange(UtilidadesCreacionLineasDetalleFlujo.ObtenerLineasDetalleFinalizacionTramitacion(proc.DatosGenerales.IdentificadorFlujo, baseDatosInfra, baseDatosDestino))
    lineas.AddRange(UtilidadesCreacionLineasDetalleFlujo.ObtenerLineasDetalleFinFlujo(proc.DatosGenerales.IdentificadorFlujo))

    Return lineas
End Function
```

---

## 2) Árbol por tipo de Trámite

```vbnet
Private Function ResolverTramite(tr As Tramite,
                                 proc As Procedimiento,
                                 catalogo As CatalogoAcciones,
                                 ByRef flowOrder As Integer,
                                 ByRef contadorReferencias As Integer,
                                 ByRef contadorJumps As Integer,
                                 ByRef contadorUnions As Integer,
                                 baseDatosInfra As String,
                                 baseDatosDestino As String) As List(Of LineaDetalle)
    Dim out As New List(Of LineaDetalle)

    ' A) Acciones aisladas (ramas paralelas)
    If tr.AccionesAisladas IsNot Nothing AndAlso tr.AccionesAisladas.Count > 0 Then
        For Each aa In tr.AccionesAisladas
            out.AddRange(aa.ObtenerLineasDetalle(If(tr.EsTramiteARTEZ, TipoTratamientoPasos.TramiteARTEZ, TipoTratamientoPasos.TramiteFicticio),
                                                 proc.DatosGenerales.IdentificadorFlujo, flowOrder, contadorReferencias,
                                                 Constantes.APLICACION, tr.DatosGenerales, tr.DatosARTEZ, catalogo, tr.Plazo))
        Next
    End If

    ' B) Trámite técnico
    If tr.DatosGenerales.TipoTramite = TipoTramite.UnionRamas Then
        flowOrder += 100
        contadorUnions += 1
        out.AddRange(UtilidadesCreacionLineasDetalleFlujo.ObtenerLineasDetalle_Accion_UNION(
            flowOrder:=flowOrder,
            path:=String.Format("UNION_{0:0000}", contadorUnions),
            numeroRamasParalelas:=2,
            level:=2,
            comentario:=String.Empty))

    ElseIf tr.DatosGenerales.TipoTramite = TipoTramite.Ficticio Then
        ' Sin UNION, solo pasos/salidas
    Else
        ' C) Trámite ARTEZ: Plazo + Crear/Ejecutar tarea + Parada + Finalizar + Salidas + Caducar

        If tr.TienePlazo Then
            flowOrder += 100 : contadorReferencias += 1
            out.AddRange(UtilidadesCreacionLineasDetalleFlujo.ObtenerLineasDetalle_Accion_APIINIENDEXT(...)) ' OBTENER_PLAZO

            flowOrder += 100
            out.AddRange(UtilidadesCreacionLineasDetalleFlujo.ObtenerLineasDetalle_Accion_IF(...)) ' caducidad

            flowOrder += 100
            out.AddRange(UtilidadesCreacionLineasDetalleFlujo.ObtenerLineasDetalle_Accion_ALERT(...))
        End If

        flowOrder += 100 : contadorReferencias += 1
        out.AddRange(UtilidadesCreacionLineasDetalleFlujo.ObtenerLineasDetalle_Accion_APIINIENDEXT(...)) ' CREAR_TAREA

        If RequiereFormParada(tr) Then
            flowOrder += 100
            out.AddRange(UtilidadesCreacionLineasDetalleFlujo.ObtenerLineasDetalle_Accion_FORM(...))
        End If

        If RequiereEjecutarTareaAutomatica(tr) Then
            flowOrder += 100 : contadorReferencias += 1
            out.AddRange(UtilidadesCreacionLineasDetalleFlujo.ObtenerLineasDetalle_Accion_APIINIENDEXT(...)) ' EJECUTAR_TAREA_AUTOMATICA
        End If
    End If

    ' D) Pasos del trámite
    If tr.Pasos IsNot Nothing AndAlso tr.Pasos.Count > 0 Then
        For Each paso In tr.Pasos
            flowOrder += 100
            out.AddRange(ResolverPaso(paso, If(tr.EsTramiteARTEZ, TipoTratamientoPasos.TramiteARTEZ, TipoTratamientoPasos.TramiteFicticio),
                                      proc, tr, catalogo, flowOrder, contadorReferencias, contadorJumps, baseDatosInfra, baseDatosDestino))
        Next
    End If

    ' E) Salidas
    out.AddRange(ResolverSalidas(tr, proc, flowOrder, contadorJumps))

    ' F) Caducidad
    If tr.TienePlazo Then
        flowOrder += 100 : contadorReferencias += 1
        out.AddRange(UtilidadesCreacionLineasDetalleFlujo.ObtenerLineasDetalle_Accion_APIINIENDEXT(...)) ' CADUCAR_TAREA

        flowOrder += 100
        out.AddRange(UtilidadesCreacionLineasDetalleFlujo.ObtenerLineasDetalle_Accion_CANCEL(...))

        flowOrder += 100 : contadorJumps += 1
        out.AddRange(UtilidadesCreacionLineasDetalleFlujo.ObtenerLineasDetalle_Accion_JUMP(...))
    End If

    Return out
End Function
```

---

## 3) Árbol por tipo de Paso

```vbnet
Private Function ResolverPaso(paso As Paso,
                              tipo As TipoTratamientoPasos,
                              proc As Procedimiento,
                              tr As Tramite,
                              catalogo As CatalogoAcciones,
                              ByRef flowOrder As Integer,
                              ByRef contadorReferencias As Integer,
                              ByRef contadorJumps As Integer,
                              baseDatosInfra As String,
                              baseDatosDestino As String) As List(Of LineaDetalle)
    Dim out As New List(Of LineaDetalle)

    Select Case paso.Tipo
        Case TipoPaso.Accion
            contadorReferencias += 1
            out.AddRange(UtilidadesCreacionLineasDetalleFlujo.ObtenerLineasDetalle_Accion_APIINIENDEXT(...))

        Case TipoPaso.InicializacionVariables
            out.AddRange(UtilidadesCreacionLineasDetalleFlujo.ObtenerLineasDetalle_Accion_LETVAR(...))

        Case TipoPaso.Evaluacion
            out.AddRange(UtilidadesCreacionLineasDetalleFlujo.ObtenerLineasDetalle_Accion_IF(...))

        Case TipoPaso.Salto
            contadorJumps += 1
            out.AddRange(UtilidadesCreacionLineasDetalleFlujo.ObtenerLineasDetalle_Accion_JUMP(...))

        Case TipoPaso.Parada
            out.AddRange(UtilidadesCreacionLineasDetalleFlujo.ObtenerLineasDetalle_Accion_FORM(...))
    End Select

    Return out
End Function
```

---

## 4) Árbol por Salidas

```vbnet
Private Function ResolverSalidas(tr As Tramite,
                                 proc As Procedimiento,
                                 ByRef flowOrder As Integer,
                                 ByRef contadorJumps As Integer) As List(Of LineaDetalle)
    Dim out As New List(Of LineaDetalle)

    If tr.Salidas Is Nothing OrElse tr.Salidas.Count = 0 Then
        Return out
    End If

    If tr.Salidas.Count = 1 Then
        Dim s = tr.Salidas(0)
        If s.Condicionada() Then
            flowOrder += 100
            out.AddRange(UtilidadesCreacionLineasDetalleFlujo.ObtenerLineasDetalle_Accion_LABEL(... SUBMIT:=IifCondiciones(s) ...))

            flowOrder += 100 : contadorJumps += 1
            out.AddRange(UtilidadesCreacionLineasDetalleFlujo.ObtenerLineasDetalle_Accion_JUMP(...))

            flowOrder += 100
            out.AddRange(UtilidadesCreacionLineasDetalleFlujo.ObtenerLineasDetalle_Accion_ENDPROC(...))
        Else
            ' Si aplica salto directo
            flowOrder += 100 : contadorJumps += 1
            out.AddRange(UtilidadesCreacionLineasDetalleFlujo.ObtenerLineasDetalle_Accion_JUMP(...))
        End If

    Else
        Dim idx As Integer = 0
        For Each s In tr.Salidas
            idx += 1
            flowOrder += 100
            out.AddRange(UtilidadesCreacionLineasDetalleFlujo.ObtenerLineasDetalle_Accion_LABEL(...
                valorParametro_SUBMIT:=If(s.Condicionada(), IifCondiciones(s), "FLY") ...))

            flowOrder += 100 : contadorJumps += 1
            out.AddRange(UtilidadesCreacionLineasDetalleFlujo.ObtenerLineasDetalle_Accion_JUMP(...))
        Next

        flowOrder += 100
        out.AddRange(UtilidadesCreacionLineasDetalleFlujo.ObtenerLineasDetalle_Accion_ENDPROC(...))
    End If

    Return out
End Function
```

---

## 5) Mapa “si encuentra X en XML, llama Y”

- `<Paso><Tipo>ACCION` -> `ObtenerLineasDetalle_Accion_APIINIENDEXT`
- `<Paso><Tipo>INICIALIZACION_VARIABLES` -> `ObtenerLineasDetalle_Accion_LETVAR`
- `<Paso><Tipo>EVALUACION` -> `ObtenerLineasDetalle_Accion_IF`
- `<Paso><Tipo>SALTO` -> `ObtenerLineasDetalle_Accion_JUMP`
- `<Paso><Tipo>PARADA` -> `ObtenerLineasDetalle_Accion_FORM`
- `<Tramite Tipo="UNION_RAMOS/FICTICIO">` técnico -> `..._UNION` (si UNION) + pasos/salidas
- `<Plazo>` -> `APIINIENDEXT(OBTENER_PLAZO)` + `IF` + `ALERT` + (caducidad) `APIINIENDEXT(CADUCAR)` + `CANCEL` + `JUMP`
- múltiples `<Salida>` -> `LABEL + JUMP` por salida + `ENDPROC`
- cierre de procedimiento -> `ObtenerLineasDetalleFinalizacionTramitacion` + `ObtenerLineasDetalleFinFlujo`

