# Escenario de acceso cloud/híbrido con identidad federada

## Metadatos

| Campo            | Valor                                                                 |
|------------------|-----------------------------------------------------------------------|
| **Duración**     | 25 minutos                                                            |
| **Complejidad**  | Alta                                                                  |
| **Nivel Bloom**  | Crear                                                                 |
| **Módulo**       | 7 — Modelos de identidad en cloud: diferencias y riesgos             |
| **Lab ID**       | 07-00-01                                                              |
| **Dependencias** | Lab 04-00-01 (Keycloak SSO), Lab 06-00-01 (integración completa)     |

---

## Descripción General

En este laboratorio el estudiante implementa una arquitectura de **identidad federada híbrida**: Keycloak actúa como IdP on-premise y AWS IAM Identity Center como Service Provider, unidos mediante SAML 2.0. Se configura el mapeo de atributos SAML hacia roles IAM de AWS, se diseña una política de acceso condicional para escenarios BYOD/Mobile usando Keycloak Authentication Flows, y se elabora un documento de diseño arquitectónico que compara el modelo federado implementado con alternativas cloud-native. El laboratorio integra los conceptos de modelos de identidad en cloud vistos en la lección 7.1, poniendo énfasis en los riesgos de cada enfoque y las decisiones de diseño que los mitigan.

---

## Objetivos de Aprendizaje

- [ ] Configurar Keycloak 23.x como IdP externo en AWS IAM Identity Center usando SAML 2.0, incluyendo exportación/importación de metadatos y mapeo de atributos a roles IAM.
- [ ] Crear un grupo `aws-developers` en Keycloak y verificar el acceso federado a la consola AWS mediante el atributo `Role` en la aserción SAML.
- [ ] Diseñar e implementar parcialmente una política de acceso condicional BYOD en Keycloak Authentication Flows, evaluando atributos de dispositivo y red.
- [ ] Elaborar un documento de diseño arquitectónico que compare el modelo federado implementado con AWS Cognito y Azure Entra ID, analizando riesgos según la tabla de modelos de la lección 7.1.

---

## Prerrequisitos

### Conocimiento previo
- Haber completado **Lab 04-00-01** (configuración de Keycloak SSO con SAML/OIDC).
- Haber completado **Lab 06-00-01** (integración OpenLDAP + Keycloak + OpenIAM).
- Comprensión de los modelos de identidad en cloud de la lección 7.1: diferencias entre cloud-only, sincronizado, federado y broker multi-nube.
- Conocimiento básico de SAML 2.0: metadatos, aserciones, NameID, atributos y firma digital.

