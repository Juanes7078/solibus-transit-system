# 💡 Desafíos Técnicos y Decisiones de Arquitectura — Solibus

Este documento recopila de manera estructurada los problemas técnicos reales afrontados durante el desarrollo y escalamiento de **Solibus**, las soluciones de ingeniería implementadas y el registro de decisiones arquitectónicas (ADR).

---

## 🛠️ Parte 1: Desafíos Técnicos y Soluciones (Challenges & Solutions)

### Desafío 1: Explosión de Costos Operativos por Telemetría Vehicular Continua

#### Problema
La flota de buses transmitía su posición GPS cada segundo de forma directa a Cloud Firestore. Al cobrar por cada operación unitaria de escritura, una flota de 150 vehículos operando en turnos completos generaba millones de operaciones mensuales, proyectando costos de nube de cientos de dólares incompatibles con el presupuesto de un piloto de ciudad intermedia.

#### Solución
Se adoptó un enfoque de dos frentes:
1. **Algoritmo de Throttling Espaciotemporal:** Se programó en el cliente un filtro basado en la fórmula del semiverseno (Haversine) que descarta paquetes redundantes cuando el bus está en reposo o no ha cambiado de paradero, exigiendo un desplazamiento mínimo de $\ge 15\text{ m}$ o un intervalo temporal mínimo de $\ge 8\text{ s}$.
2. **Desacoplamiento de Base de Datos:** Se migró el canal en tiempo real hacia Firebase Realtime Database (RTDB), cuyo modelo de facturación se rige por ancho de banda transmitido (~$1 USD/GB) y no por escrituras individuales.

#### Resultado
Se redujo en más del 80% el número de eventos de telemetría y en más del 85% el costo de facturación de base de datos, manteniendo la visualización del mapa actualizada y fluida.

---

### Desafío 2: Degradación de Rendimiento y Bloqueo de UI al Calcular Rutas de Tránsito

#### Problema
El cálculo de rutas de transporte público requería evaluar la proximidad del usuario a decenas de rutas, proyectar distancias a paraderos y cruzar geometrías para encontrar transbordos viables. Esta operación matemática de complejidad cuadrática $O(N^2)$ se ejecutaba en el hilo principal del navegador (*Main Thread*), congelando la pantalla entre 400ms y 1.2 segundos en teléfonos de gama de entrada.

#### Solución
Se extrajo la totalidad del motor matemático hacia un **Web Worker** en segundo plano (`routing.worker.js`). El hilo de la interfaz se limita a enviar las coordenadas y recibir las opciones ordenadas de forma asíncrona. Se incorporó además un temporizador de guarda (*Watchdog Timer*) de 2.5 segundos con fallback transparente para mitigar posibles fallos de inicialización del Worker.

#### Resultado
Eliminación total del bloqueo del hilo principal (*Zero Frame Drops*). El mapa y las animaciones de la interfaz se mantienen constantes a **60 FPS** mientras el motor calcula las rutas en segundo plano.

---

### Desafío 3: Sobrecarga en la Descarga Inicial de Datos en Redes Móviles

#### Problema
El Service Worker de la PWA incluía en su caché estático obligatorio la totalidad de los archivos del sistema, descargando scripts y vistas de portales administrativos (`admin.*`, `conductor.*`, `empresa.*`, `intrasog.*`) en los teléfonos de los pasajeros comunes. Esto consumía más de 2.5 MB de datos móviles por usuario y ralentizaba la instalación inicial.

#### Solución
Se depuró el arreglo de assets esenciales (`CORE_ASSETS`), limitándolo exclusivamente a los recursos requeridos por el pasajero para navegar el mapa y consultar rutas. Adicionalmente, se configuró un pipeline de compilación no destructivo con Terser que redujo el tamaño de los scripts en un 45%.

#### Resultado
Ahorro de más de 1.2 MB de descarga por usuario en la primera visita o instalación PWA, garantizando tiempos de carga casi instantáneos incluso bajo conectividad 3G.

---

### Desafío 4: Bloqueo de Placas Vehiculares por Caída de Conexión de Conductores

#### Problema
Si un conductor perdía repentinamente la conexión a Internet o cerraba abruptamente la aplicación móvil sin desloguearse, su registro vehicular permanecía activo en el sistema. Esto impedía que otro conductor pudiera tomar el control del mismo vehículo más tarde, arrojando errores de colisión de placa.

#### Solución
Se desarrolló una Cloud Function programada para ejecutarse de forma periódica cada 15 minutos. La función analiza las marcas temporales de última transmisión, detecta registros huérfanos con más de 3 minutos de inactividad y ejecuta escrituras atómicas en lote (*Batched Writes*) para liberar las placas vehiculares y marcar las sesiones como inactivas de forma idempotente.

#### Resultado
Disponibilidad operativa del 100% de la flota sin requerir intervención manual por parte de soporte técnico.

---

## 🏛️ Parte 2: Registro de Decisiones de Arquitectura (ADR)

### ADR-01: Uso de Arquitectura Serverless (Firebase Cloud Platform)
* **Decisión:** Alojar la lógica de backend en Cloud Functions y bases de datos administradas (Firestore y RTDB) en lugar de un clúster de servidores dedicados (EC2 / VPS).
* **Motivo:** Evitar los costos fijos de mantenimiento de infraestructura, parches de sistema operativo y aprovisionamiento manual para un proyecto piloto con tráfico fluctuante entre horas pico y horas valle.
* **Consecuencia:** Cero mantenimiento de servidores físicos y alta disponibilidad nativa, a cambio de sujeción al modelo de cuotas y APIs del proveedor cloud.

---

### ADR-02: Adopción de Progressive Web App (PWA) con Capacitor en lugar de React Native / Flutter
* **Decisión:** Construir una sola base de código web optimizada e interoperable, distribuyéndola vía web PWA y empaquetándola en Android mediante Capacitor.
* **Motivo:** Reducir la duplicación de código entre la versión web de escritorio y la aplicación móvil de los conductores, permitiendo iteraciones y despliegues instantáneos sin depender de ciclos de aprobación de tiendas de aplicaciones para actualizaciones críticas.
* **Consecuencia:** Menor huella de almacenamiento en el dispositivo del usuario (~5 MB vs. ~40 MB en frameworks multiplataforma pesados), requiriendo un control manual estricto del rendimiento y optimización del DOM.

---

### ADR-03: Implementación de Custom Claims para Gestión de Roles
* **Decisión:** Almacenar los permisos de rol (`admin`, `empresa`, `intrasog`, `conductor`, `pasajero`) directamente en los claims criptográficos del token JWT de Firebase Auth.
* **Motivo:** Evitar realizar una lectura previa a la colección de usuarios de Firestore en cada regla de seguridad o petición HTTP para verificar los permisos del solicitante.
* **Consecuencia:** Reducción drástica de lecturas de base de datos y latencia en consultas protegidas. La revocación de un rol requiere refrescar el token de autenticación del cliente.

---

### ADR-04: Pipeline de Compilación No Destructivo hacia `dist/`
* **Decisión:** Implementar un script de compilación automatizado con Terser que deposita los archivos minificados en una carpeta `dist/` independiente, configurando Firebase Hosting para servir únicamente desde este directorio.
* **Motivo:** Los métodos anteriores de minificación sobrescribían los archivos originales en la raíz, dificultando el control de versiones y la depuración local.
* **Consecuencia:** Código fuente en desarrollo permanece 100% legible, modular y mantenible, mientras que la distribución a producción se sirve optimizada y con nombres de versión controlados.
