# Definir reglas de sincronización y reconciliación

## 1. Metadatos

| Campo            | Valor                                      |
|------------------|--------------------------------------------|
| **Duración**     | 31 minutos                                 |
| **Complejidad**  | Alta                                       |
| **Nivel Bloom**  | Aplicar                                    |
| **Módulo**       | 3 — Sincronización y Reconciliación        |
| **Tecnologías**  | MidPoint 4.8 LTS, OpenLDAP 2.6, ppolicy, Apache Directory Studio, Docker Compose, OpenSSL |

---

## 2. Descripción General

Este laboratorio simula un escenario empresarial real donde dos directorios LDAP —**LDAP-A** (directorio corporativo principal) y **LDAP-B** (directorio de una subsidiaria)— deben mantenerse sincronizados a través de **MidPoint 4.8 LTS** como motor de aprovisionamiento. El estudiante configurará reglas de correlación por atributo `mail`, definirá mappings de transformación para normalizar diferencias de esquema entre ambos directorios (por ejemplo, `displayName` en LDAP-A vs `cn` en LDAP-B), y establecerá políticas de resolución de conflictos. Adicionalmente, se aplicarán políticas de contraseñas mediante el overlay `ppolicy` y medidas de hardening (ACLs restrictivas, deshabilitación de acceso anónimo, TLS) en ambas instancias OpenLDAP.

---

## 3. Objetivos de Aprendizaje

Al completar este laboratorio, el estudiante será capaz de:

- [ ] Configurar reglas de sincronización bidireccional entre dos instancias OpenLDAP con diferentes esquemas de atributos usando MidPoint 4.8.
- [ ] Implementar políticas de resolución de conflictos para escenarios de datos inconsistentes entre directorios.
- [ ] Aplicar políticas de contraseñas y medidas de hardening en OpenLDAP usando el overlay `ppolicy` y ACLs restrictivas.
- [ ] Validar el comportamiento de la sincronización ante escenarios de conflicto mediante pruebas controladas.

---

## 4. Prerequisitos

### Conocimiento Previo
- Haber completado el Lab 1 (02-00-01) o tener experiencia equivalente con MidPoint y OpenLDAP.
- Comprensión de la estructura DIT, DNs, atributos y esquemas LDAP (`inetOrgPerson`, `posixAccount`).
- Familiaridad con los conceptos de sincronización, reconciliación y resolución de conflictos del Módulo 3.
- Conocimiento básico de Docker Compose y edición de archivos YAML/XML.

### Acceso y Herramientas
- Docker Engine 24.x+ y Docker Compose 2.20.x+ instalados y operativos.
- Apache Directory Studio 2.0.x instalado en el equipo anfitrión.
- Acceso a Internet para descarga de imágenes Docker (si no están en caché local).
- Mínimo 8 GB RAM disponibles para este laboratorio (MidPoint + 2× OpenLDAP + PostgreSQL).
- Repositorio del curso clonado localmente con los scripts de datos ficticios.

---

## 5. Entorno del Laboratorio

### Tabla de Recursos de Software

| Componente        | Imagen / Versión                        | Puerto(s) expuesto(s) | Rol                          |
|-------------------|-----------------------------------------|-----------------------|------------------------------|
| MidPoint          | `evolveum/midpoint:4.8`                 | 8080                  | Motor de sincronización IAM  |
| PostgreSQL        | `postgres:15`                           | 5432                  | Base de datos de MidPoint    |
| OpenLDAP-A        | `osixia/openldap:1.5.0`                 | 1389 / 1636           | Directorio corporativo (A)   |
| OpenLDAP-B        | `osixia/openldap:1.5.0`                 | 2389 / 2636           | Directorio subsidiaria (B)   |
| Apache Dir Studio | (instalación local)                     | N/A                   | Cliente LDAP gráfico         |

### Tabla de Credenciales de Laboratorio (Ficticias)

| Servicio     | Usuario / DN                            | Contraseña          |
|--------------|-----------------------------------------|---------------------|
| MidPoint     | `administrator`                         | `5ecr3t`            |
| LDAP-A admin | `cn=admin,dc=corporativo,dc=lab`        | `AdminCorpA2024!`   |
| LDAP-B admin | `cn=admin,dc=subsidiaria,dc=lab`        | `AdminSubB2024!`    |
| PostgreSQL   | `midpoint`                              | `midpoint`          |

> ⚠️ **Nota de seguridad:** Todas las credenciales son ficticias y exclusivas para este entorno de laboratorio aislado. Nunca reutilizar en entornos productivos.

### Preparación del Entorno — Comandos de Configuración Inicial

**Paso 0.1 — Crear la estructura de directorios del laboratorio:**

```bash
mkdir -p ~/lab-03-00-01/{ldap-a,ldap-b,midpoint,certs,ldif}
cd ~/lab-03-00-01
```

**Paso 0.2 — Generar certificados TLS autofirmados para ambas instancias LDAP:**

```bash
# Certificado para LDAP-A
openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
  -keyout certs/ldap-a.key \
  -out certs/ldap-a.crt \
  -subj "/CN=ldap-a/O=Corporativo Lab/C=CO" \
  -addext "subjectAltName=DNS:ldap-a,IP:127.0.0.1"

# Certificado para LDAP-B
openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
  -keyout certs/ldap-b.key \
  -out certs/ldap-b.crt \
  -subj "/CN=ldap-b/O=Subsidiaria Lab/C=CO" \
  -addext "subjectAltName=DNS:ldap-b,IP:127.0.0.1"

chmod 600 certs/*.key
```

**Paso 0.3 — Crear el archivo `docker-compose.yml`:**

