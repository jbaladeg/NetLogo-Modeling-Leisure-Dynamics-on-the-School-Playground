# Explicación del Código de NetLogo: Modelo de Interacción Social

Esta sección detalla la implementación técnica del modelo en **NetLogo**, vinculando las reglas de programación con los conceptos de **Sistemas Complejos** e **Inteligencia Colectiva**.

## 1. Estructura de Datos e Indicadores Globales

Se definen variables globales para monitorizar el estado del sistema en tiempo real. Estas variables permiten observar la **emergencia de patrones** a partir de las interacciones individuales. Las variables de las "turtles" gestionan el estado interno de cada agente (su género y su propósito de movimiento).

```netlogo
globals [
  interaccion-global
  interaccion-gris
  interaccion-grada
  interaccion-amarilla
  total-en-fútbol
  total-en-grada
  pct-reposo
  pct-jugando
]

turtles-own [
  tipo               ; "nino" o "nina"
  tiempo-actividad   ; contador de tiempo en una zona
  objetivo-x         ; coordenada x del destino
  objetivo-y         ; coordenada y del destino
]
```

## 2. Configuración del Entorno (Setup)

El procedimiento `setup-espacio` construye la topología del patio. Se han implementado restricciones espaciales para garantizar la validez del modelo:

*   **Campo de fútbol (Verde):** Es opcional (`tamano-campo = 0`). Los tamaños disponibles son 10 (proporción media) o 15 (gran parte del patio).
*   **La Grada (Marrón):** Zona fija de reposo en la parte superior.
*   **Zonas de Interés (Amarillo):** Se distribuyen mediante un algoritmo de búsqueda que evita el solapamiento y previene el efecto "wrapping" en los bordes.

```netlogo
to setup
  clear-all
  setup-espacio
  crear-agentes
  reset-ticks
end

to setup-espacio
  ask patches [ set pcolor gray + 2 ]
  
  ;; 1. La Grada (Marrón) - Parte superior
  ask patches with [pycor > max-pycor - 3] [ set pcolor brown + 1 ]
  
  ;; 2. Campo de fútbol (Verde)
  if tamano-campo > 0 [
    let ancho tamano-campo
    let alto tamano-campo * 0.66
    ask patches with [
      pxcor >= desplazamiento-campo-ejex - ancho and pxcor <= desplazamiento-campo-ejex + ancho and
      pycor >= desplazamiento-campo-ejey - alto and pycor <= desplazamiento-campo-ejey + alto
    ] [ set pcolor green ]
  ]

  ;; 3. Zonas de Interés (Amarillo) - Ubicación SEGURA
  if num-zonas-interes > 0 [
    let parches-aptos patches with [
      pcolor = gray + 2 and
      pxcor > min-pxcor + 4 and pxcor < max-pxcor - 4 and
      pycor > min-pycor + 4 and pycor < max-pycor - 7 and
      all? patches in-radius 5 [ pcolor = gray + 2 ]
    ]
    ; ... (proceso de selección de centros aleatorios sin solapamiento)
  ]
end
```

## 3. Creación de Agentes y Comportamientos

Los agentes se generan con forma humana y colores diferenciados según el género (socialización de género). Cada agente inicia sin actividad pero con una predisposición conductual definida por el algoritmo de toma de decisiones.

```netlogo
to crear-agentes
  create-turtles num-ninos [
    set tipo "nino" 
    set color blue 
    set shape "person"
    set size 1.2 
    setxy random-xcor (random-ycor - 4)
    set tiempo-actividad 0
    escoger-nuevo-objetivo
  ]
  create-turtles num-ninas [
    set tipo "nina" 
    set color orange 
    set shape "person"
    set size 1.2 
    setxy random-xcor (random-ycor - 4)
    set tiempo-actividad 0
    escoger-nuevo-objetivo
  ]
end
```

## 4. Dinámica Temporal y Gestión de Estados (Go)

El bucle principal gestiona la transición entre dos estados: **Desplazamiento** (hacia un objetivo) y **Actividad** (tiempo de juego o charla). Se introducen límites de capacidad de carga para simular la saturación de recursos:
*   **Fútbol:** Máximo 22 agentes.
*   **Zonas Amarillas:** Máximo 15 agentes.

Si una zona está llena, el agente es "expulsado" y debe renegociar su objetivo, reflejando la **adaptabilidad de los sistemas sociales**.

```netlogo
to go
  if ticks >= 100 [ stop ]
  ask turtles [
    ifelse tiempo-actividad > 0 [
      ;; ESTADO: Realizando actividad
      set tiempo-actividad tiempo-actividad - 1
      rt random 40 - 20 fd 0.05
    ]
    [
      ;; ESTADO: Desplazamiento
      caminar-a-objetivo
      
      ; Verificación de llegada y capacidad de zona
      if pcolor = green [
        ifelse count turtles with [pcolor = green] < 22
          [ set tiempo-actividad (15 + random 15) ]
          [ rt 180 fd 2 escoger-nuevo-objetivo ]
      ]
      ; ... (lógica similar para otras zonas)
    ]
  ]
  calcular-indicators
  tick
end
```

## 5. Navegación e Inteligencia Individual

En lugar de un movimiento aleatorio simple, se utiliza un **Modelo de Fuerzas Sociales**. El procedimiento `escoger-nuevo-objetivo` asigna probabilidades de elección basadas en el género y la disponibilidad de atractores (Zonas de Interés).

```netlogo
to caminar-a-objetivo
  face-xy objetivo-x objetivo-y
  fd 0.4
  if not can-move? 1 [ rt 180 escoger-nuevo-objetivo ]
end

to escoger-nuevo-objetivo
  let azar random-float 100
  ifelse tipo = "nino" [
    ifelse tamano-campo > 0 and azar < 70 [
      set-destino-patch one-of patches with [pcolor = green]
    ] [
      ; Probabilidades para ninos en otras zonas...
    ]
  ] [
    ; Probabilidades diferenciadas para ninas...
  ]
end
```

## 6. Cálculo de Indicadores de Interacción y Actividad

El modelo calcula constantemente la segregación y la actividad. Se segmenta la interacción por zonas para validar la hipótesis de que las **Zonas de Interés (amarillo)** generan mayor integración que las zonas de reposo (marrón) o las áreas de paso (gris).

Se utiliza el **Ratio de contacto con el sexo opuesto** en un radio de 2 parches como indicador clave de integración.

```netlogo
to calcular-indicadores
  ;; 1. Ocupaciones y Porcentajes
  set total-en-fútbol count turtles with [pcolor = green]
  set total-en-grada count turtles with [pcolor = brown + 1]
  
  ;; 2. Interacciones (Ratio de contacto sexo opuesto radio 2)
  let contacto-global count turtles with [any? other turtles in-radius 2 with [tipo != [tipo] of myself]]
  set interaccion-global (contacto-global / count turtles)

  ; Cálculo específico por zonas (Grada, Amarilla, Gris)...
end
```
