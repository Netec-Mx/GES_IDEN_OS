# Configurar SSO con Keycloak y una App Cliente

## 1. Metadatos

| Campo            | Detalle                                                                 |
|------------------|-------------------------------------------------------------------------|
| **Duración**     | 31 minutos                                                              |
| **Complejidad**  | Media                                                                   |
| **Nivel Bloom**  | Aplicar                                                                 |
| **Módulo**       | Módulo 4 — Federación de Identidades y Protocolos Estándar             |
| **Versiones**    | Keycloak 23.x · PostgreSQL 15 · Docker Compose 2.20.x                 |

---

## 2. Descripción General

En este laboratorio desplegarás **Keycloak 23** con **PostgreSQL 15** como backend de persistencia y configurarás un realm corporativo llamado `corp-lab`. A partir de ese realm crearás usuarios de prueba con roles diferenciados, registrarás un cliente **OIDC** (Authorization Code + PKCE) y un cliente **SAML 2.0**, y verificarás el flujo completo de Single Sign-On entre ambas aplicaciones. Finalmente, usarás **Postman** y **jwt.io** para interceptar y analizar la estructura y seguridad de los tokens JWT emitidos por Keycloak, aplicando los conceptos comparativos de SAML, OAuth2 y OIDC estudiados en la lección 4.1.

---

## 3. Objetivos de Aprendizaje

- [ ] Configurar Keycloak como Identity Provider con un realm corporativo (`corp-lab`) y al menos dos clientes para SSO.
- [ ] Implementar y verificar el flujo **Authorization Code con PKCE** usando OpenID Connect con una aplicación web cliente.
- [ ] Configurar un segundo cliente **SAML 2.0**, importar metadatos SP y validar el flujo de Single Sign-On y Single Logout.
- [ ] Analizar tokens JWT (claims, scopes, expiración, refresh token) con **jwt.io** y **Postman**, e implementar mappers personalizados de claims.

---

## 4. Prerrequisitos

### Conocimiento Previo
- Comprensión de los protocolos OAuth2, OpenID Connect y SAML 2.0 (lección 4.1 de este módulo).
- Familiaridad con la estructura JWT: `header.payload.signature` y claims estándar (`iss`, `sub`, `aud`, `exp`, `iat`).
- Conocimiento básico de HTTP/S, redirecciones y cookies de sesión.
- Manejo básico de Docker Compose y línea de comandos Linux.