```bash
cat > docker-compose.yml << 'EOF'
version: "3.9"

networks:
  iam-net:
    driver: bridge

volumes:
  pg-data:
  ldap-a-data:
  ldap-a-config:
  ldap-b-data:
  ldap-b-config:
  midpoint-home:

services:

  # ── PostgreSQL para MidPoint ──────────────────────────────────
  postgres:
    image: postgres:15
    container_name: mp-postgres
    environment:
      POSTGRES_DB: midpoint
      POSTGRES_USER: midpoint
      POSTGRES_PASSWORD: midpoint
    volumes:
      - pg-data:/var/lib/postgresql/data
    networks:
      - iam-net
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U midpoint"]
      interval: 10s
      timeout: 5s
      retries: 5

  # ── OpenLDAP-A (Directorio Corporativo) ──────────────────────
  ldap-a:
    image: osixia/openldap:1.5.0
    container_name: ldap-a
    hostname: ldap-a
    environment:
      LDAP_ORGANISATION: "Corporativo Lab"
      LDAP_DOMAIN: "corporativo.lab"
      LDAP_ADMIN_PASSWORD: "AdminCorpA2024!"
      LDAP_TLS: "true"
      LDAP_TLS_CRT_FILENAME: "ldap-a.crt"
      LDAP_TLS_KEY_FILENAME: "ldap-a.key"
      LDAP_TLS_CA_CRT_FILENAME: "ldap-a.crt"
      LDAP_TLS_VERIFY_CLIENT: "never"
      LDAP_REMOVE_CONFIG_AFTER_SETUP: "false"
    volumes:
      - ldap-a-data:/var/lib/ldap
      - ldap-a-config:/etc/ldap/slapd.d
      - ./certs/ldap-a.crt:/container/service/slapd/assets/certs/ldap-a.crt:ro
      - ./certs/ldap-a.key:/container/service/slapd/assets/certs/ldap-a.key:ro
    ports:
      - "1389:389"
      - "1636:636"
    networks:
      - iam-net

  # ── OpenLDAP-B (Directorio Subsidiaria) ──────────────────────
  ldap-b:
    image: osixia/openldap:1.5.0
    container_name: ldap-b
    hostname: ldap-b
    environment:
      LDAP_ORGANISATION: "Subsidiaria Lab"
      LDAP_DOMAIN: "subsidiaria.lab"
      LDAP_ADMIN_PASSWORD: "AdminSubB2024!"
      LDAP_TLS: "true"
      LDAP_TLS_CRT_FILENAME: "ldap-b.crt"
      LDAP_TLS_KEY_FILENAME: "ldap-b.key"
      LDAP_TLS_CA_CRT_FILENAME: "ldap-b.crt"
      LDAP_TLS_VERIFY_CLIENT: "never"
      LDAP_REMOVE_CONFIG_AFTER_SETUP: "false"
    volumes:
      - ldap-b-data:/var/lib/ldap
      - ldap-b-config:/etc/ldap/slapd.d
      - ./certs/ldap-b.crt:/container/service/slapd/assets/certs/ldap-b.crt:ro
      - ./certs/ldap-b.key:/container/service/slapd/assets/certs/ldap-b.key:ro
    ports:
      - "2389:389"
      - "2636:636"
    networks:
      - iam-net

  # ── MidPoint 4.8 ─────────────────────────────────────────────
  midpoint:
    image: evolveum/midpoint:4.8
    container_name: midpoint
    depends_on:
      postgres:
        condition: service_healthy
    environment:
      MP_SET_midpoint_repository_jdbcUrl: "jdbc:postgresql://postgres/midpoint"
      MP_SET_midpoint_repository_jdbcUsername: "midpoint"
      MP_SET_midpoint_repository_jdbcPassword: "midpoint"
      MP_SET_midpoint_repository_database: "postgresql"
      MP_UNSET_midpoint_repository_hibernateHbm2ddl: ""
    volumes:
      - midpoint-home:/opt/midpoint/var
    ports:
      - "8080:8080"
    networks:
      - iam-net
EOF
```

**Paso 0.4 — Iniciar el entorno:**

```bash
docker compose up -d
# Esperar ~90 segundos para que MidPoint inicialice completamente
docker compose logs -f midpoint | grep -m1 "MidPoint started"
```

**Salida esperada:**
```
midpoint  | ... MidPoint started (build ...). Initialization complete.
```

---

## 6. Procedimiento Paso a Paso

---

### Paso 1 — Poblar los directorios LDAP con datos ficticios y esquemas diferenciados

**Objetivo:** Crear la estructura DIT en LDAP-A y LDAP-B con usuarios ficticios que representen diferencias de esquema reales: LDAP-A usará `displayName` como atributo descriptivo principal, mientras que LDAP-B usará `cn` con formato diferente.

#### Instrucciones

**1.1 — Crear el LDIF de estructura y usuarios para LDAP-A:**

```bash
cat > ldif/ldap-a-init.ldif << 'EOF'
# ── Unidades Organizativas ──────────────────────────────────────
dn: ou=Personas,dc=corporativo,dc=lab
objectClass: organizationalUnit
ou: Personas

dn: ou=Grupos,dc=corporativo,dc=lab
objectClass: organizationalUnit
ou: Grupos

# ── Usuarios ficticios (esquema con displayName) ────────────────
dn: uid=amartinez,ou=Personas,dc=corporativo,dc=lab
objectClass: inetOrgPerson
objectClass: organizationalPerson
objectClass: person
uid: amartinez
cn: Andrea Martinez
sn: Martinez
givenName: Andrea
displayName: Andrea Martinez (Corp)
mail: amartinez@corporativo.lab
userPassword: {SSHA}TempPass2024Corp!

dn: uid=clopez,ou=Personas,dc=corporativo,dc=lab
objectClass: inetOrgPerson
objectClass: organizationalPerson
objectClass: person
uid: clopez
cn: Carlos Lopez
sn: Lopez
givenName: Carlos
displayName: Carlos Lopez (Corp)
mail: clopez@corporativo.lab
userPassword: {SSHA}TempPass2024Corp!

dn: uid=lrodriguez,ou=Personas,dc=corporativo,dc=lab
objectClass: inetOrgPerson
objectClass: organizationalPerson
objectClass: person
uid: lrodriguez
cn: Laura Rodriguez
sn: Rodriguez
givenName: Laura
displayName: Laura Rodriguez (Corp)
mail: lrodriguez@corporativo.lab
userPassword: {SSHA}TempPass2024Corp!
EOF
```

**1.2 — Cargar el LDIF en LDAP-A:**

```bash
ldapadd -H ldap://localhost:1389 \
  -D "cn=admin,dc=corporativo,dc=lab" \
  -w "AdminCorpA2024!" \
  -f ldif/ldap-a-init.ldif
```

**1.3 — Crear el LDIF para LDAP-B (esquema diferente: sin `displayName`, `cn` con formato distinto):**

```bash
cat > ldif/ldap-b-init.ldif << 'EOF'
# ── Unidades Organizativas ──────────────────────────────────────
dn: ou=Empleados,dc=subsidiaria,dc=lab
objectClass: organizationalUnit
ou: Empleados

dn: ou=Equipos,dc=subsidiaria,dc=lab
objectClass: organizationalUnit
ou: Equipos

# ── Usuarios ficticios (esquema sin displayName, cn diferente) ──
dn: uid=amartinez,ou=Empleados,dc=subsidiaria,dc=lab
objectClass: inetOrgPerson
objectClass: organizationalPerson
objectClass: person
uid: amartinez
cn: MARTINEZ, Andrea
sn: Martinez
givenName: Andrea
mail: amartinez@corporativo.lab
userPassword: {SSHA}TempPass2024Sub!

dn: uid=clopez,ou=Empleados,dc=subsidiaria,dc=lab
objectClass: inetOrgPerson
objectClass: organizationalPerson
objectClass: person
uid: clopez
cn: LOPEZ, Carlos
sn: Lopez
givenName: Carlos
mail: clopez@corporativo.lab
userPassword: {SSHA}TempPass2024Sub!

# Nota: lrodriguez NO existe en LDAP-B (para probar aprovisionamiento)
# Usuario nuevo solo en subsidiaria (para probar reconciliación inversa)
dn: uid=ptorres,ou=Empleados,dc=subsidiaria,dc=lab
objectClass: inetOrgPerson
objectClass: organizationalPerson
objectClass: person
uid: ptorres
cn: TORRES, Pedro
sn: Torres
givenName: Pedro
mail: ptorres@corporativo.lab
userPassword: {SSHA}TempPass2024Sub!
EOF
```