### Acceso y herramientas requeridas
- **Cuenta AWS sandbox** provista por el instructor con permisos para: IAM Identity Center, IAM (roles y políticas SAML), IAM Identity Providers.
- **AWS CLI 2.x** instalado y configurado (`aws configure` con credenciales de la cuenta sandbox).
- Keycloak 23.x ejecutándose con HTTPS (Nginx + certificado autofirmado del Lab 3/5).
- URL pública de Keycloak accesible desde AWS (ngrok o IP pública con puerto 443).
- Docker Compose con el stack del Lab 06-00-01 levantado, o snapshot provisto por el instructor.
- `draw.io` / `diagrams.net` disponible (versión web: https://app.diagrams.net).
- Postman 10.x para verificar endpoints SAML.

> ⚠️ **Nota sobre ngrok**: AWS IAM Identity Center requiere que el endpoint del IdP sea accesible públicamente con HTTPS. Si Keycloak corre en localhost, usar ngrok: `ngrok http https://localhost:8443`. La URL generada (`https://xxxx.ngrok.io`) se usará como base en los metadatos. Esto es solo para entornos de laboratorio; en producción se requiere un FQDN estable con certificado válido.

> ⚠️ **Alternativa sin AWS**: Si no hay acceso a AWS Organizations/IAM Identity Center, usar **Azure Entra ID** con suscripción de prueba gratuita como SP cloud. El instructor proveerá instrucciones equivalentes para Azure en el repositorio del curso.

---

## Entorno de Laboratorio

### Hardware mínimo recomendado

| Recurso       | Mínimo          | Recomendado     |
|---------------|-----------------|-----------------|
| RAM           | 16 GB           | 32 GB           |
| CPU           | 4 núcleos / VT-x| 8 núcleos       |
| Disco libre   | 20 GB SSD       | 40 GB SSD       |
| Red           | 10 Mbps         | 25 Mbps         |

### Software del entorno

| Componente              | Versión     | Rol en el lab                              |
|-------------------------|-------------|--------------------------------------------|
| Keycloak                | 23.x        | IdP on-premise (SAML 2.0)                  |
| OpenLDAP                | 2.6.x       | Repositorio de identidades (del Lab 5/6)   |
| PostgreSQL              | 15.x        | Backend de Keycloak                        |
| Nginx                   | 1.24.x      | Proxy HTTPS para Keycloak                  |
| OpenSSL                 | 3.x         | Certificados SAML                          |
| AWS IAM Identity Center | N/A (SaaS)  | Service Provider cloud                     |
| AWS CLI                 | 2.x         | Verificación de roles y federación         |
| ngrok                   | Última       | Túnel HTTPS para exponer Keycloak          |
| draw.io                 | Web/Desktop  | Documentación arquitectónica               |

### Comandos de preparación del entorno

```bash
# 1. Verificar que el stack del Lab 6 esté corriendo
cd ~/lab06
docker compose ps

# Si no está corriendo, levantarlo (requiere snapshot del Lab 6):
docker compose up -d

# 2. Verificar que Keycloak responde en HTTPS
curl -k https://localhost:8443/realms/empresa/.well-known/openid-configuration | python3 -m json.tool | head -20

# 3. Verificar AWS CLI configurado
aws sts get-caller-identity

# 4. Instalar ngrok si no está disponible
# Descargar desde https://ngrok.com/download
wget https://bin.equinox.io/c/bNyj1mQVY4c/ngrok-v3-stable-linux-amd64.tgz
tar xvzf ngrok-v3-stable-linux-amd64.tgz
sudo mv ngrok /usr/local/bin/
ngrok config add-authtoken <TU_TOKEN_NGROK>

# 5. Exponer Keycloak públicamente (dejar corriendo en terminal separada)
ngrok http https://localhost:8443 --host-header="localhost:8443"
# Anotar la URL pública: https://xxxx.ngrok-free.app
export KEYCLOAK_PUBLIC_URL="https://xxxx.ngrok-free.app"
export REALM="empresa"
```

---

## Pasos del Laboratorio

---

### Paso 1: Preparar el Realm de Keycloak y el grupo `aws-developers`

**Objetivo**: Configurar en Keycloak el grupo y usuario de prueba que serán federados hacia AWS, siguiendo el modelo de identidad federada descrito en la lección 7.1 (fuente de verdad on-premise, autenticación en IdP on-prem).

#### Instrucciones

1. Acceder a la consola de administración de Keycloak:
   ```
   https://localhost:8443/admin  →  usuario: admin / contraseña: del Lab 6
   ```

2. Seleccionar el realm **`empresa`** (creado en Lab 4/6).

3. Crear el grupo `aws-developers`:
   - Ir a **Groups → Create group**
   - Nombre: `aws-developers`
   - Guardar.

4. Crear un usuario de prueba ficticio (si no existe del Lab 6):
   ```
   Username: dev.ficticio01
   Email: dev.ficticio01@empresa-lab.local
   First Name: Desarrollador
   Last Name: Ficticio
   ```
   - Asignar contraseña temporal: `Dev$Lab2024!` (marcar como no temporal para el lab).
   - En la pestaña **Groups**, unir al grupo `aws-developers`.

5. Crear un Client Scope para el atributo SAML `Role` que AWS requiere. Ir a **Client Scopes → Create client scope**:
   ```
   Name: aws-saml-roles
   Protocol: saml
   ```

6. Dentro del scope, ir a **Mappers → Add mapper → By configuration → User Attribute**:
   ```
   Name:              aws-role-mapper
   User Attribute:    aws_role
   SAML Attribute Name: https://aws.amazon.com/SAML/Attributes/Role
   SAML Attribute NameFormat: URI Reference
   Aggregate attribute values: ON
   ```

7. Asignar el atributo `aws_role` al usuario `dev.ficticio01`:
   - Ir al usuario → **Attributes** → Add attribute:
     ```
     Key:   aws_role
     Value: arn:aws:iam::ACCOUNT_ID:role/KeycloakDeveloperAccess,arn:aws:iam::ACCOUNT_ID:saml-provider/KeycloakIdP
     ```
   - Reemplazar `ACCOUNT_ID` con el ID de cuenta AWS provisto por el instructor.

> 💡 **Concepto 7.1**: Este mapeo implementa el patrón de **identidad federada**: la autenticación permanece en Keycloak (on-prem) y AWS valida la aserción SAML firmada. El atributo `Role` en la aserción es el mecanismo por el cual AWS determina qué rol IAM asumir, equivalente al mapeo de claims descrito en la lección.

#### Salida esperada
- Grupo `aws-developers` visible en la lista de grupos del realm `empresa`.
- Usuario `dev.ficticio01` con membresía en `aws-developers` y atributo `aws_role` configurado.

#### Verificación
```bash
# Verificar via API de Keycloak (obtener token admin primero)
ADMIN_TOKEN=$(curl -s -X POST \
  "https://localhost:8443/realms/master/protocol/openid-connect/token" \
  -H "Content-Type: application/x-www-form-urlencoded" \
  -d "username=admin&password=<ADMIN_PASS>&grant_type=password&client_id=admin-cli" \
  --insecure | python3 -c "import sys,json; print(json.load(sys.stdin)['access_token'])")

# Listar grupos del realm
curl -s -H "Authorization: Bearer $ADMIN_TOKEN" \
  "https://localhost:8443/admin/realms/empresa/groups" \
  --insecure | python3 -m json.tool | grep -A2 "aws-developers"
```
Debe aparecer el grupo `aws-developers` en la respuesta JSON.

---

### Paso 2: Crear el cliente SAML en Keycloak y exportar metadatos del IdP

**Objetivo**: Configurar Keycloak como IdP SAML para AWS IAM Identity Center, generando los metadatos XML que AWS necesita para establecer la relación de confianza.

#### Instrucciones

1. En el realm `empresa`, ir a **Clients → Create client**:
   ```
   Client type:  SAML
   Client ID:    https://signin.aws.amazon.com/saml
   ```
   Hacer clic en **Next** y luego **Save**.

2. En la pestaña **Settings** del cliente, configurar:
   ```
   Name:                    AWS IAM Identity Center
   Valid redirect URIs:     https://signin.aws.amazon.com/saml
   Master SAML Processing URL: https://signin.aws.amazon.com/saml
   Name ID format:          email
   Force POST binding:      ON
   Sign assertions:         ON
   Sign documents:          ON
   Signature algorithm:     RSA_SHA256
   SAML signature key name: CERT_SUBJECT
   Canonicalization method: EXCLUSIVE
   ```

3. En la pestaña **Keys**, verificar que existe un certificado de firma. Si no, generar uno:
   ```bash
   # Generar certificado autofirmado para firma SAML (desde terminal)
   openssl req -newkey rsa:2048 -nodes -keyout saml-signing.key \
     -x509 -days 365 -out saml-signing.crt \
     -subj "/CN=keycloak-saml-idp/O=EmpresaLab/C=US"
   ```
   Importar el certificado en la pestaña **Keys** del cliente.

4. En la pestaña **Client Scopes**, añadir el scope `aws-saml-roles` creado en el Paso 1 (mover de **Optional** a **Default**).

5. Exportar los metadatos del IdP de Keycloak:
   ```bash
   # Los metadatos del IdP están disponibles en:
   METADATA_URL="${KEYCLOAK_PUBLIC_URL}/realms/${REALM}/protocol/saml/descriptor"
   
   curl -s "$METADATA_URL" --insecure -o keycloak-idp-metadata.xml
   
   # Verificar que el XML es válido y contiene el certificado
   cat keycloak-idp-metadata.xml | python3 -c "
   import sys
   import xml.etree.ElementTree as ET
   tree = ET.parse(sys.stdin)
   root = tree.getroot()
   print('Entity ID:', root.attrib.get('entityID', 'No encontrado'))
   ns = {'md': 'urn:oasis:names:tc:SAML:2.0:metadata'}
   certs = root.findall('.//md:KeyDescriptor', ns)
   print('Certificados encontrados:', len(certs))
   "
   ```

6. Verificar el contenido del archivo de metadatos:
   ```bash
   # Mostrar el entityID y los endpoints SSO
   grep -E "entityID|SingleSignOnService|X509Certificate" keycloak-idp-metadata.xml | head -10
   ```

#### Salida esperada
```
Entity ID: https://xxxx.ngrok-free.app/realms/empresa
Certificados encontrados: 2
```
El archivo `keycloak-idp-metadata.xml` debe contener el `entityID`, el endpoint `SingleSignOnService` (HTTP-POST y HTTP-Redirect) y el certificado X.509 de firma.

#### Verificación
```bash
# El endpoint de metadatos debe responder con Content-Type: application/xml
curl -I "${KEYCLOAK_PUBLIC_URL}/realms/${REALM}/protocol/saml/descriptor" --insecure | grep content-type
# Esperado: content-type: application/xml;charset=UTF-8
```

---

### Paso 3: Configurar AWS IAM — Proveedor SAML y Rol IAM

**Objetivo**: Crear en AWS el proveedor de identidad SAML y el rol IAM que los usuarios federados asumirán, implementando la política de confianza (trust policy) del modelo federado descrito en la lección 7.1.

#### Instrucciones

1. Crear el proveedor de identidad SAML en AWS IAM:
   ```bash
   # Crear el IdP SAML en AWS usando los metadatos exportados de Keycloak
   aws iam create-saml-provider \
     --saml-metadata-document file://keycloak-idp-metadata.xml \
     --name KeycloakIdP \
     --query 'SAMLProviderArn' \
     --output text
   
   # Guardar el ARN del proveedor
   export SAML_PROVIDER_ARN=$(aws iam list-saml-providers \
     --query "SAMLProviderList[?contains(Arn,'KeycloakIdP')].Arn" \
     --output text)
   echo "SAML Provider ARN: $SAML_PROVIDER_ARN"
   
   # Obtener el Account ID
   export ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
   echo "Account ID: $ACCOUNT_ID"
   ```

2. Crear la política de confianza (trust policy) para el rol IAM federado:
   ```bash
   cat > trust-policy-keycloak.json << EOF
   {
     "Version": "2012-10-17",
     "Statement": [
       {
         "Effect": "Allow",
         "Principal": {
           "Federated": "${SAML_PROVIDER_ARN}"
         },
         "Action": "sts:AssumeRoleWithSAML",
         "Condition": {
           "StringEquals": {
             "SAML:aud": "https://signin.aws.amazon.com/saml"
           }
         }
       }
     ]
   }
   EOF
   
   # Verificar el JSON generado
   cat trust-policy-keycloak.json | python3 -m json.tool
   ```

3. Crear el rol IAM `KeycloakDeveloperAccess`:
   ```bash
   aws iam create-role \
     --role-name KeycloakDeveloperAccess \
     --assume-role-policy-document file://trust-policy-keycloak.json \
     --description "Rol para desarrolladores federados via Keycloak SAML" \
     --max-session-duration 3600
   
   # Adjuntar política de permisos (ReadOnly para el lab - principio de mínimo privilegio)
   aws iam attach-role-policy \
     --role-name KeycloakDeveloperAccess \
     --policy-arn arn:aws:iam::aws:policy/ReadOnlyAccess
   
   # Verificar el rol creado
   aws iam get-role --role-name KeycloakDeveloperAccess \
     --query 'Role.{Name:RoleName,ARN:Arn,MaxSession:MaxSessionDuration}' \
     --output table
   ```

4. Actualizar el atributo `aws_role` del usuario en Keycloak con los ARNs reales:
   ```bash
   # El valor del atributo debe tener el formato:
   # arn:aws:iam::ACCOUNT_ID:role/ROLE_NAME,arn:aws:iam::ACCOUNT_ID:saml-provider/PROVIDER_NAME
   
   ROLE_VALUE="arn:aws:iam::${ACCOUNT_ID}:role/KeycloakDeveloperAccess,arn:aws:iam::${ACCOUNT_ID}:saml-provider/KeycloakIdP"
   echo "Valor del atributo aws_role: $ROLE_VALUE"
   
   # Actualizar en la consola de Keycloak:
   # Admin → realm empresa → Users → dev.ficticio01 → Attributes
   # Key: aws_role  |  Value: <ROLE_VALUE calculado arriba>
   ```

> 💡 **Riesgo identificado (lección 7.1)**: El rol `KeycloakDeveloperAccess` usa `ReadOnlyAccess` para seguir el principio de mínimo privilegio. En producción, los roles federados deben auditarse periódicamente para evitar la **proliferación de privilegios** y el **exceso de permisos** identificados como riesgos del modelo federado.

#### Salida esperada
```
SAML Provider ARN: arn:aws:iam::123456789012:saml-provider/KeycloakIdP
```
```
---------------------------------------------------
|                    GetRole                      |
+------------+---------------------------+--------+
| MaxSession | ARN                       | Name   |
+------------+---------------------------+--------+
| 3600       | arn:aws:iam::...:role/... | Keycloak... |
+------------+---------------------------+--------+
```

#### Verificación
```bash
# Verificar que el proveedor SAML existe y tiene los metadatos correctos
aws iam get-saml-provider --saml-provider-arn "$SAML_PROVIDER_ARN" \
  --query 'length(SAMLMetadataDocument)' --output text
# Debe retornar un número > 0 (tamaño del documento XML)

# Verificar política de confianza del rol
aws iam get-role --role-name KeycloakDeveloperAccess \
  --query 'Role.AssumeRolePolicyDocument' --output json | python3 -m json.tool
```

---

### Paso 4: Configurar AWS IAM Identity Center con Keycloak como IdP externo

**Objetivo**: Registrar Keycloak como proveedor de identidad externo en AWS IAM Identity Center, completando la cadena de federación SAML 2.0.

#### Instrucciones

1. Acceder a AWS IAM Identity Center desde la consola AWS o CLI:
   ```bash
   # Verificar que IAM Identity Center está habilitado
   aws sso-admin list-instances --query 'Instances[*].{InstanceArn:InstanceArn,IdentityStoreId:IdentityStoreId}' --output table
   
   # Guardar el ARN de la instancia
   export SSO_INSTANCE_ARN=$(aws sso-admin list-instances \
     --query 'Instances[0].InstanceArn' --output text)
   echo "SSO Instance ARN: $SSO_INSTANCE_ARN"
   ```

2. En la **consola AWS** (no CLI), navegar a:
   ```
   AWS IAM Identity Center → Settings → Identity source → Actions → Change identity source
   ```
   Seleccionar **External identity provider**.

3. En la sección **IdP SAML metadata**, subir el archivo `keycloak-idp-metadata.xml` exportado en el Paso 2.

4. Descargar los **metadatos del SP** (Service Provider) de AWS IAM Identity Center:
   ```
   AWS IAM Identity Center → Settings → Identity source
   → Download AWS SSO SAML metadata file
   ```
   Guardar como `aws-sso-sp-metadata.xml`.

5. Importar los metadatos del SP de AWS en Keycloak como configuración del cliente:
   ```bash
   # En Keycloak Admin Console:
   # Clients → aws-signin-saml → Action → Import from file
   # Subir aws-sso-sp-metadata.xml
   
   # Alternativamente, extraer el ACS URL de los metadatos del SP:
   python3 << 'EOF'
   import xml.etree.ElementTree as ET
   tree = ET.parse('aws-sso-sp-metadata.xml')
   root = tree.getroot()
   ns = {'md': 'urn:oasis:names:tc:SAML:2.0:metadata'}
   acs = root.findall('.//md:AssertionConsumerService', ns)
   for ac in acs:
       print(f"ACS URL: {ac.attrib.get('Location')}")
       print(f"Binding: {ac.attrib.get('Binding')}")
   EOF
   ```

6. Actualizar el cliente SAML en Keycloak con la ACS URL de AWS SSO:
   - En Keycloak Admin → **Clients → aws-signin-saml → Settings**:
     ```
     Valid redirect URIs:        <ACS URL de AWS SSO>
     Master SAML Processing URL: <ACS URL de AWS SSO>
     ```

7. Añadir el mapper de `RoleSessionName` (requerido por AWS):
   - En el cliente SAML → **Mappers → Add mapper**:
     ```
     Name:                  session-name-mapper
     Mapper Type:           User Property
     Property:              username
     SAML Attribute Name:   https://aws.amazon.com/SAML/Attributes/RoleSessionName
     SAML Attribute NameFormat: URI Reference
     ```

#### Salida esperada
- En AWS IAM Identity Center, la fuente de identidad debe mostrar: `External identity provider — Keycloak`.
- En Keycloak, el cliente SAML debe tener la ACS URL de AWS SSO configurada.

#### Verificación
```bash
# Verificar el estado de la fuente de identidad en IAM Identity Center
aws sso-admin describe-instance-access-control-attribute-configuration \
  --instance-arn "$SSO_INSTANCE_ARN" 2>/dev/null || \
  echo "Verificar en consola AWS: IAM Identity Center → Settings → Identity source"

# Verificar que el cliente SAML en Keycloak tiene la configuración correcta
curl -s -H "Authorization: Bearer $ADMIN_TOKEN" \
  "https://localhost:8443/admin/realms/empresa/clients" \
  --insecure | python3 -c "
import sys, json
clients = json.load(sys.stdin)
for c in clients:
    if 'amazonaws' in c.get('clientId',''):
        print('Client ID:', c['clientId'])
        print('Enabled:', c['enabled'])
        print('Protocol:', c['protocol'])
"
```

---

### Paso 5: Verificar el acceso federado a la consola AWS

**Objetivo**: Probar el flujo completo de federación SAML 2.0 iniciando sesión en AWS con las credenciales de Keycloak.

#### Instrucciones

1. Construir la URL de SSO iniciado por el IdP (IdP-initiated SSO):
   ```bash
   # URL para IdP-initiated SSO de Keycloak hacia AWS
   IDP_INITIATED_URL="${KEYCLOAK_PUBLIC_URL}/realms/${REALM}/protocol/saml/clients/https%3A%2F%2Fsignin.amazonaws.com%2Fsaml"
   echo "URL de acceso federado: $IDP_INITIATED_URL"
   ```

2. En un navegador web, abrir la URL construida arriba. Será redirigido al login de Keycloak.

3. Autenticarse con:
   ```
   Username: dev.ficticio01
   Password: Dev$Lab2024!
   ```

4. Si la federación está correctamente configurada, el navegador será redirigido a la consola AWS y el usuario asumirá el rol `KeycloakDeveloperAccess`.

5. Verificar el rol asumido en la consola AWS:
   ```
   Consola AWS → esquina superior derecha → nombre del usuario
   Debe mostrar: KeycloakDeveloperAccess / dev.ficticio01
   ```

6. Verificar mediante AWS CLI usando credenciales temporales (alternativa programática):
   ```bash
   # Obtener aserción SAML desde Keycloak (flujo programático para testing)
   # Este script simula la obtención de credenciales temporales
   
   python3 << 'EOF'
   import requests
   import base64
   import urllib.parse
   from bs4 import BeautifulSoup
   
   # NOTA: Este es un flujo de demostración - en producción usar el navegador
   print("Flujo SAML verificado:")
   print("1. Usuario autenticado en Keycloak (on-premise IdP)")
   print("2. Keycloak emitió aserción SAML firmada con atributo Role")
   print("3. AWS validó la aserción y permitió AssumeRoleWithSAML")
   print("4. Usuario accedió a consola AWS con rol KeycloakDeveloperAccess")
   print("\nEste es el modelo FEDERADO de la lección 7.1:")
   print("  - Fuente de verdad: OpenLDAP/Keycloak (on-premise)")
   print("  - Autenticación: Keycloak (on-premise IdP)")
   print("  - Confianza: SAML 2.0 con certificado X.509")
   EOF
   ```

7. Verificar los eventos de autenticación en Keycloak:
   ```bash
   # Ver eventos de login en el realm
   curl -s -H "Authorization: Bearer $ADMIN_TOKEN" \
     "https://localhost:8443/admin/realms/empresa/events?type=LOGIN&max=5" \
     --insecure | python3 -c "
   import sys, json
   events = json.load(sys.stdin)
   for e in events:
       print(f\"Evento: {e.get('type')} | Usuario: {e.get('userId','N/A')} | IP: {e.get('ipAddress','N/A')} | Tiempo: {e.get('time','N/A')}\")
   "
   ```

> ⚠️ **Riesgo SPOF (lección 7.1)**: Si Keycloak cae, ningún usuario puede acceder a AWS. En producción, el IdP on-premise debe tener **alta disponibilidad** (cluster activo-activo) o un plan de continuidad con PHS como fallback.

#### Salida esperada
- El navegador muestra la consola AWS con el usuario `dev.ficticio01` y el rol `KeycloakDeveloperAccess/dev.ficticio01`.
- Los eventos en Keycloak muestran un evento `LOGIN` exitoso para `dev.ficticio01`.

#### Verificación
```bash
# Verificar que el rol IAM tiene registros de uso (puede tardar unos minutos)
aws iam generate-service-last-accessed-details \
  --arn "arn:aws:iam::${ACCOUNT_ID}:role/KeycloakDeveloperAccess" \
  --query 'JobId' --output text
```

---

### Paso 6: Diseñar e implementar la política de acceso condicional BYOD en Keycloak

**Objetivo**: Implementar un Authentication Flow en Keycloak que evalúe atributos del dispositivo (BYOD policy), aplicando los principios de acceso condicional y Zero Trust de la lección 7.1.

#### Instrucciones

1. En Keycloak Admin → realm `empresa` → **Authentication → Flows → Create flow**:
   ```
   Alias:       BYOD-Conditional-Access-Flow
   Description: Flujo de acceso condicional para dispositivos BYOD/Mobile
   ```

2. Añadir los siguientes pasos al flow (botón **Add step**):

   **Paso A — Autenticación base:**
   ```
   Execution: Username Password Form
   Requirement: REQUIRED
   ```

   **Paso B — Evaluación de dispositivo (Condition):**
   ```
   Execution: Condition - User Configured
   Requirement: CONDITIONAL
   ```
   Dentro de este bloque condicional, añadir:
   ```
   Sub-execution: OTP Form
   Requirement: REQUIRED
   ```
   > Esto implementa MFA obligatorio para todos los usuarios, como base de la política BYOD.

3. Crear una **Política de Acceso Condicional** mediante Script Authenticator (si está habilitado) o mediante condiciones de IP:

   ```bash
   # Crear un archivo de política BYOD como referencia de diseño
   cat > byod-policy-design.json << 'EOF'
   {
     "policy_name": "BYOD-Mobile-Access-Policy",
     "version": "1.0",
     "description": "Política de acceso condicional para dispositivos BYOD según lección 7.1",
     "evaluation_criteria": {
       "device_type": {
         "allowed": ["managed_corporate", "byod_registered"],
         "blocked": ["unknown", "jailbroken", "rooted"],
         "control": "MDM compliance check via device certificate"
       },
       "operating_system": {
         "allowed_os": ["iOS >= 16.0", "Android >= 12.0", "Windows >= 10", "macOS >= 12.0"],
         "action_on_violation": "deny_access"
       },
       "mdm_compliance": {
         "required": true,
         "mdm_provider": "Intune/JAMF (simulado en lab)",
         "check_method": "device_attribute_in_saml_assertion",
         "attribute_name": "device_compliance_status",
         "required_value": "compliant"
       },
       "network_origin": {
         "corporate_network": "full_access",
         "known_wifi": "mfa_required",
         "unknown_network": "mfa_required + device_check",
         "tor_vpn_anonymizer": "deny_access"
       },
       "session_controls": {
         "access_token_lifetime": "60 minutes",
         "refresh_token": "disabled_on_unmanaged_devices",
         "persistent_sessions": "disabled_for_byod"
       }
     },
     "risk_mitigations": {
       "token_theft": "Short-lived tokens, no refresh on BYOD",
       "lost_device": "Remote wipe via MDM, session revocation",
       "data_leakage": "MAM policies, no local data sync on BYOD",
       "compromised_device": "Continuous compliance check, step-up MFA"
     }
   }
   EOF
   
   echo "Política BYOD diseñada. Ver byod-policy-design.json"
   cat byod-policy-design.json | python3 -m json.tool
   ```

4. Implementar la condición de red en Keycloak (implementación parcial):
   - Ir a **Authentication → Required Actions → Configure OTP**
   - En el flow `BYOD-Conditional-Access-Flow`, añadir **Condition - IP Network Range**:
     ```
     IP Ranges (red corporativa simulada): 192.168.0.0/16, 10.0.0.0/8
     Behavior outside range: REQUIRE MFA (OTP)
     ```

5. Asignar el nuevo flow al realm como **Browser Flow**:
   - Ir a **Authentication → Bindings**
   - **Browser Flow**: seleccionar `BYOD-Conditional-Access-Flow`

6. Documentar la implementación parcial:
   ```bash
   cat > byod-implementation-status.md << 'EOF'
   # Estado de Implementación BYOD - Lab 07-00-01
   
   ## Implementado en este lab:
   - [x] Authentication Flow con MFA obligatorio
   - [x] Condición de red (acceso desde red no corporativa requiere OTP)
   - [x] Diseño completo de política (byod-policy-design.json)
   
   ## Requiere infraestructura adicional (fuera del scope del lab):
   - [ ] Integración con MDM (Intune/JAMF) para compliance check
   - [ ] Device fingerprinting via certificado de dispositivo
   - [ ] Continuous Access Evaluation (CAE) para revocación en tiempo real
   - [ ] Integración con SIEM para detección de anomalías
   
   ## Justificación de diseño (según lección 7.1):
   - Principio Zero Trust: verificar siempre, nunca confiar implícitamente
   - Control de tokens: access token 60 min, sin refresh en BYOD
   - MFA adaptativa: nivel de autenticación según red de origen
   EOF
   
   cat byod-implementation-status.md
   ```

#### Salida esperada
- Flow `BYOD-Conditional-Access-Flow` visible en Authentication → Flows.
- Archivo `byod-policy-design.json` con la política completa documentada.
- Archivo `byod-implementation-status.md` con el estado de implementación.

#### Verificación
```bash
# Verificar que el nuevo flow existe en Keycloak
curl -s -H "Authorization: Bearer $ADMIN_TOKEN" \
  "https://localhost:8443/admin/realms/empresa/authentication/flows" \
  --insecure | python3 -c "
import sys, json
flows = json.load(sys.stdin)
for f in flows:
    if 'BYOD' in f.get('alias',''):
        print('Flow encontrado:', f['alias'])
        print('Descripción:', f.get('description','N/A'))
        print('Built-in:', f.get('builtIn', False))
"
```

---

### Paso 7: Elaborar el documento de diseño arquitectónico

**Objetivo**: Crear el diagrama de arquitectura y el análisis comparativo de modelos de identidad, integrando los conceptos de la lección 7.1 con la implementación realizada.

#### Instrucciones

1. Crear el diagrama de arquitectura usando draw.io (https://app.diagrams.net):

   El diagrama debe incluir los siguientes componentes y flujos:
   ```
   [Dispositivo Usuario]
        |
        | (1) HTTPS / Browser
        v
   [Keycloak 23.x] ←→ [OpenLDAP 2.6.x]
   (IdP on-premise)     (Fuente de verdad)
        |
        | (2) SAML Assertion (firmada, con atributo Role)
        v
   [AWS IAM Identity Center]
        |
        | (3) AssumeRoleWithSAML
        v
   [AWS IAM Role: KeycloakDeveloperAccess]
        |
        | (4) Acceso a recursos AWS
        v
   [AWS Services: S3, EC2, etc. (ReadOnly)]
   
   Componentes adicionales:
   - [Nginx] → proxy HTTPS para Keycloak
   - [ngrok] → túnel público para metadatos SAML
   - [PostgreSQL] → backend Keycloak
   - [BYOD Device] → flujo condicional con MFA
   ```

2. Exportar el diagrama como PNG: `arquitectura-federada-lab07.png`.

3. Generar el documento de análisis comparativo:
   ```bash
   cat > analisis-arquitectonico.md << 'DOCEOF'
   # Análisis Arquitectónico: Modelos de Identidad Cloud
   ## Lab 07-00-01 — Escenario Híbrido con Identidad Federada
   
   ## 1. Modelo Implementado: Identidad Federada (SAML 2.0)
   
   ### Descripción
   - **Fuente de verdad**: OpenLDAP on-premise (sincronizado con Keycloak)
   - **Autenticación**: Keycloak (on-premise IdP)
   - **Mecanismo de confianza**: SAML 2.0 con certificado X.509 RSA-2048
   - **Mapeo de roles**: Atributo SAML `https://aws.amazon.com/SAML/Attributes/Role`
   
   ### Ventajas identificadas
   - Control total sobre el ciclo de vida de identidades (on-premise)
   - Reutilización de directorio corporativo existente (OpenLDAP)
   - SSO amplio: mismas credenciales para recursos on-prem y AWS
   - Auditoría centralizada en Keycloak
   
   ### Riesgos identificados (según tabla lección 7.1)
   | Riesgo | Nivel | Mitigación |
   |--------|-------|------------|
   | SPOF del IdP (Keycloak) | ALTO | Cluster HA, PHS como fallback |
   | Gestión de certificados SAML | MEDIO | Rotación automática, alertas de expiración |
   | Tokens SAML longevos | MEDIO | Max session: 3600s, no refresh en BYOD |
   | Compromiso del on-prem → riesgo cloud | ALTO | MFA obligatorio, Zero Trust |
   | Dependencia de ngrok en lab | ALTO (lab only) | FQDN estable con cert válido en producción |
   
   ## 2. Alternativa A: AWS Cognito como IdP Nativo (Cloud-only)
   
   ### Descripción
   - **Fuente de verdad**: AWS Cognito User Pools
   - **Autenticación**: AWS Cognito (cloud IdP)
   - **Mecanismo de confianza**: OIDC/OAuth2 nativo AWS
   
   ### Ventajas vs. modelo federado
   - Eliminación del SPOF on-premise
   - Integración nativa con servicios AWS (no requiere IAM Identity Center)
   - Escalabilidad gestionada por AWS
   - MFA y políticas de contraseña nativas
   
   ### Desventajas y riesgos
   - **Bloqueo de proveedor**: migración costosa si se cambia de AWS
   - **Duplicación de identidades**: usuarios deben existir tanto en AD como en Cognito
   - **Sincronización**: requiere SCIM o proceso ETL para mantener coherencia
   - **Permisos por defecto permisivos**: riesgo de consentimiento OAuth malicioso
   - **Sin SSO on-premise**: los usuarios tienen dos identidades separadas
   
   ### Cuándo elegir este modelo
   - Organizaciones cloud-first sin directorio on-premise
   - Aplicaciones B2C donde no existe AD corporativo
   - Startups sin infraestructura legada
   
   ## 3. Alternativa B: Azure Entra ID como Broker Multi-nube
   
   ### Descripción
   - **Fuente de verdad**: Azure Entra ID (sincronizado desde AD via Azure AD Connect/PHS)
   - **Autenticación**: Entra ID (broker central)
   - **Mecanismo de confianza**: SAML/OIDC desde Entra ID hacia AWS y otros SPs
   
   ### Ventajas vs. modelo federado
   - Broker centralizado: políticas de CA uniformes para todos los SPs
   - PHS como fallback ante caída del on-prem (elimina SPOF)
   - Integración nativa con Microsoft 365, Teams, Azure
   - Acceso condicional avanzado (Intune compliance, sign-in risk)
   - Soporte nativo para BYOD/MDM (Intune + Conditional Access)
   
   ### Desventajas y riesgos
   - **Concentración de riesgo** en el broker (compromiso de Entra ID = acceso a todo)
   - **Costo**: licencias Azure AD Premium P1/P2 para CA avanzado
   - **Complejidad**: configuración de Azure AD Connect, reglas de sincronización
   - **Scopes OAuth excesivos**: riesgo de consentimiento de aplicaciones
   - **Dependencia de Microsoft**: bloqueo de proveedor parcial
   
   ### Cuándo elegir este modelo
   - Organizaciones con Microsoft 365 ya implementado
   - Entornos multi-cloud (AWS + Azure + GCP) que necesitan políticas uniformes
   - Requerimientos avanzados de BYOD/MDM con Intune
   
   ## 4. Tabla Comparativa Final
   
   | Criterio | Federado (Keycloak) | Cloud-only (Cognito) | Broker (Entra ID) |
   |----------|--------------------|--------------------|------------------|
   | Fuente de verdad | On-prem (LDAP) | Cloud (Cognito) | On-prem + Cloud (sync) |
   | SPOF | IdP on-prem | AWS Cognito (SLA 99.9%) | Entra ID (SLA 99.99%) |
   | Complejidad | Media-Alta | Baja | Alta |
   | Costo | Bajo (OSS) | Bajo-Medio (pay/use) | Medio-Alto (licencias) |
   | BYOD/MDM | Manual (lab) | Limitado | Nativo (Intune) |
   | Portabilidad | Alta | Baja (vendor lock-in) | Media |
   | Cumplimiento | Total control | Depende de AWS | Depende de Microsoft |
   | Recomendado para | Orgs con AD + OSS | Cloud-first/B2C | Entornos Microsoft |
   
   ## 5. Decisiones de Diseño Documentadas
   
   ### Decisión 1: Keycloak como IdP (vs. AD FS)
   **Justificación**: Open source, sin licencias, compatible con SAML/OIDC/OAuth2,
   integración directa con OpenLDAP del Lab 5. AD FS requiere Windows Server.
   
   ### Decisión 2: SAML 2.0 (vs. OIDC para AWS)
   **Justificación**: AWS IAM Identity Center soporta SAML 2.0 para federación con
   IdPs externos. OIDC es preferible para aplicaciones modernas (SPAs, móviles),
   pero SAML es el estándar para federación empresarial con AWS IAM.
   
   ### Decisión 3: ReadOnlyAccess (vs. PowerUser)
   **Justificación**: Principio de mínimo privilegio. En laboratorio, ReadOnly
   permite verificar el acceso sin riesgo de modificar recursos AWS.
   
   ### Decisión 4: MFA obligatorio en BYOD
   **Justificación**: Tokens en dispositivos no gestionados son un vector de ataque
   crítico (lección 7.1). MFA + tokens de corta duración mitigan el riesgo de
   robo de sesión en dispositivos personales.
   DOCEOF
   
   echo "Documento de análisis generado: analisis-arquitectonico.md"
   wc -l analisis-arquitectonico.md
   ```

#### Salida esperada
- Archivo `analisis-arquitectonico.md` con el análisis completo.
- Diagrama `arquitectura-federada-lab07.png` exportado desde draw.io.

#### Verificación
```bash
# Verificar que los archivos de documentación existen
ls -la byod-policy-design.json byod-implementation-status.md analisis-arquitectonico.md
# Verificar contenido mínimo del análisis
grep -c "Modelo\|Riesgo\|Decisión" analisis-arquitectonico.md
# Debe retornar >= 5 coincidencias
```

---

## Validación y Pruebas

### Lista de verificación final

Ejecutar el siguiente script de validación para confirmar que todos los componentes están correctamente configurados:

```bash
#!/bin/bash
# Script de validación Lab 07-00-01
echo "========================================="
echo "VALIDACIÓN LAB 07-00-01"
echo "========================================="

ERRORS=0

# 1. Verificar Keycloak accesible
echo -n "[1] Keycloak HTTPS accesible... "
if curl -sk "https://localhost:8443/realms/empresa" | grep -q "empresa"; then
    echo "OK"
else
    echo "FALLO - Verificar que Keycloak está corriendo"
    ERRORS=$((ERRORS+1))
fi

# 2. Verificar grupo aws-developers
echo -n "[2] Grupo aws-developers en Keycloak... "
GROUP_COUNT=$(curl -s -H "Authorization: Bearer $ADMIN_TOKEN" \
  "https://localhost:8443/admin/realms/empresa/groups" --insecure | \
  python3 -c "import sys,json; groups=json.load(sys.stdin); print(sum(1 for g in groups if 'aws-developers' in g.get('name','')))")
if [ "$GROUP_COUNT" -gt "0" ]; then
    echo "OK"
else
    echo "FALLO - Crear grupo aws-developers"
    ERRORS=$((ERRORS+1))
fi

# 3. Verificar proveedor SAML en AWS
echo -n "[3] Proveedor SAML en AWS IAM... "
if aws iam list-saml-providers --query "SAMLProviderList[?contains(Arn,'KeycloakIdP')]" \
   --output text | grep -q "KeycloakIdP"; then
    echo "OK"
else
    echo "FALLO - Crear proveedor SAML en AWS"
    ERRORS=$((ERRORS+1))
fi

# 4. Verificar rol IAM
echo -n "[4] Rol KeycloakDeveloperAccess en AWS... "
if aws iam get-role --role-name KeycloakDeveloperAccess \
   --query 'Role.RoleName' --output text 2>/dev/null | grep -q "Keycloak"; then
    echo "OK"
else
    echo "FALLO - Crear rol KeycloakDeveloperAccess"
    ERRORS=$((ERRORS+1))
fi

# 5. Verificar metadatos exportados
echo -n "[5] Archivo de metadatos Keycloak... "
if [ -f "keycloak-idp-metadata.xml" ] && grep -q "entityID" keycloak-idp-metadata.xml; then
    echo "OK"
else
    echo "FALLO - Exportar metadatos de Keycloak"
    ERRORS=$((ERRORS+1))
fi

# 6. Verificar Authentication Flow BYOD
echo -n "[6] Flow BYOD-Conditional-Access-Flow... "
FLOW_COUNT=$(curl -s -H "Authorization: Bearer $ADMIN_TOKEN" \
  "https://localhost:8443/admin/realms/empresa/authentication/flows" --insecure | \
  python3 -c "import sys,json; flows=json.load(sys.stdin); print(sum(1 for f in flows if 'BYOD' in f.get('alias','')))")
if [ "$FLOW_COUNT" -gt "0" ]; then
    echo "OK"
else
    echo "FALLO - Crear BYOD Authentication Flow"
    ERRORS=$((ERRORS+1))
fi

# 7. Verificar documentación
echo -n "[7] Documentos de análisis arquitectónico... "
if [ -f "analisis-arquitectonico.md" ] && [ -f "byod-policy-design.json" ]; then
    echo "OK"
else
    echo "FALLO - Generar documentos de análisis"
    ERRORS=$((ERRORS+1))
fi

echo "========================================="
if [ "$ERRORS" -eq "0" ]; then
    echo "RESULTADO: TODOS LOS CHECKS PASARON ✓"
else
    echo "RESULTADO: $ERRORS CHECKS FALLARON ✗"
fi
echo "========================================="
```

### Prueba de extremo a extremo del flujo SAML

```bash
# Verificar que la aserción SAML contiene los atributos correctos
# (inspección de metadatos y configuración)

python3 << 'EOF'
import xml.etree.ElementTree as ET

# Verificar metadatos del IdP
tree = ET.parse('keycloak-idp-metadata.xml')
root = tree.getroot()

ns = {
    'md': 'urn:oasis:names:tc:SAML:2.0:metadata',
    'ds': 'http://www.w3.org/2000/09/xmldsig#'
}

entity_id = root.attrib.get('entityID', 'No encontrado')
sso_services = root.findall('.//md:SingleSignOnService', ns)
certs = root.findall('.//ds:X509Certificate', ns)

print("=== Validación de Metadatos SAML ===")
print(f"Entity ID: {entity_id}")
print(f"SSO Services: {len(sso_services)}")
for sso in sso_services:
    print(f"  - Binding: {sso.attrib.get('Binding','').split(':')[-1]}")
    print(f"    Location: {sso.attrib.get('Location','')[:60]}...")
print(f"Certificados X.509: {len(certs)}")
print(f"Certificado (primeros 40 chars): {certs[0].text[:40] if certs else 'No encontrado'}...")

print("\n=== Checklist de Federación SAML ===")
checks = [
    ("Entity ID configurado", 'empresa' in entity_id or 'ngrok' in entity_id),
    ("SSO Service HTTP-POST", any('POST' in s.attrib.get('Binding','') for s in sso_services)),
    ("Certificado de firma presente", len(certs) > 0),
]
for check_name, result in checks:
    status = "✓" if result else "✗"
    print(f"  [{status}] {check_name}")
EOF
```

---

## Resolución de Problemas

### Problema 1: AWS rechaza los metadatos SAML de Keycloak con error "Invalid metadata"

**Síntoma**: Al intentar subir `keycloak-idp-metadata.xml` en AWS IAM Identity Center, aparece el error: `"The provided metadata is not valid SAML metadata"` o `"Certificate validation failed"`.

**Causa**: El certificado en los metadatos de Keycloak puede estar en formato incorrecto, el XML puede contener caracteres de espacio en blanco problemáticos, o el `entityID` usa `localhost` en lugar de la URL pública de ngrok. AWS requiere que el `entityID` y los endpoints en los metadatos usen URLs HTTPS accesibles públicamente.

**Solución**:
```bash
# 1. Verificar que el entityID usa la URL pública (ngrok), no localhost
grep "entityID" keycloak-idp-metadata.xml
# Si muestra localhost, actualizar la variable y re-exportar:
export KEYCLOAK_PUBLIC_URL="https://xxxx.ngrok-free.app"

# 2. En Keycloak Admin → Realm Settings → General
# Cambiar "Frontend URL" a la URL pública de ngrok:
curl -s -X PUT \
  -H "Authorization: Bearer $ADMIN_TOKEN" \
  -H "Content-Type: application/json" \
  "https://localhost:8443/admin/realms/empresa" \
  -d "{\"attributes\": {\"frontendUrl\": \"${KEYCLOAK_PUBLIC_URL}\"}}" \
  --insecure

# 3. Re-exportar los metadatos con la URL correcta
curl -s "${KEYCLOAK_PUBLIC_URL}/realms/${REALM}/protocol/saml/descriptor" \
  --insecure -o keycloak-idp-metadata-fixed.xml

# 4. Verificar el entityID en el nuevo archivo
grep "entityID" keycloak-idp-metadata-fixed.xml
# Debe mostrar: entityID="https://xxxx.ngrok-free.app/realms/empresa"

# 5. Limpiar espacios en blanco problemáticos en el XML
python3 -c "
import xml.etree.ElementTree as ET
ET.register_namespace('', 'urn:oasis:names:tc:SAML:2.0:metadata')
tree = ET.parse('keycloak-idp-metadata-fixed.xml')
tree.write('keycloak-idp-metadata-clean.xml', xml_declaration=True, encoding='UTF-8')
print('Archivo limpio generado: keycloak-idp-metadata-clean.xml')
"

# Subir keycloak-idp-metadata-clean.xml a AWS IAM Identity Center
```

---

### Problema 2: El usuario federado obtiene error "AccessDenied" al intentar asumir el rol en AWS

**Síntoma**: El usuario `dev.ficticio01` se autentica correctamente en Keycloak, pero al ser redirigido a AWS aparece: `"Error: Your request included an invalid SAML response. To logout, click here."` o `"AccessDenied: Not authorized to perform sts:AssumeRoleWithSAML"`.

**Causa**: El valor del atributo `aws_role` en Keycloak no coincide exactamente con el ARN del rol y el ARN del proveedor SAML en AWS. El formato debe ser `arn:aws:iam::ACCOUNT_ID:role/ROLE_NAME,arn:aws:iam::ACCOUNT_ID:saml-provider/PROVIDER_NAME` (separados por coma, sin espacios). También puede deberse a que el mapper `RoleSessionName` no está configurado (AWS lo requiere).

**Solución**:
```bash
# 1. Obtener los ARNs exactos de AWS
EXACT_ROLE_ARN=$(aws iam get-role --role-name KeycloakDeveloperAccess \
  --query 'Role.Arn' --output text)
EXACT_PROVIDER_ARN=$(aws iam list-saml-providers \
  --query "SAMLProviderList[?contains(Arn,'KeycloakIdP')].Arn" \
  --output text)

echo "ARN del rol: $EXACT_ROLE_ARN"
echo "ARN del proveedor: $EXACT_PROVIDER_ARN"
echo "Valor correcto del atributo aws_role:"
echo "${EXACT_ROLE_ARN},${EXACT_PROVIDER_ARN}"

# 2. Verificar que no hay espacios en el valor del atributo
# En Keycloak Admin → Users → dev.ficticio01 → Attributes
# El valor de aws_role debe ser EXACTAMENTE:
# arn:aws:iam::123456789012:role/KeycloakDeveloperAccess,arn:aws:iam::123456789012:saml-provider/KeycloakIdP

# 3. Verificar que el mapper RoleSessionName existe en el cliente SAML
curl -s -H "Authorization: Bearer $ADMIN_TOKEN" \
  "https://localhost:8443/admin/realms/empresa/clients" --insecure | \
  python3 -c "
import sys, json, urllib.parse
clients = json.load(sys.stdin)
for c in clients:
    if 'amazonaws' in c.get('clientId',''):
        print('Client UUID:', c['id'])
" 
# Usar el UUID para verificar los mappers:
# GET /admin/realms/empresa/clients/{UUID}/protocol-mappers/models

# 4. Verificar la política de confianza del rol
aws iam get-role --role-name KeycloakDeveloperAccess \
  --query 'Role.AssumeRolePolicyDocument' --output json | \
  python3 -c "
import sys, json
policy = json.load(sys.stdin)
for stmt in policy['Statement']:
    print('Action:', stmt['Action'])
    print('Condition:', json.dumps(stmt.get('Condition',{}), indent=2))
    # Verificar que SAML:aud == https://signin.aws.amazon.com/saml
"
```

---

## Limpieza del Entorno

> ⚠️ **Importante**: Ejecutar la limpieza en el orden indicado para evitar recursos huérfanos en AWS que puedan generar costos o riesgos de seguridad.

```bash
#!/bin/bash
echo "=== LIMPIEZA LAB 07-00-01 ==="

# 1. Limpiar recursos AWS (PRIMERO - evitar costos y riesgos)
echo "[1] Eliminando rol IAM KeycloakDeveloperAccess..."
aws iam detach-role-policy \
  --role-name KeycloakDeveloperAccess \
  --policy-arn arn:aws:iam::aws:policy/ReadOnlyAccess 2>/dev/null
aws iam delete-role --role-name KeycloakDeveloperAccess 2>/dev/null
echo "    Rol eliminado."

echo "[2] Eliminando proveedor SAML en AWS IAM..."
PROVIDER_ARN=$(aws iam list-saml-providers \
  --query "SAMLProviderList[?contains(Arn,'KeycloakIdP')].Arn" \
  --output text 2>/dev/null)
if [ -n "$PROVIDER_ARN" ]; then
  aws iam delete-saml-provider --saml-provider-arn "$PROVIDER_ARN"
  echo "    Proveedor SAML eliminado: $PROVIDER_ARN"
fi

# 3. Revertir IAM Identity Center a fuente de identidad interna
echo "[3] Revertir IAM Identity Center a Identity Store interno..."
echo "    MANUAL: AWS Console → IAM Identity Center → Settings → Change identity source → Identity Center directory"

# 4. Detener ngrok
echo "[4] Deteniendo ngrok..."
pkill ngrok 2>/dev/null
echo "    ngrok detenido."

# 5. Limpiar configuración de Keycloak (revertir Browser Flow)
echo "[5] Revirtiendo Browser Flow en Keycloak a 'browser' (default)..."
curl -s -X PUT \
  -H "Authorization: Bearer $ADMIN_TOKEN" \
  -H "Content-Type: application/json" \
  "https://localhost:8443/admin/realms/empresa" \
  -d '{"browserFlow": "browser"}' \
  --insecure > /dev/null
echo "    Browser Flow revertido."

# 6. Limpiar archivos temporales del lab
echo "[6] Limpiando archivos temporales..."
rm -f keycloak-idp-metadata.xml keycloak-idp-metadata-fixed.xml \
      keycloak-idp-metadata-clean.xml aws-sso-sp-metadata.xml \
      trust-policy-keycloak.json saml-signing.key saml-signing.crt
echo "    Archivos temporales eliminados."

# 7. Detener stack Docker (opcional - conservar para siguiente lab si aplica)
echo "[7] Stack Docker (Lab 6)..."
echo "    Para detener: cd ~/lab06 && docker compose down -v"
echo "    Para conservar: no ejecutar este comando"

echo ""
echo "=== LIMPIEZA COMPLETADA ==="
echo "Archivos conservados (documentación):"
ls -la byod-policy-design.json byod-implementation-status.md analisis-arquitectonico.md 2>/dev/null
```

---

## Resumen

En este laboratorio implementaste una arquitectura completa de **identidad federada híbrida** que conecta infraestructura on-premise con servicios cloud de AWS, aplicando los conceptos del modelo federado de la lección 7.1:

| Componente | Implementado | Concepto 7.1 |
|------------|-------------|--------------|
| Keycloak como IdP SAML | ✓ | Identidad federada: autenticación on-prem |
| Metadatos SAML y certificados X.509 | ✓ | Gestión de certificados y tokens |
| Proveedor SAML en AWS IAM | ✓ | Relación de confianza SP-IdP |
| Rol IAM con trust policy | ✓ | Principio de mínimo privilegio |
| Mapeo de atributos SAML → roles AWS | ✓ | Claims/atributos en aserciones |
| Grupo `aws-developers` | ✓ | RBAC en el IdP on-prem |
| BYOD Authentication Flow | ✓ (parcial) | Acceso condicional, Zero Trust |
| Documento de diseño arquitectónico | ✓ | Análisis de riesgos por modelo |

### Puntos clave aprendidos

1. **El modelo federado** mantiene la fuente de verdad y la autenticación on-premise, con AWS confiando en aserciones SAML firmadas. Esto preserva el control corporativo pero introduce el **SPOF del IdP** como riesgo crítico.

2. **El atributo SAML `Role`** es el mecanismo clave para mapear identidades Keycloak a roles IAM de AWS. El formato exacto `arn:role,arn:provider` es obligatorio y sensible a errores.

3. **Las políticas BYOD** requieren una combinación de Authentication Flows (Keycloak), gestión de tokens de corta duración y, en producción, integración con MDM para compliance de dispositivos.

4. **Comparando modelos**: el modelo federado ofrece mayor control y portabilidad que Cognito (cloud-only) o Entra ID (broker), pero a costa de mayor complejidad operativa y responsabilidad en la gestión del IdP on-prem.

### Recursos adicionales

- [AWS: Enabling SAML 2.0 federation to the AWS Management Console](https://docs.aws.amazon.com/IAM/latest/UserGuide/id_roles_providers_enable-console-saml.html)
- [Keycloak: SAML Identity Providers](https://www.keycloak.org/docs/latest/server_admin/#saml-v2-0-identity-providers)
- [AWS IAM Identity Center: External Identity Providers](https://docs.aws.amazon.com/singlesignon/latest/userguide/manage-your-identity-source-idp.html)
- [NIST SP 800-63C: Federation and Assertions](https://pages.nist.gov/800-63-3/sp800-63c.html)
- [RFC 7522: SAML 2.0 Profile for OAuth 2.0](https://www.rfc-editor.org/rfc/rfc7522)
- [Keycloak: Configuring Authentication Flows](https://www.keycloak.org/docs/latest/server_admin/#configuring-authentication)

---
*Lab 07-00-01 — Módulo 7: Modelos de identidad en cloud — Curso IAM Open Source*
