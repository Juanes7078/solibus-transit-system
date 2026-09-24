# 🔒 Seguridad, Control de Acceso y Blindaje de Infraestructura — Solibus

Este documento resume las directrices de seguridad, control de acceso basado en roles (RBAC) y mitigación de vulnerabilidades implementadas en la plataforma **Solibus**.

---

## 1. Control de Acceso Basado en Roles (RBAC)

Para gestionar los privilegios de los diferentes perfiles del sistema de transporte sin incurrir en lecturas constantes a colecciones de usuarios en la base de datos, se adoptó el estándar de **Custom Claims en Firebase Authentication**.

### Estructura de Roles:
- `superadmin`: Acceso irrestricto a parametrización de rutas, empresas, conductores y reglas de negocio.
- `empresa`: Gestión de la flota propia y consulta de métricas de cumplimiento de horario de sus conductores.
- `intrasog` (Autoridad de Tránsito): Inspección global de cobertura, velocidades operativas y reporte de contingencias.
- `conductor`: Transmisión de telemetría de su vehículo asignado y visualización de hoja de ruta.
- `pasajero`: Consulta anónima o autenticada de catálogos de transporte y tracking en vivo de buses.

### Asignación Criptográfica en Backend
Los roles elevados nunca se auto-asignan desde el cliente. Una Cloud Function autenticada y verificada sella el rol en el token JWT del usuario:
```
Token JWT del Usuario
├── Header: Algoritmo de firma
├── Payload:
│   ├── uid: "Conductor_102837"
│   ├── email: "operador@solibus.app"
│   └── claims: { role: "conductor", empresaId: "COOP_TRANS" }
└── Signature: Clave privada de Firebase Auth
```

---

## 2. Blindaje de Reglas en Cloud Firestore

Las reglas de seguridad se definen de manera declarativa para garantizar la integridad referencial y prevenir la usurpación de datos vehiculares:

1. **Inmutabilidad de la Posesión Vehicular:**
   Un conductor solo puede actualizar el estado y ubicación del bus si su UID de autenticación coincide exactamente con el `conductorId` registrado en el documento, y está estrictamente prohibido que un conductor transfiera la propiedad del bus a otro usuario:
   ```
   allow update: if request.auth != null &&
       resource.data.conductorId == request.auth.uid &&
       request.resource.data.conductorId == resource.data.conductorId;
   ```

2. **Prevención de Elevación de Privilegios:**
   Los usuarios autenticados de forma anónima o con cuentas de nuevo registro únicamente tienen permitido registrarse bajo el rol de `pasajero`. Cualquier intento de modificar campos de privilegios en el documento personal es bloqueado inmediatamente en la capa de base de datos.

3. **Restricción de Almacenamiento (Cloud Storage):**
   Las reglas de Cloud Storage imponen límites de cuota estrictos por tipo de recurso:
   - Archivos de rutas (`/rutas/`): Solo administradores, tamaño máximo de 2 MB y validación estricta de tipo MIME `application/json` o `application/geo+json`.
   - Reportes e imágenes (`/reportes/`): Usuarios autenticados, máximo 1 MB y únicamente formatos de imagen (`image/*`).

---

## 3. Seguridad en el Empaquetado Móvil Android

Durante el proceso de auditoría y hardening del sistema, se identificaron y subsanaron los siguientes vectores de riesgo en las compilaciones móviles:

- **Exclusión Quirúrgica de Credenciales:**
  Se automatizó el script `build-apk.bat` mediante directivas de exclusión estricta (`Robocopy /XF`) para asegurar que ningún archivo de credenciales de servidor (`serviceAccountKey.json`), scripts administrativos (`admin.*`, `empresa.*`), scripts de PowerShell o logs de depuración puedan ser incluidos dentro del bundle del APK o AAB.
- **Protección de Certificados Keystore:**
  Se eliminaron las contraseñas en texto plano del archivo `build.gradle`, delegando la lectura de firmas criptográficas a variables de entorno del sistema operativo (`System.getenv("ANDROID_KEYSTORE_PASSWORD")`).

---

## 4. Política de Seguridad de Contenido (CSP)

Para mitigar riesgos de inyección de código (Cross-Site Scripting - XSS) y ataques de intermediario, la cabecera `Content-Security-Policy` en `firebase.json` define una lista blanca explícita:
- `script-src`: Restringido al origen del sitio, CDN oficiales de Google Maps, Firebase SDK y fuentes autorizadas.
- `connect-src`: Habilitado exclusivamente hacia los dominios de la API de Firebase Hosting, Cloud Functions, Cloud Firestore y sockets seguros de Firebase Realtime Database (`wss://*.firebaseio.com`).