**1.4 — Cargar el LDIF en LDAP-B:**

```bash
ldapadd -H ldap://localhost:2389 \
  -D "cn=admin,dc=subsidiaria,dc=lab" \
  -w "AdminSubB2024!" \
  -f ldif/ldap-b-init.ldif
```

#### Salida Esperada
```
adding new entry "ou=Personas,dc=corporativo,dc=lab"
adding new entry "ou=Grupos,dc=corporativo,dc=lab"
adding new entry "uid=amartinez,ou=Personas,dc=corporativo,dc=lab"
adding new entry "uid=clopez,ou=Personas,dc=corporativo,dc=lab"
adding new entry "uid=lrodriguez,ou=Personas,dc=corporativo,dc=lab"
```
*(Similar para LDAP-B con los DNs de subsidiaria)*

#### Verificación

```bash
# Verificar usuarios en LDAP-A
ldapsearch -H ldap://localhost:1389 \
  -D "cn=admin,dc=corporativo,dc=lab" -w "AdminCorpA2024!" \
  -b "ou=Personas,dc=corporativo,dc=lab" "(objectClass=inetOrgPerson)" \
  uid mail displayName | grep -E "^(uid|mail|displayName):"

# Verificar usuarios en LDAP-B
ldapsearch -H ldap://localhost:2389 \
  -D "cn=admin,dc=subsidiaria,dc=lab" -w "AdminSubB2024!" \
  -b "ou=Empleados,dc=subsidiaria,dc=lab" "(objectClass=inetOrgPerson)" \
  uid mail cn | grep -E "^(uid|mail|cn):"
```

**Resultado esperado LDAP-A:** 3 usuarios con atributo `displayName` en formato `"Nombre Apellido (Corp)"`.  
**Resultado esperado LDAP-B:** 3 usuarios con `cn` en formato `"APELLIDO, Nombre"` y sin `displayName`.

---

### Paso 2 — Configurar el overlay ppolicy y hardening en OpenLDAP-A

**Objetivo:** Aplicar políticas de contraseñas (longitud mínima, complejidad, historial) mediante el overlay `ppolicy`, deshabilitar el acceso anónimo y configurar ACLs restrictivas siguiendo las buenas prácticas de hardening del Módulo 3.

#### Instrucciones

**2.1 — Acceder al contenedor LDAP-A:**

```bash
docker exec -it ldap-a bash
```

**2.2 — Cargar el módulo ppolicy (dentro del contenedor):**

```bash
cat > /tmp/load-ppolicy.ldif << 'EOF'
dn: cn=module,cn=config
changetype: modify
add: olcModuleLoad
olcModuleLoad: ppolicy
EOF

ldapadd -Y EXTERNAL -H ldapi:/// -f /tmp/load-ppolicy.ldif
```

**2.3 — Agregar el overlay ppolicy a la base de datos (dentro del contenedor):**

```bash
cat > /tmp/ppolicy-overlay.ldif << 'EOF'
dn: olcOverlay=ppolicy,olcDatabase={1}mdb,cn=config
objectClass: olcOverlayConfig
objectClass: olcPPolicyConfig
olcOverlay: ppolicy
olcPPolicyDefault: cn=PoliticaDefault,ou=Politicas,dc=corporativo,dc=lab
olcPPolicyHashCleartext: TRUE
olcPPolicyUseLockout: TRUE
EOF

ldapadd -Y EXTERNAL -H ldapi:/// -f /tmp/ppolicy-overlay.ldif
```

**2.4 — Crear la OU de políticas y la política por defecto (dentro del contenedor):**

```bash
cat > /tmp/create-policy.ldif << 'EOF'
dn: ou=Politicas,dc=corporativo,dc=lab
objectClass: organizationalUnit
ou: Politicas

dn: cn=PoliticaDefault,ou=Politicas,dc=corporativo,dc=lab
objectClass: pwdPolicy
objectClass: person
cn: PoliticaDefault
sn: PoliticaDefault
pwdAttribute: userPassword
pwdMinLength: 10
pwdMaxAge: 7776000
pwdInHistory: 5
pwdMaxFailure: 5
pwdLockout: TRUE
pwdLockoutDuration: 300
pwdMustChange: FALSE
pwdAllowUserChange: TRUE
pwdExpireWarning: 604800
pwdGraceAuthNLimit: 0
EOF

ldapadd -H ldap://localhost \
  -D "cn=admin,dc=corporativo,dc=lab" \
  -w "AdminCorpA2024!" \
  -f /tmp/create-policy.ldif
```

**2.5 — Configurar ACLs restrictivas (deshabilitar acceso anónimo) dentro del contenedor:**

```bash
cat > /tmp/acl-hardening.ldif << 'EOF'
dn: olcDatabase={1}mdb,cn=config
changetype: modify
replace: olcAccess
olcAccess: {0}to attrs=userPassword
  by self write
  by dn="cn=admin,dc=corporativo,dc=lab" write
  by dn="cn=midpoint-svc,ou=Personas,dc=corporativo,dc=lab" read
  by anonymous auth
  by * none
olcAccess: {1}to attrs=shadowLastChange
  by self write
  by * read
olcAccess: {2}to *
  by dn="cn=admin,dc=corporativo,dc=lab" write
  by dn="cn=midpoint-svc,ou=Personas,dc=corporativo,dc=lab" read
  by self read
  by anonymous none
EOF

ldapmodify -Y EXTERNAL -H ldapi:/// -f /tmp/acl-hardening.ldif
```

**2.6 — Crear cuenta de servicio para MidPoint (dentro del contenedor):**

```bash
cat > /tmp/midpoint-svc.ldif << 'EOF'
dn: cn=midpoint-svc,ou=Personas,dc=corporativo,dc=lab
objectClass: inetOrgPerson
objectClass: organizationalPerson
objectClass: person
cn: midpoint-svc
sn: ServiceAccount
uid: midpoint-svc
mail: midpoint-svc@corporativo.lab
userPassword: MidpointSvc2024!
EOF

ldapadd -H ldap://localhost \
  -D "cn=admin,dc=corporativo,dc=lab" \
  -w "AdminCorpA2024!" \
  -f /tmp/midpoint-svc.ldif
```

**2.7 — Salir del contenedor:**

```bash
exit
```

#### Salida Esperada
```
modifying entry "olcDatabase={1}mdb,cn=config"
adding new entry "ou=Politicas,dc=corporativo,dc=lab"
adding new entry "cn=PoliticaDefault,ou=Politicas,dc=corporativo,dc=lab"
```

#### Verificación

```bash
# Verificar que el acceso anónimo está deshabilitado
ldapsearch -H ldap://localhost:1389 \
  -b "dc=corporativo,dc=lab" \
  "(objectClass=organizationalUnit)" ou 2>&1 | grep -E "(result|Operations error|Insufficient)"
```

**Resultado esperado:** El servidor debe retornar `result: 1 Operations error` o `result: 50 Insufficient access rights`, confirmando que el acceso anónimo está bloqueado.

---

### Paso 3 — Configurar los recursos LDAP en MidPoint

**Objetivo:** Registrar LDAP-A y LDAP-B como recursos en MidPoint, definiendo los conectores, credenciales de conexión y el mapeo base de atributos para cada directorio.

