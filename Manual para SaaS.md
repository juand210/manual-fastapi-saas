
# Manual de Desarrollo: SaaS Multi-Tenant con FastAPI y PostgreSQL

Este documento es una guía paso a paso para construir una aplicación de software como servicio (SaaS) multi-tenant desde cero. El objetivo es que al final de este manual, no solo tengas una aplicación funcional, sino que entiendas profundamente los patrones de arquitectura que permiten que una sola aplicación sirva a múltiples clientes de forma segura y aislada.

## Capítulo 1: Arquitectura y Visión General

### 1.1. El Problema a Resolver

Las aplicaciones SaaS deben servir a múltiples clientes (tenants). El reto es: ¿cómo almacenamos los datos de cada cliente para que estén completamente aislados entre sí? Un cliente "Empresa A" no debe, bajo ninguna circunstancia, poder ver los datos de "Empresa B".

Existen varias estrategias:
1.  **Una Base de Datos por Tenant:** Muy seguro, pero muy costoso y difícil de mantener. Cada nuevo cliente requiere aprovisionar una nueva base de datos.
2.  **Una Tabla por Tenant:** Inseguro y complejo. El esquema de la base de datos se vuelve un caos.
3.  **Discriminación por Fila (Clave Foránea):** Todas los datos de todos los clientes en las mismas tablas, con una columna `tenant_id` en cada una. Es la estrategia más común, pero tiene riesgo de errores de programación que filtren datos si una consulta `WHERE` olvida filtrar por `tenant_id`.

### 1.2. Nuestra Solución: Un Único BBDD, Múltiples Esquemas

En este proyecto implementaremos una solución elegante y robusta que ofrece lo mejor de varios mundos, utilizando una característica poderosa de PostgreSQL: los **esquemas**.

La arquitectura es la siguiente:
- **Una única instancia de aplicación:** Desplegaremos una sola aplicación FastAPI.
- **Una única base de datos PostgreSQL:** No necesitamos gestionar múltiples bases de datos.
- **Un esquema `public`:** Contendrá la información compartida y, lo más importante, una tabla `tenants` que lista todos nuestros clientes y el nombre de su esquema.
- **Un esquema por cada tenant:** Cada vez que se registre un nuevo cliente (ej. `empresa_a`), crearemos dinámicamente un nuevo esquema en la base de datos (ej. `schema_empresa_a`). Todas las tablas con los datos de ese cliente (usuarios, pedidos, facturas) vivirán dentro de su propio esquema.

El "corazón" de nuestra aplicación será un middleware que, para cada petición entrante, identificará al cliente y ejecutará un comando SQL (`SET search_path TO schema_empresa_a, public`) antes de procesar la lógica de negocio. Esto hace que todas las consultas de esa transacción "vean" únicamente las tablas del cliente y las tablas públicas, logrando un aislamiento perfecto a nivel de base de datos.

**Ventajas:**
- **Aislamiento Fuerte:** Tan seguro como tener bases de datos separadas.
- **Bajo Costo:** Todos los clientes comparten los recursos de una sola base de datos.
- **Fácil Mantenimiento:** Una sola aplicación y una sola base de datos que mantener y respaldar.

---

## Capítulo 2: Configuración del Entorno de Desarrollo

Antes de escribir una sola línea de código, debemos preparar nuestro entorno.

### 2.1. Prerrequisitos
Asegúrate de tener instalados los siguientes programas:
- **Python:** Versión 3.9 o superior.
- **PostgreSQL:** Versión 12 o superior.
- **Un editor de código:** Se recomienda Visual Studio Code con la extensión de Python.
- **Un cliente de base de datos (Opcional):** DBeaver, pgAdmin o DataGrip son excelentes opciones para visualizar la base de datos.

### 2.2. Base de Datos Inicial
1.  Abre tu cliente de PostgreSQL.
2.  Crea una nueva base de datos. Para este manual, la llamaremos `pedidosweb`.
3.  Asegúrate de tener a mano el usuario, la contraseña, el host y el puerto.

