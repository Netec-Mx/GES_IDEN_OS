# Integración de una herramienta IAM con un repositorio de identidad

## 1. Metadatos

| Campo | Valor |
|---|---|
| **Duración estimada** | 31 minutos |
| **Complejidad** | Alta |
| **Nivel Bloom** | Aplicar |
| **Módulo** | 6 — Arquitectura, componentes y APIs de OpenIAM |
| **Laboratorio previo requerido** | 02-00-01, 03-00-01, 04-00-01 |

---

## 2. Descripción General

En este laboratorio integrarás tres tecnologías en una arquitectura de identidad de tres capas: **OpenIAM** como capa de gobierno de identidades (IGA), **OpenLDAP** como repositorio autoritativo de identidades, y **Keycloak** como capa de autenticación y SSO. Configurarás el conector LDAP de OpenIAM, establecerás la federación de usuarios en Keycloak, y validarás el flujo completo de aprovisionamiento desde la creación de un usuario en OpenIAM hasta su autenticación en una aplicación cliente. El laboratorio aplica directamente los conceptos de arquitectura por capas, conectores, APIs SCIM y motor de políticas estudiados en la Lección 6.1.

---

## 3. Objetivos de Aprendizaje

Al finalizar este laboratorio, serás capaz de:

- [ ] Configurar un conector LDAP en OpenIAM apuntando a OpenLDAP (host, port, bind DN, base DN, mappings de atributos) para sincronización bidireccional de identidades.
- [ ] Integrar OpenIAM con Keycloak para federar la autenticación: OpenIAM gestiona el ciclo de vida de identidades y Keycloak gestiona la autenticación vía User Federation LDAP.
- [ ] Implementar aprovisionamiento automático desde OpenIAM hacia OpenLDAP mediante conectores y políticas de reconciliación.
- [ ] Validar el flujo completo: creación de usuario en OpenIAM → aprovisionamiento en OpenLDAP → sincronización en Keycloak → autenticación exitosa.
- [ ] Documentar la arquitectura resultante identificando el rol de cada componente según el modelo de capas de OpenIAM.

---

## 4. Prerrequisitos

### Conocimiento previo

| Área | Nivel requerido |
|---|---|
| Arquitectura de OpenIAM (conectores, recursos, tareas de reconciliación) | Comprensión conceptual (Lección 6.1) |
| User Federation en Keycloak y configuración de realms | Práctico (Lab 03-00-01) |
| Estructura de directorios LDAP (DN, OU, atributos inetOrgPerson) | Práctico (Lab 02-00-01) |
| Docker Compose y redes de contenedores | Operativo (Lab 04-00-01) |
| APIs REST y SCIM 2.0 (RFC 7643/7644) | Conceptual (Lección 6.1) |

### Acceso y recursos

| Recurso | Detalle |
|---|---|
| Máquina anfitriona | ≥ 16 GB RAM disponibles (recomendado 32 GB); Docker Engine 24.x + Compose 2.20.x |
| Imágenes Docker | `osixia/openldap:1.5.0`, `quay.io/keycloak/keycloak:23.0`, `openiam/openiam-ce:4.2` + PostgreSQL 15 |
| Puertos libres | 389, 636, 8080, 8443, 9090, 5432 |
| Herramientas cliente | Apache Directory Studio 2.x, Postman 10.x, Python 3.10+ con `ldap3`, `requests` |
| Repositorio del curso | Clonado en `~/iam-labs/` (scripts de datos ficticios disponibles) |

> ⚠️ **Advertencia de recursos:** Este laboratorio ejecuta simultáneamente OpenIAM, OpenLDAP, Keycloak y PostgreSQL. Cierra otros laboratorios y ejecuta `docker compose down -v` en stacks previos antes de continuar.

---

## 5. Entorno de Laboratorio

### 5.1 Topología de la arquitectura

```
┌─────────────────────────────────────────────────────────────┐
│                    Red Docker: iam-network                   │
│                                                             │
│  ┌──────────────┐    Conector LDAP    ┌──────────────────┐  │
│  │   OpenIAM    │ ──────────────────► │    OpenLDAP      │  │
│  │  :9090/9443  │ ◄──── reconciliación│   :389 / :636    │  │
│  └──────┬───────┘                    └────────┬─────────┘  │
│         │ API REST/SCIM                        │ User Fed.  │
│         │                             ┌────────▼─────────┐  │
│         │                             │    Keycloak      │  │
│         └────────────────────────────►│   :8080 / :8443  │  │
│                                       └──────────────────┘  │
│  ┌──────────────┐                                           │
│  │  PostgreSQL  │ ◄── OpenIAM DB                            │
│  │    :5432     │                                           │
│  └──────────────┘                                           │
└─────────────────────────────────────────────────────────────┘
```

### 5.2 Preparación del entorno

Ejecuta los siguientes comandos en tu terminal para preparar el directorio de trabajo y descargar los archivos de configuración base:

```bash
# Crear directorio de trabajo
mkdir -p ~/iam-labs/lab06 && cd ~/iam-labs/lab06

# Limpiar stacks anteriores (liberar recursos)
docker compose -f ~/iam-labs/lab05/docker-compose.yml down -v 2>/dev/null || true
docker compose -f ~/iam-labs/lab04/docker-compose.yml down -v 2>/dev/null || true

# Verificar recursos disponibles
docker system df
free -h
```

### 5.3 Archivo `docker-compose.yml` del stack integrado

Crea el archivo `~/iam-labs/lab06/docker-compose.yml` con el siguiente contenido:

```yaml
version: "3.9"

networks:
  iam-network:
    driver: bridge
    ipam:
      config:
        - subnet: 172.20.0.0/24

volumes:
  openldap-data:
  openldap-config:
  postgres-data:
  keycloak-data:

services:

  # ─── OpenLDAP ─────────────────────────────────────────────
  openldap:
    image: osixia/openldap:1.5.0
    container_name: lab06-openldap
    hostname: openldap
    environment:
      LDAP_ORGANISATION: "LabCorp"
      LDAP_DOMAIN: "labcorp.local"
      LDAP_BASE_DN: "dc=labcorp,dc=local"
      LDAP_ADMIN_PASSWORD: "Admin1234!"
      LDAP_READONLY_USER: "true"
      LDAP_READONLY_USER_USERNAME: "readonly"
      LDAP_READONLY_USER_PASSWORD: "Readonly1234!"
      LDAP_TLS: "false"
    ports:
      - "389:389"
      - "636:636"
    volumes:
      - openldap-data:/var/lib/ldap
      - openldap-config:/etc/ldap/slapd.d
      - ./ldap-seed:/container/service/slapd/assets/config/bootstrap/ldif/custom
    networks:
      iam-network:
        ipv4_address: 172.20.0.10
    restart: unless-stopped

  # ─── PostgreSQL (backend de OpenIAM) ──────────────────────
  postgres:
    image: postgres:15-alpine
    container_name: lab06-postgres
    hostname: postgres
    environment:
      POSTGRES_DB: openiam
      POSTGRES_USER: openiam
      POSTGRES_PASSWORD: OpenIAM2024!
    ports:
      - "5432:5432"
    volumes:
      - postgres-data:/var/lib/postgresql/data
    networks:
      iam-network:
        ipv4_address: 172.20.0.20
    restart: unless-stopped
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U openiam"]
      interval: 10s
      timeout: 5s
      retries: 5

  # ─── OpenIAM Community Edition ────────────────────────────
  openiam:
    image: openiam/openiam-ce:4.2
    container_name: lab06-openiam
    hostname: openiam
    depends_on:
      postgres:
        condition: service_healthy
    environment:
      DB_HOST: postgres
      DB_PORT: 5432
      DB_NAME: openiam
      DB_USER: openiam
      DB_PASSWORD: OpenIAM2024!
      OPENIAM_ADMIN_PASSWORD: Admin1234!
    ports:
      - "9090:8080"
      - "9443:8443"
    networks:
      iam-network:
        ipv4_address: 172.20.0.30
    restart: unless-stopped

  # ─── Keycloak ─────────────────────────────────────────────
  keycloak:
    image: quay.io/keycloak/keycloak:23.0
    container_name: lab06-keycloak
    hostname: keycloak
    command: start-dev
    environment:
      KC_DB: dev-mem
      KEYCLOAK_ADMIN: admin
      KEYCLOAK_ADMIN_PASSWORD: Admin1234!
      KC_HTTP_PORT: 8080
      KC_HOSTNAME_STRICT: "false"
    ports:
      - "8080:8080"
      - "8443:8443"
    networks:
      iam-network:
        ipv4_address: 172.20.0.40
    restart: unless-stopped
```