#### Instrucciones

**3.1 — Acceder a la interfaz web de MidPoint:**

Abrir navegador en `http://localhost:8080/midpoint` e iniciar sesión con:
- Usuario: `administrator`
- Contraseña: `5ecr3t`

**3.2 — Crear el archivo XML de recurso para LDAP-A:**

```bash
cat > midpoint/resource-ldap-a.xml << 'EOF'
<?xml version="1.0" encoding="UTF-8"?>
<resource xmlns="http://midpoint.evolveum.com/xml/ns/public/common/common-3"
          xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
          xmlns:icfs="http://midpoint.evolveum.com/xml/ns/public/connector/icf-1/resource-schema-3"
          xmlns:ri="http://midpoint.evolveum.com/xml/ns/public/resource/instance-3"
          oid="11111111-1111-1111-1111-111111111111">

  <name>LDAP-A Corporativo</name>
  <description>Directorio corporativo principal - OpenLDAP A</description>

  <connectorRef type="ConnectorType">
    <filter>
      <q:equal xmlns:q="http://prism.evolveum.com/xml/ns/public/query-3">
        <q:path>connectorType</q:path>
        <q:value>com.evolveum.polygon.connector.ldap.LdapConnector</q:value>
      </q:equal>
    </filter>
  </connectorRef>

  <connectorConfiguration>
    <icfs:configurationProperties>
      <icfc:host xmlns:icfc="http://midpoint.evolveum.com/xml/ns/public/connector/icf-1/connector-schema-3">ldap-a</icfc:host>
      <icfc:port xmlns:icfc="http://midpoint.evolveum.com/xml/ns/public/connector/icf-1/connector-schema-3">389</icfc:port>
      <icfc:baseContext xmlns:icfc="http://midpoint.evolveum.com/xml/ns/public/connector/icf-1/connector-schema-3">dc=corporativo,dc=lab</icfc:baseContext>
      <icfc:bindDn xmlns:icfc="http://midpoint.evolveum.com/xml/ns/public/connector/icf-1/connector-schema-3">cn=midpoint-svc,ou=Personas,dc=corporativo,dc=lab</icfc:bindDn>
      <icfc:bindPassword xmlns:icfc="http://midpoint.evolveum.com/xml/ns/public/connector/icf-1/connector-schema-3">
        <t:clearValue xmlns:t="http://prism.evolveum.com/xml/ns/public/types-3">MidpointSvc2024!</t:clearValue>
      </icfc:bindPassword>
      <icfc:usePasswordPolicy xmlns:icfc="http://midpoint.evolveum.com/xml/ns/public/connector/icf-1/connector-schema-3">false</icfc:usePasswordPolicy>
    </icfs:configurationProperties>
  </connectorConfiguration>

  <schemaHandling>
    <objectType>
      <kind>account</kind>
      <displayName>Cuenta LDAP-A</displayName>
      <default>true</default>
      <objectClass>ri:inetOrgPerson</objectClass>
      <baseContext>
        <objectClass>ri:organizationalUnit</objectClass>
        <filter>
          <q:equal xmlns:q="http://prism.evolveum.com/xml/ns/public/query-3">
            <q:path>attributes/dn</q:path>
            <q:value>ou=Personas,dc=corporativo,dc=lab</q:value>
          </q:equal>
        </filter>
      </baseContext>
      <attribute>
        <ref>icfs:name</ref>
        <displayName>DN</displayName>
        <outbound>
          <source><path>$user/name</path></source>
          <expression>
            <script>
              <code>'uid=' + name + ',ou=Personas,dc=corporativo,dc=lab'</code>
            </script>
          </expression>
        </outbound>
      </attribute>
      <attribute>
        <ref>ri:uid</ref>
        <outbound><source><path>$user/name</path></source></outbound>
        <inbound>
          <target><path>$user/name</path></target>
        </inbound>
      </attribute>
      <attribute>
        <ref>ri:mail</ref>
        <outbound><source><path>$user/emailAddress</path></source></outbound>
        <inbound>
          <target><path>$user/emailAddress</path></target>
        </inbound>
      </attribute>
      <attribute>
        <ref>ri:displayName</ref>
        <outbound>
          <source><path>$user/fullName</path></source>
        </outbound>
        <inbound>
          <target><path>$user/fullName</path></target>
        </inbound>
      </attribute>
      <attribute>
        <ref>ri:sn</ref>
        <outbound><source><path>$user/familyName</path></source></outbound>
        <inbound><target><path>$user/familyName</path></target></inbound>
      </attribute>
      <attribute>
        <ref>ri:givenName</ref>
        <outbound><source><path>$user/givenName</path></source></outbound>
        <inbound><target><path>$user/givenName</path></target></inbound>
      </attribute>
      <correlation>
        <correlators>
          <items>
            <item>
              <ref>emailAddress</ref>
            </item>
          </items>
        </correlators>
      </correlation>
      <synchronization>
        <reaction>
          <situation>linked</situation>
          <actions><synchronize/></actions>
        </reaction>
        <reaction>
          <situation>unlinked</situation>
          <actions><link/></actions>
        </reaction>
        <reaction>
          <situation>unmatched</situation>
          <actions><addFocus/></actions>
        </reaction>
      </synchronization>
    </objectType>
  </schemaHandling>
</resource>
EOF
```

**3.3 — Importar el recurso LDAP-A vía API REST de MidPoint:**

```bash
curl -s -u administrator:5ecr3t \
  -H "Content-Type: application/xml" \
  -X POST "http://localhost:8080/midpoint/ws/rest/resources" \
  -d @midpoint/resource-ldap-a.xml \
  -o /tmp/response-ldap-a.json

cat /tmp/response-ldap-a.json | python3 -m json.tool | grep -E "(oid|message|status)"
```

**3.4 — Crear y cargar el recurso LDAP-B** con mapeo de transformación para normalizar `cn` (formato `APELLIDO, Nombre`) hacia `fullName` de MidPoint:

```bash
cat > midpoint/resource-ldap-b.xml << 'EOF'
<?xml version="1.0" encoding="UTF-8"?>
<resource xmlns="http://midpoint.evolveum.com/xml/ns/public/common/common-3"
          xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
          xmlns:icfs="http://midpoint.evolveum.com/xml/ns/public/connector/icf-1/resource-schema-3"
          xmlns:ri="http://midpoint.evolveum.com/xml/ns/public/resource/instance-3"
          oid="22222222-2222-2222-2222-222222222222">

  <name>LDAP-B Subsidiaria</name>
  <description>Directorio subsidiaria - OpenLDAP B</description>

  <connectorRef type="ConnectorType">
    <filter>
      <q:equal xmlns:q="http://prism.evolveum.com/xml/ns/public/query-3">
        <q:path>connectorType</q:path>
        <q:value>com.evolveum.polygon.connector.ldap.LdapConnector</q:value>
      </q:equal>
    </filter>
  </connectorRef>

  <connectorConfiguration>
    <icfs:configurationProperties>
      <icfc:host xmlns:icfc="http://midpoint.evolveum.com/xml/ns/public/connector/icf-1/connector-schema-3">ldap-b</icfc:host>
      <icfc:port xmlns:icfc="http://midpoint.evolveum.com/xml/ns/public/connector/icf-1/connector-schema-3">389</icfc:port>
      <icfc:baseContext xmlns:icfc="http://midpoint.evolveum.com/xml/ns/public/connector/icf-1/connector-schema-3">dc=subsidiaria,dc=lab</icfc:baseContext>
      <icfc:bindDn xmlns:icfc="http://midpoint.evolveum.com/xml/ns/public/connector/icf-1/connector-schema-3">cn=admin,dc=subsidiaria,dc=lab</icfc:bindDn>
      <icfc:bindPassword xmlns:icfc="http://midpoint.evolveum.com/xml/ns/public/connector/icf-1/connector-schema-3">
        <t:clearValue xmlns:t="http://prism.evolveum.com/xml/ns/public/types-3">AdminSubB2024!</t:clearValue>
      </icfc:bindPassword>
    </icfs:configurationProperties>
  </connectorConfiguration>

  <schemaHandling>
    <objectType>
      <kind>account</kind>
      <displayName>Cuenta LDAP-B</displayName>
      <default>true</default>
      <objectClass>ri:inetOrgPerson</objectClass>
      <baseContext>
        <objectClass>ri:organizationalUnit</objectClass>
        <filter>
          <q:equal xmlns:q="http://prism.evolveum.com/xml/ns/public/query-3">
            <q:path>attributes/dn</q:path>
            <q:value>ou=Empleados,dc=subsidiaria,dc=lab</q:value>
          </q:equal>
        </filter>
      </baseContext>
      <attribute>
        <ref>icfs:name</ref>
        <outbound>
          <source><path>$user/name</path></source>
          <expression>
            <script>
              <code>'uid=' + name + ',ou=Empleados,dc=subsidiaria,dc=lab'</code>
            </script>
          </expression>
        </outbound>
      </attribute>
      <attribute>
        <ref>ri:uid</ref>
        <outbound><source><path>$user/name</path></source></outbound>
        <inbound><target><path>$user/name</path></target></inbound>
      </attribute>
      <attribute>
        <ref>ri:mail</ref>
        <outbound><source><path>$user/emailAddress</path></source></outbound>
        <inbound><target><path>$user/emailAddress</path></target></inbound>
      </attribute>
      <!-- Transformación: cn de LDAP-B (formato "APELLIDO, Nombre") → fullName en MidPoint -->
      <attribute>
        <ref>ri:cn</ref>
        <outbound>
          <source><path>$user/familyName</path></source>
          <source><path>$user/givenName</path></source>
          <expression>
            <script>
              <code>familyName.toUpperCase() + ', ' + givenName</code>
            </script>
          </expression>
        </outbound>
        <inbound>
          <expression>
            <script>
              <!-- Convierte "APELLIDO, Nombre" → "Nombre Apellido" para fullName -->
              <code>
                def parts = input?.split(', ')
                if (parts?.size() == 2) {
                  return parts[1] + ' ' + parts[0].capitalize()
                }
                return input
              </code>
            </script>
          </expression>
          <target><path>$user/fullName</path></target>
        </inbound>
      </attribute>
      <attribute>
        <ref>ri:sn</ref>
        <outbound><source><path>$user/familyName</path></source></outbound>
        <inbound><target><path>$user/familyName</path></target></inbound>
      </attribute>
      <attribute>
        <ref>ri:givenName</ref>
        <outbound><source><path>$user/givenName</path></source></outbound>
        <inbound><target><path>$user/givenName</path></target></inbound>
      </attribute>
      <correlation>
        <correlators>
          <items>
            <item>
              <ref>emailAddress</ref>
            </item>
          </items>
        </correlators>
      </correlation>
      <synchronization>
        <reaction>
          <situation>linked</situation>
          <actions><synchronize/></actions>
        </reaction>
        <reaction>
          <situation>unlinked</situation>
          <actions><link/></actions>
        </reaction>
        <reaction>
          <situation>unmatched</situation>
          <actions><addFocus/></actions>
        </reaction>
      </synchronization>
    </objectType>
  </schemaHandling>
</resource>
EOF

curl -s -u administrator:5ecr3t \
  -H "Content-Type: application/xml" \
  -X POST "http://localhost:8080/midpoint/ws/rest/resources" \
  -d @midpoint/resource-ldap-b.xml \
  -o /tmp/response-ldap-b.json

cat /tmp/response-ldap-b.json | python3 -m json.tool | grep -E "(oid|message|status)"
```

#### Salida Esperada

```json
{
  "oid": "11111111-1111-1111-1111-111111111111"
}
```
*(Para LDAP-A; similar para LDAP-B con OID `22222222-...`)*

#### Verificación

En la interfaz web de MidPoint: **Resources → All Resources** debe mostrar dos recursos:
- `LDAP-A Corporativo` — Estado: `UP`
- `LDAP-B Subsidiaria` — Estado: `UP`

Para verificar via REST:

```bash
curl -s -u administrator:5ecr3t \
  "http://localhost:8080/midpoint/ws/rest/resources?options=noFetch" \
  | python3 -c "import sys,json; [print(r['name']) for r in json.load(sys.stdin)['object']['object']]"
```

---

### Paso 4 — Ejecutar reconciliación inicial y verificar correlación por mail

**Objetivo:** Ejecutar la tarea de reconciliación desde LDAP-A hacia MidPoint para importar usuarios y correlacionarlos por el atributo `mail`. Verificar que los usuarios `amartinez` y `clopez` se correlacionen correctamente entre ambos directorios.

#### Instrucciones

**4.1 — Crear y ejecutar tarea de reconciliación para LDAP-A:**

```bash
cat > midpoint/task-recon-ldap-a.xml << 'EOF'
<?xml version="1.0" encoding="UTF-8"?>
<task xmlns="http://midpoint.evolveum.com/xml/ns/public/common/common-3"
      oid="aaaaaaaa-aaaa-aaaa-aaaa-aaaaaaaaaaaa">
  <name>Reconciliacion LDAP-A</name>
  <taskIdentifier>reconciliation-ldap-a-001</taskIdentifier>
  <ownerRef oid="00000000-0000-0000-0000-000000000002" type="UserType"/>
  <executionState>runnable</executionState>
  <schedule>
    <recurrence>single</recurrence>
  </schedule>
  <activity>
    <work>
      <reconciliation>
        <resourceRef oid="11111111-1111-1111-1111-111111111111" type="ResourceType"/>
        <objectclass>ri:inetOrgPerson</objectclass>
      </reconciliation>
    </work>
  </activity>
</task>
EOF

curl -s -u administrator:5ecr3t \
  -H "Content-Type: application/xml" \
  -X POST "http://localhost:8080/midpoint/ws/rest/tasks" \
  -d @midpoint/task-recon-ldap-a.xml

# Esperar 15 segundos para que la tarea complete
sleep 15
```

**4.2 — Verificar usuarios importados en MidPoint:**

