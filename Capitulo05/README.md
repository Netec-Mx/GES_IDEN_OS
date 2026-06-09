# Asignación de roles y revisión de accesos en una herramienta IAM

## 1. Metadatos

| Campo            | Detalle                                      |
|------------------|----------------------------------------------|
| **Duración**     | 25 minutos                                   |
| **Complejidad**  | Media                                        |
| **Nivel Bloom**  | Aplicar                                      |
| **Módulo**       | 5 — Modelos RBAC, ABAC y Control de Acceso   |
| **Lab ID**       | 05-00-01                                     |

---

## 2. Descripción General

Este laboratorio simula el departamento IAM de una organización con tres unidades de negocio (IT, RRHH, Finanzas) y tres niveles de acceso (básico, intermedio, administrador). Configurarás en **OpenIAM 4.2 Community Edition** una jerarquía de roles RBAC, asignarás recursos protegidos y aplicarás el principio de privilegio mínimo con delegación departamental. Posteriormente ejecutarás una **campaña de certificación de accesos** (*access review*), aprobarás o revocarás accesos y generarás informes de cumplimiento. Como extensión, discutirás cómo enriquecer el modelo RBAC con condiciones ABAC simples basadas en el atributo `departamento`, conectando directamente con la arquitectura PEP/PDP/PAP/PIP estudiada en la lección 5.1.

---

## 3. Objetivos de Aprendizaje

- [ ] Configurar una jerarquía de roles (RBAC1) en OpenIAM con mínimo 6 roles de negocio y técnicos, aplicando el principio de privilegio mínimo.
- [ ] Asignar recursos protegidos y políticas de delegación departamental, de forma que el manager de RRHH solo pueda gestionar roles de su propio departamento.
- [ ] Crear 15 usuarios de prueba con asignaciones variadas (incluyendo asignaciones incorrectas intencionadas) usando un script Python con datos ficticios.
- [ ] Configurar y ejecutar una campaña de certificación de accesos en OpenIAM, actuando como revisor para aprobar y revocar accesos.
- [ ] Generar e interpretar informes de usuarios por rol, roles por usuario y estado de cumplimiento (*compliance*).

---

## 4. Prerrequisitos

### Conocimiento previo
- Comprensión de los modelos RBAC (RBAC0–RBAC3), ABAC y sus diferencias (Lección 5.1).
- Familiaridad con los conceptos de delegación, privilegio mínimo y Segregación de Funciones (SoD).
- Conceptos básicos de auditoría y *compliance* de accesos.
- Haber completado o revisado el Lab 03-00-01 (OpenLDAP) y Lab 04-00-01 (Keycloak SSO) es recomendable, aunque no bloqueante para este lab.

### Acceso y herramientas
- Docker Engine 24.x y Docker Compose 2.20.x instalados y operativos.
- Mínimo **8 GB de RAM libres** para el stack OpenIAM + PostgreSQL (se recomienda 16 GB).
- Python 3.10+ con pip disponible en el host.
- Navegador Chrome o Firefox actualizado.
- Acceso a Internet para descargar imágenes Docker (primera ejecución).
- Puerto **8080** libre en el host (interfaz web OpenIAM).

> **⚠️ Nota de recursos:** Si vienes de un lab anterior, ejecuta primero la sección de limpieza de ese lab (`docker compose down -v`) para liberar RAM antes de continuar.

---

## 5. Entorno de Laboratorio

### Hardware mínimo recomendado

| Recurso         | Mínimo          | Recomendado     |
|-----------------|-----------------|-----------------|
| CPU             | 4 núcleos 64-bit| 8 núcleos       |
| RAM             | 16 GB           | 32 GB           |
| Disco libre     | 20 GB SSD       | 40 GB SSD       |
| Red             | NAT/Host-only   | Bridged + NAT   |

### Software utilizado

| Componente       | Versión          | Rol en el lab                        |
|------------------|------------------|--------------------------------------|
| OpenIAM          | 4.2 CE           | Motor IAM principal (RBAC, roles, access review) |
| PostgreSQL       | 15.x             | Base de datos de OpenIAM             |
| Docker Compose   | 2.20.x           | Orquestación de contenedores         |
| Python           | 3.10+            | Script de carga masiva de usuarios   |
| Faker (Python)   | 20.x             | Generación de datos ficticios        |
| Chrome/Firefox   | Última versión   | Interfaz web OpenIAM                 |

### Estructura de directorios del lab

```
~/lab-05-00-01/
├── docker-compose.yml
├── init/
│   └── openiam-init.sql
├── scripts/
│   ├── generate_users.py
│   └── load_users.py
└── reports/
    └── (generados durante el lab)
```

### Preparación del entorno

**Paso 0 — Crear el directorio de trabajo:**

```bash
mkdir -p ~/lab-05-00-01/{init,scripts,reports}
cd ~/lab-05-00-01
```

**Paso 0.1 — Crear el archivo `docker-compose.yml`:**

```bash
cat > ~/lab-05-00-01/docker-compose.yml << 'EOF'
version: "3.9"

services:
  postgres:
    image: postgres:15-alpine
    container_name: lab05_postgres
    environment:
      POSTGRES_DB: openiam
      POSTGRES_USER: openiam
      POSTGRES_PASSWORD: OpenIAM_Lab05!
    volumes:
      - pgdata_lab05:/var/lib/postgresql/data
      - ./init:/docker-entrypoint-initdb.d
    ports:
      - "5432:5432"
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U openiam"]
      interval: 10s
      timeout: 5s
      retries: 5
    networks:
      - lab05_net

  openiam:
    image: openiam/openiam-community:4.2
    container_name: lab05_openiam
    depends_on:
      postgres:
        condition: service_healthy
    environment:
      OPENIAM_DB_HOST: postgres
      OPENIAM_DB_PORT: 5432
      OPENIAM_DB_NAME: openiam
      OPENIAM_DB_USER: openiam
      OPENIAM_DB_PASSWORD: OpenIAM_Lab05!
      OPENIAM_ADMIN_PASSWORD: Admin@Lab05!
    ports:
      - "8080:8080"
    volumes:
      - openiam_data_lab05:/opt/openiam/data
    networks:
      - lab05_net
    healthcheck:
      test: ["CMD-SHELL", "curl -sf http://localhost:8080/openiam-ui/login || exit 1"]
      interval: 30s
      timeout: 15s
      retries: 10

volumes:
  pgdata_lab05:
  openiam_data_lab05:

networks:
  lab05_net:
    driver: bridge
EOF
```

