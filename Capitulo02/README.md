# Diseñar un flujo de aprovisionamiento y reconciliación

## 1. Metadatos

| Campo            | Valor                                                                 |
|------------------|-----------------------------------------------------------------------|
| **Duración**     | 30 minutos                                                            |
| **Complejidad**  | Media                                                                 |
| **Nivel Bloom**  | Crear                                                                 |
| **Módulo**       | 2 — Gestión del ciclo de vida, alimentación y políticas de provisión  |
| **Lab ID**       | 02-00-01                                                              |

---

## 2. Descripción General

En este laboratorio el estudiante desplegará un entorno compuesto por **MidPoint 4.8 LTS** (plataforma IGA) conectado a un repositorio **OpenLDAP 2.6**, ambos ejecutándose como contenedores Docker. Partiendo de un archivo CSV que simula una fuente de datos de Recursos Humanos con 20 usuarios ficticios, se configurará un recurso de importación en MidPoint, se definirán *mappings* de atributos y se establecerán políticas de ciclo de vida (alta, modificación, desactivación y eliminación). El laboratorio culmina con la ejecución de una reconciliación que detecta y resuelve discrepancias, y con la documentación del flujo mediante un diagrama de proceso.

---

## 3. Objetivos de Aprendizaje

Al finalizar este laboratorio, el estudiante será capaz de:

- [ ] Desplegar y verificar un entorno MidPoint + OpenLDAP usando Docker Compose.
- [ ] Generar datos de prueba ficticios con Python y configurar el recurso CSV en MidPoint.
- [ ] Definir *mappings* de atributos y políticas de ciclo de vida (Joiner / Mover / Leaver) en MidPoint.
- [ ] Ejecutar aprovisionamiento inicial y una reconciliación que detecte y resuelva discrepancias.
- [ ] Documentar el flujo de aprovisionamiento mediante un diagrama de proceso en draw.io.

---

## 4. Prerrequisitos

### Conocimiento previo

| Área                              | Nivel requerido |
|-----------------------------------|-----------------|
| Estructura LDAP (DN, OU, atributos) | Básico         |
| Formato CSV y mapeo de datos      | Básico          |
| Conceptos JML (Joiner-Mover-Leaver) | Conceptual — Módulo 2 |
| Docker y Docker Compose           | Operativo       |

### Acceso y herramientas

| Herramienta                 | Versión mínima | Notas                                      |
|-----------------------------|----------------|--------------------------------------------|
| Docker Engine               | 24.0+          | Con el servicio activo                     |
| Docker Compose              | 2.20+          | Plugin integrado (`docker compose`)        |
| Python 3                    | 3.10+          | Con `pip` disponible                       |
| Apache Directory Studio     | 2.0.0          | Para inspección LDAP                       |
| Navegador web               | Cualquier moderno | Para acceder a la UI de MidPoint        |
| draw.io / diagrams.net      | Online o Desktop | Para el diagrama final                  |

> **Nota:** Este laboratorio **no** requiere cuenta AWS ni certificados TLS. Todos los servicios se ejecutan en `localhost`.

---

## 5. Entorno de Laboratorio

### Recursos de hardware recomendados

| Recurso   | Mínimo  | Recomendado |
|-----------|---------|-------------|
| RAM       | 6 GB libres | 8 GB libres |
| CPU       | 2 núcleos | 4 núcleos  |
| Disco     | 5 GB libres | 10 GB libres |

### Arquitectura del entorno

```
┌─────────────────────────────────────────────────────────────┐
│                    Docker Network: iam-net                   │
│                                                             │
│  ┌──────────────────┐        ┌──────────────────────────┐  │
│  │   MidPoint 4.8   │───────▶│     OpenLDAP 2.6         │  │
│  │   :8080          │  LDAP  │     :389                 │  │
│  │   (IGA Engine)   │        │   dc=empresa,dc=local    │  │
│  └────────┬─────────┘        └──────────────────────────┘  │
│           │                                                  │
│           │ JDBC                                             │
│  ┌────────▼─────────┐                                       │
│  │  PostgreSQL 15   │                                       │
│  │  :5432           │                                       │
│  └──────────────────┘                                       │
└─────────────────────────────────────────────────────────────┘
         ▲
         │ CSV (HR simulado)
    [Host: ~/lab02/data/hr_users.csv]
```

### Preparación inicial del directorio de trabajo

Ejecuta los siguientes comandos en tu terminal **antes** de comenzar los pasos del laboratorio:

```bash
# Crear estructura de directorios del laboratorio
mkdir -p ~/lab02/{compose,data,midpoint-config/{resources,roles,tasks},ldap-config}
cd ~/lab02
```

---

## 6. Procedimiento Paso a Paso

---

### Paso 1: Generar datos ficticios de Recursos Humanos (CSV)

**Objetivo:** Crear el archivo CSV que simulará la fuente de verdad de RR. HH. con 20 usuarios completamente ficticios, siguiendo el patrón de ingesta por lotes descrito en la lección.

#### Instrucciones

**1.1** Instalar la librería `Faker` de Python:

```bash
pip3 install faker --quiet
```

**1.2** Crear el script de generación de datos:

```bash
cat > ~/lab02/data/generate_hr.py << 'EOF'
#!/usr/bin/env python3
"""
Script de generación de datos ficticios de RR. HH.
AVISO: Todos los datos son completamente ficticios. No representan personas reales.
"""
import csv
import random
from faker import Faker
from datetime import date, timedelta

fake = Faker('es_ES')
random.seed(42)  # Semilla fija para reproducibilidad

DEPARTMENTS = ["Finanzas", "TI", "RRHH", "Operaciones", "Legal", "Marketing"]
JOB_TITLES  = ["Analista", "Coordinador", "Gerente", "Especialista", "Director"]
EMPLOYEE_TYPES = ["FTE", "FTE", "FTE", "CONTRACTOR"]  # 75% FTE
STATUSES = ["active", "active", "active", "active", "inactive"]  # 80% activos

fieldnames = [
    "employeeNumber", "uid", "givenName", "sn", "cn",
    "mail", "department", "jobTitle", "employeeType",
    "status", "startDate", "manager"
]

users = []
used_uids = set()

for i in range(1, 21):
    first = fake.first_name()
    last  = fake.last_name()
    emp_num = f"EMP{i:04d}"

    # Generar uid único
    base_uid = f"{first[0].lower()}{last.lower().replace(' ', '')[:8]}"
    uid = base_uid
    counter = 1
    while uid in used_uids:
        uid = f"{base_uid}{counter}"
        counter += 1
    used_uids.add(uid)

    start = date.today() - timedelta(days=random.randint(30, 1825))
    dept  = random.choice(DEPARTMENTS)
    mgr_num = f"EMP{random.randint(1, 5):04d}" if i > 5 else "EMP0001"

    users.append({
        "employeeNumber": emp_num,
        "uid":            uid,
        "givenName":      first,
        "sn":             last,
        "cn":             f"{first} {last}",
        "mail":           f"{uid}@empresa.local",
        "department":     dept,
        "jobTitle":       random.choice(JOB_TITLES),
        "employeeType":   random.choice(EMPLOYEE_TYPES),
        "status":         random.choice(STATUSES),
        "startDate":      start.isoformat(),
        "manager":        mgr_num,
    })

output_file = "hr_users.csv"
with open(output_file, "w", newline="", encoding="utf-8") as f:
    writer = csv.DictWriter(f, fieldnames=fieldnames)
    writer.writeheader()
    writer.writerows(users)

print(f"[OK] Generados {len(users)} usuarios ficticios en '{output_file}'")
for u in users[:3]:
    print(f"     {u['employeeNumber']} | {u['uid']} | {u['cn']} | {u['status']}")
print("     ...")
EOF
```

**1.3** Ejecutar el script:

```bash
cd ~/lab02/data
python3 generate_hr.py
```

**1.4** Verificar el CSV generado:

```bash
head -5 ~/lab02/data/hr_users.csv
wc -l ~/lab02/data/hr_users.csv
```

#### Salida esperada