### 2.3. Entorno Virtual y Dependencias
Vamos a crear una carpeta para el proyecto y un entorno virtual de Python para mantener las dependencias aisladas.

```bash
# 1. Crea la carpeta del proyecto
mkdir pedidos-actual
cd pedidos-actual

# 2. Crea un entorno virtual
python -m venv venv

# 3. Activa el entorno virtual
# En Windows:
venv\Scripts\activate
# En macOS/Linux:
source venv/bin/activate
```

Ahora, crea un archivo llamado `requirements.txt` en la raíz de tu proyecto y pega el siguiente contenido. Estas son todas las librerías de Python que necesitaremos.

**requirements.txt:**
```
fastapi==0.135.2
uvicorn[standard]
sqlalchemy==2.0.49
psycopg[binary]
python-dotenv
passlib[bcrypt]
Jinja2==3.1.6
email-validator
python-multipart
```

Instala todas las dependencias con un solo comando:
```bash
pip install -r requirements.txt
```

### 2.4. Archivo de Configuración
La aplicación necesita saber cómo conectarse a la base de datos. Para esto usamos variables de entorno, que gestionaremos con un archivo `.env`.

Crea un archivo llamado `.env` en la raíz del proyecto. **¡Importante!** Este archivo nunca debe subirse a un repositorio de código (Git), ya que contiene información sensible.

**.env:**
```
# Formato: postgresql+psycopg://<user>:<password>@<host>:<port>/<dbname>
DATABASE_URL="postgresql+psycopg://postgres:123456@127.0.0.1:5432/pedidosweb"
SESSION_SECRET="puedes-generar-una-cadena-aleatoria-aqui"
```
*Reemplaza los valores con tus propias credenciales de base de datos.*

¡Listo! Nuestro entorno está configurado. En el próximo capítulo, empezaremos a construir la base de nuestra aplicación.

---

## Capítulo 3: La Base de Datos y el Esquema `public`

Con el entorno listo, es hora de definir cómo se conectará nuestra aplicación a la base de datos y cuál será la estructura de nuestros datos.

### 3.1. El Orquestador de Sesiones (`database.py`)

SQLAlchemy es la librería ORM (Object-Relational Mapper) por excelencia en Python. Nos permite interactuar con la base de datos usando objetos de Python en lugar de escribir SQL crudo.

Crea un archivo `database.py` en la raíz del proyecto. Este archivo será el responsable de configurar la conexión.

**database.py:**
```python
import os
from sqlalchemy import create_engine
from sqlalchemy.orm import sessionmaker, declarative_base
from dotenv import load_dotenv

# Cargar variables desde archivo .env
load_dotenv()

# URL por defecto (para desarrollo local)
DEFAULT_DB_URL = "postgresql+psycopg://postgres:123456@127.0.0.1:5432/pedidosweb"
DATABASE_URL = os.getenv("DATABASE_URL", DEFAULT_DB_URL)

# Corrección para compatibilidad con servicios de hosting como Render
if DATABASE_URL.startswith("postgres://"):
    DATABASE_URL = DATABASE_URL.replace("postgres://", "postgresql+psycopg://", 1)

# El motor que gestiona la conexión a la base de datos
engine = create_engine(
    DATABASE_URL,
    pool_pre_ping=True  # Revalida conexiones antes de usarlas
)

# La fábrica de sesiones que usaremos para interactuar con la BBDD
SessionLocal = sessionmaker(
    autocommit=False,
    autoflush=False,
    bind=engine
)

# La clase base de la que heredarán todos nuestros modelos de datos
Base = declarative_base()
```

### 3.2. El Modelo `Tenant`: Nuestro Directorio Maestro

Nuestra primera tabla será `tenants`. Esta tabla es especial porque es la única que vivirá en el esquema `public`. Su propósito es actuar como un directorio central para saber qué clientes existen y cómo se llama el esquema de base de datos de cada uno.