> **Nota:** Si tu instructor provee una imagen Docker de OpenIAM preconfigurada o un snapshot de VM, usa esa en lugar de la imagen pública. Consulta el repositorio del curso para la URL exacta de la imagen validada para este lab.

**Paso 0.2 — Instalar dependencias Python:**

```bash
pip3 install faker requests --quiet
```

**Paso 0.3 — Levantar el stack:**

```bash
cd ~/lab-05-00-01
docker compose up -d
```

**Esperar a que OpenIAM esté listo** (puede tardar 2–3 minutos en el primer arranque):

```bash
docker compose logs -f openiam | grep -m1 "Started OpenIAM"
# Ctrl+C cuando aparezca el mensaje de inicio
```

**Verificar acceso:**
```bash
curl -s -o /dev/null -w "%{http_code}" http://localhost:8080/openiam-ui/login
# Debe devolver: 200
```

---

## 6. Procedimiento Paso a Paso

---

### Paso 1 — Configurar la Jerarquía Organizacional

**Objetivo:** Crear las tres unidades organizativas (IT, RRHH, Finanzas) en OpenIAM, que servirán como contenedores para la delegación departamental y como atributo `departamento` para las condiciones ABAC posteriores.

#### Instrucciones

1. Abre el navegador y navega a: `http://localhost:8080/openiam-ui/login`

2. Inicia sesión con las credenciales de administrador:
   - **Usuario:** `sysadmin`
   - **Contraseña:** `Admin@Lab05!`

3. En el menú lateral, ve a **Organization Management → Organizations**.

4. Haz clic en **+ New Organization** y crea la primera unidad:
   - **Name:** `Departamento IT`
   - **Organization Type:** `Department`
   - **Description:** `Unidad de Tecnologías de la Información`
   - **Metadata / Attribute:** Agrega atributo personalizado: `dept_code = IT`
   - Haz clic en **Save**.

5. Repite el proceso para las otras dos unidades:

   | Name               | Type       | dept_code |
   |--------------------|------------|-----------|
   | `Departamento RRHH`| Department | RRHH      |
   | `Departamento Finanzas` | Department | FIN  |

6. Verifica que las tres organizaciones aparecen en la lista con estado **Active**.

#### Resultado esperado

```
Organizations (3):
  ✓ Departamento IT       [Active] dept_code=IT
  ✓ Departamento RRHH     [Active] dept_code=RRHH
  ✓ Departamento Finanzas [Active] dept_code=FIN
```

#### Verificación

```bash
# Verificar vía API REST de OpenIAM (ajusta el token si es necesario)
curl -s -u sysadmin:Admin@Lab05! \
  "http://localhost:8080/openiam-rest/api/1/org/search" \
  -H "Content-Type: application/json" \
  -d '{"name": "Departamento"}' | python3 -m json.tool | grep '"name"'
```

---

### Paso 2 — Diseñar y Crear la Jerarquía de Roles RBAC

**Objetivo:** Implementar un modelo RBAC1 (con jerarquía de roles) con 6 roles organizados en dos niveles: roles de negocio (Business Roles) que heredan de roles técnicos (Technical Roles), aplicando el principio de privilegio mínimo.

#### Diseño de roles (referencia)

```
Jerarquía RBAC1 — Organización Lab05
─────────────────────────────────────────────────────────────────────
ROLES TÉCNICOS (Technical Roles — permisos atómicos):
  TR_ACCESO_BASICO        → leer_portal, ver_perfil_propio
  TR_ACCESO_INTERMEDIO    → TR_ACCESO_BASICO + editar_documentos, generar_reportes
  TR_ADMIN_DEPT           → TR_ACCESO_INTERMEDIO + gestionar_usuarios_dept

ROLES DE NEGOCIO (Business Roles — asignados a usuarios):
  BR_EMPLEADO             → hereda TR_ACCESO_BASICO
  BR_ESPECIALISTA         → hereda TR_ACCESO_INTERMEDIO
  BR_MANAGER_DEPT         → hereda TR_ADMIN_DEPT
─────────────────────────────────────────────────────────────────────
Restricción SoD: BR_MANAGER_DEPT y BR_ESPECIALISTA en Finanzas
  no pueden coexistir en el mismo usuario (crear_pago + aprobar_pago).
```

#### Instrucciones

1. En OpenIAM, ve a **Role Management → Roles**.

2. Crea los **Roles Técnicos** (tipo: `Technical`). Para cada uno:
   - Haz clic en **+ New Role**.
   - Completa los campos según la tabla y guarda.

   | Role Name              | Type       | Description                                    |
   |------------------------|------------|------------------------------------------------|
   | `TR_ACCESO_BASICO`     | Technical  | Acceso de lectura básico al portal             |
   | `TR_ACCESO_INTERMEDIO` | Technical  | Acceso intermedio con edición de documentos    |
   | `TR_ADMIN_DEPT`        | Technical  | Administración de usuarios del departamento    |

