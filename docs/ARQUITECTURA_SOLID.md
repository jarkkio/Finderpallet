# FinderPallet — diagnóstico de arquitectura y plan de evolución

**Estado:** diagnóstico inicial  
**Base revisada:** rama `main`, archivo `index.html`  
**Objetivo:** mejorar mantenibilidad y seguridad sin perder funciones ni datos.

## 1. Situación actual

FinderPallet se publica como sitio estático en GitHub Pages y su interfaz principal está concentrada en `index.html`. Ese archivo contiene estilos, estructura HTML, lógica de interfaz, operaciones de inventario y llamadas a Firebase. La aplicación usa funciones globales (`window.*`) y manipula directamente elementos del DOM.

La solución se apoya en dos proyectos Firebase:
- **Usuarios:** colección `usuarios`, con número de empleado, nombre, contraseña, puesto, rol y planta.
- **Información operativa:** colecciones `inventario`, `historial`, `proveedores` y documento de configuración de permisos.

El campo `tecnicoAsignado` del inventario representa el **nombre del técnico**, no su número de empleado. Las futuras migraciones deben conservar esta compatibilidad.

## 2. Diagnóstico SOLID

### S — Responsabilidad única (Single Responsibility Principle)
**Situación:** el HTML combina presentación, validación de formularios, permisos, reglas de negocio, persistencia y auditoría.

**Dirección propuesta:** separar gradualmente:
- presentación y componentes de interfaz;
- servicios de aplicación (casos de uso);
- repositorios de acceso a Firebase;
- validaciones y reglas de dominio;
- configuración y permisos.

### O — Abierto/cerrado (Open/Closed Principle)
**Situación:** las reglas por rol y varias operaciones se resuelven con condiciones repartidas por la interfaz.

**Dirección propuesta:** centralizar las capacidades de cada rol en una política de permisos reutilizable. Esto mejora la consistencia de la interfaz, pero no sustituye las reglas de seguridad de Firebase.

### L — Sustitución de Liskov (Liskov Substitution Principle)
**Aplicabilidad:** no es la prioridad inmediata de una aplicación procedural con funciones globales. Se considerará cuando existan implementaciones intercambiables de repositorios o servicios, que deberán respetar los mismos contratos.

### I — Segregación de interfaces (Interface Segregation Principle)
**Aplicabilidad:** JavaScript no exige interfaces formales para este caso. La idea práctica es evitar un único servicio que concentre todas las operaciones y ofrecer módulos pequeños por área: inventario, usuarios, historial, proveedores y retrabajos.

### D — Inversión de dependencias (Dependency Inversion Principle)
**Situación:** las funciones de interfaz conocen detalles de Firebase y de sus operaciones.

**Dirección propuesta:** mover las operaciones de persistencia a repositorios y hacer que los casos de uso dependan de esas funciones, no de detalles de la pantalla. No se introducirá una capa abstracta si no reduce acoplamiento real.

## 3. Riesgo prioritario: seguridad

GitHub Pages entrega código al navegador; por tanto, cualquier validación de rol hecha solamente en JavaScript puede modificarse o saltarse desde el cliente. La aplicación no debe confiar en que ocultar un botón o comprobar una variable global autorice una operación.

Antes de considerar FinderPallet listo para uso operativo, se deben revisar las reglas de seguridad de **ambos proyectos Firebase** y comprobar que:
- los usuarios no puedan leer ni modificar cuentas ajenas sin autorización;
- un usuario no pueda elevar su propio rol;
- cada rol solo pueda leer y escribir los documentos y campos permitidos;
- las operaciones sensibles se validen en un backend confiable cuando las reglas de Firestore no puedan expresar la política necesaria.

El inicio de sesión basado en contraseñas guardadas en documentos de Firestore y validado desde el cliente no equivale a autenticación robusta. La migración a Firebase Authentication y reglas basadas en identidad/claims debe planearse sin bloquear el trabajo de mejora visual y modular.