```bash
curl -s -u administrator:5ecr3t \
  "http://localhost:8080/midpoint/ws/rest/users?options=noFetch" \
  | python3 -c "
import sys, json
data = json.load(sys.stdin)
users = data.get('object', {}).get('object', [])
for u in users:
    print(f\"Usuario: {u.get('name')} | Email: {u.get('emailAddress','N/A')}\")"
```

**4.3 — Ejecutar reconciliación para LDAP-B:**

```bash
cat > midpoint/task-recon-ldap-b.xml << 'EOF'
<?xml version="1.0" encoding="UTF-8"?>
<task xmlns="http://midpoint.evolveum.com/xml/ns/public/common/common-3"
      oid="bbbbbbbb-bbbb-bbbb-bbbb-bbbbbbbbbbbb">
  <name>Reconciliacion LDAP-B</name>
  <taskIdentifier>reconciliation-ldap-b-001</taskIdentifier>
  <ownerRef oid="00000000-0000-0000-0000-000000000002" type="UserType"/>
  <executionState>runnable</executionState>
  <schedule>
    <recurrence>single</recurrence>
  </schedule>
  <activity>
    <work>
      <reconciliation>
        <resourceRef oid="22222222-2222-2222-2222-222222222222" type="ResourceType"/>
        <objectclass>ri:inetOrgPerson</objectclass>
      </reconciliation>
    </work>
  </activity>
</task>
EOF

curl -s -u administrator:5ecr3t \
  -H "Content-Type: application/xml" \
  -X POST "http://localhost:8080/midpoint/ws/rest/tasks" \
  -d @midpoint/task-recon-ldap-b.xml

sleep 20
```

#### Salida Esperada

La consulta de usuarios en MidPoint debe mostrar al menos:
```
Usuario: amartinez | Email: amartinez@corporativo.lab
Usuario: clopez    | Email: clopez@corporativo.lab
Usuario: lrodriguez| Email: lrodriguez@corporativo.lab
Usuario: ptorres   | Email: ptorres@corporativo.lab
```

#### Verificación

En la interfaz web de MidPoint: **Users → All Users** debe listar los 4 usuarios. Hacer clic en `amartinez` y verificar que en la sección **Projections** aparezcan dos entradas: una para `LDAP-A Corporativo` y otra para `LDAP-B Subsidiaria`, ambas en estado `LINKED`.

> 💡 **Punto de aprendizaje:** La correlación por `mail` permite que MidPoint identifique que `uid=amartinez,ou=Personas,dc=corporativo,dc=lab` y `uid=amartinez,ou=Empleados,dc=subsidiaria,dc=lab` son la misma identidad, aunque residan en árboles DIT completamente diferentes. Esto implementa el concepto de **correlación de identidades** discutido en el Módulo 3.

---

### Paso 5 — Probar escenario de conflicto y verificar política de resolución

**Objetivo:** Simular una modificación simultánea del atributo `fullName`/`displayName` en ambos directorios para un mismo usuario, y verificar que la política de resolución de conflictos (LDAP-A tiene prioridad como fuente autoritativa) funciona correctamente.

#### Instrucciones

**5.1 — Introducir el conflicto: modificar `displayName` en LDAP-A y `cn` en LDAP-B para `amartinez`:**

```bash
# Modificar en LDAP-A (directorio autoritativo)
cat > /tmp/conflict-ldap-a.ldif << 'EOF'
dn: uid=amartinez,ou=Personas,dc=corporativo,dc=lab
changetype: modify
replace: displayName
displayName: Andrea Martinez ACTUALIZADO-CORP
EOF

ldapmodify -H ldap://localhost:1389 \
  -D "cn=admin,dc=corporativo,dc=lab" \
  -w "AdminCorpA2024!" \
  -f /tmp/conflict-ldap-a.ldif

# Modificar en LDAP-B (directorio secundario — valor conflictivo)
cat > /tmp/conflict-ldap-b.ldif << 'EOF'
dn: uid=amartinez,ou=Empleados,dc=subsidiaria,dc=lab
changetype: modify
replace: cn
cn: MARTINEZ-CONFLICTO, Andrea
EOF

ldapmodify -H ldap://localhost:2389 \
  -D "cn=admin,dc=subsidiaria,dc=lab" \
  -w "AdminSubB2024!" \
  -f /tmp/conflict-ldap-b.ldif
```

**5.2 — Ejecutar reconciliación de LDAP-A (fuente autoritativa) primero:**

```bash
# Re-ejecutar reconciliación LDAP-A para que MidPoint tome el valor autoritativo
curl -s -u administrator:5ecr3t \
  -H "Content-Type: application/xml" \
  -X POST "http://localhost:8080/midpoint/ws/rest/tasks" \
  -d @midpoint/task-recon-ldap-a.xml

sleep 15
```

**5.3 — Ejecutar reconciliación de LDAP-B para propagar el valor correcto:**

```bash
curl -s -u administrator:5ecr3t \
  -H "Content-Type: application/xml" \
  -X POST "http://localhost:8080/midpoint/ws/rest/tasks" \
  -d @midpoint/task-recon-ldap-b.xml

sleep 20
```

**5.4 — Verificar el resultado del conflicto en LDAP-B:**

```bash
ldapsearch -H ldap://localhost:2389 \
  -D "cn=admin,dc=subsidiaria,dc=lab" \
  -w "AdminSubB2024!" \
  -b "ou=Empleados,dc=subsidiaria,dc=lab" \
  "(uid=amartinez)" cn givenName sn | grep -E "^(cn|givenName|sn):"
```

#### Salida Esperada

```
cn: MARTINEZ ACTUALIZADO-CORP, Andrea
sn: Martinez
givenName: Andrea
```

> El valor de `cn` en LDAP-B debe reflejar el dato proveniente de LDAP-A (fuente autoritativa), demostrando que la política de resolución de conflictos funcionó: LDAP-A sobrescribió el valor conflictivo de LDAP-B.

#### Verificación

```bash
# Verificar también en MidPoint que fullName refleja el valor de LDAP-A
curl -s -u administrator:5ecr3t \
  "http://localhost:8080/midpoint/ws/rest/users?query=%7B%22filter%22%3A%7B%22text%22%3A%22amartinez%22%7D%7D" \
  | python3 -c "
import sys, json
data = json.load(sys.stdin)
users = data.get('object',{}).get('object',[])
for u in users:
    print(f\"fullName: {u.get('fullName','N/A')}\")"
```

**Resultado esperado:** `fullName: Andrea Martinez ACTUALIZADO-CORP`

---

### Paso 6 — Verificar aprovisionamiento de usuarios únicos entre directorios

**Objetivo:** Confirmar que `lrodriguez` (solo en LDAP-A) fue aprovisionado en LDAP-B, y que `ptorres` (solo en LDAP-B) fue creado en MidPoint y está disponible para aprovisionamiento en LDAP-A.

#### Instrucciones

**6.1 — Verificar que `lrodriguez` fue creado en LDAP-B:**

```bash
ldapsearch -H ldap://localhost:2389 \
  -D "cn=admin,dc=subsidiaria,dc=lab" \
  -w "AdminSubB2024!" \
  -b "ou=Empleados,dc=subsidiaria,dc=lab" \
  "(uid=lrodriguez)" uid mail cn
```