**models/tenants_model.py:**
```python
from sqlalchemy import Column, Integer, String, Boolean
from database import Base

class Tenant(Base):
    __tablename__ = "tenants"
    __table_args__ = {"schema": "public"}

    id = Column(Integer, primary_key=True, index=True)
    code = Column(String(50), unique=True, nullable=False, index=True)
    name = Column(String(150), nullable=False)
    schema_name = Column(String(63), unique=True, nullable=False, index=True)
    is_active = Column(Boolean, nullable=False, default=True)
```

### 3.3. Contratos de Datos con Pydantic

Crea el archivo `schemas/tenants_schema.py`:

**schemas/tenants_schema.py:**
```python
from pydantic import BaseModel, ConfigDict

class TenantBase(BaseModel):
    code: str
    name: str
    schema_name: str
    is_active: bool = True

class TenantCreate(TenantBase):
    pass

class TenantResponse(TenantBase):
    id: int
    model_config = ConfigDict(from_attributes=True)
```

---

## Capítulo 4: El Corazón del Multi-Tenancy (Dependencias)

Llegamos a la pieza central de nuestra arquitectura: un sistema de **Inyección de Dependencias** que se ejecuta antes de nuestra lógica de negocio.

**deps.py:**
```python
import re
from fastapi import HTTPException, Request, Depends
from sqlalchemy import text
from sqlalchemy.orm import Session
from database import SessionLocal

TENANT_RE = re.compile(r"^[a-zA-Z0-9_]+$")

def get_tenant(cliente: str, request: Request):
    if not TENANT_RE.match(cliente):
        raise HTTPException(status_code=400, detail="Cliente inválido")
    db = SessionLocal()
    try:
        tenant = db.execute(text("SELECT schema_name FROM public.tenants WHERE code = :code AND is_active = true"), {"code": cliente}).scalar()
        if not tenant:
            raise HTTPException(status_code=404, detail="Cliente no existe o está inactivo")
        request.state.tenant_schema = tenant
        return tenant
    finally:
        db.close()

def get_db(request: Request, tenant=Depends(get_tenant)) -> Session:
    schema = request.state.tenant_schema
    db = SessionLocal()
    try:
        db.execute(text(f'SET search_path TO "{schema}", public'))
        yield db
    finally:
        db.close()
```
- **`get_tenant`**: Valida el cliente y guarda su `schema_name` en el estado de la petición.
- **`get_db`**: Usa el `schema_name` guardado para ejecutar `SET search_path`, aislando la sesión de base de datos antes de inyectarla en el endpoint.

---

## Capítulo 5: Gestión de Tenants (Onboarding de Clientes)

Para registrar nuevos clientes, usamos una potente estrategia de **clonación de un esquema plantilla** llamado `demo`.

**routers/tenant_router.py (extracto):**
```python
@router.post("/new")
async def create_tenant(db: Session = Depends(get_public_db), ...):
    # ... (validaciones)
    new_tenant = Tenant(code=code, name=name, schema_name=schema_name)
    db.add(new_tenant)
    db.commit()

    try:
        # 1. Crear el nuevo esquema
        db.execute(text(f'CREATE SCHEMA IF NOT EXISTS "{schema_name}"'))
        # 2. Clonar tablas desde el esquema 'demo'
        tables = db.execute(text("..."), {"schema": "demo"}).fetchall()
        for (table_name,) in tables:
            db.execute(text(f'CREATE TABLE "{schema_name}"."{table_name}" (LIKE "demo"."{table_name}" INCLUDING ALL)'))
        # 3. Copiar datos iniciales (seed data)
        db.execute(text(f'INSERT INTO "{schema_name}".permissions ... SELECT ... FROM demo.permissions'))
        # 4. Crear usuario admin por defecto para el nuevo tenant
        # ...
        db.commit()
    except Exception as e:
        db.rollback()
        # ... (código de limpieza)
        raise e
```
Este enfoque asegura que cada nuevo cliente comience con una estructura y datos base idénticos, todo dentro de una transacción atómica.

