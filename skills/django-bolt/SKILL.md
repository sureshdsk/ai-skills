---
name: django-bolt
description: Use this skill when the user is building with Django Bolt (django-bolt package), mentions "runbolt", uses BoltAPI, or asks about django-bolt routing, serializers, auth, or guards. Django Bolt is NOT Django REST Framework and NOT Django Ninja — it has its own Rust-powered server, serializers, and patterns that differ significantly from standard Django.
version: 1.1.0
---

# Django Bolt Skill

Use this skill when building APIs with **Django Bolt** (`django-bolt`). Django Bolt is a Rust-powered API framework that sits on top of Django. It is NOT Django REST Framework and NOT Django Ninja — it has its own patterns, serializers, and server.

**Docs**: https://bolt.farhana.li | **GitHub**: https://github.com/dj-bolt/django-bolt

---

## What Django Bolt Is

Django Bolt adds a high-performance Rust HTTP layer (Actix Web + PyO3) to Django. Key differences from plain Django:

- **No Gunicorn/Uvicorn needed** — `runbolt` is the server
- **No DRF or Django Ninja** — Bolt has its own routing, serializers, auth, and middleware
- **Rust-processed auth/guards** — permission checks run before Python code executes
- **`msgspec.Struct`-based serializers** — not DRF `Serializer`, not Pydantic models
- **Async-first** — `async def` handlers are preferred; sync handlers run in a thread pool
- **Full Django compatibility** — ORM, Admin, signals, sessions, and packages all work

---

## Installation & Setup

```bash
pip install django-bolt
# or
uv add django-bolt
```

**`settings.py`**:
```python
INSTALLED_APPS = [
    ...
    "django_bolt",
]
```

**Run the server**:
```bash
python manage.py runbolt --dev   # development
python manage.py runbolt          # production
```

Do NOT use `runserver`, `gunicorn`, or `uvicorn` with Django Bolt.

---

## Routing

`BoltAPI` is a standalone ASGI app — it does **not** wire into Django's `urls.py`. `runbolt` autodiscovers all `BoltAPI` instances named `api` in each installed app's `api.py` file. Django's `urls.py` is only needed for Django admin.

**Standard pattern**: define `api` in `<app>/api.py`, import views at the bottom to register viewsets/routes:

```python
# <app>/api.py
from django_bolt import BoltAPI

api = BoltAPI(prefix="/api/users")

# Import views here to register their routes on this api instance
import myapp.views  # noqa: E402, F401
```

```python
# <app>/views.py  (or define routes directly in api.py for simple cases)
from django_bolt import ViewSet, Request
from .api import api

@api.viewset("/")
class UserViewSet(ViewSet):
    async def list(self, request: Request):
        return {"users": []}
```

For function-based routes, define them directly in `api.py`:

```python
# api.py
from django_bolt import BoltAPI

api = BoltAPI(prefix="/api")

@api.get("/users")
async def list_users():
    return {"users": []}

@api.post("/users", status_code=201)
async def create_user(data: UserSchema):
    ...

@api.get("/users/{user_id}")
async def get_user(user_id: int):
    return {"user_id": user_id}
```

Django's `urls.py` only needs admin (Bolt handles everything else):

```python
# finnlens/urls.py
from django.contrib import admin
from django.urls import path

urlpatterns = [
    path("admin/", admin.site.urls),
]
```

### BoltAPI Options

```python
api = BoltAPI(
    prefix="/api/v1",           # URL prefix for all routes
    trailing_slash="strip",     # "strip" (default), "append", or "keep"
    middleware=[...],           # global Bolt middleware
    enable_logging=True,
)
```

### HTTP Methods

```python
@api.get(path)
@api.post(path)
@api.put(path)
@api.patch(path)
@api.delete(path)
@api.head(path)
@api.options(path)
```

### Route Decorator Options

```python
@api.get(
    "/users/{user_id}",
    status_code=200,
    response_model=UserSchema,
    summary="Get user",
    description="Fetch a user by ID",
    tags=["users"],
    auth=[JWTAuth()],
    guards=[IsAuthenticated],
)
async def get_user(user_id: int): ...
```

### Mounting Sub-APIs

```python
v1 = BoltAPI(prefix="/v1")
v2 = BoltAPI(prefix="/v2")

main = BoltAPI()
main.mount("/v1", v1)
main.mount("/v2", v2)
```

---

## Serializers / Schemas

Django Bolt uses **`msgspec.Struct`** for request/response schemas. There is no `Serializer` class exported from `django_bolt`. Do NOT use DRF serializers or Pydantic models.

