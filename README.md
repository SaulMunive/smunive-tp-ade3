# TP Integrador — TransAndino Express
**Módulo:** Azure para Ingeniería de Datos — Edición 3  
**Alumno:** Saul Alejandro Munive  
**Alias:** smunive  
**Fecha de entrega:** 2026-06-24  

## Estructura del repositorio

- `tp_notebook.ipynb` — Notebook Databricks (Parte 4)
- `arquitectura/diagrama_arquitectura.png` — Diagrama de arquitectura
- `evidencias/` — Screenshots de evidencias

## Recursos Azure utilizados

| Recurso | Nombre |
|---|---|
| Storage Account | smunivestad |
| Azure Databricks | smunive-dbw-dataengineered |
| Azure Data Factory | smunive-adf-dataengineered3 |
| Key Vault | smunive-kv-dataengi3 |
| Microsoft Fabric Workspace | smunive-ws-dataengineered |
| UAMI | smunive-uami-dataengineered3 |

## Parte 1.2 — Justificaciones técnicas

**a) ¿Por qué usar Job Cluster y no All-Purpose Cluster en la Notebook Activity de ADF?**

El Job Cluster se crea automáticamente cuando arranca el pipeline y se destruye al terminar, por lo que solo se paga por el tiempo de ejecución real. El All-Purpose Cluster está pensado para trabajo interactivo y queda encendido esperando, lo que generaría costos innecesarios en un proceso automatizado que corre una vez por día.

**b) ¿Por qué autenticar el Linked Service con UAMI y no con la Storage Account Key?**

La Storage Account Key es una credencial estática que da acceso total a la cuenta y, si se filtra, compromete todo el storage. La UAMI no tiene contraseña que pueda robarse: Azure gestiona la identidad internamente y se le asignan solo los permisos mínimos necesarios mediante RBAC, lo que es mucho más seguro y fácil de auditar.

**c) ¿Qué problema genera guardar el token de Databricks en texto plano en el Linked Service?**

Cualquier persona con acceso al ADF puede ver el token directamente, y además queda expuesto en logs y exports de ARM templates. La alternativa correcta es guardar el token como secreto en Azure Key Vault y referenciarlo desde el Linked Service, de modo que ADF lo recupera en tiempo de ejecución sin que nadie lo vea en texto claro.

## Decisiones técnicas

**Autenticación hacia ADLS Gen2 desde Databricks:**
En vez de usar storage keys directamente en el notebook, se configuraron External Locations en Unity Catalog con una Storage Credential basada en el Access Connector de Azure. Esto permite que Databricks acceda al storage de forma segura sin exponer credenciales en el código.

**Cluster interactivo en lugar de Job Cluster:**
Por limitaciones de quota de vCPUs en la cuenta free de Azure (máximo 4 cores disponibles), se utilizó un cluster interactivo existente en lugar de un Job Cluster dedicado para la Notebook Activity de ADF. En un entorno productivo lo correcto sería usar Job Cluster para optimizar costos.

**try_to_date vs to_date para normalización de fechas:**
El cluster interactivo con Photon habilitado soporta `try_to_date` de SQL, pero cuando ADF dispara el notebook en un contexto diferente esta función no estaba disponible. Se implementó un bloque try/except que intenta `try_to_date` primero y cae en `coalesce(to_date(...))` como fallback, garantizando compatibilidad en ambos contextos.

**Fabric con cuenta institucional:**
Microsoft Fabric requiere cuenta organizacional para acceder. Se utilizó la cuenta institucional del curso para activar el trial de 60 días y crear el workspace, Lakehouse y shortcuts hacia el Storage Account de Azure.

## Puntos de mejora

- Migrar la autenticación de Databricks a OAuth con UAMI en lugar de Account Key para mayor seguridad
- Usar Job Cluster en ADF en lugar del cluster interactivo cuando la quota de Azure lo permita
- Agregar manejo de errores más robusto en el notebook para casos de datos faltantes
- Implementar particionamiento en las tablas Gold por año/mes para mejorar performance de queries
- Configurar alertas en ADF para notificar fallos del pipeline automáticamente