---

## Capítulo 6: Módulos de Negocio (CRUDs Tenant-Aware)

Ahora vemos los frutos de la arquitectura. Al construir un CRUD para "Items", el código es simple y limpio.

**routers/item_router.py (extracto):**
```python
router = APIRouter(prefix="/{cliente}/items")

@router.get("", response_model=list[ItemResponse])
def get_all_items_for_tenant(
    cliente: str,
    db: Session = Depends(get_db) # <-- ¡LA CLAVE!
):
    # No hay 'WHERE tenant_id'. ¡La dependencia `get_db` ya aisló la sesión!
    # SQLAlchemy genera 'SELECT * FROM items', pero Postgres lo ejecuta
    # en el esquema del cliente gracias al search_path.
    items = db.query(Item).order_by(Item.description).all()
    return items
```
La complejidad del multi-tenancy está totalmente abstraída de la lógica de negocio.

---

## Capítulo 7: Autenticación y Permisos (RBAC)

La autenticación sigue el mismo patrón: un usuario se autentica *contra* un tenant específico.

### 7.1. Modelo de Datos RBAC
Para un sistema de permisos basado en roles (RBAC) flexible, necesitamos varias tablas que se relacionan entre sí:
- `users`: Contiene los usuarios de un tenant.
- `roles`: Define los roles (ej. 'Administrador', 'Vendedor').
- `permissions`: Lista todas las acciones posibles (ej. 'crear_pedido', 'ver_dashboard').
- `user_roles`: Tabla intermedia que asigna usuarios a roles (Muchos a Muchos).
- `role_permissions`: Tabla intermedia que asigna permisos a roles (Muchos a Muchos).

### 7.2. El Flujo de Autenticación
El endpoint de login es el centro de este flujo.

**routers/auth_router.py (extracto):**
```python
@router.post("/login")
def login(
    request: Request,
    cliente: str,
    username: str = Form(...),
    password: str = Form(...),
    db: Session = Depends(get_db_for_cliente) # <-- Dependencia especial
):
    # 1. Busca el usuario en la BBDD del tenant
    user = db.query(UserModel.User).filter(UserModel.User.username == username).first()

    # 2. Verifica la contraseña hasheada con passlib
    if not user or not verify_password(password, user.password):
        # Error
        ...

    # 3. Si es válido, carga los permisos del usuario
    permissions = get_user_permissions(user.id, db)

    # 4. Guarda la información en la sesión del navegador
    request.session["user"] = { "id": user.id, "name": user.name, ... }
    request.session["tenant"] = cliente
    request.session["permissions"] = permissions # <-- ¡Lista de permisos guardada!

    return RedirectResponse(url=f"/{cliente}/home", status_code=303)

def get_user_permissions(user_id: int, db: Session) -> list[str]:
    # Realiza un JOIN a través de users -> user_roles -> roles -> role_permissions -> permissions
    permissions = (
        db.query(Permission.codename)
        .join(RolePermission, RolePermission.permission_id == Permission.id)
        .join(UserRole, UserRole.role_id == RolePermission.role_id)
        .filter(UserRole.user_id == user_id)
        .distinct()
        .all()
    )
    return [p.codename for p in permissions]
```
Cada vez que un usuario inicia sesión, el sistema valida sus credenciales contra su propio esquema y carga una lista de sus permisos en la sesión.

---

## Capítulo 8: Integración del Frontend con Jinja2

FastAPI puede servir tanto APIs como páginas HTML tradicionales. En este proyecto, usamos plantillas Jinja2 para el frontend.

### 8.1. Configuración de Plantillas
En los routers que devuelven HTML, se inicializa un objeto `Jinja2Templates`:
```python
from fastapi.templating import Jinja2Templates
templates = Jinja2Templates(directory="templates")
```

