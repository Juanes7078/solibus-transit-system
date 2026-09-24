# 📐 Arquitectura del Sistema — Solibus

Este documento describe la arquitectura técnica de alto nivel, la separación de responsabilidades entre capas y los patrones de integración utilizados en la plataforma **Solibus**.

---

## 1. Visión General de Capas

El sistema se estructura en cuatro capas desacopladas diseñadas para maximizar la resiliencia en redes móviles intermitentes y minimizar el costo operativo de la infraestructura serverless:

```
┌────────────────────────────────────────────────────────────────────────┐
│                       1. CAPA DE CLIENTES                              │
│   • PWA Pasajeros        • App Conductor (Android)   • Dashboards      │
└───────────────────────────────────┬────────────────────────────────────┘
                                    │
                                    ▼
┌────────────────────────────────────────────────────────────────────────┐
│                       2. MOTOR EN EL CLIENTE                           │
│   • Hilo de UI (60 FPS)   • Web Worker (Ruteo O(N²)) • Service Worker │
└───────────────────────────────────┬────────────────────────────────────┘
                                    │
                                    ▼
┌────────────────────────────────────────────────────────────────────────┐
│                  3. INGESTA Y OPTIMIZACIÓN DE RED                      │
│   • Throttling Haversine  • Dual RTDB / Firestore Fallback             │
└───────────────────────────────────┬────────────────────────────────────┘
                                    │
                                    ▼
┌────────────────────────────────────────────────────────────────────────┐
│                   4. NUBE SERVERLESS (FIREBASE)                        │
│   • Cloud Functions Cron  • Custom Claims JWT        • Cloud Storage   │
└────────────────────────────────────────────────────────────────────────┘
```

---

## 2. Descripción de Componentes

### 2.1. Aplicación de Pasajeros (PWA)
- **Responsabilidad:** Proporcionar al usuario final la visualización interactiva del mapa, búsqueda de rutas hacia un destino y renderizado del movimiento de los vehículos en tiempo real.
- **Estrategia de Caché:** El catálogo de rutas y geometrías se carga una única vez mediante `getDocs` y se almacena en `localStorage` con versionado criptográfico, eliminando las lecturas repetitivas a la base de datos documental.
- **Integración Cartográfica:** Implementación de marcadores vectoriales mediante la API `google.maps.marker.AdvancedMarkerElement`, reduciendo la sobrecarga de repintado del DOM.

### 2.2. Aplicación de Conducción y Telemetría
- **Responsabilidad:** Capturar la posición geodésica del bus mediante la API de Geolocalización (GPS del hardware), evaluar umbrales espaciotemporales y emitir el paquete de telemetría hacia la nube.
- **Empaquetado Android:** Envoltura con Capacitor para permitir ejecución como servicio de ubicación en primer plano en terminales móviles de los conductores.

### 2.3. Portales Operativos (Empresas y Autoridad de Tránsito)
- **Responsabilidad:** Visualización consolidada de la flota, cálculo de intervalos de paso entre vehículos y auditoría de cobertura.
- **Seguridad:** Acceso protegido mediante validación de Custom Claims en el token de autenticación.

---

## 3. Patrón de Persistencia Dual

Una de las decisiones más críticas del sistema fue el desacoplamiento de la base de datos en dos motores con modelos de facturación complementarios:

| Motor | Propósito | Modelo de Facturación | Ventaja Técnica |
| :--- | :--- | :--- | :--- |
| **Firebase Realtime Database (RTDB)** | Streaming de posición en vivo de buses (`buses_live/{busId}`). | Por ancho de banda de red transferido (~$1 USD/GB). | Millones de pings GPS por una fracción de centavo. Conexión persistente multiplexada vía WebSockets. |
| **Cloud Firestore** | Almacenamiento de catálogos de rutas, estadísticas agregadas (`/route_stats/`) y perfiles de usuario. | Por operaciones de lectura, escritura y borrado de documentos. | Consultas complejas indexadas, reglas de seguridad declarativas por colección y persistencia offline estructurada. |

---

## 4. Tareas Serverless Automatizadas (Cron Jobs)

En la capa de backend serverless, Cloud Functions ejecuta tareas programadas idempotentes cada 15 minutos:
1. **Detección de Salud de Flota:** Clasifica los registros vehiculares en tres estados:
   - `Activo`: Última transmisión $< 35$ segundos.
   - `Inactivo / Stale`: Transmisión interrumpida entre 35 segundos y 3 minutos.
   - `Offline / Desconectado`: Sin reporte durante más de 3 minutos.
2. **Liberación Atómica de Placas:** Si un conductor cierra inesperadamente la aplicación o pierde conectividad total, el cron de backend procesa las instancias inactivas en lotes atómicos (*Batched Writes*) liberando la asignación del bus para que no quede bloqueado ante otros operadores.
