# 🎥 Demostración y Escenarios de Prueba — Solibus

Este documento describe la experiencia de usuario y los escenarios operativos clave de la plataforma **Solibus**.

---

## 1. Escenarios de Uso y Flujos de Usuario

### Escenario A: Pasajero en Búsqueda de Ruta
1. El usuario accede a la aplicación web o PWA desde su teléfono móvil.
2. La interfaz detecta la ubicación actual y centra el mapa interactivo de Sogamoso.
3. El usuario pulsa sobre un destino comercial o ingresa una dirección.
4. El motor de ruteo en segundo plano (Web Worker) proyecta las coordenadas y devuelve dos alternativas:
   - *Alternativa Directa:* Línea de ruta con caminata mínima hasta el paradero más cercano.
   - *Alternativa con Transbordo:* Conexión entre dos líneas con tiempo estimado de transbordo.
5. El usuario selecciona la alternativa y visualiza el trazado en el mapa junto a la posición en vivo del bus más próximo.

### Escenario B: Conductor en Ruta
1. El conductor inicia sesión con sus credenciales asignadas por la empresa.
2. Selecciona la ruta programada del día e ingresa la placa del vehículo.
3. El sistema valida en Firestore que la placa no esté en uso por otra sesión activa.
4. Se activa el modo de telemetría:
   - El chip GPS reporta coordenadas al módulo de telemetría.
   - El filtro espaciotemporal descarta transmisiones si el bus está en reposo.
   - Al detectar movimiento significativo o cambio de paradero, despacha la actualización a Realtime Database.
5. Al finalizar el recorrido, pulsa "Finalizar Ruta", liberando automáticamente la asignación del bus.

---

## 2. Demostración Visual y Sanitización de Datos

Para proteger la privacidad de los conductores y las empresas transportadoras:
- Los identificadores de vehículos mostrados en capturas y demostraciones utilizan identificadores sintéticos de prueba (`BUS-01`, `BUS-02`).
- Las coordenadas de prueba corresponden a vías públicas principales de Sogamoso sin registrar domicilios privados.
- Los paneles administrativos de fiscalización se ilustran con datos agregados anónimos.

---

## 3. Contacto para Demostración Guiada

Si eres un reclutador, evaluador técnico o directivo interesado en coordinar una demostración en vivo de la plataforma o revisar especificaciones complementarias, puedes ponerte en contacto a través de los canales listados en el README principal.