### 5.4 Datos LDAP semilla (ficticios)

Crea el directorio y el archivo LDIF con usuarios ficticios generados con Faker:

```bash
mkdir -p ~/iam-labs/lab06/ldap-seed
```

Crea `~/iam-labs/lab06/ldap-seed/00-base.ldif`:

```ldif
# Unidades organizativas base
dn: ou=people,dc=labcorp,dc=local
objectClass: organizationalUnit
ou: people

dn: ou=groups,dc=labcorp,dc=local
objectClass: organizationalUnit
ou: groups

dn: ou=services,dc=labcorp,dc=local
objectClass: organizationalUnit
ou: services
```

Crea `~/iam-labs/lab06/ldap-seed/01-groups.ldif`:

```ldif
dn: cn=developers,ou=groups,dc=labcorp,dc=local
objectClass: groupOfNames
cn: developers
member: uid=placeholder,ou=people,dc=labcorp,dc=local

dn: cn=analysts,ou=groups,dc=labcorp,dc=local
objectClass: groupOfNames
cn: analysts
member: uid=placeholder,ou=people,dc=labcorp,dc=local
```

### 5.5 Levantar el stack

```bash
cd ~/iam-labs/lab06

# Levantar todos los servicios
docker compose up -d

# Verificar estado de los contenedores (esperar ~90 segundos para OpenIAM)
docker compose ps

# Monitorear logs de OpenIAM hasta que esté listo
docker compose logs -f openiam | grep -i "started\|ready\|error"
# Presiona Ctrl+C cuando veas "Started" o "Application ready"
```

**Salida esperada de `docker compose ps`:**

```
NAME                IMAGE                        STATUS
lab06-keycloak      quay.io/keycloak/keycloak    Up (healthy)
lab06-openiam       openiam/openiam-ce:4.2       Up
lab06-openldap      osixia/openldap:1.5.0        Up
lab06-postgres      postgres:15-alpine           Up (healthy)
```

---

## 6. Procedimiento Paso a Paso

---

### Paso 1: Verificar la conectividad entre contenedores

**Objetivo:** Confirmar que OpenIAM, OpenLDAP y Keycloak pueden comunicarse en la red Docker antes de configurar integraciones.

#### Instrucciones

1. Verifica la resolución de nombres DNS entre contenedores:

```bash
# Desde OpenIAM, hacer ping a OpenLDAP y Keycloak
docker exec lab06-openiam ping -c 3 openldap
docker exec lab06-openiam ping -c 3 keycloak
docker exec lab06-openiam ping -c 3 postgres
```

2. Verifica que el puerto LDAP 389 es accesible desde OpenIAM:

```bash
docker exec lab06-openiam bash -c "nc -zv openldap 389 && echo 'LDAP OK'"
```

3. Verifica que OpenLDAP responde con una búsqueda anónima básica:

```bash
docker exec lab06-openldap ldapsearch \
  -x \
  -H ldap://localhost:389 \
  -D "cn=admin,dc=labcorp,dc=local" \
  -w "Admin1234!" \
  -b "dc=labcorp,dc=local" \
  "(objectClass=organizationalUnit)" \
  dn
```

4. Verifica que las interfaces web están activas:

```bash
# OpenIAM Admin Console
curl -s -o /dev/null -w "OpenIAM HTTP: %{http_code}\n" http://localhost:9090/openiam-ui-static/

# Keycloak Admin Console
curl -s -o /dev/null -w "Keycloak HTTP: %{http_code}\n" http://localhost:8080/
```

#### Salida esperada

```
# ping openldap
PING openldap (172.20.0.10): 56 data bytes
3 packets transmitted, 3 received, 0% packet loss

# ldapsearch
dn: ou=people,dc=labcorp,dc=local
dn: ou=groups,dc=labcorp,dc=local
dn: ou=services,dc=labcorp,dc=local

OpenIAM HTTP: 200
Keycloak HTTP: 200
```

#### Verificación

```bash
docker network inspect lab06_iam-network --format '{{range .Containers}}{{.Name}}: {{.IPv4Address}}{{"\n"}}{{end}}'
```

Debes ver las cuatro IPs asignadas (172.20.0.10, .20, .30, .40).

---

### Paso 2: Configurar el conector LDAP en OpenIAM

**Objetivo:** Registrar OpenLDAP como recurso gestionado en OpenIAM y configurar el conector LDAP con los parámetros de conexión, bind DN y mapeos de atributos.

#### Instrucciones

1. Accede a la consola de administración de OpenIAM en tu navegador:
   - URL: `http://localhost:9090/openiam-ui-static/`
   - Usuario: `sysadmin`
   - Contraseña: `Admin1234!`

2. Navega a **Resources → Managed Systems → Add New Managed System**.

3. Completa el formulario con los siguientes valores:

| Campo | Valor |
|---|---|
| **Name** | `OpenLDAP-LabCorp` |
| **Connector Type** | `LDAP` |
| **Host** | `openldap` |
| **Port** | `389` |
| **Use SSL** | `No` |
| **Bind DN** | `cn=admin,dc=labcorp,dc=local` |
| **Bind Password** | `Admin1234!` |
| **Base DN** | `dc=labcorp,dc=local` |
| **User Base DN** | `ou=people,dc=labcorp,dc=local` |
| **Group Base DN** | `ou=groups,dc=labcorp,dc=local` |
| **Object Class (Users)** | `inetOrgPerson` |
| **User ID Attribute** | `uid` |

4. En la sección **Attribute Mappings**, configura los siguientes mapeos entre atributos OpenIAM ↔ LDAP:

| Atributo OpenIAM | Atributo LDAP | Obligatorio |
|---|---|---|
| `firstName` | `givenName` | Sí |
| `lastName` | `sn` | Sí |
| `login` | `uid` | Sí |
| `email` | `mail` | Sí |
| `displayName` | `displayName` | No |
| `telephoneNumber` | `telephoneNumber` | No |
| `department` | `departmentNumber` | No |