```
[OK] Generados 20 usuarios ficticios en 'hr_users.csv'
     EMP0001 | jgarcia | Juan García | active
     EMP0002 | mlopez  | María López | active
     EMP0003 | cmarti  | Carlos Martí | inactive
     ...
employeeNumber,uid,givenName,sn,cn,mail,department,jobTitle,employeeType,status,startDate,manager
EMP0001,jgarcia,Juan,García,Juan García,jgarcia@empresa.local,TI,Analista,FTE,active,2022-03-15,EMP0001
...
21 hr_users.csv   ← (1 cabecera + 20 registros)
```

#### Verificación

```bash
# Contar registros (debe ser 20, sin contar cabecera)
tail -n +2 ~/lab02/data/hr_users.csv | wc -l
# Verificar que no hay UIDs duplicados
cut -d',' -f2 ~/lab02/data/hr_users.csv | sort | uniq -d
```

La segunda línea no debe producir ninguna salida (sin duplicados).

---

### Paso 2: Desplegar el entorno con Docker Compose

**Objetivo:** Levantar los contenedores de PostgreSQL, OpenLDAP y MidPoint en una red Docker aislada, replicando la arquitectura de integración descrita en la lección (conectores LDAP on-premise).

#### Instrucciones

**2.1** Crear el archivo de configuración inicial de OpenLDAP:

```bash
cat > ~/lab02/ldap-config/bootstrap.ldif << 'EOF'
# Unidades organizativas base para empresa.local
dn: ou=People,dc=empresa,dc=local
objectClass: organizationalUnit
ou: People

dn: ou=Groups,dc=empresa,dc=local
objectClass: organizationalUnit
ou: Groups

dn: ou=Disabled,dc=empresa,dc=local
objectClass: organizationalUnit
ou: Disabled
EOF
```

**2.2** Crear el archivo `docker-compose.yml`:

```bash
cat > ~/lab02/compose/docker-compose.yml << 'EOF'
version: "3.9"

networks:
  iam-net:
    driver: bridge

volumes:
  postgres-data:
  openldap-data:
  openldap-config:
  midpoint-home:

services:

  # ─── PostgreSQL (repositorio de MidPoint) ───────────────────────────────────
  postgres:
    image: postgres:15-alpine
    container_name: lab02-postgres
    environment:
      POSTGRES_DB:       midpoint
      POSTGRES_USER:     midpoint
      POSTGRES_PASSWORD: midpoint_secret
    volumes:
      - postgres-data:/var/lib/postgresql/data
    networks:
      - iam-net
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U midpoint"]
      interval: 10s
      timeout: 5s
      retries: 5

  # ─── OpenLDAP ────────────────────────────────────────────────────────────────
  openldap:
    image: osixia/openldap:1.5.0
    container_name: lab02-openldap
    environment:
      LDAP_ORGANISATION:    "Empresa Local"
      LDAP_DOMAIN:          "empresa.local"
      LDAP_BASE_DN:         "dc=empresa,dc=local"
      LDAP_ADMIN_PASSWORD:  "ldap_admin_secret"
      LDAP_CONFIG_PASSWORD: "ldap_config_secret"
      LDAP_READONLY_USER:   "true"
      LDAP_READONLY_USER_USERNAME: "readonly"
      LDAP_READONLY_USER_PASSWORD: "readonly_secret"
    volumes:
      - openldap-data:/var/lib/ldap
      - openldap-config:/etc/ldap/slapd.d
      - ../ldap-config/bootstrap.ldif:/container/service/slapd/assets/config/bootstrap/ldif/custom/bootstrap.ldif
    ports:
      - "389:389"
    networks:
      - iam-net
    healthcheck:
      test: ["CMD", "ldapsearch", "-x", "-H", "ldap://localhost", "-b", "dc=empresa,dc=local", "-D", "cn=admin,dc=empresa,dc=local", "-w", "ldap_admin_secret"]
      interval: 15s
      timeout: 10s
      retries: 5

  # ─── MidPoint ────────────────────────────────────────────────────────────────
  midpoint:
    image: evolveum/midpoint:4.8-alpine
    container_name: lab02-midpoint
    depends_on:
      postgres:
        condition: service_healthy
      openldap:
        condition: service_healthy
    environment:
      MP_SET_midpoint_repository_database:             postgresql
      MP_SET_midpoint_repository_jdbcUrl:              jdbc:postgresql://postgres:5432/midpoint
      MP_SET_midpoint_repository_jdbcUsername:         midpoint
      MP_SET_midpoint_repository_jdbcPassword:         midpoint_secret
      MP_SET_midpoint_repository_hibernateHbm2ddl:     update
      MIDPOINT_ADMINISTRATOR_INITIAL_PASSWORD:         "Admin1234!"
    volumes:
      - midpoint-home:/opt/midpoint/var
      - ../midpoint-config:/opt/midpoint/var/import:ro
      - ../data:/opt/midpoint/var/hr-data
    ports:
      - "8080:8080"
    networks:
      - iam-net
    healthcheck:
      test: ["CMD-SHELL", "curl -sf http://localhost:8080/midpoint/actuator/health | grep -q UP || exit 1"]
      interval: 30s
      timeout: 15s
      retries: 10
      start_period: 120s
EOF
```

**2.3** Iniciar el entorno:

```bash
cd ~/lab02/compose
docker compose up -d
echo "Esperando inicialización de MidPoint (puede tardar 2-3 minutos)..."
```

**2.4** Monitorear el inicio de MidPoint:

```bash
# Seguir los logs hasta ver "MidPoint started"
docker logs -f lab02-midpoint 2>&1 | grep -E "(started|ERROR|WARN|MidPoint)" | head -20
```

> **Paciencia:** MidPoint tarda entre 2 y 4 minutos en inicializar completamente la primera vez.

**2.5** Verificar el estado de todos los contenedores:

```bash
docker compose ps
```

#### Salida esperada

```
NAME               IMAGE                        STATUS
lab02-postgres     postgres:15-alpine           Up (healthy)
lab02-openldap     osixia/openldap:1.5.0        Up (healthy)
lab02-midpoint     evolveum/midpoint:4.8-alpine  Up (healthy)
```

#### Verificación

```bash
# Verificar conectividad LDAP
ldapsearch -x -H ldap://localhost:389 \
  -D "cn=admin,dc=empresa,dc=local" \
  -w "ldap_admin_secret" \
  -b "dc=empresa,dc=local" \
  "(objectClass=organizationalUnit)" dn 2>/dev/null | grep "dn:"

# Verificar UI de MidPoint
curl -s -o /dev/null -w "%{http_code}" http://localhost:8080/midpoint/
```

La búsqueda LDAP debe mostrar las tres OUs (`People`, `Groups`, `Disabled`). El curl debe devolver `200` o `302`.

---

### Paso 3: Configurar el recurso CSV (fuente HR) en MidPoint

**Objetivo:** Definir el recurso CSV en MidPoint que representa la fuente autoritativa de RR. HH., estableciendo los *mappings* de atributos que correlacionan campos del CSV con atributos del esquema interno de MidPoint. Esto implementa el patrón de **ingesta por lotes** descrito en la lección.

#### Instrucciones

**3.1** Acceder a la interfaz web de MidPoint:

Abre en tu navegador: `http://localhost:8080/midpoint`

- **Usuario:** `administrator`
- **Contraseña:** `Admin1234!`

**3.2** Crear el archivo de definición del recurso CSV. Este archivo XML define el conector, los *mappings* de atributos y la política de correlación:

```bash
cat > ~/lab02/midpoint-config/resources/resource-csv-hr.xml << 'EOF'
<?xml version="1.0" encoding="UTF-8"?>
<!--
  Recurso CSV: Fuente HR simulada
  Implementa el patrón de ingesta por lotes (CSV) descrito en la lección 2.1
  Atributos mapeados: employeeNumber (correlación), uid, cn, sn, givenName, mail,
                      department, employeeType, status (política de activación)
-->
<resource xmlns="http://midpoint.evolveum.com/xml/ns/public/common/common-3"
          xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
          xmlns:icfs="http://midpoint.evolveum.com/xml/ns/public/connector/icf-1/resource-schema-3"
          xmlns:ri="http://midpoint.evolveum.com/xml/ns/public/resource/instance-3"
          oid="a1b2c3d4-0001-0000-0000-000000000001">

  <name>HR-CSV-Source</name>
  <description>Fuente autoritativa de Recursos Humanos (CSV simulado). Patrón: ingesta por lotes.</description>

  <connectorRef type="ConnectorType">
    <filter>
      <q:equal xmlns:q="http://prism.evolveum.com/xml/ns/public/query-3">
        <q:path>connectorType</q:path>
        <q:value>com.evolveum.polygon.connector.csv.CsvConnector</q:value>
      </q:equal>
    </filter>
  </connectorRef>

  <connectorConfiguration xmlns:icfc="http://midpoint.evolveum.com/xml/ns/public/connector/icf-1/connector-schema-3">
    <icfc:configurationProperties>
      <!-- Ruta al CSV dentro del contenedor MidPoint -->
      <icfi:filePath xmlns:icfi="http://midpoint.evolveum.com/xml/ns/public/connector/icf-1/bundle/com.evolveum.polygon.connector-csv/com.evolveum.polygon.connector.csv.CsvConnector">
        /opt/midpoint/var/hr-data/hr_users.csv
      </icfi:filePath>
      <icfi:uniqueAttribute xmlns:icfi="http://midpoint.evolveum.com/xml/ns/public/connector/icf-1/bundle/com.evolveum.polygon.connector-csv/com.evolveum.polygon.connector.csv.CsvConnector">
        employeeNumber
      </icfi:uniqueAttribute>
      <icfi:passwordAttribute xmlns:icfi="http://midpoint.evolveum.com/xml/ns/public/connector/icf-1/bundle/com.evolveum.polygon.connector-csv/com.evolveum.polygon.connector.csv.CsvConnector">
        __PASSWORD__
      </icfi:passwordAttribute>
    </icfc:configurationProperties>
  </connectorConfiguration>

  <!-- ── Esquema del recurso ─────────────────────────────────────────────── -->
  <schemaHandling>
    <objectType>
      <kind>account</kind>
      <intent>default</intent>
      <displayName>Cuenta HR</displayName>
      <default>true</default>
      <objectClass>ri:AccountObjectClass</objectClass>

      <!-- ── Mappings INBOUND: CSV → MidPoint User ────────────────────── -->
      <attribute>
        <ref>ri:employeeNumber</ref>
        <inbound>
          <target><path>employeeNumber</path></target>
        </inbound>
      </attribute>

      <attribute>
        <ref>ri:uid</ref>
        <inbound>
          <target><path>name</path></target>
        </inbound>
      </attribute>

      <attribute>
        <ref>ri:givenName</ref>
        <inbound>
          <target><path>givenName</path></target>
        </inbound>
      </attribute>

      <attribute>
        <ref>ri:sn</ref>
        <inbound>
          <target><path>familyName</path></target>
        </inbound>
      </attribute>

      <attribute>
        <ref>ri:cn</ref>
        <inbound>
          <target><path>fullName</path></target>
        </inbound>
      </attribute>

      <attribute>
        <ref>ri:mail</ref>
        <inbound>
          <target><path>emailAddress</path></target>
        </inbound>
      </attribute>

      <attribute>
        <ref>ri:department</ref>
        <inbound>
          <target><path>organizationalUnit</path></target>
        </inbound>
      </attribute>

      <attribute>
        <ref>ri:employeeType</ref>
        <inbound>
          <target><path>employeeType</path></target>
        </inbound>
      </attribute>

      <!-- ── Política de activación basada en campo 'status' ──────────── -->
      <!-- Implementa el estado Active/Suspended de la máquina de estados ILM -->
      <attribute>
        <ref>ri:status</ref>
        <inbound>
          <expression>
            <script>
              <code>
                // Política de ciclo de vida: 'active' → habilitado; cualquier otro → deshabilitado
                input == 'active'
              </code>
            </script>
          </expression>
          <target>
            <path>activation/administrativeStatus</path>
            <set>
              <condition>
                <script>
                  <code>input == true ? 'enabled' : 'disabled'</code>
                </script>
              </condition>
            </set>
          </target>
        </inbound>
      </attribute>

      <!-- ── Correlación: identificador inmutable = employeeNumber ────── -->
      <!-- Principio de la lección: usar IDs inmutables para correlación estable -->
      <correlation>
        <correlators>
          <items>
            <item>
              <ref>employeeNumber</ref>
            </item>
          </items>
        </correlators>
      </correlation>

      <!-- ── Política de sincronización (reacción a situaciones) ──────── -->
      <synchronization>
        <!-- Joiner: nuevo en HR, no existe en MidPoint → crear -->
        <reaction>
          <situation>unmatched</situation>
          <action><handlerUri>http://midpoint.evolveum.com/xml/ns/public/model/action-3#addFocus</handlerUri></action>
        </reaction>
        <!-- Mover: existe en ambos → sincronizar atributos -->
        <reaction>
          <situation>linked</situation>
          <action><handlerUri>http://midpoint.evolveum.com/xml/ns/public/model/action-3#synchronize</handlerUri></action>
        </reaction>
        <!-- Leaver: existe en MidPoint pero no en HR → deshabilitar (no eliminar) -->
        <reaction>
          <situation>deleted</situation>
          <action><handlerUri>http://midpoint.evolveum.com/xml/ns/public/model/action-3#inactivateFocus</handlerUri></action>
        </reaction>
      </synchronization>

    </objectType>
  </schemaHandling>

</resource>
EOF
```

**3.3** Importar el recurso CSV en MidPoint vía la UI:

1. En la UI de MidPoint, navega a **Configuration → Import object**
2. Selecciona el archivo `~/lab02/midpoint-config/resources/resource-csv-hr.xml`
3. Haz clic en **Import object**
4. Verifica que aparezca el mensaje: *"Object imported successfully"*

**3.4** Verificar el recurso importado:

1. Navega a **Resources → All resources**
2. Debe aparecer **HR-CSV-Source** en la lista
3. Haz clic sobre él y selecciona **Test connection** → debe mostrar **Success**

#### Salida esperada

En la UI de MidPoint, el recurso `HR-CSV-Source` debe mostrar estado de conexión en verde con el mensaje:
```
Connection test result: SUCCESS
```

#### Verificación

```bash
# Verificar que el CSV es accesible desde dentro del contenedor
docker exec lab02-midpoint ls -la /opt/midpoint/var/hr-data/hr_users.csv
docker exec lab02-midpoint wc -l /opt/midpoint/var/hr-data/hr_users.csv
```

---

### Paso 4: Configurar el recurso OpenLDAP (destino) en MidPoint

**Objetivo:** Definir OpenLDAP como recurso de destino en MidPoint con los *mappings* de atributos LDAP y la política de ciclo de vida que determina en qué OU se crean las cuentas según el estado del usuario.

#### Instrucciones

**4.1** Crear el archivo de definición del recurso LDAP:

```bash
cat > ~/lab02/midpoint-config/resources/resource-openldap.xml << 'EOF'
<?xml version="1.0" encoding="UTF-8"?>
<!--
  Recurso OpenLDAP: Directorio de destino
  Implementa el patrón de conector LDAP on-premise descrito en la lección 2.1
  Mappings OUTBOUND: MidPoint User → atributos inetOrgPerson en LDAP
-->
<resource xmlns="http://midpoint.evolveum.com/xml/ns/public/common/common-3"
          xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
          xmlns:icfs="http://midpoint.evolveum.com/xml/ns/public/connector/icf-1/resource-schema-3"
          xmlns:ri="http://midpoint.evolveum.com/xml/ns/public/resource/instance-3"
          oid="a1b2c3d4-0002-0000-0000-000000000002">

  <name>OpenLDAP-Target</name>
  <description>Directorio LDAP de destino. Recibe cuentas aprovisionadas desde la fuente HR.</description>

  <connectorRef type="ConnectorType">
    <filter>
      <q:equal xmlns:q="http://prism.evolveum.com/xml/ns/public/query-3">
        <q:path>connectorType</q:path>
        <q:value>com.evolveum.polygon.connector.ldap.LdapConnector</q:value>
      </q:equal>
    </filter>
  </connectorRef>

  <connectorConfiguration xmlns:icfc="http://midpoint.evolveum.com/xml/ns/public/connector/icf-1/connector-schema-3">
    <icfc:configurationProperties>
      <icfi:host xmlns:icfi="http://midpoint.evolveum.com/xml/ns/public/connector/icf-1/bundle/com.evolveum.polygon.connector-ldap/com.evolveum.polygon.connector.ldap.LdapConnector">
        openldap
      </icfi:host>
      <icfi:port xmlns:icfi="http://midpoint.evolveum.com/xml/ns/public/connector/icf-1/bundle/com.evolveum.polygon.connector-ldap/com.evolveum.polygon.connector.ldap.LdapConnector">
        389
      </icfi:port>
      <icfi:baseContext xmlns:icfi="http://midpoint.evolveum.com/xml/ns/public/connector/icf-1/bundle/com.evolveum.polygon.connector-ldap/com.evolveum.polygon.connector.ldap.LdapConnector">
        dc=empresa,dc=local
      </icfi:baseContext>
      <icfi:bindDn xmlns:icfi="http://midpoint.evolveum.com/xml/ns/public/connector/icf-1/bundle/com.evolveum.polygon.connector-ldap/com.evolveum.polygon.connector.ldap.LdapConnector">
        cn=admin,dc=empresa,dc=local
      </icfi:bindDn>
      <icfi:bindPassword xmlns:icfi="http://midpoint.evolveum.com/xml/ns/public/connector/icf-1/bundle/com.evolveum.polygon.connector-ldap/com.evolveum.polygon.connector.ldap.LdapConnector">
        <clearValue>ldap_admin_secret</clearValue>
      </icfi:bindPassword>
    </icfc:configurationProperties>
  </connectorConfiguration>

  <schemaHandling>
    <objectType>
      <kind>account</kind>
      <intent>default</intent>
      <displayName>Cuenta LDAP</displayName>
      <default>true</default>
      <objectClass>ri:inetOrgPerson</objectClass>

      <!-- ── DN dinámico según estado de activación ───────────────────── -->
      <!-- Política de ciclo de vida: activos en ou=People; deshabilitados en ou=Disabled -->
      <attribute>
        <ref>ri:dn</ref>
        <outbound>
          <source><path>name</path></source>
          <source><path>activation/administrativeStatus</path></source>
          <expression>
            <script>
              <code>
                def ou = (administrativeStatus == com.evolveum.midpoint.xml.ns._public.common.common_3.ActivationStatusType.ENABLED) \
                         ? 'People' : 'Disabled'
                "uid=${name},ou=${ou},dc=empresa,dc=local"
              </code>
            </script>
          </expression>
        </outbound>
      </attribute>

      <!-- ── Mappings OUTBOUND: MidPoint User → LDAP ──────────────────── -->
      <attribute>
        <ref>ri:uid</ref>
        <outbound><source><path>name</path></source></outbound>
      </attribute>

      <attribute>
        <ref>ri:cn</ref>
        <outbound><source><path>fullName</path></source></outbound>
      </attribute>

      <attribute>
        <ref>ri:sn</ref>
        <outbound><source><path>familyName</path></source></outbound>
      </attribute>

      <attribute>
        <ref>ri:givenName</ref>
        <outbound><source><path>givenName</path></source></outbound>
      </attribute>

      <attribute>
        <ref>ri:mail</ref>
        <outbound><source><path>emailAddress</path></source></outbound>
      </attribute>

      <attribute>
        <ref>ri:departmentNumber</ref>
        <outbound><source><path>organizationalUnit</path></source></outbound>
      </attribute>

      <attribute>
        <ref>ri:employeeType</ref>
        <outbound><source><path>employeeType</path></source></outbound>
      </attribute>

      <!-- ── Contraseña inicial aleatoria ─────────────────────────────── -->
      <credentials>
        <password>
          <outbound>
            <expression>
              <generate/>
            </expression>
          </outbound>
        </password>
      </credentials>

    </objectType>
  </schemaHandling>

</resource>
EOF
```

**4.2** Importar el recurso LDAP en MidPoint:

1. Navega a **Configuration → Import object**
2. Selecciona `~/lab02/midpoint-config/resources/resource-openldap.xml`
3. Haz clic en **Import object**
4. Verifica: **Resources → All resources** → `OpenLDAP-Target` → **Test connection** → **Success**

#### Verificación

```bash
# Verificar que OpenLDAP está vacío (sin usuarios aún)
ldapsearch -x -H ldap://localhost:389 \
  -D "cn=admin,dc=empresa,dc=local" \
  -w "ldap_admin_secret" \
  -b "ou=People,dc=empresa,dc=local" \
  "(objectClass=inetOrgPerson)" uid 2>/dev/null | grep "uid:"
```

No debe haber salida (directorio vacío, listo para aprovisionamiento).

---

### Paso 5: Ejecutar el aprovisionamiento inicial (Joiner)

**Objetivo:** Ejecutar la importación inicial desde el CSV hacia MidPoint y el aprovisionamiento subsecuente hacia OpenLDAP, materializando el evento **Joiner** del ciclo de vida JML para los 20 usuarios ficticios.

#### Instrucciones

**5.1** Crear la tarea de importación desde CSV:

```bash
cat > ~/lab02/midpoint-config/tasks/task-import-csv.xml << 'EOF'
<?xml version="1.0" encoding="UTF-8"?>
<!-- Tarea: Importación inicial desde fuente HR (CSV) -->
<task xmlns="http://midpoint.evolveum.com/xml/ns/public/common/common-3"
      oid="t0000001-0000-0000-0000-000000000001">
  <name>Import-HR-CSV-Initial</name>
  <description>Importación inicial de usuarios desde fuente HR simulada (CSV). Evento Joiner.</description>
  <taskIdentifier>import-hr-csv-001</taskIdentifier>
  <ownerRef oid="00000000-0000-0000-0000-000000000002" type="UserType"/>
  <executionState>runnable</executionState>
  <schedule>
    <recurrence>single</recurrence>
  </schedule>
  <activity>
    <work>
      <import>
        <resourceObjects>
          <resourceRef oid="a1b2c3d4-0001-0000-0000-000000000001"/>
          <kind>account</kind>
          <intent>default</intent>
        </resourceObjects>
      </import>
    </work>
  </activity>
</task>
EOF
```

**5.2** Importar y ejecutar la tarea:

1. En MidPoint UI: **Configuration → Import object** → selecciona `task-import-csv.xml` → **Import object**
2. Navega a **Server tasks → All tasks**
3. Busca `Import-HR-CSV-Initial` y haz clic en **Run now** (ícono de play)
4. Espera a que el estado cambie a **Success** (puede tardar 30-60 segundos)

**5.3** Verificar usuarios creados en MidPoint:

1. Navega a **Identity Management → Users**
2. Deben aparecer 20 usuarios con los nombres generados por Faker
3. Observa que los usuarios con `status=inactive` en el CSV tienen `Administrative status: Disabled`

**5.4** Verificar aprovisionamiento en OpenLDAP:

```bash
# Contar usuarios activos en ou=People
echo "=== Usuarios en ou=People (activos) ==="
ldapsearch -x -H ldap://localhost:389 \
  -D "cn=admin,dc=empresa,dc=local" \
  -w "ldap_admin_secret" \
  -b "ou=People,dc=empresa,dc=local" \
  "(objectClass=inetOrgPerson)" uid cn mail 2>/dev/null | grep -E "^(uid|cn|mail):"

# Contar usuarios deshabilitados en ou=Disabled
echo ""
echo "=== Usuarios en ou=Disabled (inactivos) ==="
ldapsearch -x -H ldap://localhost:389 \
  -D "cn=admin,dc=empresa,dc=local" \
  -w "ldap_admin_secret" \
  -b "ou=Disabled,dc=empresa,dc=local" \
  "(objectClass=inetOrgPerson)" uid 2>/dev/null | grep "uid:"
```