**Limitación de este diagnóstico:** no se han inspeccionado ni probado las reglas desplegadas de Firestore, los permisos reales de los proyectos ni el sitio publicado en un entorno autenticado. No se afirma que esos controles estén bien configurados.

## 4. Plan incremental recomendado

### Fase 0 — Línea base y seguridad
1. Conservar la versión actual como referencia y no reemplazarla durante la preparación.
2. Identificar reglas de Firestore de ambos proyectos y revisar permisos por rol.
3. Registrar las operaciones críticas y sus invariantes: alta, edición, eliminación, movimientos, asignación rápida, retrabajos, historial y gestión de usuarios.
4. Confirmar que no se hayan publicado credenciales privadas. La configuración cliente de Firebase no es por sí misma una clave secreta; las reglas de acceso siguen siendo esenciales.

**Criterio de salida:** riesgos documentados y controles de acceso comprobados o con tareas correctivas definidas.

### Fase 1 — Pruebas de regresión
Crear pruebas repetibles para los flujos críticos antes de mover código. Las pruebas deben verificar resultados en la interfaz y escrituras esperadas, incluyendo errores y conflictos concurrentes. Los dobles de prueba de Firebase no sustituyen una validación posterior contra un proyecto de pruebas real.

**Criterio de salida:** existe una línea base reproducible para comparar la versión anterior con cada cambio.

### Fase 2 — Modularización sin cambiar comportamiento
Estructura objetivo inicial:

```text
/
├── index.html
├── css/
│   └── app.css
├── js/
│   ├── app.js
│   ├── firebase/
│   │   └── clients.js
│   ├── repositories/
│   │   ├── inventoryRepository.js
│   │   ├── usersRepository.js
│   │   └── historyRepository.js
│   ├── services/
│   │   ├── permissionsService.js
│   │   ├── inventoryService.js
│   │   └── reworkService.js
│   └── ui/
│       ├── modals.js
│       └── notifications.js
└── docs/
    └── ARQUITECTURA_SOLID.md
```

La estructura es un destino gradual, no una instrucción para mover todo en un solo commit. Primero se extraerá CSS y después módulos pequeños, manteniendo temporalmente adaptadores para las funciones globales que la interfaz ya utiliza. Se evitará introducir un framework o un proceso de compilación sin una necesidad demostrada.

**Criterio de salida:** GitHub Pages carga la aplicación y las pruebas críticas continúan pasando.

### Fase 3 — Refactor por dominio
Separar primero una operación acotada y de bajo riesgo, por ejemplo lectura/listado del inventario; después edición, movimientos, retrabajos y gestión de usuarios. Cada extracción debe ser un cambio pequeño con pruebas y un commit independiente.

### Fase 4 — Despliegue controlado
Probar en un entorno de prueba, revisar errores de consola, validar en escritorio y móvil, y publicar solo cuando los flujos críticos estén verificados. Mantener commits previos como punto de retorno.

## 5. Reglas para no romper el sistema

- No eliminar campos de Firestore ni cambiar sus nombres sin una migración explícita.
- Mantener compatibilidad con los registros históricos y los campos heredados.
- No cambiar el significado de `tecnicoAsignado`: es el nombre del técnico.
- Mantener la escritura de auditoría asociada a cada operación que la requiera.
- No realizar una reescritura total ni cambiar de framework como primer paso.
- Distinguir siempre entre pruebas simuladas y pruebas contra Firebase real.
- Cada cambio debe indicar qué se modificó, qué pruebas se ejecutaron y qué queda sin verificar.

## 6. Próxima acción

La siguiente acción técnica debe ser **revisar las reglas de Firestore y definir una matriz de permisos por rol**. Es el mayor riesgo arquitectónico porque la aplicación es pública y se ejecuta en el navegador. Para revisar las reglas reales hacen falta los archivos de reglas o acceso al proyecto Firebase; no deben inferirse a partir de la interfaz.