3. Configura la **jerarquía de herencia** (RBAC1):
   - Edita `TR_ACCESO_INTERMEDIO` → pestaña **Parent Roles** → agrega `TR_ACCESO_BASICO`.
   - Edita `TR_ADMIN_DEPT` → pestaña **Parent Roles** → agrega `TR_ACCESO_INTERMEDIO`.

4. Crea los **Roles de Negocio** (tipo: `Business`):

   | Role Name          | Type     | Child Technical Role   | Description                          |
   |--------------------|----------|------------------------|--------------------------------------|
   | `BR_EMPLEADO`      | Business | `TR_ACCESO_BASICO`     | Empleado base — todos los dptos.     |
   | `BR_ESPECIALISTA`  | Business | `TR_ACCESO_INTERMEDIO` | Especialista funcional               |
   | `BR_MANAGER_DEPT`  | Business | `TR_ADMIN_DEPT`        | Manager con delegación departamental |

5. Configura la **restricción SoD** para Finanzas:
   - Ve a **Role Management → Segregation of Duties**.
   - Haz clic en **+ New SoD Rule**.
   - **Name:** `SoD_Finanzas_CrearAprobar`
   - **Role 1:** `BR_ESPECIALISTA` (contexto: Finanzas — aplica filtro por organización `Departamento Finanzas`)
   - **Role 2:** `BR_MANAGER_DEPT` (contexto: Finanzas)
   - **Conflict Type:** `Exclusive` (mutuamente excluyentes)
   - Guarda la regla.

6. Asocia cada rol de negocio a su organización correspondiente:
   - Edita `BR_MANAGER_DEPT` → pestaña **Organizations** → asigna las tres organizaciones pero marca la opción **Scope: Own Department Only** para habilitar la delegación.

#### Resultado esperado

```
Roles configurados (6):
  Technical Roles:
    ✓ TR_ACCESO_BASICO       [Active]
    ✓ TR_ACCESO_INTERMEDIO   [Active] parent: TR_ACCESO_BASICO
    ✓ TR_ADMIN_DEPT          [Active] parent: TR_ACCESO_INTERMEDIO

  Business Roles:
    ✓ BR_EMPLEADO            [Active] → TR_ACCESO_BASICO
    ✓ BR_ESPECIALISTA        [Active] → TR_ACCESO_INTERMEDIO
    ✓ BR_MANAGER_DEPT        [Active] → TR_ADMIN_DEPT

  SoD Rules:
    ✓ SoD_Finanzas_CrearAprobar [Active] EXCLUSIVE
```

#### Verificación

En el árbol de roles, selecciona `TR_ADMIN_DEPT` y verifica que en la pestaña **Effective Permissions** aparecen los permisos de `TR_ACCESO_BASICO` e `TR_ACCESO_INTERMEDIO` por herencia.

---

### Paso 3 — Crear Recursos Protegidos (Aplicaciones Simuladas)

**Objetivo:** Registrar tres recursos protegidos que representan aplicaciones departamentales, y asignar los roles que tienen acceso a cada uno. Esto implementa el mapeo **Rol → Permiso → Recurso** del modelo RBAC.

#### Instrucciones

1. Ve a **Resource Management → Resources**.

2. Crea los siguientes tres recursos (tipo: `Application`):

   | Resource Name          | Description                        | Roles con acceso                            |
   |------------------------|------------------------------------|---------------------------------------------|
   | `APP_PORTAL_EMPLEADO`  | Portal de autoservicio empleados   | `BR_EMPLEADO`, `BR_ESPECIALISTA`, `BR_MANAGER_DEPT` |
   | `APP_GESTION_RRHH`     | Sistema de gestión de RRHH         | `BR_ESPECIALISTA` (RRHH), `BR_MANAGER_DEPT` (RRHH) |
   | `APP_FINANZAS_ERP`     | ERP módulo financiero              | `BR_ESPECIALISTA` (FIN), `BR_MANAGER_DEPT` (FIN) |

3. Para cada recurso:
   - Haz clic en **+ New Resource**.
   - Completa **Name**, **Resource Type: Application**, **Description**.
   - En la pestaña **Entitlements**, haz clic en **+ Add Role** y agrega los roles indicados.
   - Para `APP_GESTION_RRHH` y `APP_FINANZAS_ERP`, filtra por organización al agregar roles (pestaña **Scope**), de modo que solo los roles del departamento correspondiente tengan acceso.
   - Guarda.

4. Configura la **delegación de administración** para el manager de RRHH:
   - Ve a **Role Management → Delegation Policies**.
   - Haz clic en **+ New Policy**.
   - **Name:** `DELEG_MANAGER_RRHH`
   - **Delegator Role:** `BR_MANAGER_DEPT`
   - **Scope Organization:** `Departamento RRHH`
   - **Allowed Actions:** `assign_role`, `revoke_role`, `view_user`
   - **Restricted Roles:** Solo roles con organización `Departamento RRHH` (excluye IT y Finanzas).
   - Guarda la política.

#### Resultado esperado

```
Resources (3):
  ✓ APP_PORTAL_EMPLEADO   [Active] — 3 roles asignados
  ✓ APP_GESTION_RRHH      [Active] — 2 roles asignados (scope: RRHH)
  ✓ APP_FINANZAS_ERP      [Active] — 2 roles asignados (scope: FIN)

Delegation Policies (1):
  ✓ DELEG_MANAGER_RRHH    [Active] — scope: RRHH only
```

#### Verificación

Intenta (como ejercicio mental o con un usuario de prueba en el Paso 4) que un usuario con `BR_MANAGER_DEPT` en RRHH **no pueda** ver ni modificar roles de Finanzas. La política de delegación debe bloquearlo.

---

### Paso 4 — Generar y Cargar 15 Usuarios de Prueba