### Acceso y Software Requerido
- Docker Engine 24.x y Docker Compose 2.20.x instalados y operativos.
- Postman 10.x instalado con acceso a la interfaz web.
- Navegador web moderno (Firefox o Chrome) con las DevTools disponibles.
- Acceso a [https://jwt.io](https://jwt.io) desde el navegador del laboratorio.
- Mínimo **4 GB de RAM** disponibles para los contenedores de este lab (Keycloak + PostgreSQL).
- Puerto **8080** y **8443** libres en el host.

> **Nota:** Este laboratorio puede realizarse de forma independiente, pero se recomienda haber completado el Lab 02-00-01 (OpenLDAP) para familiaridad con el entorno Docker del curso.

---

## 5. Entorno de Laboratorio

### Recursos de Hardware Recomendados

| Recurso         | Mínimo        | Recomendado   |
|-----------------|---------------|---------------|
| RAM             | 4 GB libres   | 8 GB libres   |
| CPU             | 2 vCPU        | 4 vCPU        |
| Almacenamiento  | 5 GB libres   | 10 GB libres  |

### Arquitectura del Laboratorio

```
┌─────────────────────────────────────────────────────────────┐
│                    Host / VM Ubuntu 22.04                    │
│                                                             │
│  ┌──────────────────┐    ┌──────────────────────────────┐  │
│  │  PostgreSQL 15   │◄───│     Keycloak 23 (IdP)        │  │
│  │  :5432           │    │     :8080 / :8443            │  │
│  └──────────────────┘    └──────────────┬───────────────┘  │
│                                          │ OIDC / SAML      │
│  ┌──────────────────┐    ┌──────────────▼───────────────┐  │
│  │  App OIDC        │    │  App SAML (whoami-saml)      │  │
│  │  (Flask :5000)   │    │  (:5001)                     │  │
│  └──────────────────┘    └──────────────────────────────┘  │
│                                                             │
│  Postman ──► Keycloak Token Endpoint                        │
│  Browser ──► jwt.io (análisis de tokens)                    │
└─────────────────────────────────────────────────────────────┘
```

### Preparación del Entorno — Comandos de Configuración Inicial

Ejecuta los siguientes comandos en tu terminal antes de comenzar los pasos del laboratorio:

```bash
# 1. Crear directorio de trabajo del laboratorio
mkdir -p ~/lab-04-sso/{config,certs,apps/oidc-app,apps/saml-app,data}
cd ~/lab-04-sso

# 2. Clonar o crear los archivos de la aplicación de demostración
# (El repositorio del curso provee los archivos; aquí se crean manualmente)

# 3. Verificar que los puertos necesarios estén libres
ss -tlnp | grep -E ':(8080|8443|5000|5001|5432)'
# Si algún puerto está ocupado, detén el proceso correspondiente antes de continuar
```

---

## 6. Pasos del Laboratorio

---

### Paso 1: Desplegar Keycloak 23 con PostgreSQL mediante Docker Compose

**Objetivo:** Levantar la infraestructura base del laboratorio con Keycloak y su base de datos PostgreSQL, verificando que ambos servicios estén operativos.

#### Instrucciones

**1.1** Crea el archivo `docker-compose.yml` en `~/lab-04-sso/`:

```yaml
# ~/lab-04-sso/docker-compose.yml
version: "3.9"

services:
  postgres:
    image: postgres:15-alpine
    container_name: lab04-postgres
    environment:
      POSTGRES_DB: keycloak
      POSTGRES_USER: keycloak
      POSTGRES_PASSWORD: Kc_L4b_P4ss!
    volumes:
      - postgres_data:/var/lib/postgresql/data
    networks:
      - lab04-net
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U keycloak"]
      interval: 10s
      timeout: 5s
      retries: 5

  keycloak:
    image: quay.io/keycloak/keycloak:23.0
    container_name: lab04-keycloak
    command: start-dev
    environment:
      KC_DB: postgres
      KC_DB_URL: jdbc:postgresql://postgres:5432/keycloak
      KC_DB_USERNAME: keycloak
      KC_DB_PASSWORD: Kc_L4b_P4ss!
      KC_HOSTNAME: localhost
      KC_HTTP_PORT: 8080
      KEYCLOAK_ADMIN: admin
      KEYCLOAK_ADMIN_PASSWORD: Admin_L4b!2024
    ports:
      - "8080:8080"
    depends_on:
      postgres:
        condition: service_healthy
    networks:
      - lab04-net

  oidc-app:
    image: python:3.11-slim
    container_name: lab04-oidc-app
    working_dir: /app
    volumes:
      - ./apps/oidc-app:/app
    command: >
      sh -c "pip install flask requests authlib --quiet &&
             python app.py"
    ports:
      - "5000:5000"
    environment:
      FLASK_ENV: development
      KC_BASE_URL: http://keycloak:8080
      KC_REALM: corp-lab
      CLIENT_ID: oidc-web-client
      CLIENT_SECRET: ""   # se actualiza en el Paso 3
      REDIRECT_URI: http://localhost:5000/callback
    networks:
      - lab04-net
    depends_on:
      - keycloak

networks:
  lab04-net:
    driver: bridge

volumes:
  postgres_data:
```

**1.2** Crea la aplicación Flask OIDC de demostración en `~/lab-04-sso/apps/oidc-app/app.py`:

```python
# ~/lab-04-sso/apps/oidc-app/app.py
import os, json
from flask import Flask, redirect, url_for, session, request, jsonify
from authlib.integrations.flask_client import OAuth

app = Flask(__name__)
app.secret_key = "lab04-super-secret-flask-key"

KC_BASE    = os.getenv("KC_BASE_URL", "http://localhost:8080")
KC_REALM   = os.getenv("KC_REALM", "corp-lab")
CLIENT_ID  = os.getenv("CLIENT_ID", "oidc-web-client")
CLIENT_SEC = os.getenv("CLIENT_SECRET", "")

OIDC_BASE = f"{KC_BASE}/realms/{KC_REALM}/protocol/openid-connect"

oauth = OAuth(app)
oauth.register(
    name="keycloak",
    client_id=CLIENT_ID,
    client_secret=CLIENT_SEC,
    server_metadata_url=f"{KC_BASE}/realms/{KC_REALM}/.well-known/openid-configuration",
    client_kwargs={
        "scope": "openid email profile roles",
        "code_challenge_method": "S256",  # PKCE
    },
)

@app.route("/")
def index():
    user = session.get("user")
    if user:
        return f"""<h2>✅ Sesión activa</h2>
        <pre>{json.dumps(user, indent=2)}</pre>
        <a href='/logout'>Cerrar sesión</a>"""
    return '<h2>App OIDC Demo</h2><a href="/login">Iniciar sesión con Keycloak</a>'

@app.route("/login")
def login():
    return oauth.keycloak.authorize_redirect(url_for("callback", _external=True))

@app.route("/callback")
def callback():
    token = oauth.keycloak.authorize_access_token()
    session["user"]         = token.get("userinfo")
    session["access_token"] = token.get("access_token")
    session["id_token"]     = token.get("id_token")
    return redirect("/")

@app.route("/token-info")
def token_info():
    return jsonify({
        "access_token": session.get("access_token"),
        "id_token":     session.get("id_token"),
        "userinfo":     session.get("user"),
    })

@app.route("/logout")
def logout():
    session.clear()
    logout_url = (
        f"{OIDC_BASE}/logout"
        f"?post_logout_redirect_uri={url_for('index', _external=True)}"
        f"&client_id={CLIENT_ID}"
    )
    return redirect(logout_url)

if __name__ == "__main__":
    app.run(host="0.0.0.0", port=5000, debug=True)
```

**1.3** Inicia los servicios:

```bash
cd ~/lab-04-sso
docker compose up -d postgres keycloak
```

**1.4** Espera a que Keycloak esté completamente iniciado (aproximadamente 60–90 segundos):

```bash
# Monitorear el inicio de Keycloak
docker compose logs -f keycloak | grep -E "(started|error|WARN)"
# Presiona Ctrl+C cuando veas: "Keycloak X.X.X on JVM ... started in"
```

**1.5** Verifica que ambos servicios estén en estado `healthy` / `running`:

```bash
docker compose ps
```

#### Salida Esperada

```
NAME               IMAGE                              STATUS          PORTS
lab04-postgres     postgres:15-alpine                 Up (healthy)    5432/tcp
lab04-keycloak     quay.io/keycloak/keycloak:23.0     Up              0.0.0.0:8080->8080/tcp
```

#### Verificación

Abre el navegador en `http://localhost:8080` — deberías ver la pantalla de bienvenida de Keycloak. Accede a la consola de administración en `http://localhost:8080/admin` con las credenciales `admin` / `Admin_L4b!2024`.

---

### Paso 2: Crear el Realm `corp-lab` y Usuarios de Prueba

**Objetivo:** Configurar el realm corporativo con usuarios ficticios y roles que representen una organización real de laboratorio.

#### Instrucciones

**2.1** En la consola de administración de Keycloak (`http://localhost:8080/admin`), crea el realm `corp-lab`:
- Haz clic en el menú desplegable **"Keycloak"** (esquina superior izquierda) → **"Create Realm"**.
- **Realm name:** `corp-lab`
- **Enabled:** ON
- Haz clic en **"Create"**.

**2.2** Dentro del realm `corp-lab`, crea los roles del realm. Ve a **Realm roles** → **"Create role"**:

| Rol              | Descripción                        |
|------------------|------------------------------------|
| `corp-user`      | Usuario estándar de la organización |
| `corp-analyst`   | Analista con acceso extendido       |
| `corp-admin`     | Administrador de aplicaciones       |

Para cada rol: ingresa el nombre, descripción y haz clic en **"Save"**.

**2.3** Crea los usuarios de prueba. Ve a **Users** → **"Add user"**:

**Usuario 1 — Ana Ficticia (analista):**
```
Username:    ana.ficticia
Email:       ana.ficticia@corp-lab.local
First name:  Ana
Last name:   Ficticia
Email verified: ON
```
Tras guardar, ve a la pestaña **Credentials** → **"Set password"**:
```
Password:     CorpLab#Ana2024
Temporary:    OFF
```
Ve a la pestaña **Role mapping** → **"Assign role"** → selecciona `corp-analyst` y `corp-user`.

**Usuario 2 — Carlos Inventado (usuario estándar):**
```
Username:    carlos.inventado
Email:       carlos.inventado@corp-lab.local
First name:  Carlos
Last name:   Inventado
Email verified: ON
Password:    CorpLab#Carlos2024  (Temporary: OFF)
Roles:       corp-user
```

**2.4** Verifica la creación de usuarios desde la CLI del contenedor:

```bash
docker exec lab04-keycloak /opt/keycloak/bin/kcadm.sh \
  config credentials \
  --server http://localhost:8080 \
  --realm master \
  --user admin \
  --password 'Admin_L4b!2024'

docker exec lab04-keycloak /opt/keycloak/bin/kcadm.sh \
  get users -r corp-lab --fields username,email,enabled
```

#### Salida Esperada

```json
[ {
  "username" : "ana.ficticia",
  "email" : "ana.ficticia@corp-lab.local",
  "enabled" : true
}, {
  "username" : "carlos.inventado",
  "email" : "carlos.inventado@corp-lab.local",
  "enabled" : true
} ]
```

#### Verificación

En la consola de Keycloak, navega a **Users** del realm `corp-lab` y confirma que ambos usuarios aparecen listados con el estado **Enabled**.

---

### Paso 3: Configurar el Cliente OIDC y Verificar el Flujo Authorization Code con PKCE

**Objetivo:** Registrar la aplicación Flask como cliente OIDC en Keycloak, levantar la app y verificar el flujo completo de autenticación con PKCE.

#### Instrucciones

**3.1** En la consola de Keycloak (realm `corp-lab`), ve a **Clients** → **"Create client"**:

```
Client type:    OpenID Connect
Client ID:      oidc-web-client
Name:           OIDC Web Demo Client
```
Haz clic en **"Next"**.

**3.2** En la pantalla de **Capability config**:
```
Client authentication:  ON   (confidential client)
Authorization:          OFF
Authentication flow:    ✅ Standard flow (Authorization Code)
                        ✅ Direct access grants (para pruebas con Postman)
```
Haz clic en **"Next"**.

**3.3** En **Login settings**:
```
Root URL:                   http://localhost:5000
Home URL:                   http://localhost:5000
Valid redirect URIs:        http://localhost:5000/callback
                            http://localhost:5000/*
Valid post logout redirect URIs:  http://localhost:5000/
Web origins:                http://localhost:5000
```
Haz clic en **"Save"**.

**3.4** Obtén el **Client Secret**. Ve a la pestaña **Credentials** del cliente `oidc-web-client`:
- Copia el valor del campo **"Client secret"** (formato UUID).

**3.5** Actualiza el `docker-compose.yml` con el secreto obtenido. Modifica la variable de entorno del servicio `oidc-app`:

```yaml
# En el servicio oidc-app, actualiza:
CLIENT_SECRET: "PEGA_AQUI_EL_CLIENT_SECRET_COPIADO"
```

**3.6** Configura el scope personalizado para incluir roles. Ve a **Client scopes** → **"Create client scope"**:
```
Name:           corp-roles
Type:           Default
Protocol:       OpenID Connect
```
Dentro del scope creado, ve a la pestaña **Mappers** → **"Add mapper"** → **"By configuration"** → **"User Realm Role"**:
```
Name:           realm-roles-mapper
Token Claim Name: roles
Add to ID token:      ON
Add to access token:  ON
Add to userinfo:      ON
```
Guarda el mapper y luego asigna el scope al cliente: ve a **Clients** → `oidc-web-client` → **Client scopes** → **"Add client scope"** → selecciona `corp-roles` → **"Add (Default)"**.

**3.7** Inicia la aplicación OIDC:

```bash
cd ~/lab-04-sso
docker compose up -d oidc-app
docker compose logs -f oidc-app
# Espera hasta ver: "Running on http://0.0.0.0:5000"
# Presiona Ctrl+C
```

**3.8** Prueba el flujo SSO en el navegador:
1. Abre `http://localhost:5000` → verás el enlace **"Iniciar sesión con Keycloak"**.
2. Haz clic → serás redirigido a la página de login de Keycloak.
3. Inicia sesión con `ana.ficticia` / `CorpLab#Ana2024`.
4. Tras autenticarse, Keycloak redirige a `http://localhost:5000/callback` con el código de autorización.
5. La app intercambia el código por tokens y muestra la información del usuario.

**3.9** Verifica el parámetro PKCE en el flujo. Abre las **DevTools** del navegador (F12) → pestaña **Network**, filtra por `openid-connect/auth`. Observa los parámetros de la solicitud inicial:

```
response_type=code
client_id=oidc-web-client
scope=openid+email+profile+roles
code_challenge=<BASE64URL_SHA256_del_verifier>
code_challenge_method=S256
redirect_uri=http://localhost:5000/callback
state=<random_state>
```

#### Salida Esperada

La página `http://localhost:5000` mostrará:

```
✅ Sesión activa
{
  "sub": "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx",
  "email": "ana.ficticia@corp-lab.local",
  "name": "Ana Ficticia",
  "given_name": "Ana",
  "family_name": "Ficticia",
  "roles": ["corp-analyst", "corp-user", "default-roles-corp-lab"]
}
```

#### Verificación

Navega a `http://localhost:5000/token-info` — deberías ver el `access_token` JWT y el `id_token` en la respuesta JSON. Confirma que el campo `roles` contiene `corp-analyst`.

---

### Paso 4: Inspeccionar Tokens JWT con jwt.io y Postman

**Objetivo:** Analizar la estructura y claims de los tokens JWT emitidos por Keycloak, verificando firma, scopes y tiempos de expiración.

#### Instrucciones

**4.1** Obtén el `access_token` de la sesión activa:

```bash
# Desde el navegador, accede a:
# http://localhost:5000/token-info
# Copia el valor del campo "access_token"
```

**4.2** Analiza el token en **jwt.io**:
1. Abre `https://jwt.io` en el navegador.
2. Pega el `access_token` en el campo **"Encoded"**.
3. Observa el **Header** decodificado:
```json
{
  "alg": "RS256",
  "typ": "JWT",
  "kid": "<key-id>"
}
```
4. Observa el **Payload** decodificado e identifica los claims:

```json
{
  "exp": 1752086400,
  "iat": 1752082800,
  "jti": "<uuid>",
  "iss": "http://localhost:8080/realms/corp-lab",
  "aud": "account",
  "sub": "<user-uuid>",
  "typ": "Bearer",
  "azp": "oidc-web-client",
  "session_state": "<session-uuid>",
  "scope": "openid email profile roles",
  "email_verified": true,
  "roles": ["corp-analyst", "corp-user"],
  "name": "Ana Ficticia",
  "preferred_username": "ana.ficticia",
  "email": "ana.ficticia@corp-lab.local"
}
```

**4.3** Verifica la firma del token usando el endpoint JWKS de Keycloak. Obtén las claves públicas:

```bash
curl -s http://localhost:8080/realms/corp-lab/.well-known/openid-configuration | \
  python3 -m json.tool | grep jwks_uri

# Luego obtén las claves:
curl -s http://localhost:8080/realms/corp-lab/protocol/openid-connect/certs | \
  python3 -m json.tool
```

**4.4** Configura Postman para obtener tokens directamente. Abre Postman y crea una nueva solicitud:

- **Método:** POST
- **URL:** `http://localhost:8080/realms/corp-lab/protocol/openid-connect/token`
- **Body** (x-www-form-urlencoded):

```
grant_type:    password
client_id:     oidc-web-client
client_secret: <TU_CLIENT_SECRET>
username:      carlos.inventado
password:      CorpLab#Carlos2024
scope:         openid email profile roles
```

Envía la solicitud y examina la respuesta completa:

```json
{
  "access_token": "eyJhbGciOiJSUzI1NiIsInR5cCI6IkpXVCJ9...",
  "expires_in": 300,
  "refresh_expires_in": 1800,
  "refresh_token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...",
  "token_type": "Bearer",
  "id_token": "eyJhbGciOiJSUzI1NiIsInR5cCI6IkpXVCJ9...",
  "not-before-policy": 0,
  "session_state": "<uuid>",
  "scope": "openid email profile roles"
}
```

**4.5** Configura tiempos de expiración en Keycloak. Ve a **Realm settings** → **Tokens**:

```
Access Token Lifespan:          5 minutes   (valor por defecto)
Refresh Token Max Reuse:        0
SSO Session Idle:               30 minutes
SSO Session Max:                10 hours
```

Anota los valores y experimenta cambiando el **Access Token Lifespan** a **1 minuto** para observar la expiración.

**4.6** Prueba el refresh token con Postman. Crea una segunda solicitud POST al mismo endpoint:

```
grant_type:     refresh_token
client_id:      oidc-web-client
client_secret:  <TU_CLIENT_SECRET>
refresh_token:  <REFRESH_TOKEN_DEL_PASO_4.4>
```

Verifica que obtienes un nuevo `access_token` con un `iat` actualizado.

**4.7** Verifica el endpoint de discovery OIDC:

```bash
curl -s http://localhost:8080/realms/corp-lab/.well-known/openid-configuration | \
  python3 -m json.tool | grep -E '"(issuer|authorization_endpoint|token_endpoint|jwks_uri|userinfo_endpoint)"'
```

#### Salida Esperada (fragmento del discovery)

```json
"issuer": "http://localhost:8080/realms/corp-lab",
"authorization_endpoint": "http://localhost:8080/realms/corp-lab/protocol/openid-connect/auth",
"token_endpoint": "http://localhost:8080/realms/corp-lab/protocol/openid-connect/token",
"jwks_uri": "http://localhost:8080/realms/corp-lab/protocol/openid-connect/certs",
"userinfo_endpoint": "http://localhost:8080/realms/corp-lab/protocol/openid-connect/userinfo"
```

#### Verificación

En jwt.io, pega el JWKS (clave pública RSA) en el campo **"Public Key or Certificate"** y confirma que el indicador de verificación de firma muestra **"Signature Verified"** en verde.

---

### Paso 5: Configurar un Cliente SAML 2.0 e Intercambiar Metadatos

**Objetivo:** Registrar un segundo cliente usando SAML 2.0, importar metadatos del SP y verificar el flujo de autenticación SAML.

#### Instrucciones

**5.1** Crea la aplicación SAML de demostración. Crea el archivo `~/lab-04-sso/apps/saml-app/app.py`:

```python
# ~/lab-04-sso/apps/saml-app/app.py
# Aplicación minimalista que actúa como SP SAML usando python3-saml
import os, json
from flask import Flask, request, redirect, session, url_for, Response
from onelogin.saml2.auth import OneLogin_Saml2_Auth
from onelogin.saml2.utils import OneLogin_Saml2_Utils

app = Flask(__name__)
app.secret_key = "lab04-saml-secret-key"

SAML_PATH = os.path.join(os.path.dirname(__file__), "saml")

def prepare_flask_request(req):
    url_data = req.url.split("?")
    return {
        "https": "off",
        "http_host": req.host,
        "server_port": req.environ.get("SERVER_PORT", "5001"),
        "script_name": req.path,
        "get_data": req.args.copy(),
        "post_data": req.form.copy(),
    }

def init_saml_auth(req):
    return OneLogin_Saml2_Auth(prepare_flask_request(req), custom_base_path=SAML_PATH)

@app.route("/")
def index():
    user = session.get("saml_user")
    if user:
        return f"""<h2>✅ Sesión SAML activa</h2>
        <pre>{json.dumps(user, indent=2)}</pre>
        <a href='/saml/slo'>Single Logout (SLO)</a>"""
    return '<h2>App SAML Demo</h2><a href="/saml/login">Iniciar sesión con SAML</a>'

@app.route("/saml/login")
def saml_login():
    auth = init_saml_auth(request)
    return redirect(auth.login())

@app.route("/saml/acs", methods=["POST"])
def saml_acs():
    auth = init_saml_auth(request)
    auth.process_response()
    errors = auth.get_errors()
    if not errors:
        session["saml_user"] = {
            "name_id":    auth.get_nameid(),
            "attributes": auth.get_attributes(),
            "session_idx": auth.get_session_index(),
        }
        return redirect("/")
    return f"Error SAML: {', '.join(errors)}", 400

@app.route("/saml/metadata")
def saml_metadata():
    auth = init_saml_auth(request)
    settings = auth.get_settings()
    metadata = settings.get_sp_metadata()
    errors = settings.validate_metadata(metadata)
    if errors:
        return f"Metadata inválida: {', '.join(errors)}", 500
    return Response(metadata, content_type="text/xml")

@app.route("/saml/slo")
def saml_slo():
    auth = init_saml_auth(request)
    name_id = session.get("saml_user", {}).get("name_id")
    session_idx = session.get("saml_user", {}).get("session_idx")
    session.clear()
    return redirect(auth.logout(name_id=name_id, session_index=session_idx))

if __name__ == "__main__":
    app.run(host="0.0.0.0", port=5001, debug=True)
```

**5.2** Crea la configuración SAML del SP. Primero genera un certificado autofirmado para el SP:

```bash
mkdir -p ~/lab-04-sso/apps/saml-app/saml/certs
cd ~/lab-04-sso/apps/saml-app/saml/certs

openssl req -x509 -newkey rsa:2048 -keyout sp.key -out sp.crt \
  -days 365 -nodes \
  -subj "/C=US/ST=Lab/L=LabCity/O=CorpLab/CN=localhost"

# Verificar que se crearon los archivos
ls -la
```

**5.3** Crea el archivo de configuración SAML `~/lab-04-sso/apps/saml-app/saml/settings.json`:

> **Nota:** El certificado del IdP (Keycloak) se obtiene en el paso 5.5. Por ahora usa un placeholder.

```json
{
  "strict": false,
  "debug": true,
  "sp": {
    "entityId": "http://localhost:5001/saml/metadata",
    "assertionConsumerService": {
      "url": "http://localhost:5001/saml/acs",
      "binding": "urn:oasis:names:tc:SAML:2.0:bindings:HTTP-POST"
    },
    "singleLogoutService": {
      "url": "http://localhost:5001/saml/slo",
      "binding": "urn:oasis:names:tc:SAML:2.0:bindings:HTTP-Redirect"
    },
    "NameIDFormat": "urn:oasis:names:tc:SAML:1.1:nameid-format:emailAddress",
    "x509cert": "",
    "privateKey": ""
  },
  "idp": {
    "entityId": "http://localhost:8080/realms/corp-lab",
    "singleSignOnService": {
      "url": "http://localhost:8080/realms/corp-lab/protocol/saml",
      "binding": "urn:oasis:names:tc:SAML:2.0:bindings:HTTP-Redirect"
    },
    "singleLogoutService": {
      "url": "http://localhost:8080/realms/corp-lab/protocol/saml",
      "binding": "urn:oasis:names:tc:SAML:2.0:bindings:HTTP-Redirect"
    },
    "x509cert": "PLACEHOLDER_REEMPLAZAR_EN_PASO_5.5"
  }
}
```

**5.4** Crea el cliente SAML en Keycloak. Ve a **Clients** → **"Create client"**:

```
Client type:   SAML
Client ID:     http://localhost:5001/saml/metadata
```
Haz clic en **"Next"** y luego **"Save"**.

**5.5** Configura el cliente SAML. En la pestaña **Settings** del cliente recién creado:

```
Name:                    SAML Demo Client
Valid redirect URIs:     http://localhost:5001/*
Master SAML Processing URL: http://localhost:5001/saml/acs
IDP-Initiated SSO URL name: saml-demo
```

En la sección **SAML capabilities**:
```
Sign assertions:         ON
Sign documents:          ON
Signature algorithm:     RSA_SHA256
Name ID format:          email
Force name ID format:    ON
```

Haz clic en **"Save"**.

**5.6** Obtén el certificado del IdP (Keycloak) para el SP. Ve a **Realm settings** → **Keys** → pestaña **Active** → busca la clave de tipo **RSA** y haz clic en **"Certificate"**. Copia el valor del certificado (sin cabeceras PEM).

Actualiza `settings.json` reemplazando `PLACEHOLDER_REEMPLAZAR_EN_PASO_5.5` con el certificado copiado.

**5.7** Agrega el servicio SAML al `docker-compose.yml`:

```yaml
  saml-app:
    image: python:3.11-slim
    container_name: lab04-saml-app
    working_dir: /app
    volumes:
      - ./apps/saml-app:/app
    command: >
      sh -c "pip install flask python3-saml --quiet &&
             python app.py"
    ports:
      - "5001:5001"
    networks:
      - lab04-net
    depends_on:
      - keycloak
```

```bash
cd ~/lab-04-sso
docker compose up -d saml-app
docker compose logs -f saml-app
# Espera a: "Running on http://0.0.0.0:5001"
```

**5.8** Importa los metadatos del SP en Keycloak. Abre `http://localhost:5001/saml/metadata` en el navegador y copia el XML. En Keycloak, ve al cliente SAML → **Action** → **"Import from file"** (o crea un mapper manualmente).

Alternativamente, agrega un mapper de roles SAML. Ve al cliente SAML → **Client scopes** → `<client-id>-dedicated` → **Mappers** → **"Add mapper"** → **"Role list"**:
```
Name:           saml-role-list
Role attribute name: roles
Single Role Attribute: ON
```

**5.9** Prueba el flujo SAML:
1. Abre `http://localhost:5001` → haz clic en **"Iniciar sesión con SAML"**.
2. Keycloak detecta que ya existe una sesión activa para `ana.ficticia` (del Paso 3) y realiza SSO automático sin solicitar credenciales.
3. Si la sesión expiró, ingresa las credenciales de `ana.ficticia`.
4. Verifica que la app SAML muestra los atributos del usuario.

#### Salida Esperada

```
✅ Sesión SAML activa
{
  "name_id": "ana.ficticia@corp-lab.local",
  "attributes": {
    "roles": ["corp-analyst", "corp-user"],
    "email": ["ana.ficticia@corp-lab.local"],
    "firstName": ["Ana"],
    "lastName": ["Ficticia"]
  },
  "session_idx": "_xxxxxxxxxxxxxxxx"
}
```

#### Verificación

Con ambas aplicaciones abiertas en pestañas separadas del navegador:
- Verifica que **una sola autenticación** da acceso a ambas apps (SSO).
- Haz clic en **"Single Logout (SLO)"** en la app SAML y verifica que también cierra la sesión en la app OIDC (navega a `http://localhost:5000` — debe mostrar el enlace de login nuevamente).

---

### Paso 6: Configurar Mappers de Claims Personalizados y Políticas de Seguridad de Tokens

**Objetivo:** Extender los claims del token JWT con atributos personalizados del usuario y ajustar las políticas de seguridad de tokens.

#### Instrucciones

**6.1** Agrega un atributo personalizado al usuario `ana.ficticia`. En Keycloak, ve a **Users** → `ana.ficticia` → **Attributes** → **"Add attribute"**:

```
Key:    department
Value:  Cybersecurity
```
Haz clic en **"Save"**.

**6.2** Crea un mapper de atributo en el cliente OIDC. Ve a **Clients** → `oidc-web-client` → **Client scopes** → `oidc-web-client-dedicated` → **Mappers** → **"Add mapper"** → **"By configuration"** → **"User Attribute"**:

```
Name:                   department-mapper
User Attribute:         department
Token Claim Name:       department
Claim JSON Type:        String
Add to ID token:        ON
Add to access token:    ON
Add to userinfo:        ON
```

**6.3** Crea un scope personalizado de audiencia. Ve a **Client scopes** → **"Create client scope"**:
```
Name:     corp-internal-api
Type:     Optional
Protocol: OpenID Connect
```
Dentro del scope, ve a **Mappers** → **"Add mapper"** → **"Audience"**:
```
Name:                   corp-api-audience
Included Client Audience: oidc-web-client
Add to access token:    ON
```
Asigna el scope al cliente `oidc-web-client` como **Optional**.

**6.4** Obtén un nuevo token que incluya el scope opcional y verifica el claim `department`:

En Postman, realiza una nueva solicitud POST con scope ampliado:

```
grant_type:    password
client_id:     oidc-web-client
client_secret: <TU_CLIENT_SECRET>
username:      ana.ficticia
password:      CorpLab#Ana2024
scope:         openid email profile roles corp-internal-api
```

**6.5** Analiza el nuevo token en jwt.io. Decodifica el `access_token` y verifica:

```json
{
  "iss": "http://localhost:8080/realms/corp-lab",
  "sub": "<uuid>",
  "aud": ["account", "oidc-web-client"],
  "scope": "openid email profile roles corp-internal-api",
  "department": "Cybersecurity",
  "roles": ["corp-analyst", "corp-user"],
  "exp": <timestamp>,
  "iat": <timestamp>
}
```

**6.6** Configura la política de contraseñas del realm. Ve a **Authentication** → **Policies** → **"Password policy"** → **"Add policy"**:

```
Minimum Length:         12
Uppercase Characters:   1
Special Characters:     1
Not Username:           ON
Password History:       5
```
Haz clic en **"Save"**.

**6.7** Verifica el endpoint de introspección de tokens con Postman:

```
POST http://localhost:8080/realms/corp-lab/protocol/openid-connect/token/introspect

Body (x-www-form-urlencoded):
  client_id:     oidc-web-client
  client_secret: <TU_CLIENT_SECRET>
  token:         <ACCESS_TOKEN_DEL_PASO_6.4>
```

#### Salida Esperada

```json
{
  "exp": 1752086700,
  "iat": 1752086400,
  "jti": "<uuid>",
  "iss": "http://localhost:8080/realms/corp-lab",
  "aud": ["account", "oidc-web-client"],
  "sub": "<uuid>",
  "typ": "Bearer",
  "azp": "oidc-web-client",
  "scope": "openid email profile roles corp-internal-api",
  "department": "Cybersecurity",
  "active": true
}
```

#### Verificación

Confirma que:
1. El campo `"active": true` aparece en la respuesta de introspección.
2. El claim `"department": "Cybersecurity"` está presente en el payload del token.
3. La audiencia (`aud`) incluye `"oidc-web-client"` gracias al mapper de audiencia.

---

## 7. Validación y Pruebas Finales

Ejecuta el siguiente checklist completo para confirmar que el laboratorio fue completado exitosamente:

```bash
# ── VALIDACIÓN 1: Servicios en ejecución ──────────────────────────────────
docker compose ps --format "table {{.Name}}\t{{.Status}}\t{{.Ports}}"

# Esperado: lab04-postgres (healthy), lab04-keycloak (Up), 
#           lab04-oidc-app (Up), lab04-saml-app (Up)

# ── VALIDACIÓN 2: Discovery OIDC disponible ───────────────────────────────
curl -s http://localhost:8080/realms/corp-lab/.well-known/openid-configuration \
  | python3 -c "import sys,json; d=json.load(sys.stdin); \
    print('✅ Issuer:', d['issuer']); \
    print('✅ JWKS URI:', d['jwks_uri'])"

# ── VALIDACIÓN 3: Metadatos SAML del SP disponibles ───────────────────────
curl -s http://localhost:5001/saml/metadata | grep -c "EntityDescriptor"
# Esperado: 1

# ── VALIDACIÓN 4: Metadatos SAML del IdP (Keycloak) ──────────────────────
curl -s "http://localhost:8080/realms/corp-lab/protocol/saml/descriptor" \
  | grep -c "IDPSSODescriptor"
# Esperado: 1

# ── VALIDACIÓN 5: Obtención de token vía Postman/curl ────────────────────
KC_SECRET=$(docker exec lab04-keycloak /opt/keycloak/bin/kcadm.sh \
  get clients -r corp-lab --fields clientId,secret \
  2>/dev/null | python3 -c "
import sys,json
clients = json.load(sys.stdin)
for c in clients:
    if c.get('clientId') == 'oidc-web-client':
        print(c.get('secret',''))
" 2>/dev/null || echo "OBTENER_MANUALMENTE")

echo "Client Secret: $KC_SECRET"

# ── VALIDACIÓN 6: Verificar roles en el token ────────────────────────────
TOKEN=$(curl -s -X POST \
  "http://localhost:8080/realms/corp-lab/protocol/openid-connect/token" \
  -d "grant_type=password" \
  -d "client_id=oidc-web-client" \
  -d "client_secret=${KC_SECRET}" \
  -d "username=ana.ficticia" \
  -d "password=CorpLab#Ana2024" \
  -d "scope=openid email profile roles" \
  | python3 -c "import sys,json; print(json.load(sys.stdin)['access_token'])")

# Decodificar payload del JWT (sin verificar firma - solo para lab)
echo $TOKEN | cut -d. -f2 | \
  python3 -c "
import sys, base64, json
payload = sys.stdin.read().strip()
# Añadir padding si es necesario
payload += '=' * (4 - len(payload) % 4)
decoded = base64.urlsafe_b64decode(payload)
data = json.loads(decoded)
print('✅ Issuer:', data.get('iss'))
print('✅ Usuario:', data.get('preferred_username'))
print('✅ Roles:', data.get('roles', []))
print('✅ Scope:', data.get('scope'))
print('✅ Expira en (epoch):', data.get('exp'))
"
```

### Resumen de Resultados Esperados

| Prueba                                      | Resultado Esperado                              |
|---------------------------------------------|-------------------------------------------------|
| Todos los contenedores activos              | 4 servicios en estado `Up`                      |
| Discovery OIDC accesible                    | Issuer = `http://localhost:8080/realms/corp-lab` |
| Metadatos SAML del SP                       | XML válido con `EntityDescriptor`               |
| Token JWT con roles                         | `corp-analyst`, `corp-user` presentes           |
| Claim `department` en token                 | `"Cybersecurity"` presente                      |
| SSO entre app OIDC y app SAML              | Una sola autenticación, acceso a ambas apps     |
| SLO funcional                               | Logout en una app cierra sesión en ambas        |

---

## 8. Resolución de Problemas

### Problema 1: Keycloak no inicia — Error de conexión a PostgreSQL

**Síntomas:**
```
lab04-keycloak | ERROR: Failed to obtain JDBC Connection
lab04-keycloak | Caused by: org.postgresql.util.PSQLException: Connection refused
lab04-keycloak | Keycloak exiting with error
```
El contenedor de Keycloak entra en estado `Exited` o en bucle de reinicios.

**Causa:**
Keycloak intenta conectarse a PostgreSQL antes de que el healthcheck del contenedor `postgres` haya pasado. Esto ocurre si Docker Compose no respeta la condición `service_healthy`, si la imagen de PostgreSQL tarda más de lo esperado en inicializar, o si hay un conflicto de volúmenes de una ejecución anterior.

**Solución:**
```bash
# 1. Detener todos los contenedores del lab
cd ~/lab-04-sso
docker compose down

# 2. Verificar que PostgreSQL inicia correctamente de forma aislada
docker compose up -d postgres
sleep 15
docker compose ps postgres
# Debe mostrar: (healthy)

# 3. Verificar conectividad a la base de datos
docker exec lab04-postgres pg_isready -U keycloak -d keycloak
# Esperado: /var/run/postgresql:5432 - accepting connections

# 4. Si hay datos corruptos de una ejecución anterior, limpiar el volumen
docker compose down -v
docker volume rm lab04-sso_postgres_data 2>/dev/null || true

# 5. Reiniciar todo el stack
docker compose up -d
docker compose logs -f keycloak
```

---

### Problema 2: Error de firma SAML — "Invalid Signature" o respuesta SAML rechazada

**Síntomas:**
```
Error SAML: invalid_response, Signature validation failed. 
SAML Response rejected
```
La aplicación SAML en `http://localhost:5001` muestra el error al procesar la respuesta del IdP.

**Causa:**
El certificado X.509 del IdP (Keycloak) en el archivo `settings.json` del SP está desactualizado, tiene formato incorrecto (con o sin cabeceras PEM), o Keycloak rotó sus claves después de la configuración inicial. También puede ocurrir si el certificado fue copiado con caracteres de nueva línea o espacios adicionales.

**Solución:**
```bash
# 1. Obtener el certificado actualizado del IdP
curl -s "http://localhost:8080/realms/corp-lab/protocol/saml/descriptor" \
  | python3 -c "
import sys, re
xml = sys.stdin.read()
# Extraer el certificado X509Certificate
match = re.search(r'<ds:X509Certificate>(.*?)</ds:X509Certificate>', xml, re.DOTALL)
if match:
    cert = match.group(1).strip().replace('\n', '').replace(' ', '')
    print(cert)
else:
    print('No se encontró el certificado')
"

# 2. Actualizar settings.json con el nuevo certificado
# Edita ~/lab-04-sso/apps/saml-app/saml/settings.json
# Reemplaza el valor de "idp" -> "x509cert" con el certificado obtenido
# IMPORTANTE: NO incluir cabeceras -----BEGIN CERTIFICATE----- ni saltos de línea

# 3. Verificar que el modo strict está desactivado durante el lab
# En settings.json: "strict": false

# 4. Reiniciar el contenedor de la app SAML
docker compose restart saml-app
docker compose logs -f saml-app

# 5. Verificar los metadatos del SP
curl -s http://localhost:5001/saml/metadata | python3 -m json.tool 2>/dev/null || \
curl -s http://localhost:5001/saml/metadata | grep -c "EntityDescriptor"
```

---

## 9. Limpieza del Entorno

Ejecuta los siguientes comandos al finalizar el laboratorio para liberar recursos del sistema:

```bash
cd ~/lab-04-sso

# ── Detener y eliminar contenedores, redes y volúmenes ────────────────────
docker compose down -v

# ── Verificar que no quedan contenedores activos del lab ─────────────────
docker ps --filter "name=lab04" --format "table {{.Names}}\t{{.Status}}"
# Esperado: tabla vacía (no output)

# ── Eliminar imágenes descargadas (opcional — libera ~2 GB) ──────────────
docker rmi quay.io/keycloak/keycloak:23.0 postgres:15-alpine python:3.11-slim 2>/dev/null || true

# ── Limpiar archivos temporales del lab (opcional) ────────────────────────
# ADVERTENCIA: esto elimina todos los archivos del laboratorio
# Comenta esta línea si deseas conservar la configuración para revisión
# rm -rf ~/lab-04-sso

# ── Verificar espacio liberado ────────────────────────────────────────────
docker system df
```

> **Importante para Labs Posteriores:** Si planeas realizar el Lab 05-00-01 (OpenIAM), es **obligatorio** ejecutar `docker compose down -v` para liberar los puertos 8080 y 5432 que serán reutilizados. Considera hacer un snapshot de VM o exportar la configuración de Keycloak antes de limpiar si necesitas referenciarla.

```bash
# Exportar configuración del realm antes de limpiar (opcional pero recomendado)
docker exec lab04-keycloak /opt/keycloak/bin/kc.sh export \
  --dir /tmp/realm-export \
  --realm corp-lab \
  --users same_file 2>/dev/null

docker cp lab04-keycloak:/tmp/realm-export ~/lab-04-sso/corp-lab-backup.json 2>/dev/null || \
echo "Exportar manualmente desde Admin Console: Realm settings → Action → Export"
```

---

## 10. Resumen y Recursos Adicionales

### Resumen del Laboratorio

En este laboratorio aplicaste de forma práctica los tres protocolos comparados en la lección 4.1:

| Protocolo       | Lo que implementaste                                                    |
|-----------------|-------------------------------------------------------------------------|
| **OIDC/OAuth2** | Cliente `oidc-web-client` con Authorization Code + PKCE, inspección de `id_token` y `access_token` JWT, refresh token, discovery automático vía `.well-known` |
| **SAML 2.0**    | Cliente SAML con intercambio de metadatos XML, assertions con roles y atributos, flujo SSO y Single Logout (SLO) |
| **JWT**         | Análisis de claims (`iss`, `sub`, `aud`, `exp`, `roles`), verificación de firma con JWKS, mappers personalizados de claims, introspección vía RFC 7662 |

**Conceptos clave consolidados:**
- **PKCE** (`code_challenge_method=S256`) protege el flujo Authorization Code contra interceptación del código de autorización, esencial para clientes públicos y recomendado para confidenciales.
- El **discovery endpoint** (`/.well-known/openid-configuration`) y **JWKS** permiten que los Relying Parties se configuren dinámicamente, a diferencia de SAML que requiere intercambio manual de metadatos XML.
- Los **mappers de claims** en Keycloak permiten enriquecer los tokens con atributos de usuario (`department`), roles del realm y audiencias adicionales, controlando qué información se expone por scope.
- La **introspección de tokens** (RFC 7662) permite que Resource Servers validen tokens opacos o JWT sin necesidad de conocer la clave pública del IdP.
- **SSO y SLO** funcionan a través de la sesión central en Keycloak: cuando el usuario se autentica una vez, todos los clientes registrados en el realm se benefician del SSO; el SLO propaga el cierre de sesión a todos los SPs registrados.

### Recursos Adicionales

- [Keycloak 23 Documentation — Server Administration Guide](https://www.keycloak.org/docs/latest/server_admin/)
- [OpenID Connect Core 1.0](https://openid.net/specs/openid-connect-core-1_0.html)
- [OAuth 2.0 PKCE — RFC 7636](https://www.rfc-editor.org/rfc/rfc7636)
- [SAML 2.0 Web SSO Profile (OASIS)](https://docs.oasis-open.org/security/saml/v2.0/saml-profiles-2.0-os.pdf)
- [jwt.io — Herramienta de decodificación y verificación de JWT](https://jwt.io)
- [OAuth 2.0 Token Introspection — RFC 7662](https://www.rfc-editor.org/rfc/rfc7662)
- [python3-saml Library Documentation](https://github.com/SAML-Toolkits/python3-saml)
- [Authlib — Flask OIDC Integration](https://docs.authlib.org/en/latest/client/flask.html)

---
*Lab 04-00-01 · Módulo 4 · Curso de Servicios de Directorio e IAM Open Source*