```python
import msgspec

class UserSchema(msgspec.Struct):
    id: int
    username: str
    email: str
    age: int = 0  # default value

# Use as request body type annotation:
@api.post("/users")
async def create_user(data: UserSchema):
    return data

# Use as response schema:
@api.get("/users/{id}", response_model=UserSchema)
async def get_user(id: int): ...
```

### Constructing and Returning Schemas

```python
# Construct directly
return UserSchema(id=user.pk, username=user.username, email=user.email)

# Or return a plain dict — Bolt serializes it automatically
return {"id": user.pk, "username": user.username}
```

---

## Request Handling

Function parameters are automatically resolved:

- **Path params**: match by name, typed automatically (`user_id: int`)
- **Query params**: any typed param not in the path is a query param
- **Request body**: annotate with a `Serializer` subclass
- **Request object**: annotate with `Request`

```python
from django_bolt import Request

@api.get("/users")
async def list_users(request: Request, page: int = 1, limit: int = 20):
    # page and limit are query params
    return {"page": page, "limit": limit}
```

### Explicit Parameter Markers

```python
from typing import Annotated
from django_bolt import UploadFile
from django_bolt.params import File, Header, Cookie

# File upload — use Annotated[UploadFile, File()] as parameter type
@api.post("/upload")
async def upload(file: Annotated[UploadFile, File()]):
    content = await file.read()   # bytes
    filename = file.filename
    ...

# Other markers
@api.post("/form")
async def form(
    token: str = Header("Authorization"),
    session: str = Cookie("sessionid"),
):
    ...
```

---

## Responses

```python
from django_bolt import JSON, HTML, PlainText, Redirect, File, FileResponse, StreamingResponse

# Auto JSON (return dict/list/Struct)
@api.get("/data")
async def data():
    return {"key": "value"}

# Explicit JSON with status/headers
@api.get("/data")
async def data():
    return JSON({"key": "value"}, status_code=200, headers={"X-Custom": "val"})

# HTML with Django templates
from django_bolt import render
@api.get("/page")
async def page(request: Request):
    return render(request, "template.html", {"key": "value"})

# Redirect
return Redirect("/new-path", status_code=302)

# File streaming (large files)
return FileResponse("/path/to/file.pdf")

# Streaming / SSE
from django_bolt import no_compress

@no_compress
@api.get("/stream")
async def stream():
    async def generate():
        for i in range(10):
            yield f"data: {i}\n\n"
    return StreamingResponse(generate())
```

### Cookie Management

```python
response = JSON({"ok": True})
response.set_cookie("session", "value", httponly=True, secure=True, samesite="Lax")
response.delete_cookie("old_cookie")
return response
```

---

## Authentication

Import auth classes from `django_bolt` top-level (not `django_bolt.auth`):

```python
from django_bolt import JWTAuthentication, APIKeyAuthentication, create_jwt_for_user
```

### JWT Auth

```python
from django_bolt import JWTAuthentication, create_jwt_for_user

# Generate token (pass a Django User instance)
token = create_jwt_for_user(user)

# Protect a route
@api.get("/me", auth=[JWTAuthentication()])
async def me(request: Request):
    ctx = request.context  # plain dict — no DB query, data from JWT claims
    return {"user_id": ctx["user_id"]}
```

Configure in `settings.py`:
```python
BOLT_JWT_SECRET = "your-secret-key"
BOLT_JWT_ALGORITHM = "HS256"  # HS256, RS256, ES256, etc.
BOLT_JWT_EXPIRATION = 3600  # seconds
```

### API Key Auth

```python
from django_bolt import APIKeyAuthentication

@api.get("/data", auth=[APIKeyAuthentication()])
async def data():
    ...
```
API key is read from `X-API-Key` header by default.

### `request.context` vs `request.user`

- `request.context` — plain **dict** of JWT claims, no DB hit (`ctx["user_id"]`, not `ctx.user_id`)
- `request.context["user_id"]` is always a **string** — cast to int before ORM lookups: `int(ctx["user_id"])`
- `request.user` — full Django `User` object, requires `await`, hits the DB

### Request attributes

| Attribute | Type | Notes |
|-----------|------|-------|
| `request.query` | dict | Query string params — use `.get("key", default)` |
| `request.body` | bytes | Raw request body |
| `request.headers` | dict | HTTP headers |
| `request.context` | dict | Auth context (JWT claims etc.) |
| `request.method` | str | HTTP method |
| `request.path` | str | Request path |
| `request.cookies` | dict | Cookies |
| `request.files` | dict | Uploaded files (multipart) |
| `request.session` | dict-like | Django session (requires SessionMiddleware) |
| `request.state` | dict | Middleware state |

