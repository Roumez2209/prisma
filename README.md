# Prisma

Sistema de gestión para ópticas. Ver `Prisma – Guía por partes escalables.md` para el alcance por partes.

## Estructura (Parte 0: Fundamentos)

```
prisma/
├── servidor/                         Spring Boot + MySQL (lógica, BD, tareas programadas)
│   └── src/
│       ├── main/java/co/prisma/
│       │   ├── comun/                Servicios que reutiliza cada parte
│       │   │   ├── seguridad/        Login, hash de contraseñas, permisos por rol, bloqueo (RNF-01/03/04)
│       │   │   ├── error/            Manejo global de errores y log técnico sin datos personales (RNF-16/22)
│       │   │   ├── persistencia/     Entidad base: sucursal_id, versión para concurrencia, fechas (RNF-13/20)
│       │   │   └── config/           Configuración de Spring
│       │   └── modulos/              Un paquete por dominio; una parte nueva = un módulo nuevo
│       │       ├── usuarios/         HU-01, HU-02 · RF-01/02/03 · RN-53
│       │       ├── configuracion/    HU-50 · RF-04
│       │       ├── auditoria/        HU-51 · RF-63 · RN-52 · RNF-21
│       │       └── respaldo/         HU-52 · RF-64 · RNF-14
│       ├── main/resources/
│       │   └── db/migration/         Migraciones versionadas (Flyway): V1__..., V2__... (RNF-18)
│       └── test/java/co/prisma/
│           └── modulos/<módulo>/     Pruebas de las reglas de negocio de cada módulo (RNF-19)
│
├── cliente/                          Interfaz (JavaFX o React, por decidir)
│   └── src/
│       ├── comun/
│       │   ├── api/                  Llamadas al servidor y aviso si el servidor no responde (RNF-12)
│       │   ├── sesion/               Usuario actual, rol y bloqueo por inactividad
│       │   ├── componentes/          Tablas, formularios y diálogos de confirmación reutilizables
│       │   └── estilos/
│       └── modulos/
│           ├── inicio-sesion/
│           ├── usuarios/
│           ├── configuracion/
│           ├── auditoria/
│           └── respaldo/
│
├── despliegue/
│   ├── instalador/                   Instalador para Windows 11 (RNF-17/18)
│   ├── respaldo/                     Scripts de copia diaria y restauración (RNF-14)
│   └── config/                       Plantillas de configuración por equipo (sin credenciales)
│
└── docs/
    ├── arquitectura/                 Cómo levantar servidor, BD e interfaz (criterio de salida Parte 0)
    ├── decisiones/                   Decisiones registradas (p. ej. JavaFX vs React)
    ├── prototipos/                   Prototipos de pantallas (Sprint 0)
    └── siigo-poc/                    Prueba de concepto de la API de Siigo (Sprint 0)
```

## Convenciones

**Capas dentro de cada módulo del servidor:**

| Capa | Contenido |
| --- | --- |
| `api/` | Controladores REST y DTOs de entrada/salida. Validan permisos del rol. |
| `servicio/` | Casos de uso y reglas de negocio (RN). Registran en auditoría. |
| `dominio/` | Entidades y enums. |
| `repositorio/` | Acceso a la base de datos (Spring Data). |

- Un módulo solo usa a otro a través de su `servicio/`, nunca de su `repositorio/`.
- Todo cambio de la base de datos es una migración nueva; nunca se edita una ya aplicada.
- Las credenciales (MySQL, Siigo) van en variables de entorno o archivos fuera del repositorio.

## Cómo crecer: Parte 1

La Parte 1 suma dos módulos sin tocar los anteriores:

```
servidor/.../modulos/inventario/     {api, servicio, dominio, repositorio}   Producto, categoría, movimiento
servidor/.../modulos/proveedores/    {api, servicio, dominio, repositorio}   Proveedor, relación producto-proveedor
servidor/src/main/resources/db/migration/V<n>__inventario_proveedores.sql
cliente/src/modulos/inventario/
cliente/src/modulos/proveedores/
```