5. Haz clic en **Test Connection** y verifica que el resultado sea `Connection Successful`.

6. Guarda el recurso con **Save**.

7. Verifica la configuración via API REST de OpenIAM (desde terminal):

```bash
# Obtener token de administración de OpenIAM
OPENIAM_TOKEN=$(curl -s -X POST http://localhost:9090/openiam-rest/api/auth/login \
  -H "Content-Type: application/json" \
  -d '{"login":"sysadmin","password":"Admin1234!","managedSysId":"0"}' \
  | python3 -c "import sys,json; print(json.load(sys.stdin).get('token','NO_TOKEN'))")

echo "Token obtenido: ${OPENIAM_TOKEN:0:30}..."

# Listar recursos gestionados
curl -s -X GET "http://localhost:9090/openiam-rest/api/resources/managed" \
  -H "Authorization: Bearer $OPENIAM_TOKEN" \
  -H "Accept: application/json" \
  | python3 -m json.tool | grep -A3 "OpenLDAP-LabCorp"
```

#### Salida esperada

```json
{
  "name": "OpenLDAP-LabCorp",
  "connectorType": "LDAP",
  "host": "openldap",
  "port": 389,
  "status": "ACTIVE"
}
```

#### Verificación

```bash
# Probar conexión LDAP desde el contenedor OpenIAM usando las credenciales del conector
docker exec lab06-openiam bash -c \
  "ldapsearch -x -H ldap://openldap:389 \
   -D 'cn=admin,dc=labcorp,dc=local' \
   -w 'Admin1234!' \
   -b 'ou=people,dc=labcorp,dc=local' \
   '(objectClass=inetOrgPerson)' uid mail 2>&1 | head -20"
```

---

### Paso 3: Crear un usuario en OpenIAM y verificar el aprovisionamiento en OpenLDAP

**Objetivo:** Crear un usuario ficticio en OpenIAM y confirmar que el motor de aprovisionamiento lo crea automáticamente en OpenLDAP a través del conector LDAP configurado.

#### Instrucciones

1. En la consola de OpenIAM, navega a **Identity → Users → Add User**.

2. Completa el formulario con el siguiente usuario ficticio:

| Campo | Valor |
|---|---|
| **First Name** | `Carlos` |
| **Last Name** | `Mendoza` |
| **Login** | `carlos.mendoza` |
| **Email** | `carlos.mendoza@labcorp.local` |
| **Department** | `Engineering` |
| **Phone** | `+1-555-0101` |
| **Password** | `LabUser2024!` |
| **Confirm Password** | `LabUser2024!` |

3. En la sección **Resource Assignments**, asigna el recurso `OpenLDAP-LabCorp` al usuario y haz clic en **Save**.

4. Alternativamente, crea el usuario mediante la API SCIM de OpenIAM (método programático, recomendado para validar la integración):

```bash
# Usar el token obtenido en el paso anterior
# Si el token expiró, volver a autenticarse:
OPENIAM_TOKEN=$(curl -s -X POST http://localhost:9090/openiam-rest/api/auth/login \
  -H "Content-Type: application/json" \
  -d '{"login":"sysadmin","password":"Admin1234!","managedSysId":"0"}' \
  | python3 -c "import sys,json; print(json.load(sys.stdin).get('token',''))")

# Crear usuario via SCIM 2.0 (RFC 7644)
USER_RESPONSE=$(curl -s -X POST http://localhost:9090/scim/v2/Users \
  -H "Authorization: Bearer $OPENIAM_TOKEN" \
  -H "Content-Type: application/scim+json" \
  -d '{
    "schemas": [
      "urn:ietf:params:scim:schemas:core:2.0:User",
      "urn:ietf:params:scim:schemas:extension:enterprise:2.0:User"
    ],
    "userName": "ana.torres",
    "name": {
      "givenName": "Ana",
      "familyName": "Torres"
    },
    "displayName": "Ana Torres",
    "active": true,
    "emails": [
      {"value": "ana.torres@labcorp.local", "primary": true, "type": "work"}
    ],
    "phoneNumbers": [
      {"value": "+1-555-0202", "type": "mobile"}
    ],
    "urn:ietf:params:scim:schemas:extension:enterprise:2.0:User": {
      "department": "Analytics",
      "organization": "LabCorp"
    }
  }')

echo $USER_RESPONSE | python3 -m json.tool

# Extraer el ID del usuario creado
USER_ID=$(echo $USER_RESPONSE | python3 -c "import sys,json; print(json.load(sys.stdin).get('id',''))")
echo "Usuario creado con ID: $USER_ID"
```

5. Verifica que el usuario fue aprovisionado en OpenLDAP:

```bash
# Buscar el usuario en OpenLDAP (esperar ~5-10 segundos para que el conector procese)
sleep 10

docker exec lab06-openldap ldapsearch \
  -x \
  -H ldap://localhost:389 \
  -D "cn=admin,dc=labcorp,dc=local" \
  -w "Admin1234!" \
  -b "ou=people,dc=labcorp,dc=local" \
  "(uid=ana.torres)" \
  uid givenName sn mail departmentNumber
```

6. Verifica el estado del job de aprovisionamiento en OpenIAM:

```bash
curl -s -X GET "http://localhost:9090/openiam-rest/api/provisioning/jobs?userId=$USER_ID" \
  -H "Authorization: Bearer $OPENIAM_TOKEN" \
  | python3 -m json.tool | grep -E "status|taskName|completedDate"
```

#### Salida esperada

```ldif
# ana.torres, people, labcorp.local
dn: uid=ana.torres,ou=people,dc=labcorp,dc=local
uid: ana.torres
givenName: Ana
sn: Torres
mail: ana.torres@labcorp.local
departmentNumber: Analytics
```

#### Verificación

```bash
# Contar usuarios en LDAP (debe ser >= 1 después del aprovisionamiento)
docker exec lab06-openldap ldapsearch \
  -x -H ldap://localhost:389 \
  -D "cn=admin,dc=labcorp,dc=local" \
  -w "Admin1234!" \
  -b "ou=people,dc=labcorp,dc=local" \
  "(objectClass=inetOrgPerson)" dn \
  | grep "^dn:" | wc -l
```

---

### Paso 4: Configurar User Federation LDAP en Keycloak

**Objetivo:** Conectar Keycloak con OpenLDAP mediante User Federation para que Keycloak pueda autenticar usuarios cuyas identidades residen en OpenLDAP (aprovisionadas por OpenIAM).

#### Instrucciones

1. Accede a la consola de administración de Keycloak:
   - URL: `http://localhost:8080/`
   - Usuario: `admin`
   - Contraseña: `Admin1234!`

2. Selecciona el realm `master` (o crea uno nuevo llamado `labcorp` para mejor aislamiento — recomendado).

   Para crear el realm `labcorp`:
   - Haz clic en el menú desplegable de realm (arriba a la izquierda) → **Create realm**
   - Name: `labcorp`
   - Enabled: `ON`
   - Haz clic en **Create**

3. Dentro del realm `labcorp`, navega a **User Federation → Add provider → ldap**.