#### Salida esperada

```
=== Usuarios en ou=People (activos) ===
uid: jgarcia
cn: Juan García
mail: jgarcia@empresa.local
uid: mlopez
cn: María López
mail: mlopez@empresa.local
... (aprox. 16 usuarios activos)

=== Usuarios en ou=Disabled (inactivos) ===
uid: cmarti
uid: rperez
... (aprox. 4 usuarios inactivos)
```

#### Verificación

```bash
# Total de usuarios LDAP debe ser 20
TOTAL=$(ldapsearch -x -H ldap://localhost:389 \
  -D "cn=admin,dc=empresa,dc=local" \
  -w "ldap_admin_secret" \
  -b "dc=empresa,dc=local" \
  "(objectClass=inetOrgPerson)" uid 2>/dev/null | grep -c "uid:")
echo "Total usuarios LDAP: $TOTAL (esperado: 20)"
```

---

### Paso 6: Introducir cambios en el CSV y ejecutar reconciliación

**Objetivo:** Simular eventos **Mover** y **Leaver** modificando el CSV de RR. HH., luego ejecutar una reconciliación en MidPoint para observar cómo detecta y resuelve las discrepancias. Este paso implementa el concepto de **detección de desvíos (drift)** y **convergencia** descrito en la lección.

#### Instrucciones

**6.1** Crear un script Python para modificar el CSV (simular cambios de HR):

```bash
cat > ~/lab02/data/modify_hr.py << 'EOF'
#!/usr/bin/env python3
"""
Script de modificación del CSV de HR para simular eventos del ciclo de vida.
Implementa los tres tipos de cambios: Joiner (alta), Mover (modificación), Leaver (baja).
AVISO: Todos los datos son completamente ficticios.
"""
import csv
import copy
from faker import Faker

fake = Faker('es_ES')
fake.seed_instance(99)

input_file  = "hr_users.csv"
output_file = "hr_users_v2.csv"

with open(input_file, newline="", encoding="utf-8") as f:
    reader = csv.DictReader(f)
    fieldnames = reader.fieldnames
    rows = list(reader)

original_count = len(rows)
changes_log = []

# ── CAMBIO 1: LEAVER ─────────────────────────────────────────────────────────
# EMP0003 sale de la empresa → cambiar status a 'inactive'
for row in rows:
    if row["employeeNumber"] == "EMP0003":
        old_status = row["status"]
        row["status"] = "inactive"
        changes_log.append(f"[LEAVER]  {row['employeeNumber']} ({row['cn']}): status {old_status} → inactive")
        break

# ── CAMBIO 2: MOVER ──────────────────────────────────────────────────────────
# EMP0005 cambia de departamento → actualizar department y jobTitle
for row in rows:
    if row["employeeNumber"] == "EMP0005":
        old_dept = row["department"]
        row["department"] = "Legal"
        row["jobTitle"]   = "Coordinador"
        changes_log.append(f"[MOVER]   {row['employeeNumber']} ({row['cn']}): dept {old_dept} → Legal, jobTitle → Coordinador")
        break

# ── CAMBIO 3: LEAVER (completo) ───────────────────────────────────────────────
# EMP0007 es eliminado del CSV (ya no aparece en HR) → MidPoint debe deshabilitarlo
rows_filtered = [r for r in rows if r["employeeNumber"] != "EMP0007"]
removed = [r for r in rows if r["employeeNumber"] == "EMP0007"]
if removed:
    changes_log.append(f"[LEAVER]  {removed[0]['employeeNumber']} ({removed[0]['cn']}): ELIMINADO del CSV")
rows = rows_filtered

# ── CAMBIO 4: JOINER ─────────────────────────────────────────────────────────
# Nuevo empleado se incorpora
new_first = fake.first_name()
new_last  = fake.last_name()
new_uid   = f"{new_first[0].lower()}{new_last.lower().replace(' ','')[:8]}99"
new_user = {
    "employeeNumber": "EMP0021",
    "uid":            new_uid,
    "givenName":      new_first,
    "sn":             new_last,
    "cn":             f"{new_first} {new_last}",
    "mail":           f"{new_uid}@empresa.local",
    "department":     "TI",
    "jobTitle":       "Especialista",
    "employeeType":   "FTE",
    "status":         "active",
    "startDate":      "2024-01-15",
    "manager":        "EMP0001",
}
rows.append(new_user)
changes_log.append(f"[JOINER]  {new_user['employeeNumber']} ({new_user['cn']}): NUEVO en HR")

# ── Escribir CSV modificado ───────────────────────────────────────────────────
with open(output_file, "w", newline="", encoding="utf-8") as f:
    writer = csv.DictWriter(f, fieldnames=fieldnames)
    writer.writeheader()
    writer.writerows(rows)

print(f"[OK] CSV original: {original_count} usuarios → CSV v2: {len(rows)} usuarios")
print(f"[OK] Cambios aplicados:")
for c in changes_log:
    print(f"     {c}")
print(f"[OK] Archivo guardado: {output_file}")
EOF
```

**6.2** Ejecutar el script de modificación:

```bash
cd ~/lab02/data
python3 modify_hr.py
```

**6.3** Reemplazar el CSV activo con la versión modificada:

```bash
cp ~/lab02/data/hr_users.csv ~/lab02/data/hr_users_v1_backup.csv
cp ~/lab02/data/hr_users_v2.csv ~/lab02/data/hr_users.csv
echo "[OK] CSV actualizado. Verificando cambios:"
diff ~/lab02/data/hr_users_v1_backup.csv ~/lab02/data/hr_users.csv | head -20
```

**6.4** Crear y ejecutar la tarea de reconciliación en MidPoint:

```bash
cat > ~/lab02/midpoint-config/tasks/task-reconciliation.xml << 'EOF'
<?xml version="1.0" encoding="UTF-8"?>
<!-- Tarea: Reconciliación entre CSV HR y MidPoint -->
<!-- Detecta: Joiners (unmatched), Movers (linked→sync), Leavers (deleted→inactivate) -->
<task xmlns="http://midpoint.evolveum.com/xml/ns/public/common/common-3"
      oid="t0000002-0000-0000-0000-000000000002">
  <name>Reconciliation-HR-CSV</name>
  <description>
    Reconciliación entre fuente HR (CSV) y MidPoint.
    Detecta y resuelve: Joiners, Movers y Leavers.
    Principio: idempotencia — reejecutar produce el mismo resultado final.
  </description>
  <taskIdentifier>reconciliation-hr-csv-001</taskIdentifier>
  <ownerRef oid="00000000-0000-0000-0000-000000000002" type="UserType"/>
  <executionState>runnable</executionState>
  <schedule>
    <recurrence>single</recurrence>
  </schedule>
  <activity>
    <work>
      <reconciliation>
        <resourceObjects>
          <resourceRef oid="a1b2c3d4-0001-0000-0000-000000000001"/>
          <kind>account</kind>
          <intent>default</intent>
        </resourceObjects>
      </reconciliation>
    </work>
  </activity>
</task>
EOF
```

1. En MidPoint UI: **Configuration → Import object** → selecciona `task-reconciliation.xml` → **Import object**
2. Navega a **Server tasks → All tasks** → busca `Reconciliation-HR-CSV`
3. Haz clic en **Run now** y espera a que finalice (estado **Success**)

**6.5** Revisar los resultados de la reconciliación en la UI:

1. Haz clic en la tarea `Reconciliation-HR-CSV` → pestaña **Operation statistics**
2. Observa los contadores:
   - **Users processed** — total de registros evaluados
   - **Users added** — nuevos Joiners (EMP0021)
   - **Users modified** — Movers (EMP0005)
   - **Users disabled/deleted** — Leavers (EMP0003, EMP0007)

#### Verificación