### Sync Django auth functions in async handlers

`django.contrib.auth.authenticate` is sync — wrap it with `sync_to_async`:

```python
from asgiref.sync import sync_to_async
from django.contrib.auth import authenticate

user = await sync_to_async(authenticate)(username=data.username, password=data.password)
```

---

## Permissions (Guards)

Guards run before your handler. Import them from `django_bolt` top-level — there is no `django_bolt.guards` module.

```python
from django_bolt import IsAuthenticated, IsAdminUser, IsStaff, HasPermission

@api.get("/admin", guards=[IsAdminUser])
async def admin_panel(): ...

@api.get("/report", guards=[IsAuthenticated, HasPermission("app.view_report")])
async def report(): ...
```

Available guards:
- `IsAuthenticated` — valid token required (401 if not)
- `IsAdminUser` — superuser required (403 if not)
- `IsStaff` — staff status required
- `HasPermission("app.codename")` — specific Django permission
- `HasAnyPermission([...])` — OR logic
- `HasAllPermissions([...])` — AND logic
- `AllowAny` — explicit open access

Guards use JWT claims — permissions must be embedded in the token via `extra_claims` when using `create_jwt_for_user()`.

---

## Class-Based Views

### APIView

```python
from django_bolt import APIView, JWTAuthentication, IsAuthenticated

@api.view("/users/{user_id}")
class UserView(APIView):
    auth = [JWTAuthentication()]
    guards = [IsAuthenticated]

    async def get(self, request: Request, user_id: int):
        return {"user_id": user_id}

    async def delete(self, request: Request, user_id: int):
        ...
```

### ViewSet (auto CRUD routing)

```python
from django_bolt import ViewSet

@api.viewset("/users")
class UserViewSet(ViewSet):
    async def list(self, request):          # GET /users
        ...
    async def retrieve(self, request, pk):  # GET /users/{pk}
        ...
    async def create(self, request):        # POST /users
        ...
    async def update(self, request, pk):    # PUT /users/{pk}
        ...
    async def partial_update(self, request, pk):  # PATCH /users/{pk}
        ...
    async def destroy(self, request, pk):   # DELETE /users/{pk}
        ...
```

### ModelViewSet (ORM-integrated)

```python
from django_bolt import ModelViewSet
from .models import User
from .serializers import UserSerializer

@api.viewset("/users")
class UserViewSet(ModelViewSet):
    queryset = User.objects.all()
    serializer_class = UserSerializer

    async def get_queryset(self):
        return self.queryset.filter(is_active=True)
```

### Custom Actions on ViewSets

```python
from django_bolt import action

@api.viewset("/users")
class UserViewSet(ModelViewSet):
    queryset = User.objects.all()
    serializer_class = UserSerializer

    @action(methods=["post"], detail=True, path="activate")
    async def activate(self, request, pk):  # POST /users/{pk}/activate
        ...

    @action(methods=["get"], detail=False)
    async def stats(self, request):         # GET /users/stats
        ...
```

Note: `@action` only works with `@api.viewset()`, not `@api.view()`.

---

## Middleware

### Built-in Middleware

Available in `django_bolt.middleware`: `BaseMiddleware`, `Middleware`, `TimingMiddleware`, `LoggingMiddleware`, `ErrorHandlerMiddleware`, `CompressionConfig`.

There is **no `CORSMiddleware`** in `django_bolt.middleware`. Use `django-cors-headers` for CORS:

```bash
pip install django-cors-headers
```

```python
# settings.py
INSTALLED_APPS = [..., "corsheaders"]
MIDDLEWARE = [
    "corsheaders.middleware.CorsMiddleware",  # must be first
    ...
]
CORS_ALLOWED_ORIGINS = ["http://localhost:5173"]
```

```python
# Per-route rate limiting
from django_bolt import rate_limit

@rate_limit(rps=10, burst=20)
@api.post("/submit")
async def submit(): ...
```

```python
# Global compression
from django_bolt import CompressionConfig

api = BoltAPI(
    compression=CompressionConfig(algorithm="brotli", min_size=1024),
)
```

### Custom Middleware

```python
from django_bolt import middleware, Middleware

# Function-style (per-route decorator)
@middleware
async def my_middleware(request, call_next):
    # before
    response = await call_next(request)
    # after
    return response

# Class-style (global)
class MyMiddleware(Middleware):
    async def process_request(self, request):
        ...
```