4. Configura el proveedor LDAP con los siguientes parámetros:

| Campo | Valor |
|---|---|
| **UI display name** | `OpenLDAP-LabCorp` |
| **Vendor** | `Other` |
| **Connection URL** | `ldap://openldap:389` |
| **Enable StartTLS** | `OFF` |
| **Use Truststore SPI** | `Never` |
| **Bind Type** | `simple` |
| **Bind DN** | `cn=admin,dc=labcorp,dc=local` |
| **Bind Credential** | `Admin1234!` |
| **Edit Mode** | `READ_ONLY` |
| **Users DN** | `ou=people,dc=labcorp,dc=local` |
| **Username LDAP attribute** | `uid` |
| **RDN LDAP attribute** | `uid` |
| **UUID LDAP attribute** | `entryUUID` |
| **User Object Classes** | `inetOrgPerson, organizationalPerson` |
| **Search Scope** | `One Level` |
| **Read Timeout** | `5000` |
| **Pagination** | `ON` |
| **Sync Registrations** | `OFF` |
| **Periodic Full Sync** | `ON` |
| **Full Sync Period** | `86400` (24h) |
| **Periodic Changed Users Sync** | `ON` |
| **Changed Users Sync Period** | `3600` (1h) |

5. Haz clic en **Test connection** → debe mostrar `Successfully connected to LDAP`.

6. Haz clic en **Test authentication** → debe mostrar `Successfully authenticated to LDAP`.

7. Haz clic en **Save**.

8. Configura los **Mappers** de atributos LDAP. Navega a la pestaña **Mappers** del proveedor LDAP y verifica/agrega los siguientes mappers:

| Mapper Name | Mapper Type | LDAP Attribute | User Model Attribute |
|---|---|---|---|
| `username` | user-attribute-ldap-mapper | `uid` | `username` |
| `first name` | user-attribute-ldap-mapper | `givenName` | `firstName` |
| `last name` | user-attribute-ldap-mapper | `sn` | `lastName` |
| `email` | user-attribute-ldap-mapper | `mail` | `email` |

9. Ejecuta la sincronización inicial desde la UI de Keycloak:
   - Navega a **User Federation → OpenLDAP-LabCorp**
   - Haz clic en **Synchronize all users**

10. Verifica via API de Keycloak que los usuarios se sincronizaron:

```bash
# Obtener token de admin de Keycloak
KC_TOKEN=$(curl -s -X POST http://localhost:8080/realms/master/protocol/openid-connect/token \
  -d "grant_type=password" \
  -d "client_id=admin-cli" \
  -d "username=admin" \
  -d "password=Admin1234!" \
  | python3 -c "import sys,json; print(json.load(sys.stdin)['access_token'])")

# Listar usuarios en el realm labcorp
curl -s -X GET "http://localhost:8080/admin/realms/labcorp/users" \
  -H "Authorization: Bearer $KC_TOKEN" \
  | python3 -m json.tool | grep -E '"username"|"email"|"firstName"'
```

#### Salida esperada

```
Successfully connected to LDAP
Successfully authenticated to LDAP
Sync of users finished successfully. X imported users, 0 failed.
```

```json
"username": "ana.torres",
"email": "ana.torres@labcorp.local",
"firstName": "Ana",
```

#### Verificación

```bash
# Verificar que el usuario puede autenticarse contra Keycloak
# (Keycloak delegará la validación de contraseña a OpenLDAP)
curl -s -X POST http://localhost:8080/realms/labcorp/protocol/openid-connect/token \
  -d "grant_type=password" \
  -d "client_id=admin-cli" \
  -d "username=ana.torres" \
  -d "password=LabUser2024!" \
  | python3 -c "import sys,json; d=json.load(sys.stdin); print('AUTH OK - Token type:', d.get('token_type','ERROR'), '| Expires in:', d.get('expires_in','N/A'))"
```

---

### Paso 5: Configurar la reconciliación en OpenIAM

**Objetivo:** Activar la tarea de reconciliación bidireccional para que los cambios realizados directamente en OpenLDAP se reflejen en OpenIAM, garantizando consistencia entre ambos sistemas.

#### Instrucciones

1. En la consola de OpenIAM, navega a **Provisioning → Reconciliation → Add Reconciliation Task**.

2. Configura la tarea con los siguientes parámetros:

| Campo | Valor |
|---|---|
| **Task Name** | `Reconcile-OpenLDAP-LabCorp` |
| **Managed System** | `OpenLDAP-LabCorp` |
| **Reconciliation Type** | `FULL` |
| **Schedule** | `0 */30 * * * ?` (cada 30 minutos) |
| **On Match** | `UPDATE` |
| **On No Match in IDM** | `CREATE` |
| **On No Match in Target** | `DISABLE` |

3. Guarda la tarea y ejecútala manualmente por primera vez:
   - Haz clic en **Run Now** junto a `Reconcile-OpenLDAP-LabCorp`.

4. Simula un cambio directo en OpenLDAP (como si un administrador LDAP actualizara un atributo fuera de OpenIAM) y verifica que la reconciliación lo detecta:

```bash
# Modificar el número de teléfono de ana.torres directamente en OpenLDAP
docker exec lab06-openldap ldapmodify \
  -x \
  -H ldap://localhost:389 \
  -D "cn=admin,dc=labcorp,dc=local" \
  -w "Admin1234!" <<EOF
dn: uid=ana.torres,ou=people,dc=labcorp,dc=local
changetype: modify
replace: telephoneNumber
telephoneNumber: +1-555-9999
EOF

echo "Modificación directa en LDAP aplicada."
```

5. Espera 30 segundos y ejecuta la reconciliación manualmente:

```bash
# Disparar reconciliación via API (endpoint ilustrativo de OpenIAM)
curl -s -X POST "http://localhost:9090/openiam-rest/api/reconciliation/run" \
  -H "Authorization: Bearer $OPENIAM_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"taskName": "Reconcile-OpenLDAP-LabCorp"}' \
  | python3 -m json.tool
```

6. Verifica en OpenIAM que el atributo `telephoneNumber` del usuario `ana.torres` se actualizó:

```bash
curl -s -X GET "http://localhost:9090/scim/v2/Users?filter=userName%20eq%20%22ana.torres%22" \
  -H "Authorization: Bearer $OPENIAM_TOKEN" \
  | python3 -m json.tool | grep -A2 "phoneNumbers"
```

#### Salida esperada

```json
{
  "taskName": "Reconcile-OpenLDAP-LabCorp",
  "status": "COMPLETED",
  "usersProcessed": 2,
  "usersUpdated": 1,
  "errors": 0
}
```

```json
"phoneNumbers": [
  {"value": "+1-555-9999", "type": "mobile"}
]
```

#### Verificación

```bash
# Consultar log de auditoría de la reconciliación
curl -s -X GET "http://localhost:9090/openiam-rest/api/audit/events?eventType=RECONCILIATION&limit=5" \
  -H "Authorization: Bearer $OPENIAM_TOKEN" \
  | python3 -m json.tool | grep -E "eventType|timestamp|status|userId"
```

---

### Paso 6: Probar el flujo completo de aprovisionamiento con script Python