```bash
# Verificar que EMP0021 (nuevo Joiner) está en LDAP
echo "=== Verificando Joiner (EMP0021) ==="
ldapsearch -x -H ldap://localhost:389 \
  -D "cn=admin,dc=empresa,dc=local" \
  -w "ldap_admin_secret" \
  -b "ou=People,dc=empresa,dc=local" \
  "(mail=*@empresa.local)" uid cn departmentNumber 2>/dev/null | grep -A3 "EMP0021\|uid: .*99"

# Verificar que EMP0005 (Mover) tiene department=Legal en LDAP
echo "=== Verificando Mover (EMP0005) ==="
ldapsearch -x -H ldap://localhost:389 \
  -D "cn=admin,dc=empresa,dc=local" \
  -w "ldap_admin_secret" \
  -b "dc=empresa,dc=local" \
  "(departmentNumber=Legal)" uid cn 2>/dev/null | grep -E "^(uid|cn):"

# Verificar que Leavers están en ou=Disabled
echo "=== Verificando Leavers en ou=Disabled ==="
ldapsearch -x -H ldap://localhost:389 \
  -D "cn=admin,dc=empresa,dc=local" \
  -w "ldap_admin_secret" \
  -b "ou=Disabled,dc=empresa,dc=local" \
  "(objectClass=inetOrgPerson)" uid 2>/dev/null | grep "uid:"
```

---

### Paso 7: Inspeccionar el directorio con Apache Directory Studio

**Objetivo:** Validar visualmente el estado final del directorio LDAP usando Apache Directory Studio, confirmando que la estructura de OUs refleja correctamente las políticas de ciclo de vida implementadas.

#### Instrucciones

**7.1** Abrir Apache Directory Studio y crear una nueva conexión:

1. Ir a **File → New → LDAP Connection**
2. Configurar:
   - **Connection name:** `Lab02-OpenLDAP`
   - **Hostname:** `localhost`
   - **Port:** `389`
   - **Encryption:** No encryption
3. Clic en **Next**
4. **Bind DN:** `cn=admin,dc=empresa,dc=local`
5. **Bind password:** `ldap_admin_secret`
6. Clic en **Finish**

**7.2** Explorar la estructura del directorio:

Expande el árbol LDAP y verifica:

```
dc=empresa,dc=local
├── ou=People          ← Usuarios activos (status=active)
│   ├── uid=jgarcia,...
│   ├── uid=mlopez,...
│   └── ... (~17 usuarios)
├── ou=Disabled        ← Usuarios inactivos/Leavers
│   ├── uid=cmarti,...
│   └── ... (~4 usuarios)
└── ou=Groups
```

**7.3** Verificar atributos de un usuario específico:

1. Haz clic en cualquier usuario en `ou=People`
2. En el panel de atributos, verifica la presencia de: `uid`, `cn`, `sn`, `givenName`, `mail`, `departmentNumber`, `employeeType`

#### Verificación

```bash
# Resumen final del estado del directorio
echo "=== ESTADO FINAL DEL DIRECTORIO LDAP ==="
echo "Usuarios activos (ou=People):"
ldapsearch -x -H ldap://localhost:389 \
  -D "cn=admin,dc=empresa,dc=local" \
  -w "ldap_admin_secret" \
  -b "ou=People,dc=empresa,dc=local" \
  "(objectClass=inetOrgPerson)" uid 2>/dev/null | grep -c "uid:"

echo "Usuarios inactivos (ou=Disabled):"
ldapsearch -x -H ldap://localhost:389 \
  -D "cn=admin,dc=empresa,dc=local" \
  -w "ldap_admin_secret" \
  -b "ou=Disabled,dc=empresa,dc=local" \
  "(objectClass=inetOrgPerson)" uid 2>/dev/null | grep -c "uid:"
```

---

### Paso 8: Documentar el flujo con un diagrama de proceso

**Objetivo:** Crear el diagrama de proceso que documenta el flujo de aprovisionamiento y reconciliación implementado, cumpliendo el nivel Bloom **Crear** del laboratorio.

#### Instrucciones

**8.1** Abrir draw.io en el navegador: `https://app.diagrams.net`

**8.2** Crear un nuevo diagrama con el siguiente contenido (puedes importar el XML directamente en draw.io vía **Extras → Edit Diagram**):

```xml
<!-- Importar en draw.io: Extras → Edit Diagram → pegar este XML -->
<mxGraphModel>
  <root>
    <mxCell id="0"/><mxCell id="1" parent="0"/>

    <!-- Título -->
    <mxCell id="2" value="Flujo de Aprovisionamiento y Reconciliación&#xa;Lab 02-00-01 — Ciclo de Vida JML" style="text;html=1;strokeColor=none;fillColor=none;align=center;verticalAlign=middle;whiteSpace=wrap;rounded=0;fontSize=14;fontStyle=1;" vertex="1" parent="1"><mxGeometry x="100" y="20" width="600" height="50" as="geometry"/></mxCell>

    <!-- Fuente HR -->
    <mxCell id="10" value="📄 Fuente HR&#xa;(hr_users.csv)&#xa;Patrón: Lote diario" style="rounded=1;whiteSpace=wrap;html=1;fillColor=#dae8fc;strokeColor=#6c8ebf;fontSize=11;" vertex="1" parent="1"><mxGeometry x="50" y="120" width="140" height="70" as="geometry"/></mxCell>

    <!-- MidPoint: Correlación -->
    <mxCell id="20" value="🔗 MidPoint&#xa;Motor de Correlación&#xa;(employeeNumber)" style="rounded=1;whiteSpace=wrap;html=1;fillColor=#fff2cc;strokeColor=#d6b656;fontSize=11;" vertex="1" parent="1"><mxGeometry x="250" y="120" width="140" height="70" as="geometry"/></mxCell>

    <!-- Decision: Situación -->
    <mxCell id="30" value="¿Situación?" style="rhombus;whiteSpace=wrap;html=1;fillColor=#f8cecc;strokeColor=#b85450;fontSize=11;" vertex="1" parent="1"><mxGeometry x="460" y="110" width="120" height="90" as="geometry"/></mxCell>

    <!-- Joiner -->
    <mxCell id="40" value="✅ JOINER&#xa;Crear usuario&#xa;MidPoint + LDAP" style="rounded=1;whiteSpace=wrap;html=1;fillColor=#d5e8d4;strokeColor=#82b366;fontSize=11;" vertex="1" parent="1"><mxGeometry x="650" y="60" width="130" height="60" as="geometry"/></mxCell>

    <!-- Mover -->
    <mxCell id="50" value="🔄 MOVER&#xa;Actualizar atributos&#xa;en LDAP" style="rounded=1;whiteSpace=wrap;html=1;fillColor=#fff2cc;strokeColor=#d6b656;fontSize=11;" vertex="1" parent="1"><mxGeometry x="650" y="145" width="130" height="60" as="geometry"/></mxCell>

    <!-- Leaver -->
    <mxCell id="60" value="🚫 LEAVER&#xa;Deshabilitar cuenta&#xa;Mover a ou=Disabled" style="rounded=1;whiteSpace=wrap;html=1;fillColor=#f8cecc;strokeColor=#b85450;fontSize=11;" vertex="1" parent="1"><mxGeometry x="650" y="230" width="130" height="60" as="geometry"/></mxCell>

    <!-- OpenLDAP -->
    <mxCell id="70" value="🗂️ OpenLDAP&#xa;ou=People (activos)&#xa;ou=Disabled (inactivos)" style="rounded=1;whiteSpace=wrap;html=1;fillColor=#e1d5e7;strokeColor=#9673a6;fontSize=11;" vertex="1" parent="1"><mxGeometry x="850" y="130" width="150" height="70" as="geometry"/></mxCell>

    <!-- Auditoría -->
    <mxCell id="80" value="📋 Auditoría&#xa;Registro de cambios&#xa;(quién/cuándo/qué)" style="rounded=1;whiteSpace=wrap;html=1;fillColor=#f5f5f5;strokeColor=#666666;fontSize=11;" vertex="1" parent="1"><mxGeometry x="450" y="280" width="140" height="60" as="geometry"/></mxCell>

    <!-- Flechas -->
    <mxCell id="e1" style="edgeStyle=orthogonalEdgeStyle;" edge="1" source="10" target="20" parent="1"><mxGeometry relative="1" as="geometry"/></mxCell>
    <mxCell id="e2" style="edgeStyle=orthogonalEdgeStyle;" edge="1" source="20" target="30" parent="1"><mxGeometry relative="1" as="geometry"/></mxCell>
    <mxCell id="e3" value="unmatched" style="edgeStyle=orthogonalEdgeStyle;" edge="1" source="30" target="40" parent="1"><mxGeometry relative="1" as="geometry"/></mxCell>
    <mxCell id="e4" value="linked" style="edgeStyle=orthogonalEdgeStyle;" edge="1" source="30" target="50" parent="1"><mxGeometry relative="1" as="geometry"/></mxCell>
    <mxCell id="e5" value="deleted" style="edgeStyle=orthogonalEdgeStyle;" edge="1" source="30" target="60" parent="1"><mxGeometry relative="1" as="geometry"/></mxCell>
    <mxCell id="e6" style="edgeStyle=orthogonalEdgeStyle;" edge="1" source="40" target="70" parent="1"><mxGeometry relative="1" as="geometry"/></mxCell>
    <mxCell id="e7" style="edgeStyle=orthogonalEdgeStyle;" edge="1" source="50" target="70" parent="1"><mxGeometry relative="1" as="geometry"/></mxCell>
    <mxCell id="e8" style="edgeStyle=orthogonalEdgeStyle;" edge="1" source="60" target="70" parent="1"><mxGeometry relative="1" as="geometry"/></mxCell>
    <mxCell id="e9" style="edgeStyle=orthogonalEdgeStyle;dashed=1;" edge="1" source="20" target="80" parent="1"><mxGeometry relative="1" as="geometry"/></mxCell>
  </root>
</mxGraphModel>
```