### 8.2. Renderizando una Página
Un endpoint que renderiza una página devuelve un `TemplateResponse`.

**routers/home_router.py (extracto):**
```python
@router.get("/home", response_class=HTMLResponse)
def home(request: Request, cliente: str, db: Session = Depends(get_db_for_cliente)):
    # ... (validar sesión)
    user = request.session.get("user")
    user_permissions = request.session.get("permissions", []) # Recuperamos permisos

    return templates.TemplateResponse(
        request=request,
        name="home.html", # El archivo a renderizar
        context={ # El diccionario de datos para la plantilla
            "request": request,
            "cliente": cliente,
            "user": user,
            "user_permissions": user_permissions,
        }
    )
```

### 8.3. Uso en la Plantilla HTML
Dentro del archivo `home.html` (o cualquier otra plantilla), podemos usar estos datos para personalizar la interfaz.

**templates/home.html (ejemplo):**
```html
<nav>
    <a href="/{{ cliente }}/items">Productos</a>
    
    {# Solo muestra este enlace si el usuario tiene el permiso #}
    {% if 'ver_reporte_ventas' in user_permissions %}
        <a href="/{{ cliente }}/reports">Reporte de Ventas</a>
    {% endif %}
</nav>

<h1>Bienvenido, {{ user.name }}!</h1>
```
Este mecanismo nos permite tener una interfaz dinámica y segura, donde las opciones del menú y los botones se muestran u ocultan según los permisos del usuario que ha iniciado sesión.

---

## Capítulo 9: Despliegue y Consideraciones Finales

Llevar nuestra aplicación a producción requiere unos últimos pasos clave.

### 9.1. Ejecución con un Servidor ASGI
En desarrollo usamos `uvicorn main:app --reload`. En producción, debemos usar un gestor de procesos como Gunicorn para lanzar varios trabajadores de Uvicorn y gestionar el servidor de forma robusta.
```bash
gunicorn -w 4 -k uvicorn.workers.UvicornWorker main:app
```
Este comando inicia 4 procesos trabajadores, permitiendo a la aplicación manejar múltiples peticiones en paralelo.

### 9.2. Variables de Entorno
Es **crítico** configurar las variables de entorno en el servidor de producción. Nunca se debe subir el archivo `.env` a producción.
- `DATABASE_URL`: La cadena de conexión a la base de datos de producción.
- `SESSION_SECRET`: Una cadena larga y aleatoria para firmar las cookies de sesión de forma segura.

### 9.3. Inicialización de la Base de Datos (Bootstrap)
Para una base de datos nueva en producción, debes realizar el siguiente proceso **una sola vez**:
1.  **Crear Tablas Públicas**: Conectar a la BBDD y ejecutar un script que cree las tablas del esquema `public`. La forma más sencilla es definir todos los modelos que pertenecen a `public` (como `Tenant`) y usar:
    ```python
    # script_inicial.py
    from database import engine, Base
    from models.tenants_model import Tenant # Importar todos los modelos públicos
    
    Base.metadata.create_all(bind=engine)
    ```
2.  **Crear y Poblar el Esquema `demo`**: Este es el paso más importante. Necesitas crear el esquema `demo` y llenarlo con la estructura y datos iniciales. Puedes adaptar el código del `tenant_router` para que funcione como un script independiente que cree el esquema `demo` y lo pueble.

### 9.4. Migraciones de Base de Datos
Para futuros cambios en la estructura de la base de datos (ej. añadir una columna a la tabla `items`), usar el método de clonación de esquema presenta un reto. La mejor solución a largo plazo es integrar una herramienta de migraciones como **Alembic**. Alembic puede gestionar el versionado de la base de datos y aplicar cambios de forma programática a todos los esquemas de los tenants existentes. La integración de Alembic en esta arquitectura es un tema avanzado que va más allá de esta guía inicial.

---