**6.2 — Verificar que `ptorres` existe en MidPoint:**

```bash
curl -s -u administrator:5ecr3t \
  "http://localhost:8080/midpoint/ws/rest/users" \
  | python3 -c "
import sys, json
data = json.load(sys.stdin)
users = data.get('object',{}).get('object',[])
ptorres = [u for u in users if u.get('name') == 'ptorres']
print('ptorres encontrado:', bool(ptorres))
if ptorres:
    print('Email:', ptorres[0].get('emailAddress','N/A'))"
```

#### Salida Esperada

**Para `lrodriguez` en LDAP-B:**
```
dn: uid=lrodriguez,ou=Empleados,dc=subsidiaria,dc=lab
uid: lrodriguez
mail: lrodriguez@corporativo.lab
cn: RODRIGUEZ, Laura
```

**Para `ptorres` en MidPoint:**
```
ptorres encontrado: True
Email: ptorres@corporativo.lab
```

---

## 7. Validación y Pruebas

### Lista de Verificación de Completitud

Ejecutar el siguiente script de validación integral:

```bash
#!/bin/bash
# Script de validación Lab 03-00-01
echo "=== VALIDACIÓN LAB 03-00-01 ==="
PASS=0; FAIL=0

check() {
  local desc="$1"; local cmd="$2"; local expected="$3"
  result=$(eval "$cmd" 2>&1)
  if echo "$result" | grep -q "$expected"; then
    echo "  ✅ PASS: $desc"
    ((PASS++))
  else
    echo "  ❌ FAIL: $desc"
    echo "     Resultado: $result"
    ((FAIL++))
  fi
}

echo ""
echo "── Verificando contenedores activos ──"
check "ldap-a corriendo" "docker ps --filter name=ldap-a --format '{{.Status}}'" "Up"
check "ldap-b corriendo" "docker ps --filter name=ldap-b --format '{{.Status}}'" "Up"
check "midpoint corriendo" "docker ps --filter name=midpoint --format '{{.Status}}'" "Up"

echo ""
echo "── Verificando acceso anónimo bloqueado en LDAP-A ──"
check "Anónimo bloqueado LDAP-A" \
  "ldapsearch -H ldap://localhost:1389 -b 'dc=corporativo,dc=lab' '(objectClass=*)' 2>&1" \
  "Operations error\|Insufficient access\|result: 1\|result: 50"

echo ""
echo "── Verificando usuarios en LDAP-A ──"
check "amartinez en LDAP-A" \
  "ldapsearch -H ldap://localhost:1389 -D 'cn=admin,dc=corporativo,dc=lab' -w 'AdminCorpA2024!' -b 'ou=Personas,dc=corporativo,dc=lab' '(uid=amartinez)' uid 2>&1" \
  "uid: amartinez"
check "lrodriguez en LDAP-A" \
  "ldapsearch -H ldap://localhost:1389 -D 'cn=admin,dc=corporativo,dc=lab' -w 'AdminCorpA2024!' -b 'ou=Personas,dc=corporativo,dc=lab' '(uid=lrodriguez)' uid 2>&1" \
  "uid: lrodriguez"

echo ""
echo "── Verificando sincronización en LDAP-B ──"
check "amartinez en LDAP-B" \
  "ldapsearch -H ldap://localhost:2389 -D 'cn=admin,dc=subsidiaria,dc=lab' -w 'AdminSubB2024!' -b 'ou=Empleados,dc=subsidiaria,dc=lab' '(uid=amartinez)' uid 2>&1" \
  "uid: amartinez"
check "lrodriguez aprovisionado en LDAP-B" \
  "ldapsearch -H ldap://localhost:2389 -D 'cn=admin,dc=subsidiaria,dc=lab' -w 'AdminSubB2024!' -b 'ou=Empleados,dc=subsidiaria,dc=lab' '(uid=lrodriguez)' uid 2>&1" \
  "uid: lrodriguez"

echo ""
echo "── Verificando resolución de conflicto ──"
check "Valor autoritativo en LDAP-B post-conflicto" \
  "ldapsearch -H ldap://localhost:2389 -D 'cn=admin,dc=subsidiaria,dc=lab' -w 'AdminSubB2024!' -b 'ou=Empleados,dc=subsidiaria,dc=lab' '(uid=amartinez)' cn 2>&1" \
  "ACTUALIZADO-CORP"

echo ""
echo "── Verificando ppolicy cargado ──"
check "ppolicy overlay activo" \
  "docker exec ldap-a ldapsearch -Y EXTERNAL -H ldapi:/// -b 'cn=config' '(objectClass=olcPPolicyConfig)' olcOverlay 2>&1" \
  "olcOverlay: ppolicy"

echo ""
echo "═══════════════════════════════════════"
echo "  RESULTADO: $PASS pasaron | $FAIL fallaron"
echo "═══════════════════════════════════════"
```

```bash
chmod +x validate-lab.sh && bash validate-lab.sh
```

**Resultado esperado:** `RESULTADO: 10 pasaron | 0 fallaron`

---

## 8. Resolución de Problemas

### Problema 1: La reconciliación de MidPoint falla con error "Cannot connect to resource"

**Síntomas:**
- En la interfaz de MidPoint, la tarea de reconciliación muestra estado `FATAL_ERROR`.
- El log muestra: `com.evolveum.midpoint.util.exception.CommunicationException: Cannot connect to resource LDAP-A Corporativo`.
- El recurso aparece con estado `DOWN` en la lista de recursos.

**Causa:**
MidPoint intenta conectarse a `ldap-a` usando el nombre de host del servicio Docker, pero si el contenedor MidPoint no está en la misma red Docker (`iam-net`) que los contenedores LDAP, la resolución DNS interna falla. Esto puede ocurrir si los contenedores se iniciaron en momentos diferentes o si el archivo `docker-compose.yml` fue modificado.

**Solución:**

```bash
# 1. Verificar que todos los contenedores están en la misma red
docker network inspect iam-net --format '{{range .Containers}}{{.Name}} {{end}}'
# Debe mostrar: mp-postgres ldap-a ldap-b midpoint

# 2. Si algún contenedor no aparece, reconectarlo
docker network connect iam-net midpoint
docker network connect iam-net ldap-a
docker network connect iam-net ldap-b

# 3. Verificar resolución DNS desde MidPoint hacia ldap-a
docker exec midpoint ping -c 2 ldap-a

# 4. Si el ping falla, reiniciar el stack completo
docker compose down && docker compose up -d
sleep 90

# 5. Probar conexión LDAP desde dentro del contenedor MidPoint
docker exec midpoint bash -c \
  "apt-get install -y ldap-utils -qq && \
   ldapsearch -H ldap://ldap-a:389 \
   -D 'cn=midpoint-svc,ou=Personas,dc=corporativo,dc=lab' \
   -w 'MidpointSvc2024!' \
   -b 'dc=corporativo,dc=lab' '(objectClass=*)' dn 2>&1 | head -5"
```

---

### Problema 2: El overlay ppolicy no aplica la política y los usuarios pueden usar contraseñas cortas