**Objetivo:** Crear 15 usuarios ficticios con asignaciones de roles variadas, incluyendo **tres asignaciones incorrectas intencionadas** que serán detectadas en la campaña de revisión.

#### Instrucciones

1. Crea el script de generación de datos ficticios:

```bash
cat > ~/lab-05-00-01/scripts/generate_users.py << 'PYEOF'
#!/usr/bin/env python3
"""
Lab 05-00-01 — Generador de usuarios ficticios con Faker.
ADVERTENCIA: Todos los datos son completamente ficticios.
"""
import json
import random
from faker import Faker

fake = Faker('es_ES')
random.seed(42)  # Reproducible

DEPARTMENTS = ["IT", "RRHH", "FIN"]
DEPT_NAMES = {"IT": "Departamento IT", "RRHH": "Departamento RRHH", "FIN": "Departamento Finanzas"}

# Asignaciones normales: (dept, role)
NORMAL_ASSIGNMENTS = [
    ("IT",   "BR_EMPLEADO"),
    ("IT",   "BR_ESPECIALISTA"),
    ("IT",   "BR_MANAGER_DEPT"),
    ("RRHH", "BR_EMPLEADO"),
    ("RRHH", "BR_EMPLEADO"),
    ("RRHH", "BR_ESPECIALISTA"),
    ("RRHH", "BR_MANAGER_DEPT"),
    ("FIN",  "BR_EMPLEADO"),
    ("FIN",  "BR_EMPLEADO"),
    ("FIN",  "BR_ESPECIALISTA"),
    ("FIN",  "BR_MANAGER_DEPT"),
    ("IT",   "BR_EMPLEADO"),
]

# Asignaciones INCORRECTAS intencionadas (para la revisión)
# Caso 1: Empleado IT con acceso a APP_FINANZAS_ERP (acceso cruzado indebido)
# Caso 2: Empleado RRHH con BR_MANAGER_DEPT de Finanzas (elevación de privilegios)
# Caso 3: Empleado FIN con BR_ESPECIALISTA + BR_MANAGER_DEPT (violación SoD)
INCORRECT_ASSIGNMENTS = [
    ("IT",   "BR_ESPECIALISTA", "FIN",  True,  "CROSS_DEPT_ACCESS"),
    ("RRHH", "BR_MANAGER_DEPT", "FIN",  True,  "PRIVILEGE_ESCALATION"),
    ("FIN",  "BR_ESPECIALISTA", "FIN",  True,  "SOD_VIOLATION"),
]

users = []

# Usuarios normales
for dept, role in NORMAL_ASSIGNMENTS:
    first = fake.first_name()
    last = fake.last_name()
    username = f"{first.lower()}.{last.lower().replace(' ', '')}"[:20]
    users.append({
        "username": username,
        "firstName": first,
        "lastName": last,
        "email": fake.email(),
        "password": "Temporal@2024!",
        "department": dept,
        "departmentName": DEPT_NAMES[dept],
        "role": role,
        "roleOrg": dept,
        "isIncorrect": False,
        "incorrectReason": None
    })

# Usuarios con asignaciones incorrectas
for dept, role, role_org, is_incorrect, reason in INCORRECT_ASSIGNMENTS:
    first = fake.first_name()
    last = fake.last_name()
    username = f"{first.lower()}.{last.lower().replace(' ', '')}"[:20]
    users.append({
        "username": username,
        "firstName": first,
        "lastName": last,
        "email": fake.email(),
        "password": "Temporal@2024!",
        "department": dept,
        "departmentName": DEPT_NAMES[dept],
        "role": role,
        "roleOrg": role_org,
        "isIncorrect": is_incorrect,
        "incorrectReason": reason
    })

# Guardar
with open("/tmp/lab05_users.json", "w", encoding="utf-8") as f:
    json.dump(users, f, ensure_ascii=False, indent=2)

print(f"✓ Generados {len(users)} usuarios ficticios en /tmp/lab05_users.json")
print(f"  - Usuarios normales: {len(NORMAL_ASSIGNMENTS)}")
print(f"  - Usuarios con asignaciones incorrectas: {len(INCORRECT_ASSIGNMENTS)}")

# Mostrar resumen
for u in users:
    marker = " ⚠️  INCORRECTO" if u["isIncorrect"] else ""
    print(f"  {u['username']:25s} | {u['department']:4s} | {u['role']:20s}{marker}")
PYEOF

python3 ~/lab-05-00-01/scripts/generate_users.py
```

2. Crea el script de carga masiva en OpenIAM vía API REST:

```bash
cat > ~/lab-05-00-01/scripts/load_users.py << 'PYEOF'
#!/usr/bin/env python3
"""
Lab 05-00-01 — Carga masiva de usuarios en OpenIAM vía REST API.
"""
import json
import sys
import requests
import time

BASE_URL = "http://localhost:8080/openiam-rest/api/1"
ADMIN_USER = "sysadmin"
ADMIN_PASS = "Admin@Lab05!"

session = requests.Session()
session.auth = (ADMIN_USER, ADMIN_PASS)
session.headers.update({"Content-Type": "application/json"})

def create_user(user_data: dict) -> str | None:
    """Crea un usuario en OpenIAM y devuelve su ID."""
    payload = {
        "login": user_data["username"],
        "firstName": user_data["firstName"],
        "lastName": user_data["lastName"],
        "email": user_data["email"],
        "password": user_data["password"],
        "organizationName": user_data["departmentName"],
        "userAttributes": [
            {"name": "dept_code", "value": user_data["department"]},
            {"name": "isIncorrectAssignment", "value": str(user_data["isIncorrect"])},
            {"name": "incorrectReason", "value": user_data.get("incorrectReason", "")}
        ]
    }
    try:
        resp = session.post(f"{BASE_URL}/user/add", json=payload, timeout=15)
        resp.raise_for_status()
        user_id = resp.json().get("userId") or resp.json().get("id")
        return user_id
    except requests.RequestException as e:
        print(f"  ✗ Error creando {user_data['username']}: {e}")
        return None

def assign_role(user_id: str, role_name: str, org_dept: str) -> bool:
    """Asigna un rol a un usuario en el contexto de una organización."""
    payload = {
        "userId": user_id,
        "roleName": role_name,
        "organizationDept": org_dept
    }
    try:
        resp = session.post(f"{BASE_URL}/role/user/add", json=payload, timeout=15)
        resp.raise_for_status()
        return True
    except requests.RequestException as e:
        print(f"  ✗ Error asignando rol {role_name} a {user_id}: {e}")
        return False

# Cargar usuarios
with open("/tmp/lab05_users.json", "r", encoding="utf-8") as f:
    users = json.load(f)

print(f"Cargando {len(users)} usuarios en OpenIAM...")
results = {"ok": 0, "error": 0}

for u in users:
    marker = " [INCORRECTO]" if u["isIncorrect"] else ""
    print(f"  → {u['username']}{marker}", end=" ")
    uid = create_user(u)
    if uid:
        role_ok = assign_role(uid, u["role"], u["roleOrg"])
        if role_ok:
            print(f"✓ (ID: {uid[:8]}...)")
            results["ok"] += 1
        else:
            print(f"⚠ usuario creado pero rol no asignado")
            results["error"] += 1
    else:
        print("✗")
        results["error"] += 1
    time.sleep(0.3)  # Rate limiting

print(f"\nResumen: {results['ok']} OK, {results['error']} errores")
PYEOF

python3 ~/lab-05-00-01/scripts/load_users.py
```

#### Resultado esperado

```
Cargando 15 usuarios en OpenIAM...
  → ana.garcia           ✓ (ID: a1b2c3d4...)
  → carlos.lopez         ✓ (ID: e5f6g7h8...)
  ...
  → miguel.torres [INCORRECTO] ✓ (ID: x1y2z3w4...)
  → lucia.martin  [INCORRECTO] ✓ (ID: p9q8r7s6...)
  → pedro.santos  [INCORRECTO] ✓ (ID: m5n4o3p2...)

Resumen: 15 OK, 0 errores
```

#### Verificación

En OpenIAM UI → **User Management → Users**, filtra por organización y verifica que aparecen los 15 usuarios distribuidos entre los tres departamentos.

---

### Paso 5 — Configurar y Ejecutar una Campaña de Certificación de Accesos

**Objetivo:** Crear una campaña de *access review* (certificación de accesos) que cubra todos los usuarios y roles de la organización, ejecutarla como revisor y documentar las decisiones de aprobación/revocación.

#### Instrucciones

1. Ve a **Compliance → Access Certifications → Campaigns**.

2. Haz clic en **+ New Campaign** y configura:

   | Campo                   | Valor                                              |
   |-------------------------|----------------------------------------------------|
   | **Campaign Name**       | `CERT_Q1_2024_FullOrg`                             |
   | **Description**         | Revisión trimestral Q1 2024 — Todos los departamentos |
   | **Campaign Type**       | `User Access Review`                               |
   | **Scope**               | All Users — All Roles                              |
   | **Reviewer**            | `sysadmin` (en producción: manager de cada dpto.)  |
   | **Escalation Reviewer** | `sysadmin`                                         |
   | **Due Date**            | Hoy + 7 días                                       |
   | **Reminder Frequency**  | Daily                                              |

3. En la pestaña **Filters**, agrega el filtro:
   - **Include Roles:** `BR_EMPLEADO`, `BR_ESPECIALISTA`, `BR_MANAGER_DEPT`
   - **Include Organizations:** Las tres organizaciones creadas.

4. Haz clic en **Launch Campaign**. OpenIAM generará automáticamente las tareas de revisión (*review items*) para cada par usuario–rol.

5. Ve a **Compliance → My Certifications** para ver tu bandeja de revisión.

6. Revisa cada ítem siguiendo estas reglas de decisión:

   | Condición detectada                                    | Acción          | Justificación                                |
   |--------------------------------------------------------|-----------------|----------------------------------------------|
   | Usuario en dept. correcto con rol apropiado            | **Approve** ✓   | Asignación válida según política RBAC        |
   | Usuario IT con acceso a APP_FINANZAS_ERP               | **Revoke** ✗    | Acceso cruzado no autorizado (CROSS_DEPT)    |
   | Usuario RRHH con BR_MANAGER_DEPT en Finanzas           | **Revoke** ✗    | Escalación de privilegios no autorizada      |
   | Usuario FIN con BR_ESPECIALISTA + BR_MANAGER_DEPT      | **Revoke** ✗    | Violación de regla SoD_Finanzas_CrearAprobar |

   Para cada ítem en la UI:
   - Haz clic en el ítem.
   - Selecciona **Approve** o **Revoke**.
   - En el campo **Comments**, escribe una justificación (obligatorio para Revoke).
   - Haz clic en **Submit Decision**.

7. Una vez revisados todos los ítems, ve a **Compliance → Access Certifications → Campaigns** → selecciona `CERT_Q1_2024_FullOrg` → haz clic en **Close Campaign**.

8. OpenIAM ejecutará automáticamente las acciones de remediación (revocación de roles marcados como Revoke).

#### Resultado esperado

```
Campaign: CERT_Q1_2024_FullOrg
  Status: CLOSED
  Total Review Items: 15
  ✓ Approved: 12
  ✗ Revoked:   3
    - [CROSS_DEPT_ACCESS]      → rol revocado
    - [PRIVILEGE_ESCALATION]   → rol revocado
    - [SOD_VIOLATION]          → rol revocado
  Completion: 100%
```

#### Verificación