**8.3** Agregar al diagrama las siguientes anotaciones adicionales:

- **Nota en el flujo CSV → MidPoint:** *"Patrón: ingesta por lotes. Correlación por employeeNumber (identificador inmutable)."*
- **Nota en la decisión:** *"Situaciones: unmatched=Joiner | linked=Mover | deleted=Leaver"*
- **Nota en LDAP:** *"Política de activación: status='active' → ou=People; status≠'active' → ou=Disabled"*

**8.4** Exportar el diagrama:

1. **File → Export as → PNG** → guardar como `~/lab02/diagrama-aprovisionamiento.png`
2. **File → Export as → XML** → guardar como `~/lab02/diagrama-aprovisionamiento.xml`

#### Verificación

```bash
# Verificar que el archivo del diagrama fue guardado
ls -lh ~/lab02/diagrama-aprovisionamiento.png 2>/dev/null && echo "[OK] Diagrama PNG guardado" || echo "[PENDIENTE] Guardar el diagrama"
```

---

## 7. Validación y Pruebas

Ejecuta este bloque de validación completa al finalizar todos los pasos:

```bash
#!/bin/bash
# Script de validación final — Lab 02-00-01
echo "════════════════════════════════════════════════════"
echo "  VALIDACIÓN FINAL — Lab 02-00-01"
echo "════════════════════════════════════════════════════"

PASS=0; FAIL=0

check() {
    local desc="$1"; local cmd="$2"; local expected="$3"
    local result=$(eval "$cmd" 2>/dev/null)
    if echo "$result" | grep -q "$expected"; then
        echo "[✓] $desc"
        ((PASS++))
    else
        echo "[✗] $desc (obtenido: '$result', esperado contiene: '$expected')"
        ((FAIL++))
    fi
}

# 1. Contenedores en ejecución
check "PostgreSQL running"  "docker inspect lab02-postgres --format '{{.State.Status}}'"  "running"
check "OpenLDAP running"    "docker inspect lab02-openldap --format '{{.State.Status}}'"  "running"
check "MidPoint running"    "docker inspect lab02-midpoint --format '{{.State.Status}}'"  "running"

# 2. CSV generado correctamente
check "CSV tiene 21 líneas (header+20 usuarios o header+21 con EMP0021)" \
    "wc -l < ~/lab02/data/hr_users.csv" "2[01]"

# 3. OpenLDAP tiene usuarios en ou=People
PEOPLE_COUNT=$(ldapsearch -x -H ldap://localhost:389 \
    -D "cn=admin,dc=empresa,dc=local" -w "ldap_admin_secret" \
    -b "ou=People,dc=empresa,dc=local" "(objectClass=inetOrgPerson)" uid 2>/dev/null | grep -c "uid:")
check "Usuarios activos en ou=People (≥1)" "echo $PEOPLE_COUNT" "[1-9]"

# 4. OpenLDAP tiene usuarios en ou=Disabled
DISABLED_COUNT=$(ldapsearch -x -H ldap://localhost:389 \
    -D "cn=admin,dc=empresa,dc=local" -w "ldap_admin_secret" \
    -b "ou=Disabled,dc=empresa,dc=local" "(objectClass=inetOrgPerson)" uid 2>/dev/null | grep -c "uid:")
check "Usuarios inactivos en ou=Disabled (≥1)" "echo $DISABLED_COUNT" "[1-9]"

# 5. Total de usuarios LDAP = 21 (20 originales + 1 Joiner, menos 0 eliminados físicamente)
TOTAL_LDAP=$(ldapsearch -x -H ldap://localhost:389 \
    -D "cn=admin,dc=empresa,dc=local" -w "ldap_admin_secret" \
    -b "dc=empresa,dc=local" "(objectClass=inetOrgPerson)" uid 2>/dev/null | grep -c "uid:")
check "Total usuarios LDAP = 21 (20 originales + 1 Joiner)" "echo $TOTAL_LDAP" "21"

# 6. MidPoint UI responde
check "MidPoint UI accesible" \
    "curl -s -o /dev/null -w '%{http_code}' http://localhost:8080/midpoint/" "30[02]"

# 7. Backup del CSV original existe
check "Backup CSV v1 existe" "ls ~/lab02/data/hr_users_v1_backup.csv" "backup"

echo ""
echo "════════════════════════════════════════════════════"
echo "  Resultados: $PASS pasaron | $FAIL fallaron"
echo "════════════════════════════════════════════════════"
```

```bash
# Ejecutar el script de validación
chmod +x /tmp/validate_lab02.sh 2>/dev/null
bash <(cat << 'VALIDATION_EOF'
# [pegar el script de validación aquí o ejecutarlo directamente desde el archivo]
VALIDATION_EOF
)
```

### Lista de verificación manual

| Ítem                                                                 | Estado |
|----------------------------------------------------------------------|--------|
| CSV generado con 20 usuarios ficticios (sin datos reales)            | ☐      |
| Contenedores PostgreSQL, OpenLDAP y MidPoint en estado `healthy`     | ☐      |
| Recurso `HR-CSV-Source` importado y con Test Connection = Success    | ☐      |
| Recurso `OpenLDAP-Target` importado y con Test Connection = Success  | ☐      |
| Tarea de importación inicial completada (20 usuarios en MidPoint)    | ☐      |
| Usuarios activos en `ou=People`, inactivos en `ou=Disabled`          | ☐      |
| Reconciliación ejecutada: Joiner (EMP0021), Mover (EMP0005), Leaver (EMP0003/EMP0007) detectados | ☐ |
| Diagrama de proceso exportado en PNG y XML                           | ☐      |

---

## 8. Solución de Problemas

### Problema 1: MidPoint no puede conectarse a OpenLDAP (Test Connection falla)

**Síntoma:**
En la UI de MidPoint, al hacer **Test connection** en el recurso `OpenLDAP-Target`, aparece el error:
```
Connection test result: FAILURE
com.evolveum.midpoint.util.exception.SystemException: LDAP connection refused to openldap:389
```

**Causa:**
El contenedor MidPoint intenta resolver el hostname `openldap` dentro de la red Docker `iam-net`, pero el contenedor OpenLDAP no terminó de inicializarse (el *healthcheck* aún no pasó) o el volumen `bootstrap.ldif` tiene un error de sintaxis que impidió que slapd arrancara correctamente.

**Solución:**