**Síntomas:**
- Al intentar cambiar la contraseña de un usuario a un valor de 4 caracteres, la operación tiene éxito en lugar de ser rechazada.
- El comando `ldapsearch` sobre `cn=config` no muestra la entrada `olcOverlay=ppolicy`.
- Los logs de `slapd` muestran: `overlay "ppolicy" not found`.

**Causa:**
El módulo `ppolicy` no fue cargado correctamente porque el archivo `.so` no está disponible en la imagen Docker `osixia/openldap:1.5.0`, o el módulo fue cargado pero la ruta de la política por defecto en `olcPPolicyDefault` no corresponde a una entrada existente en el DIT (la OU `ou=Politicas` no se creó antes del overlay).

**Solución:**

```bash
# 1. Verificar que el módulo ppolicy está disponible en el contenedor
docker exec ldap-a find /usr/lib -name "ppolicy.so" 2>/dev/null

# 2. Verificar que la OU de políticas existe
docker exec ldap-a ldapsearch \
  -H ldap://localhost \
  -D "cn=admin,dc=corporativo,dc=lab" \
  -w "AdminCorpA2024!" \
  -b "dc=corporativo,dc=lab" \
  "(ou=Politicas)" dn

# 3. Si la OU no existe, crearla primero
docker exec ldap-a bash -c "cat > /tmp/fix-policy-ou.ldif << 'LDIF'
dn: ou=Politicas,dc=corporativo,dc=lab
objectClass: organizationalUnit
ou: Politicas
LDIF
ldapadd -H ldap://localhost \
  -D 'cn=admin,dc=corporativo,dc=lab' \
  -w 'AdminCorpA2024!' \
  -f /tmp/fix-policy-ou.ldif"

# 4. Verificar que el overlay está registrado en cn=config
docker exec ldap-a ldapsearch \
  -Y EXTERNAL -H ldapi:/// \
  -b "olcDatabase={1}mdb,cn=config" \
  "(objectClass=olcPPolicyConfig)" olcOverlay olcPPolicyDefault

# 5. Probar la política manualmente
docker exec ldap-a ldappasswd \
  -H ldap://localhost \
  -D "cn=admin,dc=corporativo,dc=lab" \
  -w "AdminCorpA2024!" \
  -s "abc" \
  "uid=amartinez,ou=Personas,dc=corporativo,dc=lab"
# Debe retornar: Result: Constraint violation (19) - Password fails quality checking policy
```

---

## 9. Limpieza del Entorno

> ⚠️ **Importante:** Ejecutar la limpieza completa antes de iniciar el siguiente laboratorio para liberar recursos de memoria y almacenamiento.

```bash
# Desde el directorio ~/lab-03-00-01

# 1. Detener y eliminar todos los contenedores y volúmenes del lab
docker compose down -v

# 2. Eliminar imágenes descargadas (opcional, solo si se necesita espacio)
docker rmi osixia/openldap:1.5.0 evolveum/midpoint:4.8 postgres:15 2>/dev/null || true

# 3. Limpiar archivos temporales generados durante el lab
rm -rf /tmp/conflict-ldap-*.ldif \
       /tmp/load-ppolicy.ldif \
       /tmp/ppolicy-overlay.ldif \
       /tmp/create-policy.ldif \
       /tmp/acl-hardening.ldif \
       /tmp/midpoint-svc.ldif \
       /tmp/response-ldap-*.json

# 4. Verificar que no quedan contenedores activos del lab
docker ps --filter "name=ldap-a" --filter "name=ldap-b" \
          --filter "name=midpoint" --filter "name=mp-postgres"

# 5. (Opcional) Conservar archivos de configuración para referencia futura
tar -czf ~/backup-lab-03-00-01-$(date +%Y%m%d).tar.gz ~/lab-03-00-01/
echo "Backup guardado en ~/backup-lab-03-00-01-$(date +%Y%m%d).tar.gz"
```

**Resultado esperado de limpieza:**
```
[+] Running 5/5
 ✔ Container midpoint     Removed
 ✔ Container ldap-b       Removed
 ✔ Container ldap-a       Removed
 ✔ Container mp-postgres  Removed
 ✔ Network iam-net        Removed
```

---

## 10. Resumen

### Conceptos Aplicados en este Laboratorio

En este laboratorio aplicaste de forma práctica los conceptos centrales del Módulo 3 sobre sincronización y reconciliación de identidades:

| Concepto | Implementación Realizada |
|----------|--------------------------|
| **Arquitectura DIT diferenciada** | LDAP-A usó `ou=Personas` con `displayName`; LDAP-B usó `ou=Empleados` con `cn` en formato distinto |
| **Correlación de identidades** | Regla de correlación por atributo `mail` en MidPoint para identificar usuarios equivalentes entre directorios |
| **Transformación de atributos** | Mapping Groovy que convierte `"APELLIDO, Nombre"` ↔ `"Nombre Apellido"` entre esquemas |
| **Resolución de conflictos** | LDAP-A establecido como fuente autoritativa; su valor sobrescribe datos conflictivos en LDAP-B |
| **Hardening LDAP** | ACLs restrictivas, deshabilitación de acceso anónimo, cuenta de servicio con privilegios mínimos |
| **Política de contraseñas** | Overlay `ppolicy` con longitud mínima 10, historial de 5 contraseñas, bloqueo tras 5 intentos fallidos |
| **Aprovisionamiento bidireccional** | `lrodriguez` (solo en LDAP-A) fue aprovisionado en LDAP-B; `ptorres` (solo en LDAP-B) fue importado a MidPoint |

### Puntos Clave

- Los **esquemas LDAP heterogéneos** son la norma en entornos empresariales reales; MidPoint resuelve estas diferencias mediante mappings de transformación expresados en Groovy.
- La **correlación por atributo estable** (como `mail`) es fundamental: usar atributos que cambien con frecuencia (como `cn`) genera correlaciones incorrectas.
- El **overlay ppolicy** de OpenLDAP es la herramienta estándar para aplicar políticas de contraseñas directamente en el directorio, independientemente de la aplicación que realice el cambio.
- Las **ACLs restrictivas** y la deshabilitación del bind anónimo son medidas de hardening de bajo costo y alto impacto, alineadas con las buenas prácticas de la lección 3.1.

### Recursos Adicionales

- [MidPoint 4.8 Documentation — Synchronization](https://docs.evolveum.com/midpoint/reference/synchronization/)
- [MidPoint Correlation — Official Guide](https://docs.evolveum.com/midpoint/reference/synchronization/correlation/)
- [OpenLDAP Password Policy Overlay (ppolicy)](https://www.openldap.org/software/man.cgi?query=slapo-ppolicy)
- [RFC 3112 — LDAP Authentication Password Schema](https://www.rfc-editor.org/rfc/rfc3112)
- [OpenLDAP Admin Guide — Access Control](https://www.openldap.org/doc/admin24/access-control.html)
- [MidPoint Groovy Scripting in Mappings](https://docs.evolveum.com/midpoint/reference/expressions/expressions/script/groovy/)

---
*Lab 03-00-01 — Módulo 3: Sincronización y Reconciliación de Identidades | Versión 1.0*