Verifica que los 3 usuarios con asignaciones incorrectas ya no tienen los roles revocados:

```bash
# Verificar vía API que los roles incorrectos fueron revocados
curl -s -u sysadmin:Admin@Lab05! \
  "http://localhost:8080/openiam-rest/api/1/user/roles?username=<usuario_incorrecto>" \
  -H "Content-Type: application/json" | python3 -m json.tool
```

---

### Paso 6 — Generar y Analizar Informes de Accesos

**Objetivo:** Generar los cuatro informes de cumplimiento requeridos y analizar los resultados para identificar brechas y el estado post-revisión.

#### Instrucciones

1. Ve a **Reports → Access Reports** en OpenIAM.

2. **Informe 1 — Usuarios por Rol:**
   - Selecciona reporte: `Users by Role`
   - Filtros: Todas las organizaciones, Todos los roles de negocio.
   - Formato: PDF y CSV.
   - Haz clic en **Generate** → **Download**.
   - Guarda en `~/lab-05-00-01/reports/01_usuarios_por_rol.csv`.

3. **Informe 2 — Roles por Usuario:**
   - Selecciona reporte: `Roles by User`
   - Filtros: Todos los usuarios activos.
   - Formato: PDF y CSV.
   - Guarda en `~/lab-05-00-01/reports/02_roles_por_usuario.csv`.

4. **Informe 3 — Accesos Pendientes / Revisados:**
   - Selecciona reporte: `Certification Campaign Summary`
   - Filtros: Campaign = `CERT_Q1_2024_FullOrg`.
   - Formato: PDF.
   - Guarda en `~/lab-05-00-01/reports/03_certification_summary.pdf`.

5. **Informe 4 — Reporte de Cumplimiento (Compliance):**
   - Selecciona reporte: `Compliance Status Report`
   - Filtros: Incluir violaciones SoD, accesos cruzados.
   - Formato: PDF.
   - Guarda en `~/lab-05-00-01/reports/04_compliance_report.pdf`.

6. Analiza el CSV del Informe 1 con Python para obtener estadísticas rápidas:

```bash
python3 << 'PYEOF'
import csv
from collections import Counter

with open("/root/lab-05-00-01/reports/01_usuarios_por_rol.csv", "r") as f:
    reader = csv.DictReader(f)
    rows = list(reader)

role_counts = Counter(row.get("role_name", row.get("roleName", "unknown")) for row in rows)
dept_counts = Counter(row.get("department", row.get("dept_code", "unknown")) for row in rows)

print("=== Usuarios por Rol ===")
for role, count in sorted(role_counts.items()):
    print(f"  {role:30s}: {count} usuarios")

print("\n=== Usuarios por Departamento ===")
for dept, count in sorted(dept_counts.items()):
    print(f"  {dept:10s}: {count} usuarios")

print(f"\nTotal usuarios activos: {len(rows)}")
PYEOF
```

#### Resultado esperado

```
=== Usuarios por Rol ===
  BR_EMPLEADO                   : 6 usuarios
  BR_ESPECIALISTA               : 4 usuarios   (post-revocación: 3 en sus dptos.)
  BR_MANAGER_DEPT               : 3 usuarios   (post-revocación: 3 válidos)

=== Usuarios por Departamento ===
  FIN       : 5 usuarios
  IT        : 5 usuarios
  RRHH      : 5 usuarios

Total usuarios activos: 15
```

> **Nota de análisis:** Los 3 usuarios con asignaciones revocadas siguen existiendo como usuarios activos, pero con sus roles incorrectos eliminados. Esto refleja el principio de que la revisión de accesos no elimina identidades, solo ajusta privilegios.

#### Verificación

Confirma que los reportes se han guardado correctamente:

```bash
ls -lh ~/lab-05-00-01/reports/
# Debe mostrar los 4 archivos de reporte
```

---

### Paso 7 — Extensión: Añadir Condiciones ABAC Simples sobre la Base RBAC

**Objetivo conceptual:** Discutir y configurar una condición ABAC básica que enriquece el modelo RBAC con el atributo `dept_code`, implementando el patrón híbrido estudiado en la Lección 5.1 (RBAC para derechos base + ABAC para condiciones contextuales).

#### Instrucciones

1. En OpenIAM, ve a **Policy Management → Authorization Policies**.

2. Haz clic en **+ New Policy** y configura la política ABAC:

   | Campo              | Valor                                                        |
   |--------------------|--------------------------------------------------------------|
   | **Policy Name**    | `ABAC_DEPT_SCOPE_FINANZAS`                                   |
   | **Description**    | Solo usuarios con dept_code=FIN pueden acceder a APP_FINANZAS_ERP |
   | **Resource**       | `APP_FINANZAS_ERP`                                           |
   | **Effect**         | `Permit`                                                     |

3. Agrega la condición ABAC:
   - **Attribute Source:** User Attribute
   - **Attribute Name:** `dept_code`
   - **Operator:** `equals`
   - **Value:** `FIN`
   - **Combine With:** `AND` (con el rol existente)

4. Guarda la política. Esto implementa el patrón híbrido:

```
Decisión de acceso a APP_FINANZAS_ERP:
  RBAC (base):  usuario tiene BR_ESPECIALISTA o BR_MANAGER_DEPT
  AND
  ABAC (condición): usuario.dept_code == "FIN"
  ──────────────────────────────────────────────────────
  Resultado: PERMIT solo si AMBAS condiciones son verdaderas
```

5. Relaciona este diseño con la arquitectura PEP/PDP/PAP/PIP de la Lección 5.1:

   ```
   PEP → OpenIAM API Gateway (intercepta solicitud de acceso)
   PDP → OpenIAM Policy Engine (evalúa RBAC + ABAC_DEPT_SCOPE_FINANZAS)
   PAP → OpenIAM Admin UI (donde acabas de configurar la política)
   PIP → OpenIAM User Store (fuente del atributo dept_code)
   ```