### Django Middleware Integration

```python
from django_bolt import DjangoMiddleware, DjangoMiddlewareStack

# Single
api = BoltAPI(middleware=[DjangoMiddleware(SomeDjangoMiddleware)])

# Multiple (more efficient)
api = BoltAPI(middleware=[DjangoMiddlewareStack([Mid1, Mid2, Mid3])])
```

---

## Error Handling

```python
from django_bolt.exceptions import HTTPException, BadRequest, NotFound, Unauthorized, Forbidden

@api.get("/users/{id}")
async def get_user(id: int):
    user = await User.objects.filter(pk=id).afirst()
    if not user:
        raise NotFound(detail="User not found")
    return user

# Custom error with extra data
raise BadRequest(detail="Invalid input", extra={"field": "email", "issue": "already taken"})

# Full control
raise HTTPException(status_code=429, detail="Too many requests", headers={"Retry-After": "60"})
```

**Exception signature** — all arguments are keyword-only:
```python
# WRONG — positional string is interpreted as status_code, crashes with ValueError
raise NotFound("User not found")

# RIGHT — always use detail= keyword
raise NotFound(detail="User not found")
raise BadRequest(detail="bank_name is required")
raise Unauthorized(detail="Invalid token")
```

The full signature is:
```python
HTTPException(status_code=None, detail=None, headers=None, extra=None)
```

Validation errors (422) are raised automatically from serializer failures and include all field errors at once.

---

## Async Django ORM

Use Django's async ORM methods inside `async def` handlers — no extra setup needed:

```python
@api.get("/users")
async def list_users():
    users = await User.objects.filter(is_active=True).aall()
    return UserSerializer.dump_many(users)

@api.get("/users/{pk}")
async def get_user(pk: int):
    user = await User.objects.aget(pk=pk)  # raises DoesNotExist if not found
    return UserSerializer.from_model(user)

@api.post("/users")
async def create_user(data: UserSerializer):
    user = await User.objects.acreate(
        username=data.username,
        email=data.email,
    )
    return UserSerializer.from_model(user)
```

Avoid N+1: use `select_related` / `prefetch_related` before awaiting.

---

## WebSocket

```python
from django_bolt import WebSocket

@api.websocket("/ws/chat")
async def chat(websocket: WebSocket):
    await websocket.accept()
    async for message in websocket.iter_text():
        await websocket.send_text(f"Echo: {message}")
```

---

## OpenAPI / Docs

Swagger UI is auto-generated at `/docs`. Configure:

```python
from django_bolt import OpenAPIConfig

api = BoltAPI(
    openapi_config=OpenAPIConfig(
        title="My API",
        version="1.0.0",
        docs_url="/docs",       # Swagger UI
        redoc_url="/redoc",     # ReDoc
    )
)
```

---

## Common Mistakes to Avoid

| Wrong | Right |
|-------|-------|
| `python manage.py runserver` | `python manage.py runbolt` |
| Gunicorn/Uvicorn | `runbolt` |
| `from django_bolt import Serializer` | Use `msgspec.Struct` directly |
| Pydantic `BaseModel` for schemas | `msgspec.Struct` subclass |
| `from django_bolt.auth import JWTAuth` | `from django_bolt import JWTAuthentication` |
| `from django_bolt.guards import IsAuthenticated` | `from django_bolt import IsAuthenticated` |
| `from django_bolt.middleware import CORSMiddleware` | Use `django-cors-headers` package |
| `path("", api.urls)` in `urls.py` | No wiring needed — `runbolt` autodiscovers `api.py` |
| `django.http.JsonResponse` | Return dict or use `JSON()` |
| `django.http.HttpResponse` | Use `HTML()`, `PlainText()`, etc. |
| `@permission_required` decorator | `guards=[HasPermission(...)]` |
| Sync DB calls in async handlers | `await Model.objects.aget(...)` |
| Sync Django functions (e.g. `authenticate`) in async handlers | `await sync_to_async(fn)(...)` |
| `request.user` for token data | `request.context` (no DB, from JWT) |
| `request.query_params` | `request.query` |
| `ctx.user_id` on request.context | `ctx["user_id"]` — it's a plain dict |
| `User.objects.aget(pk=ctx["user_id"])` | `User.objects.aget(pk=int(ctx["user_id"]))` — user_id is always a string |
| `raise NotFound("message")` | `raise NotFound(detail="message")` — positional arg is `status_code` |
| `request.files["file"]` for uploads | `file: Annotated[UploadFile, File()]` as handler parameter — `request.files` is always `None` |
