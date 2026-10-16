# Octágono – Crear Usuario con Área

> API REST con Bearer Token que crea un usuario de WordPress y lo asigna a un área de Octágono.

| Dato | Valor |
|---|---|
| Nombre del plugin | Octagono - Crear Usuario con Área |
| Slug / carpeta | `octagono-create-user-area` |
| Versión | 1.0 |
| Autor | Cristian |
| Requisitos | WordPress con REST API activa · PHP 8.0+ (usa `str_starts_with`) · tablas `{prefix}areas` y `{prefix}user_members` existentes |

---

## ¿Qué hace?

Expone un único endpoint REST que, en una sola llamada:

1. Verifica que el área exista en la tabla `{prefix}areas`.
2. Toma la **fecha de expiración** de esa área.
3. Crea un **usuario de WordPress** con el correo y la contraseña recibidos (rol `customer`).
4. Registra la **membresía** del usuario en esa área (tabla `{prefix}user_members`), activa y con vencimiento igual al del área.

Está pensado para que sistemas externos (formularios, automatizaciones, otros sitios del ecosistema WMS) den de alta usuarios en Octágono ya vinculados a su área, sin pasar por el panel de administración.

---

## Endpoint

```
POST /wp-json/octagono/v1/create-user-area
```

### Autenticación

Token fijo tipo **Bearer**, definido en la constante `OCTAGONO_API_TOKEN` dentro del archivo del plugin.

```
Authorization: Bearer <OCTAGONO_API_TOKEN>
Content-Type: application/json
```

| Situación | HTTP | Código |
|---|---|---|
| Falta el header `Authorization` o no empieza por `Bearer ` | 401 | `unauthorized` – *Missing Authorization header* |
| El token no coincide (comparación con `hash_equals`) | 403 | `unauthorized` – *Invalid API token* |

### Body (JSON)

| Campo | Tipo | Obligatorio | Descripción |
|---|---|---|---|
| `email` | string | Sí | Correo del nuevo usuario. Se usa como `user_login` y `user_email`. |
| `password` | string | Sí | Contraseña del usuario (WordPress la guarda con hash). |
| `area_id` | integer | Sí | ID del área en `{prefix}areas`. |

```json
{
  "email": "juan.perez@empresa.com",
  "password": "unaClave123",
  "area_id": 5
}
```

### Ejemplo con cURL

```bash
curl -X POST "https://octagono.wmslatam.com/wp-json/octagono/v1/create-user-area" \
  -H "Authorization: Bearer TU_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"email":"juan.perez@empresa.com","password":"unaClave123","area_id":5}'
```

---

## Flujo interno

```
Petición POST
   │
   ├─ ¿Token válido? ──────────────── no → 401 / 403
   ├─ ¿email, password y area_id? ─── no → 400
   ├─ ¿email con formato válido? ──── no → 400
   ├─ ¿Existe el área? ────────────── no → 404
   ├─ ¿Área con fecha_expiracion
   │   configurada y válida? ──────── no → 422
   ├─ ¿El correo ya existe? ───────── sí → 409
   ├─ Crear usuario (wp_insert_user) ─ error → 500
   ├─ Insertar membresía ──────────── error → 500 (usuario ya creado)
   └─ 201 Created
```

### Detalles de la creación del usuario

- **user_login / user_email:** el correo recibido (sanitizado con `sanitize_email`).
- **user_nicename:** parte del correo antes de la `@`, pasada por `sanitize_title`, más un guion y 5 letras minúsculas aleatorias (`random_int`).
  Ejemplo: `juan.perez@empresa.com` → `juanperez-xkqrt`.
- **Rol:** `customer`.

### Registro en `{prefix}user_members`

| Columna | Valor |
|---|---|
| `area_id` | El `area_id` recibido |
| `user_id` | ID del usuario recién creado |
| `status` | `ACTIVO` |
| `days` | `0` |
| `fecha_final_membresia` | `fecha_expiracion` del área, formateada como `dd-mm-yyyy` |

---

## Respuestas

### 201 – Éxito

```json
{
  "success": true,
  "message": "Usuario y membresía creados correctamente",
  "user_id": 123,
  "user_nicename": "juanperez-xkqrt",
  "area_id": 5,
  "status": "ACTIVO",
  "fecha_final_membresia": "30-06-2027"
}
```

### Errores

| HTTP | Mensaje | Causa |
|---|---|---|
| 400 | `Faltan datos requeridos: email, password, area_id` | Algún campo vacío o ausente |
| 400 | `El correo electrónico no es válido` | Formato de correo inválido |
| 404 | `El área especificada no existe` | `area_id` no está en `{prefix}areas` |
| 422 | `El área no tiene una fecha de expiración configurada` | `fecha_expiracion` vacía o `0000-00-00` |
| 422 | `La fecha de expiración del área no es válida` | La fecha no se puede interpretar |
| 409 | `Ya existe un usuario con ese correo electrónico` | `email_exists()` devuelve un usuario |
| 500 | `Error al crear el usuario: …` | `wp_insert_user` devolvió `WP_Error` |
| 500 | `El usuario se creó, pero ocurrió un error al registrar la membresía del área` | Falló el `INSERT`; incluye `user_id` |

Todas las respuestas de error tienen la forma `{ "success": false, "message": "..." }`.

---

## Funciones del plugin

| Función | Propósito |
|---|---|
| `octagono_api_check_token_area()` | `permission_callback`: valida el Bearer Token |
| `octagono_generar_letras_random( $length = 5 )` | Genera letras a–z aleatorias para el `user_nicename` |
| `octagono_api_create_user_area()` | Callback principal: valida, crea usuario e inserta membresía |

---

## Instalación

1. Subir la carpeta `octagono-create-user-area` a `wp-content/plugins/` (o el `.zip` desde *Plugins → Añadir nuevo*).
2. Activar el plugin.
3. Cambiar el valor de `OCTAGONO_API_TOKEN` por un token propio y secreto.
4. Probar el endpoint con el ejemplo de cURL.

---

## Consideraciones y mejoras sugeridas

- **Token en el código:** el token está escrito en el archivo PHP. Conviene moverlo a `wp-config.php` (`define('OCTAGONO_API_TOKEN', '...')` envuelto en `if ( ! defined(...) )`) o a una opción, y no subir el archivo a repositorios.
- **Sin rollback:** si falla el registro de la membresía, el usuario queda creado sin área. Se podría borrar con `wp_delete_user()` para dejar la operación atómica.
- **No valida cupos:** no comprueba si el área tiene cupos disponibles antes de agregar al usuario.
- **No verifica áreas vencidas:** acepta áreas cuya `fecha_expiracion` ya pasó.
- **Formato de fecha:** guarda `fecha_final_membresia` como texto `dd-mm-yyyy`; debe coincidir con lo que esperan los demás plugins de Octágono que leen esa columna.
- **Sin notificación:** no envía correo de bienvenida ni credenciales al usuario creado.
- **Sin límite de peticiones:** no hay rate limiting ni registro (log) de las llamadas.