**Objetivo:** Automatizar y validar el flujo completo end-to-end usando un script Python que crea un usuario en OpenIAM via SCIM, verifica su existencia en OpenLDAP y confirma que puede autenticarse en Keycloak.

#### Instrucciones

1. Instala las dependencias Python necesarias:

```bash
pip3 install requests ldap3 faker --quiet
```

2. Crea el script de prueba `~/iam-labs/lab06/test_full_flow.py`:

```python
#!/usr/bin/env python3
"""
Lab 06-00-01: Script de prueba del flujo completo de aprovisionamiento.
Todos los datos de usuario son FICTICIOS (generados con Faker).
"""
import time
import json
import requests
import sys
from ldap3 import Server, Connection, ALL, SUBTREE
from faker import Faker

# ─── Configuración ────────────────────────────────────────────────────────────
OPENIAM_BASE   = "http://localhost:9090"
KEYCLOAK_BASE  = "http://localhost:8080"
LDAP_HOST      = "localhost"
LDAP_PORT      = 389
LDAP_BIND_DN   = "cn=admin,dc=labcorp,dc=local"
LDAP_BIND_PW   = "Admin1234!"
LDAP_BASE_DN   = "ou=people,dc=labcorp,dc=local"
KC_REALM       = "labcorp"
USER_PASSWORD  = "TestUser2024!"

requests.packages.urllib3.disable_warnings()
fake = Faker('es_ES')

# ─── Paso 1: Generar datos ficticios ─────────────────────────────────────────
first_name = fake.first_name()
last_name  = fake.last_name()
username   = f"{first_name.lower()}.{last_name.lower()}".replace(" ", "")
email      = f"{username}@labcorp.local"
department = fake.job()[:30]

print(f"\n{'='*60}")
print(f"USUARIO FICTICIO GENERADO:")
print(f"  Nombre:     {first_name} {last_name}")
print(f"  Login:      {username}")
print(f"  Email:      {email}")
print(f"  Dpto:       {department}")
print(f"{'='*60}\n")

# ─── Paso 2: Autenticarse en OpenIAM ─────────────────────────────────────────
print("[1/5] Autenticando en OpenIAM...")
auth_resp = requests.post(
    f"{OPENIAM_BASE}/openiam-rest/api/auth/login",
    json={"login": "sysadmin", "password": "Admin1234!", "managedSysId": "0"},
    verify=False, timeout=15
)
if auth_resp.status_code != 200:
    print(f"ERROR: No se pudo autenticar en OpenIAM. Status: {auth_resp.status_code}")
    sys.exit(1)

token = auth_resp.json().get("token")
headers = {
    "Authorization": f"Bearer {token}",
    "Content-Type": "application/scim+json",
    "Accept": "application/scim+json"
}
print(f"   ✓ Token obtenido: {token[:20]}...")

# ─── Paso 3: Crear usuario en OpenIAM via SCIM 2.0 ───────────────────────────
print(f"\n[2/5] Creando usuario '{username}' en OpenIAM via SCIM 2.0...")
scim_payload = {
    "schemas": [
        "urn:ietf:params:scim:schemas:core:2.0:User",
        "urn:ietf:params:scim:schemas:extension:enterprise:2.0:User"
    ],
    "userName": username,
    "name": {"givenName": first_name, "familyName": last_name},
    "displayName": f"{first_name} {last_name}",
    "active": True,
    "emails": [{"value": email, "primary": True, "type": "work"}],
    "phoneNumbers": [{"value": fake.phone_number(), "type": "mobile"}],
    "urn:ietf:params:scim:schemas:extension:enterprise:2.0:User": {
        "department": department,
        "organization": "LabCorp"
    }
}

create_resp = requests.post(
    f"{OPENIAM_BASE}/scim/v2/Users",
    headers=headers,
    json=scim_payload,
    verify=False, timeout=15
)

if create_resp.status_code not in (200, 201):
    print(f"ERROR al crear usuario. Status: {create_resp.status_code}")
    print(create_resp.text[:500])
    sys.exit(1)

user_id = create_resp.json().get("id")
print(f"   ✓ Usuario creado en OpenIAM. ID: {user_id}")

# ─── Paso 4: Verificar aprovisionamiento en OpenLDAP ─────────────────────────
print(f"\n[3/5] Esperando aprovisionamiento en OpenLDAP (15s)...")
time.sleep(15)

server = Server(LDAP_HOST, port=LDAP_PORT, get_info=ALL)
conn   = Connection(server, user=LDAP_BIND_DN, password=LDAP_BIND_PW, auto_bind=True)

conn.search(
    search_base=LDAP_BASE_DN,
    search_filter=f"(uid={username})",
    search_scope=SUBTREE,
    attributes=["uid", "givenName", "sn", "mail", "departmentNumber"]
)

if not conn.entries:
    print(f"   ✗ Usuario '{username}' NO encontrado en OpenLDAP.")
    print("     Verifica el conector LDAP y los logs de aprovisionamiento.")
    sys.exit(1)

ldap_user = conn.entries[0]
print(f"   ✓ Usuario encontrado en OpenLDAP:")
print(f"     DN:    {ldap_user.entry_dn}")
print(f"     uid:   {ldap_user.uid}")
print(f"     mail:  {ldap_user.mail}")
conn.unbind()

# ─── Paso 5: Sincronizar Keycloak y verificar autenticación ──────────────────
print(f"\n[4/5] Disparando sincronización en Keycloak...")
kc_admin_token_resp = requests.post(
    f"{KEYCLOAK_BASE}/realms/master/protocol/openid-connect/token",
    data={
        "grant_type": "password",
        "client_id": "admin-cli",
        "username": "admin",
        "password": "Admin1234!"
    },
    timeout=15
)
kc_admin_token = kc_admin_token_resp.json().get("access_token")

# Buscar el ID del proveedor LDAP en Keycloak
fed_resp = requests.get(
    f"{KEYCLOAK_BASE}/admin/realms/{KC_REALM}/components?type=org.keycloak.storage.UserStorageProvider",
    headers={"Authorization": f"Bearer {kc_admin_token}"},
    timeout=15
)
providers = fed_resp.json()
ldap_provider_id = next(
    (p["id"] for p in providers if "OpenLDAP" in p.get("name", "")), None
)

if ldap_provider_id:
    sync_resp = requests.post(
        f"{KEYCLOAK_BASE}/admin/realms/{KC_REALM}/user-storage/{ldap_provider_id}/sync?action=triggerChangedUsersSync",
        headers={"Authorization": f"Bearer {kc_admin_token}"},
        timeout=30
    )
    print(f"   ✓ Sincronización disparada. Status: {sync_resp.status_code}")
else:
    print("   ⚠ Proveedor LDAP no encontrado en Keycloak. Sincronizando manualmente.")

time.sleep(5)

# ─── Paso 6: Verificar autenticación en Keycloak ─────────────────────────────
print(f"\n[5/5] Verificando autenticación de '{username}' en Keycloak...")
auth_test = requests.post(
    f"{KEYCLOAK_BASE}/realms/{KC_REALM}/protocol/openid-connect/token",
    data={
        "grant_type": "password",
        "client_id": "admin-cli",
        "username": username,
        "password": USER_PASSWORD
    },
    timeout=15
)

if auth_test.status_code == 200:
    token_data = auth_test.json()
    print(f"   ✓ AUTENTICACIÓN EXITOSA")
    print(f"     Token type:  {token_data.get('token_type')}")
    print(f"     Expires in:  {token_data.get('expires_in')}s")
    print(f"     Scope:       {token_data.get('scope')}")
else:
    print(f"   ✗ Autenticación fallida. Status: {auth_test.status_code}")
    print(f"     Respuesta: {auth_test.text[:300]}")
    print("     Nota: Puede requerir sincronización manual o cambio de contraseña en LDAP.")

print(f"\n{'='*60}")
print("FLUJO COMPLETO FINALIZADO")
print(f"{'='*60}\n")
```

