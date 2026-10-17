# Campus API segura

API educativa desarrollada con FastAPI para practicar registro de usuarios, autenticación con OAuth2 y protección de rutas mediante tokens JWT.

## Tecnologías

- **FastAPI**: endpoints, validación y documentación interactiva.
- **Pydantic**: validación de los datos de registro.
- **PyJWT**: creación y validación de tokens JWT con HS256.
- **pwdlib con Argon2**: hashing y verificación de contraseñas.
- **python-dotenv**: carga de configuración desde `.env`.

## Estructura

```text
sesion 1/
├── main.py           # Aplicación, modelos y endpoints
├── requirements.txt  # Dependencias de Python
├── .env.example      # Ejemplo de configuración
├── .gitignore
└── README.md
```

El archivo `.env` y el entorno virtual `venv/` se crean localmente.

## Instalación y configuración

Requisitos: Python 3.10 o superior y pip. Los siguientes comandos están preparados para PowerShell en Windows.

### 1. Abrir el proyecto y preparar el entorno

```powershell
cd "C:\Users\crist\Documents\Campus\FastAPI\Ultima sesion\sesion 1"
python -m venv venv
.\venv\Scripts\Activate.ps1
python -m pip install -r requirements.txt
```

Si ya existe el entorno virtual, basta con activarlo e instalar las dependencias.

### 2. Configurar la clave JWT

Copia el archivo de ejemplo y genera una clave aleatoria:

```powershell
Copy-Item .env.example .env
python -c "import secrets; print(secrets.token_hex(32))"
```

Reemplaza el valor de ejemplo en `.env` por la clave generada:

```dotenv
JWT_SECRET_KEY=REEMPLAZAR_POR_LA_CLAVE_GENERADA
```

Conserva esta clave en privado. La aplicación no inicia si `JWT_SECRET_KEY` no está definida o está vacía. Al cambiarla, los tokens anteriores dejan de ser válidos.

### 3. Iniciar el servidor

Desde la carpeta del proyecto, con el entorno activado:

```powershell
python -m uvicorn main:app --reload
```

- API: <http://127.0.0.1:8000>
- Swagger UI: <http://127.0.0.1:8000/docs>
- ReDoc: <http://127.0.0.1:8000/redoc>
- Esquema OpenAPI: <http://127.0.0.1:8000/openapi.json>

`--reload` reinicia el servidor al modificar el código y se utiliza durante el desarrollo. No hay un endpoint definido para `/`; utiliza `/publico` o `/docs`.

## Endpoints

| Método | Ruta | Autenticación | Descripción |
| --- | --- | --- | --- |
| POST | `/usuarios/registro` | No | Registra un usuario con contraseña hasheada. |
| POST | `/token` | No | Valida las credenciales y entrega un token de acceso. |
| GET | `/publico` | No | Devuelve un mensaje de bienvenida. |
| GET | `/perfil` | Bearer JWT | Devuelve el nombre del usuario autenticado. |
| GET | `/cursos` | Bearer JWT | Devuelve el usuario y los cursos Python, Java y FastAPI. |

### Registro

`POST /usuarios/registro` recibe JSON:

```json
{
  "username": "estudiante",
  "password": "ClaveDemo123!"
}
```

El nombre de usuario debe tener entre **3 y 50 caracteres** y la contraseña un mínimo de **8 caracteres**. El registro exitoso devuelve **201**; un nombre ya registrado devuelve **409**; los datos que incumplen la validación producen **422**.

### Inicio de sesión

`POST /token` recibe `username` y `password` como formulario `application/x-www-form-urlencoded`, no como JSON.

Respuesta exitosa:

```json
{
  "access_token": "<token_jwt>",
  "token_type": "bearer"
}
```

El token contiene el usuario en `sub` y la expiración en `exp`. Su vigencia es de **60 minutos**. Las credenciales incorrectas producen **401**.

### Rutas protegidas

Envía el token en el encabezado:

```http
Authorization: Bearer <token_jwt>
```

Las rutas protegidas rechazan peticiones sin autenticación válida. Los tokens inválidos o expirados y los usuarios que ya no existen en memoria producen **401**.

## Ejemplo completo en PowerShell

Con el servidor en ejecución, abre otra terminal y ejecuta:

```powershell
$baseUrl = "http://127.0.0.1:8000"

# Consultar una ruta pública.
Invoke-RestMethod -Uri "$baseUrl/publico"

# Registrar un usuario.
$registro = @{
    username = "estudiante"
    password = "ClaveDemo123!"
} | ConvertTo-Json

Invoke-RestMethod -Uri "$baseUrl/usuarios/registro" `
    -Method Post -ContentType "application/json" -Body $registro

# Solicitar el token mediante un formulario.
$sesion = Invoke-RestMethod -Uri "$baseUrl/token" `
    -Method Post -ContentType "application/x-www-form-urlencoded" `
    -Body @{ username = "estudiante"; password = "ClaveDemo123!" }

# Usar el token para consultar las rutas protegidas.
$cabeceras = @{ Authorization = "Bearer $($sesion.access_token)" }
Invoke-RestMethod -Uri "$baseUrl/perfil" -Headers $cabeceras
Invoke-RestMethod -Uri "$baseUrl/cursos" -Headers $cabeceras
```

Si repites el registro con el mismo nombre mientras el servidor conserva sus datos, recibirás **409**.

También puedes probar el flujo en Swagger UI: registra un usuario, pulsa **Authorize**, introduce sus credenciales y ejecuta `/perfil` o `/cursos`.

## Limitaciones actuales

- **Almacenamiento en memoria:** `usuarios_db` es un diccionario; todos los usuarios se pierden al reiniciar el proceso o al recargar el servidor. No hay base de datos ni persistencia. Los procesos independientes tampoco comparten estos datos.
- **Respuesta de registro:** actualmente incluye un campo `password` que devuelve el diccionario completo de usuarios con sus hashes de contraseña. Aunque no contiene las contraseñas en texto plano, esta información no debe exponerse; debe retirarse de la respuesta antes de utilizar la aplicación con datos reales.
- **Sesiones:** no hay endpoints de renovación, cierre de sesión ni revocación de tokens.
- **Dependencias:** `requirements.txt` no fija versiones concretas.
- **Pruebas:** el proyecto no incluye una suite de pruebas automatizadas.

## Problemas frecuentes

| Situación | Qué revisar |
| --- | --- |
| Error al iniciar por falta de clave JWT | Crear `.env` y establecer `JWT_SECRET_KEY` con un valor no vacío. |
| Error 422 al solicitar un token | Enviar las credenciales como formulario, no como JSON. |
| Error 401 en una ruta protegida | Usar un token vigente en `Authorization: Bearer <token>` y verificar que el usuario siga registrado. |
| Los usuarios desaparecen | El almacenamiento es temporal; es necesario registrarlos de nuevo después de reiniciar. |
| Error 404 en `/` | Abrir `/publico` o `/docs`; no existe una ruta raíz. |