```bash
# 1. Verificar el estado de salud de OpenLDAP
docker inspect lab02-openldap --format '{{.State.Health.Status}}'

# 2. Revisar los logs de OpenLDAP en busca de errores
docker logs lab02-openldap 2>&1 | tail -30

# 3. Si el contenedor está en estado "unhealthy", verificar el LDIF de bootstrap
# El error más común es un DN incorrecto o espacio extra al inicio del archivo
cat -A ~/lab02/ldap-config/bootstrap.ldif | head -5
# No deben aparecer caracteres ^M (retorno de carro Windows) ni espacios al inicio

# 4. Si hay caracteres Windows, convertir a Unix
sed -i 's/\r//' ~/lab02/ldap-config/bootstrap.ldif

# 5. Reiniciar solo el contenedor OpenLDAP
docker compose -f ~/lab02/compose/docker-compose.yml restart openldap

# 6. Esperar 20 segundos y verificar estado
sleep 20
docker inspect lab02-openldap --format '{{.State.Health.Status}}'

# 7. Probar conectividad LDAP desde el host
ldapsearch -x -H ldap://localhost:389 \
  -D "cn=admin,dc=empresa,dc=local" \
  -w "ldap_admin_secret" \
  -b "dc=empresa,dc=local" "(objectClass=*)" dn 2>&1 | head -5
```

---

### Problema 2: La reconciliación no detecta el nuevo Joiner (EMP0021) o no actualiza el Mover (EMP0005)

**Síntoma:**
Tras ejecutar la tarea `Reconciliation-HR-CSV`, los contadores de la tarea muestran 0 usuarios añadidos o modificados, aunque el CSV fue actualizado correctamente.

**Causa:**
MidPoint puede estar leyendo el CSV desde una caché o desde la ruta incorrecta dentro del contenedor. El volumen Docker monta `~/lab02/data` en `/opt/midpoint/var/hr-data`, pero si el archivo fue modificado después del inicio del contenedor sin que el conector CSV haya refrescado su caché de esquema, puede leer la versión anterior. También puede ocurrir si el CSV `hr_users.csv` no fue reemplazado correctamente (el archivo `hr_users_v2.csv` existe pero el activo sigue siendo el original).

**Solución:**

```bash
# 1. Verificar que el CSV activo es la versión modificada (debe tener 21 líneas: header + 20)
wc -l ~/lab02/data/hr_users.csv
# Si muestra 21 (header + 20 usuarios), el CSV está correcto

# 2. Verificar que EMP0021 está en el CSV
grep "EMP0021" ~/lab02/data/hr_users.csv

# 3. Verificar que el archivo es visible desde dentro del contenedor
docker exec lab02-midpoint cat /opt/midpoint/var/hr-data/hr_users.csv | grep "EMP0021"

# 4. Si el archivo no es visible dentro del contenedor, el volumen no se montó correctamente
# Solución: recrear el contenedor MidPoint (sin perder datos de PostgreSQL)
docker compose -f ~/lab02/compose/docker-compose.yml up -d --force-recreate midpoint

# 5. Esperar a que MidPoint reinicie (2-3 minutos) y volver a ejecutar la reconciliación
# En la UI: Server tasks → Reconciliation-HR-CSV → Run now

# 6. Si el problema persiste, verificar que el recurso CSV apunta a la ruta correcta
# En MidPoint UI: Resources → HR-CSV-Source → Configuration → filePath
# Debe ser: /opt/midpoint/var/hr-data/hr_users.csv

# 7. Alternativa: forzar reimportación completa en lugar de reconciliación
# En MidPoint UI: Resources → HR-CSV-Source → Import → Run import
```

---

## 9. Limpieza del Entorno

> ⚠️ **Importante:** Ejecuta la limpieza **solo al finalizar** el laboratorio. Los Labs 5 y 6 del curso no dependen de este entorno, pero si deseas conservar el estado para referencia futura, omite el paso de eliminación de volúmenes.

```bash
# Paso 1: Detener todos los contenedores
cd ~/lab02/compose
docker compose down
echo "[OK] Contenedores detenidos"

# Paso 2: Eliminar volúmenes (libera espacio en disco)
docker compose down -v
echo "[OK] Volúmenes eliminados"

# Paso 3: Verificar que no quedan contenedores del lab
docker ps -a | grep "lab02" && echo "[AVISO] Aún hay contenedores lab02" || echo "[OK] Sin contenedores lab02"

# Paso 4 (opcional): Eliminar imágenes descargadas para liberar más espacio
# docker rmi evolveum/midpoint:4.8-alpine osixia/openldap:1.5.0 postgres:15-alpine
# NOTA: Omitir si se usarán en otros labs del curso

# Paso 5 (opcional): Archivar los archivos de configuración generados
tar -czf ~/lab02-backup-$(date +%Y%m%d).tar.gz ~/lab02/
echo "[OK] Archivos del lab comprimidos en ~/lab02-backup-$(date +%Y%m%d).tar.gz"

echo ""
echo "Recursos liberados aproximadamente:"
echo "  RAM: ~2-4 GB (MidPoint + OpenLDAP + PostgreSQL)"
echo "  Disco: ~1-2 GB (volúmenes Docker)"
```

---

## 10. Resumen

### Conceptos aplicados en este laboratorio

En este laboratorio has implementado de forma práctica los conceptos centrales de la lección 2.1:

| Concepto de la lección                        | Implementación en el lab                                                    |
|-----------------------------------------------|-----------------------------------------------------------------------------|
| Ciclo de vida JML (Joiner-Mover-Leaver)       | Políticas de sincronización en MidPoint: `unmatched`, `linked`, `deleted`   |
| Fuente de verdad y patrón de ingesta por lotes | CSV simulando RR. HH., importado periódicamente por MidPoint                |
| Correlación determinística                    | Campo `employeeNumber` como identificador inmutable para correlación         |
| Política de activación condicional            | Campo `status` del CSV determina si la cuenta va a `ou=People` o `ou=Disabled` |
| Máquina de estados ILM                        | `Active` → `ou=People`; `Suspended/Leaver` → `ou=Disabled`                  |
| Idempotencia                                  | Re-ejecutar la reconciliación produce el mismo estado final                 |
| Auditoría                                     | Registros de operaciones en MidPoint (task results, operation statistics)   |
| Diagrama de proceso                           | Flujo documentado en draw.io con fuentes, decisiones y destinos             |

### Lo que has logrado

1. ✅ Desplegaste un entorno IGA completo (MidPoint + OpenLDAP + PostgreSQL) con Docker Compose.
2. ✅ Generaste datos de prueba ficticios con Python/Faker siguiendo las políticas de privacidad del curso.
3. ✅ Configuraste un recurso CSV como fuente autoritativa con correlación por `employeeNumber`.
4. ✅ Definiste *mappings* de atributos y una política de activación condicional basada en `status`.
5. ✅ Ejecutaste el aprovisionamiento inicial (20 Joiners) y una reconciliación que detectó y resolvió cambios.
6. ✅ Documentaste el flujo mediante un diagrama de proceso en draw.io.

### Próximos pasos

En el **Lab 03-00-01** profundizarás en sincronización bidireccional y resolución de conflictos entre repositorios, aplicando estrategias de detección de desvíos (*drift*) y reconciliación basada en evidencia para mantener la integridad de las identidades en múltiples destinos simultáneamente.

### Recursos adicionales

| Recurso                                          | URL                                                                      |
|--------------------------------------------------|--------------------------------------------------------------------------|
| MidPoint 4.8 — Guía de recursos y conectores    | https://docs.evolveum.com/midpoint/reference/resources/                  |
| MidPoint — Sincronización y reconciliación       | https://docs.evolveum.com/midpoint/reference/synchronization/            |
| OpenLDAP 2.6 — Administración                   | https://www.openldap.org/doc/admin26/                                    |
| RFC 7644 — SCIM Protocol                        | https://www.rfc-editor.org/rfc/rfc7644                                   |
| draw.io — Documentación oficial                  | https://www.drawio.com/doc/                                              |
| Python Faker — Generación de datos ficticios    | https://faker.readthedocs.io/en/master/                                  |

---