6. Prueba la política con un usuario IT que tenga `BR_ESPECIALISTA` pero `dept_code=IT`:
   - En la UI, ve a **Policy Management → Policy Tester**.
   - **Subject:** usuario IT con BR_ESPECIALISTA.
   - **Resource:** `APP_FINANZAS_ERP`.
   - **Action:** `read`.
   - Haz clic en **Evaluate**.
   - **Resultado esperado:** `DENY` (tiene el rol pero no el atributo dept_code=FIN).

#### Resultado esperado

```
Policy Evaluation Test:
  Subject:  [usuario IT, BR_ESPECIALISTA, dept_code=IT]
  Resource: APP_FINANZAS_ERP
  Action:   read
  ─────────────────────────────────────────────
  RBAC check:  PERMIT (tiene BR_ESPECIALISTA)
  ABAC check:  DENY   (dept_code=IT ≠ FIN)
  Final:       DENY
  ─────────────────────────────────────────────
  Matching Policy: ABAC_DEPT_SCOPE_FINANZAS
  Rule: dept_code == "FIN" → FALSE
```

---

## 7. Validación y Pruebas Finales

Ejecuta el siguiente checklist de validación para confirmar que el lab está completo:

```bash
cat << 'EOF'
╔══════════════════════════════════════════════════════════════════╗
║         CHECKLIST DE VALIDACIÓN — Lab 05-00-01                  ║
╠══════════════════════════════════════════════════════════════════╣
║  [ ] 1. Tres organizaciones creadas (IT, RRHH, FIN)             ║
║  [ ] 2. 6 roles configurados (3 técnicos + 3 de negocio)        ║
║  [ ] 3. Jerarquía RBAC1 verificada (herencia de permisos)       ║
║  [ ] 4. Regla SoD_Finanzas_CrearAprobar activa                  ║
║  [ ] 5. 3 recursos protegidos con roles asignados               ║
║  [ ] 6. Política de delegación DELEG_MANAGER_RRHH activa        ║
║  [ ] 7. 15 usuarios ficticios cargados (datos Faker)            ║
║  [ ] 8. Campaña CERT_Q1_2024_FullOrg cerrada (100%)             ║
║  [ ] 9. 3 accesos revocados (incorrectos detectados)            ║
║  [ ] 10. 4 informes generados y guardados en /reports/          ║
║  [ ] 11. Política ABAC_DEPT_SCOPE_FINANZAS configurada          ║
║  [ ] 12. Policy Tester devuelve DENY para usuario IT en FIN ERP ║
╚══════════════════════════════════════════════════════════════════╝
EOF
```

**Prueba de regresión rápida** — verifica que el stack sigue operativo:

```bash
# Verificar contenedores
docker compose -f ~/lab-05-00-01/docker-compose.yml ps

# Verificar acceso a la UI
curl -s -o /dev/null -w "OpenIAM UI: %{http_code}\n" http://localhost:8080/openiam-ui/login

# Verificar PostgreSQL
docker exec lab05_postgres pg_isready -U openiam
```

**Resultado esperado:**

```
NAME              STATUS          PORTS
lab05_postgres    running (healthy)   0.0.0.0:5432->5432/tcp
lab05_openiam     running (healthy)   0.0.0.0:8080->8080/tcp

OpenIAM UI: 200
/var/run/postgresql:5432 - accepting connections
```

---

## 8. Resolución de Problemas

### Problema 1 — OpenIAM no arranca: error de conexión a PostgreSQL

**Síntomas:**
```
lab05_openiam | ERROR: Connection refused to postgres:5432
lab05_openiam | Caused by: org.postgresql.util.PSQLException: Connection refused
```
El contenedor `lab05_openiam` entra en bucle de reinicio (`Restarting`).

**Causa:**
OpenIAM intenta conectarse a PostgreSQL antes de que el healthcheck lo marque como listo. Puede ocurrir si el volumen de PostgreSQL tiene datos corruptos de un lab anterior, o si el contenedor de postgres tardó más de lo esperado en inicializar el schema.

**Solución:**
```bash
# 1. Detener todo el stack
docker compose -f ~/lab-05-00-01/docker-compose.yml down

# 2. Eliminar volúmenes (ADVERTENCIA: borra todos los datos del lab)
docker compose -f ~/lab-05-00-01/docker-compose.yml down -v

# 3. Verificar que no quedan volúmenes huérfanos
docker volume ls | grep lab05

# 4. Reiniciar solo PostgreSQL primero y esperar el healthcheck
docker compose -f ~/lab-05-00-01/docker-compose.yml up -d postgres
docker compose -f ~/lab-05-00-01/docker-compose.yml ps postgres
# Esperar hasta ver: (healthy)

# 5. Luego levantar OpenIAM
docker compose -f ~/lab-05-00-01/docker-compose.yml up -d openiam
docker compose -f ~/lab-05-00-01/docker-compose.yml logs -f openiam | grep -E "(Started|ERROR)"
```

---

### Problema 2 — El script `load_users.py` falla con error 401 o 403

**Síntomas:**
```
  ✗ Error creando ana.garcia: 401 Client Error: Unauthorized
  ✗ Error creando carlos.lopez: 403 Client Error: Forbidden
Resumen: 0 OK, 15 errores
```

**Causa:**
La API REST de OpenIAM requiere autenticación por sesión (cookie/token) en lugar de Basic Auth estándar en algunas versiones de la Community Edition. El script usa `requests.Session` con Basic Auth, que puede no ser compatible si OpenIAM requiere un token de sesión previo, o si la contraseña del admin fue modificada durante la instalación.

