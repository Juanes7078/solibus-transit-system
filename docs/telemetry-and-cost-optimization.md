# ⚡ Telemetría de Alta Frecuencia y Optimización de Costos de Nube

Este documento analiza en profundidad el desafío técnico de procesar telemetría vehicular continua en una plataforma de transporte urbano masivo y la solución matemática y arquitectónica implementada en **Solibus**.

---

## 1. El Problema de la Telemetría Continua en Nubes Documentales

En los sistemas tradicionales de seguimiento de vehículos, la aplicación móvil del conductor emite un evento cada vez que el chip GPS del dispositivo reporta un cambio de coordenadas (frecuentemente una vez por segundo).

Si una flota promedio cuenta con 150 buses operando durante 14 horas al día:

$$\text{Escrituras por día} = 150 \times 14 \times 3,600 = 7,560,000 \text{ escrituras/día}$$

En servicios de bases de datos documentales como Cloud Firestore, donde se factura por cada operación de escritura unitaria (a razón de ~$0.18 USD por cada 100,000 escrituras), este flujo generaría:

$$\text{Costo diario aproximado} = \frac{7,560,000}{100,000} \times \$0.18 \approx \$13.60 \text{ USD/día} \approx \$408 \text{ USD/mes}$$

Para un sistema de transporte en una ciudad intermedia, este costo recurrente de infraestructura resultaba inviable para la sostenibilidad del proyecto.

---

## 2. La Solución: Filtrado Espaciotemporal (Throttling)

Se implementó un algoritmo de evaluación local en el cliente antes de despachar cualquier solicitud de red hacia el backend.

### Criterios de Disparo de Telemetría:
Un nuevo paquete de coordenadas solo es enviado si se cumple al menos una de las siguientes tres condiciones:
1. **Umbral Temporal:** Han transcurrido $\ge 8$ segundos desde la última transmisión confirmada.
2. **Umbral Espacial:** El vehículo se ha desplazado $\ge 15$ metros lineales con respecto al punto anterior, calculado mediante la fórmula del semiverseno (Haversine).
3. **Evento Crítico:** El vehículo detecta la entrada o salida de un paradero oficial.

Si el bus está detenido en un semáforo o en un trancón, el sistema descarta transmisiones redundantes, evitando el spamming a la base de datos.

### Formulación Matemática de Haversine:
$$d = 2r \arcsin \left( \sqrt{\sin^2\left(\frac{\Delta \phi}{2}\right) + \cos(\phi_1)\cos(\phi_2)\sin^2\left(\frac{\Delta \lambda}{2}\right)} \right)$$

Donde $r = 6,371,000\text{ m}$ (radio medio terrestre), $\phi$ corresponde a la latitud y $\lambda$ a la longitud en radianes.

---

## 3. Desacoplamiento a Firebase Realtime Database (RTDB)

Además de reducir la frecuencia de transmisiones mediante el filtro espaciotemporal, se modificó el canal de ingestión primario:
- **RTDB factura por ancho de banda descargado (~$1.00 USD por Gigabyte)**, sin cobrar por operaciones de lectura ni escritura unitarias.
- Un payload de telemetría optimizado contiene únicamente:
  ```json
  {
    "lat": 5.71421,
    "lng": -72.93215,
    "speed": 34.2,
    "heading": 182,
    "ts": 1727218400000,
    "stop": "P-04"
  }
  ```
  Peso promedio: $\sim 200\text{ bytes}$ por paquete comprimido.

### Comparativa Cuantitativa de Costos:

| Métrica | Antes (Escritura Ciega a Firestore) | Después (Throttling + RTDB) | Reducción Lograda |
| :--- | :--- | :--- | :--- |
| **Frecuencia por bus** | 1 transmisión/segundo | 1 transmisión cada 8-12 segundos | **-88% en eventos** |
| **Operaciones mensuales** | ~226 millones de escrituras | ~27 millones de pings RTDB | **-88% en volumen** |
| **Costo mensual estimado** | ~$400 - $450 USD | **<$15 USD** | **📉 ~85% a 90% de ahorro** |

---

## 4. Mecanismo de Contingencia y Failover Automático

La conexión con RTDB se realiza a través de WebSockets persistentes. Si el cliente detecta una pérdida de conexión de socket o un error de red prolongado por más de 2.5 segundos:
1. El cliente conmuta en caliente (*hot failover*) hacia una escritura de emergencia en Cloud Firestore.
2. Continúa intentando restablecer el socket de RTDB en segundo plano.
3. Una vez restablecido el canal primario, revierte automáticamente a RTDB sin intervención del usuario.