3. Ejecuta el script:

```bash
cd ~/iam-labs/lab06
python3 test_full_flow.py
```

#### Salida esperada

```
============================================================
USUARIO FICTICIO GENERADO:
  Nombre:     Lucía Fernández
  Login:      lucia.fernandez
  Email:      lucia.fernandez@labcorp.local
  Dpto:       Analista de sistemas
============================================================

[1/5] Autenticando en OpenIAM...
   ✓ Token obtenido: eyJhbGciOiJSUzI1NiIs...

[2/5] Creando usuario 'lucia.fernandez' en OpenIAM via SCIM 2.0...
   ✓ Usuario creado en OpenIAM. ID: a7f3c2d1-...

[3/5] Esperando aprovisionamiento en OpenLDAP (15s)...
   ✓ Usuario encontrado en OpenLDAP:
     DN:    uid=lucia.fernandez,ou=people,dc=labcorp,dc=local
     uid:   lucia.fernandez
     mail:  lucia.fernandez@labcorp.local

[4/5] Disparando sincronización en Keycloak...
   ✓ Sincronización disparada. Status: 200

[5/5] Verificando autenticación de 'lucia.fernandez' en Keycloak...
   ✓ AUTENTICACIÓN EXITOSA
     Token type:  Bearer
     Expires in:  300s
     Scope:       openid profile email

============================================================
FLUJO COMPLETO FINALIZADO
============================================================
```

#### Verificación

```bash
# Verificar que el usuario aparece en Keycloak con atributos LDAP
KC_TOKEN=$(curl -s -X POST http://localhost:8080/realms/master/protocol/openid-connect/token \
  -d "grant_type=password&client_id=admin-cli&username=admin&password=Admin1234!" \
  | python3 -c "import sys,json; print(json.load(sys.stdin)['access_token'])")

curl -s "http://localhost:8080/admin/realms/labcorp/users?search=lucia.fernandez" \
  -H "Authorization: Bearer $KC_TOKEN" \
  | python3 -m json.tool | grep -E '"username"|"email"|"firstName"|"lastName"'
```

---

### Paso 7: Probar escenarios de ciclo de vida (desactivación y cambio de contraseña)

**Objetivo:** Validar que las operaciones de desactivación de usuario y cambio de contraseña en OpenIAM se propagan correctamente a OpenLDAP y se reflejan en Keycloak.

#### Instrucciones

1. **Escenario A — Desactivación de usuario:**

```bash
# Obtener el ID del usuario ana.torres en OpenIAM
USER_ID=$(curl -s -X GET \
  "http://localhost:9090/scim/v2/Users?filter=userName%20eq%20%22ana.torres%22" \
  -H "Authorization: Bearer $OPENIAM_TOKEN" \
  | python3 -c "import sys,json; d=json.load(sys.stdin); print(d['Resources'][0]['id'] if d.get('Resources') else '')")

echo "ID de ana.torres: $USER_ID"

# Desactivar usuario via SCIM PATCH (RFC 7644 - actualización parcial)
curl -s -X PATCH "http://localhost:9090/scim/v2/Users/$USER_ID" \
  -H "Authorization: Bearer $OPENIAM_TOKEN" \
  -H "Content-Type: application/scim+json" \
  -d '{
    "schemas": ["urn:ietf:params:scim:api:messages:2.0:PatchOp"],
    "Operations": [
      {
        "op": "replace",
        "path": "active",
        "value": false
      }
    ]
  }' | python3 -m json.tool | grep '"active"'
```

2. Verifica que el usuario fue desactivado en OpenLDAP (atributo `pwdAccountLockedTime` o eliminación del objeto según configuración del conector):

```bash
sleep 10
docker exec lab06-openldap ldapsearch \
  -x -H ldap://localhost:389 \
  -D "cn=admin,dc=labcorp,dc=local" \
  -w "Admin1234!" \
  -b "ou=people,dc=labcorp,dc=local" \
  "(uid=ana.torres)" \
  uid pwdAccountLockedTime shadowExpire
```

3. **Escenario B — Cambio de contraseña:**

```bash
# Cambiar contraseña de carlos.mendoza via API OpenIAM
CARLOS_ID=$(curl -s -X GET \
  "http://localhost:9090/scim/v2/Users?filter=userName%20eq%20%22carlos.mendoza%22" \
  -H "Authorization: Bearer $OPENIAM_TOKEN" \
  | python3 -c "import sys,json; d=json.load(sys.stdin); print(d['Resources'][0]['id'] if d.get('Resources') else '')")

curl -s -X PATCH "http://localhost:9090/scim/v2/Users/$CARLOS_ID" \
  -H "Authorization: Bearer $OPENIAM_TOKEN" \
  -H "Content-Type: application/scim+json" \
  -d '{
    "schemas": ["urn:ietf:params:scim:api:messages:2.0:PatchOp"],
    "Operations": [
      {
        "op": "replace",
        "path": "password",
        "value": "NuevaPass2024!"
      }
    ]
  }' | python3 -c "import sys,json; d=json.load(sys.stdin); print('Contraseña actualizada' if d.get('id') else 'Error:', d)"
```

4. Verifica la autenticación con la nueva contraseña en Keycloak:

```bash
curl -s -X POST http://localhost:8080/realms/labcorp/protocol/openid-connect/token \
  -d "grant_type=password&client_id=admin-cli&username=carlos.mendoza&password=NuevaPass2024!" \
  | python3 -c "import sys,json; d=json.load(sys.stdin); print('LOGIN OK' if d.get('access_token') else f'ERROR: {d.get(\"error_description\",d)}')"
```

#### Salida esperada

```
"active": false

LOGIN OK
```

#### Verificación

```bash
# Intentar autenticación con contraseña ANTIGUA (debe fallar)
curl -s -X POST http://localhost:8080/realms/labcorp/protocol/openid-connect/token \
  -d "grant_type=password&client_id=admin-cli&username=carlos.mendoza&password=LabUser2024!" \
  | python3 -c "import sys,json; d=json.load(sys.stdin); print('ESPERADO: Error -', d.get('error_description',''))"
```

---

## 7. Validación y Pruebas

### 7.1 Checklist de validación del laboratorio

Ejecuta el siguiente script de validación integral:

```bash
cat > ~/iam-labs/lab06/validate_lab.sh << 'EOF'
#!/bin/bash
echo "=================================================="
echo "  VALIDACIÓN LAB 06-00-01"
echo "=================================================="
PASS=0; FAIL=0

check() {
  local desc="$1"; local cmd="$2"; local expected="$3"
  local result
  result=$(eval "$cmd" 2>&1)
  if echo "$result" | grep -qi "$expected"; then
    echo "  ✓ $desc"
    ((PASS++))
  else
    echo "  ✗ $desc"
    echo "    Resultado: $(echo $result | head -c 100)"
    ((FAIL++))
  fi
}

echo ""
echo "── Contenedores activos ──────────────────────────"
check "OpenLDAP corriendo" "docker inspect lab06-openldap --format '{{.State.Status}}'" "running"
check "OpenIAM corriendo"  "docker inspect lab06-openiam --format '{{.State.Status}}'"  "running"
check "Keycloak corriendo" "docker inspect lab06-keycloak --format '{{.State.Status}}'" "running"
check "PostgreSQL corriendo" "docker inspect lab06-postgres --format '{{.State.Status}}'" "running"

echo ""
echo "── Conectividad LDAP ─────────────────────────────"
check "Puerto 389 accesible" \
  "docker exec lab06-openiam bash -c 'nc -zv openldap 389 2>&1'" "open\|succeeded\|Connected"
check "OUs base creadas" \
  "docker exec lab06-openldap ldapsearch -x -H ldap://localhost:389 -D 'cn=admin,dc=labcorp,dc=local' -w 'Admin1234!' -b 'dc=labcorp,dc=local' '(objectClass=organizationalUnit)' dn" \
  "ou=people"

echo ""
echo "── Usuarios aprovisionados en LDAP ──────────────"
check "Usuarios en LDAP >= 1" \
  "docker exec lab06-openldap ldapsearch -x -H ldap://localhost:389 -D 'cn=admin,dc=labcorp,dc=local' -w 'Admin1234!' -b 'ou=people,dc=labcorp,dc=local' '(objectClass=inetOrgPerson)' dn | grep -c '^dn:'" \
  "[1-9]"

echo ""
echo "── Keycloak User Federation ──────────────────────"
check "Keycloak responde" \
  "curl -s -o /dev/null -w '%{http_code}' http://localhost:8080/" "200\|303"
check "Realm labcorp existe" \
  "curl -s http://localhost:8080/realms/labcorp | python3 -c \"import sys,json; print(json.load(sys.stdin).get('realm',''))\"" \
  "labcorp"

echo ""
echo "── APIs OpenIAM ──────────────────────────────────"
check "OpenIAM UI responde" \
  "curl -s -o /dev/null -w '%{http_code}' http://localhost:9090/openiam-ui-static/" "200\|302"

echo ""
echo "=================================================="
echo "  RESULTADO: $PASS pasaron | $FAIL fallaron"
echo "=================================================="
EOF
chmod +x ~/iam-labs/lab06/validate_lab.sh
bash ~/iam-labs/lab06/validate_lab.sh
```

### 7.2 Verificación del flujo de auditoría

```bash
# Consultar los últimos eventos de aprovisionamiento en OpenIAM
curl -s -X GET "http://localhost:9090/openiam-rest/api/audit/events?limit=10&eventType=PROVISIONING" \
  -H "Authorization: Bearer $OPENIAM_TOKEN" \
  | python3 -m json.tool 2>/dev/null | grep -E "eventType|userId|timestamp|status|targetSystem" \
  || echo "Nota: Consultar logs en la consola web de OpenIAM → Reports → Audit"

# Verificar logs del conector LDAP
docker compose logs openiam | grep -i "ldap\|provision\|connector" | tail -20
```

### 7.3 Verificación de tokens OAuth2/OIDC con Postman

Importa la siguiente colección de Postman para verificar el flujo de autenticación:

1. Abre Postman → **Import** → **Raw text** y pega:

```json
{
  "info": {"name": "Lab06 - IAM Integration Validation", "schema": "https://schema.getpostman.com/json/collection/v2.1.0/collection.json"},
  "item": [
    {
      "name": "1. Keycloak - Get Token (Resource Owner)",
      "request": {
        "method": "POST",
        "url": "http://localhost:8080/realms/labcorp/protocol/openid-connect/token",
        "body": {
          "mode": "urlencoded",
          "urlencoded": [
            {"key": "grant_type", "value": "password"},
            {"key": "client_id", "value": "admin-cli"},
            {"key": "username", "value": "carlos.mendoza"},
            {"key": "password", "value": "NuevaPass2024!"}
          ]
        }
      }
    },
    {
      "name": "2. Keycloak - Introspect Token",
      "request": {
        "method": "GET",
        "url": "http://localhost:8080/realms/labcorp/protocol/openid-connect/userinfo",
        "header": [{"key": "Authorization", "value": "Bearer {{access_token}}"}]
      }
    }
  ]
}
```

2. Ejecuta la petición `1` y copia el `access_token` en la variable de entorno `access_token`.
3. Ejecuta la petición `2` y verifica que el `preferred_username` corresponde al usuario LDAP.

---

## 8. Resolución de Problemas

### Problema 1: El conector LDAP de OpenIAM falla con "Connection refused" o "LDAP bind failed"

**Síntomas:**
- El botón **Test Connection** en OpenIAM muestra error `Connection refused to openldap:389` o `LDAP bind failed: invalid credentials`.
- Los logs de OpenIAM muestran `com.openiam.connector.ldap.LDAPConnectorException: Cannot connect`.
- Los usuarios creados en OpenIAM no aparecen en OpenLDAP tras esperar 30 segundos.

**Causa:**
Hay dos causas frecuentes: (a) el nombre de host `openldap` no es resuelto correctamente porque el contenedor de OpenIAM no está en la misma red Docker, o (b) las credenciales del Bind DN están incorrectas (mayúsculas/minúsculas en el DN, espacios extra).

**Solución:**

```bash
# 1. Verificar que ambos contenedores están en la misma red
docker network inspect lab06_iam-network | python3 -c \
  "import sys,json; nets=json.load(sys.stdin); [print(c['Name'],c.get('IPv4Address')) for c in nets[0]['Containers'].values()]"

# 2. Si OpenIAM no está en la red, reconectarlo
docker network connect lab06_iam-network lab06-openiam

# 3. Probar la conexión LDAP manualmente desde dentro del contenedor OpenIAM
docker exec lab06-openiam bash -c \
  "ldapsearch -x -H ldap://openldap:389 \
   -D 'cn=admin,dc=labcorp,dc=local' \
   -w 'Admin1234!' \
   -b 'dc=labcorp,dc=local' \
   '(objectClass=*)' dn 2>&1 | head -10"

# 4. Si el error es de credenciales, verificar la contraseña del admin LDAP
docker exec lab06-openldap ldapsearch \
  -x -H ldap://localhost:389 \
  -D "cn=admin,dc=labcorp,dc=local" \
  -w "Admin1234!" \
  -b "dc=labcorp,dc=local" "(objectClass=*)" dn 2>&1 | head -5

# 5. Reiniciar solo OpenLDAP si es necesario
docker compose restart openldap
sleep 15
# Volver a probar la conexión en la consola de OpenIAM
```

---

### Problema 2: Keycloak no sincroniza usuarios desde OpenLDAP (User Federation muestra 0 usuarios importados)