**Solución:**
```bash
# 1. Verificar la contraseña del admin (puede diferir si el contenedor ya existía)
docker exec lab05_openiam cat /opt/openiam/conf/admin.properties | grep password

# 2. Obtener token de sesión primero
TOKEN=$(curl -s -X POST http://localhost:8080/openiam-rest/api/1/auth/login \
  -H "Content-Type: application/json" \
  -d '{"login":"sysadmin","password":"Admin@Lab05!"}' \
  | python3 -c "import sys,json; print(json.load(sys.stdin).get('token',''))")

echo "Token obtenido: ${TOKEN:0:20}..."

# 3. Modificar el script para usar Bearer Token en lugar de Basic Auth
# Edita load_users.py: reemplaza session.auth = (...) por:
# session.headers.update({"Authorization": f"Bearer {TOKEN}"})

# 4. Re-ejecutar el script
python3 ~/lab-05-00-01/scripts/load_users.py

# Alternativa: crear usuarios manualmente en la UI para los 15 usuarios
# y usar el script solo para asignar roles
```

---

## 9. Limpieza del Entorno

Ejecuta los siguientes comandos para liberar recursos al finalizar el lab. Esto es **obligatorio** antes de iniciar el Lab 06-00-01.

```bash
cd ~/lab-05-00-01

# 1. Exportar los informes antes de limpiar (por si los necesitas para el portafolio)
cp -r ~/lab-05-00-01/reports ~/Desktop/lab05_reports_backup 2>/dev/null || true

# 2. Detener y eliminar contenedores, redes y volúmenes
docker compose down -v

# 3. Verificar que los contenedores fueron eliminados
docker ps -a | grep lab05
# No debe mostrar ningún resultado

# 4. Verificar que los volúmenes fueron eliminados
docker volume ls | grep lab05
# No debe mostrar ningún resultado

# 5. (Opcional) Eliminar imágenes para liberar espacio en disco
# ADVERTENCIA: Solo si no necesitas reutilizarlas en el Lab 06-00-01
# docker rmi openiam/openiam-community:4.2 postgres:15-alpine

# 6. Verificar memoria liberada
free -h
```

**Resultado esperado de limpieza:**

```bash
$ docker compose down -v
[+] Running 4/4
 ✔ Container lab05_openiam   Removed
 ✔ Container lab05_postgres  Removed
 ✔ Volume pgdata_lab05       Removed
 ✔ Volume openiam_data_lab05 Removed
 ✔ Network lab05_net         Removed

$ docker ps -a | grep lab05
# (sin resultados)

$ free -h
              total    used    free
Mem:           32Gi    8.2Gi   23.8Gi   # Memoria recuperada
```

---

## 10. Resumen

En este laboratorio implementaste un ciclo completo de gobierno de accesos RBAC en OpenIAM:

1. **Jerarquía organizacional** — Tres departamentos con el atributo `dept_code` como base para la delegación y las condiciones ABAC.

2. **Modelo RBAC1** — Seis roles (3 técnicos + 3 de negocio) con herencia de permisos, restricción SoD y política de delegación departamental que limita al manager de RRHH a su propio ámbito.

3. **Datos de prueba ficticios** — 15 usuarios generados con Faker, incluyendo tres asignaciones intencionalmente incorrectas que representan escenarios reales de auditoría: acceso cruzado entre departamentos, escalación de privilegios y violación de SoD.

4. **Campaña de certificación** — Ejecutaste un proceso de *access review* completo, tomando decisiones documentadas de aprobación y revocación, y verificando la remediación automática post-campaña.

5. **Informes de cumplimiento** — Generaste y analizaste cuatro tipos de informes que proporcionan visibilidad sobre el estado de accesos y sirven como evidencia para auditorías.

6. **Extensión ABAC** — Configuraste una condición ABAC simple (`dept_code`) sobre la base RBAC, implementando el patrón híbrido de la Lección 5.1 y relacionándolo con la arquitectura PEP/PDP/PAP/PIP.

### Conexión con el contenido teórico

| Concepto (Lección 5.1)                | Implementación en el lab                                      |
|---------------------------------------|---------------------------------------------------------------|
| RBAC1 — Jerarquía de roles            | `TR_ACCESO_BASICO → TR_ACCESO_INTERMEDIO → TR_ADMIN_DEPT`    |
| RBAC2 — Restricciones SoD             | `SoD_Finanzas_CrearAprobar` (Exclusive)                       |
| Principio de privilegio mínimo        | Roles técnicos atómicos + delegación por scope departamental  |
| Patrón híbrido RBAC + ABAC            | `ABAC_DEPT_SCOPE_FINANZAS` — condición `dept_code == "FIN"`  |
| Arquitectura PEP/PDP/PAP/PIP          | Mapeado a componentes OpenIAM (API GW / Policy Engine / UI / User Store) |
| Auditoría y trazabilidad              | Campaña de certificación con decisiones documentadas + informes |

### Recursos adicionales

- [OpenIAM 4.2 Community Edition — Documentación oficial](https://wiki.openiam.com)
- [NIST SP 800-162: Guide to Attribute Based Access Control (ABAC)](https://csrc.nist.gov/publications/detail/sp/800-162/final)
- [NIST RBAC Project — Modelos y estándares](https://csrc.nist.gov/projects/role-based-access-control)
- [Repositorio del curso — Scripts de datos ficticios con Faker](https://github.com/curso-iam/lab-scripts)
- **Próximo lab (06-00-01):** Aprovisionamiento e integración de identidades con MidPoint y OpenLDAP — requiere conocimiento práctico de este lab.

---
*Todos los datos de usuarios, contraseñas y nombres organizacionales utilizados en este laboratorio son completamente ficticios y generados mediante la librería Python Faker. Ningún dato corresponde a personas reales.*