**Síntomas:**
- La sincronización en Keycloak muestra `0 imported users, 0 failed` pero OpenLDAP tiene usuarios.
- Al intentar autenticar un usuario LDAP en Keycloak, aparece `Invalid user credentials` o `User not found`.
- El log de Keycloak muestra `WARN [org.keycloak.storage.ldap.LDAPStorageProvider]` con `Can't retrieve LDAP users`.

**Causa:**
El atributo **Users DN** configurado en Keycloak User Federation no coincide exactamente con la ubicación de los usuarios en OpenLDAP, o el **Username LDAP attribute** (`uid`) no está presente en los objetos LDAP porque la clase de objeto no incluye `inetOrgPerson` (que requiere `uid` como atributo MAY).

**Solución:**

```bash
# 1. Verificar la estructura real de un usuario en OpenLDAP
docker exec lab06-openldap ldapsearch \
  -x -H ldap://localhost:389 \
  -D "cn=admin,dc=labcorp,dc=local" \
  -w "Admin1234!" \
  -b "ou=people,dc=labcorp,dc=local" \
  "(objectClass=*)" \
  objectClass uid cn 2>&1 | head -30

# 2. Verificar que los objectClass incluyen inetOrgPerson
# Si los usuarios tienen solo 'person' o 'organizationalPerson', añadir inetOrgPerson:
docker exec lab06-openldap ldapmodify \
  -x -H ldap://localhost:389 \
  -D "cn=admin,dc=labcorp,dc=local" \
  -w "Admin1234!" << 'EOF'
dn: uid=ana.torres,ou=people,dc=labcorp,dc=local
changetype: modify
add: objectClass
objectClass: inetOrgPerson
EOF

# 3. En Keycloak, verificar/corregir la configuración de User Federation:
#    - Users DN: debe ser exactamente "ou=people,dc=labcorp,dc=local"
#    - Username LDAP attribute: "uid"
#    - User Object Classes: "inetOrgPerson, organizationalPerson"
#    Navegar a: Keycloak → labcorp realm → User Federation → OpenLDAP-LabCorp → Save

# 4. Forzar sincronización completa via API
KC_TOKEN=$(curl -s -X POST http://localhost:8080/realms/master/protocol/openid-connect/token \
  -d "grant_type=password&client_id=admin-cli&username=admin&password=Admin1234!" \
  | python3 -c "import sys,json; print(json.load(sys.stdin)['access_token'])")

PROVIDER_ID=$(curl -s \
  "http://localhost:8080/admin/realms/labcorp/components?type=org.keycloak.storage.UserStorageProvider" \
  -H "Authorization: Bearer $KC_TOKEN" \
  | python3 -c "import sys,json; providers=json.load(sys.stdin); print(providers[0]['id'] if providers else 'NOT_FOUND')")

echo "Provider ID: $PROVIDER_ID"

curl -s -X POST \
  "http://localhost:8080/admin/realms/labcorp/user-storage/$PROVIDER_ID/sync?action=triggerFullSync" \
  -H "Authorization: Bearer $KC_TOKEN" \
  | python3 -m json.tool

# 5. Verificar resultado
curl -s "http://localhost:8080/admin/realms/labcorp/users" \
  -H "Authorization: Bearer $KC_TOKEN" \
  | python3 -c "import sys,json; users=json.load(sys.stdin); print(f'Usuarios en Keycloak: {len(users)}')"
```

---

## 9. Limpieza del Entorno

Ejecuta los siguientes comandos al finalizar el laboratorio para liberar recursos del sistema:

```bash
cd ~/iam-labs/lab06

# Detener y eliminar todos los contenedores, redes y volúmenes del lab
docker compose down -v

# Verificar que los contenedores se detuvieron
docker ps | grep lab06 && echo "ADVERTENCIA: Aún hay contenedores activos" || echo "✓ Todos los contenedores detenidos"

# Limpiar imágenes no utilizadas (opcional, libera espacio en disco)
docker image prune -f

# Verificar espacio liberado
docker system df

echo "✓ Limpieza completada. Recursos del Lab 06 liberados."
```

> ⚠️ **Nota importante:** El comando `docker compose down -v` **elimina los volúmenes** de datos (OpenLDAP, PostgreSQL). Si necesitas conservar el estado para el Lab 07-00-01, usa `docker compose down` (sin `-v`) para conservar los datos, o crea un snapshot con `docker compose pause` antes de continuar.

```bash
# Alternativa: pausar sin eliminar datos (para continuar en el Lab 07)
# docker compose pause

# Para reanudar:
# docker compose unpause
```

---

## 10. Resumen

### Conceptos aplicados

En este laboratorio implementaste una arquitectura IAM de tres capas funcionales que refleja directamente el modelo descrito en la Lección 6.1:

| Capa (Lección 6.1) | Componente implementado | Función en el laboratorio |
|---|---|---|
| **Gestión de identidades** | OpenIAM 4.2 CE | Ciclo de vida de usuarios, aprovisionamiento, reconciliación |
| **Repositorio autoritativo** | OpenLDAP 2.6 | Almacén de identidades, fuente de verdad para atributos |
| **Gestión de acceso / SSO** | Keycloak 23.x | Autenticación federada, emisión de tokens OAuth2/OIDC |
| **Conectividad** | Conector LDAP (OpenIAM) | Propagación bidireccional OpenIAM ↔ OpenLDAP |
| **APIs externas** | SCIM 2.0 + REST | Creación programática de usuarios, integración automatizable |
| **Persistencia** | PostgreSQL 15 | Metadatos de identidad y auditoría de OpenIAM |

### Flujo validado

```
Administrador
    │
    ▼ (1) Crea usuario via SCIM 2.0 (RFC 7644)
OpenIAM
    │
    ▼ (2) Conector LDAP → aprovisiona uid en ou=people
OpenLDAP
    │
    ▼ (3) User Federation → sincroniza usuario al realm
Keycloak
    │
    ▼ (4) Usuario se autentica → Keycloak delega bind a LDAP → emite JWT
Aplicación cliente
```

### Puntos clave

- **Separación de responsabilidades:** OpenIAM es el punto único de gobierno del ciclo de vida; Keycloak no gestiona identidades directamente, solo autentica delegando a LDAP.
- **APIs SCIM 2.0:** El estándar RFC 7643/7644 permite automatizar aprovisionamiento sin acoplamiento a la implementación interna de OpenIAM.
- **Reconciliación:** La tarea programada garantiza que desviaciones introducidas directamente en LDAP sean detectadas y corregidas por OpenIAM.
- **Datos ficticios:** Todos los usuarios de prueba fueron generados con Faker; nunca usar datos reales en entornos de laboratorio.

### Recursos adicionales

| Recurso | URL |
|---|---|
| Documentación OpenIAM | https://docs.openiam.com |
| SCIM 2.0 Protocol (RFC 7644) | https://www.rfc-editor.org/rfc/rfc7644 |
| SCIM 2.0 Core Schema (RFC 7643) | https://www.rfc-editor.org/rfc/rfc7643 |
| Keycloak User Federation LDAP | https://www.keycloak.org/docs/latest/server_admin/#_ldap |
| ConnId Framework (conectores) | https://github.com/Tirasa/ConnId |
| OpenLDAP Admin Guide | https://www.openldap.org/doc/admin26/ |

---
