# Architecture
This file explains how the Open POS API is organised and how a request travels through the code.
The project is a Flask REST API for a point-of-sale system (products, orders, users, shops). The code is split into small modules, and every module follows the same 4 layers.
---
## 1. Folder structure
```text
open-pos-api-master/
├── manager.py            # starts the app, adds the flask CLI commands
├── .env                  # config values (db url, flask settings)
├── migrations/           # database migration files (flask db)
└── src/
    ├── __init__.py       # imports everything in the right order
    ├── config.py         # Dev / Test / Prod config classes
    │
    ├── utils/            # the base code that every module reuses
    │   ├── factory.py        # create_app()
    │   ├── blue_prints.py    # the blueprint, url prefix /api/v1
    │   ├── api.py            # @api.register(), BaseView, AssociationView
    │   ├── resource.py       # ModelResource, AssociationModelResource
    │   ├── models.py         # db and BaseMixin (id, created_on, updated_on)
    │   ├── schema.py         # ma and BaseSchema
    │   ├── security.py       # login / roles / user datastore
    │   ├── methods.py        # List, Fetch, Create, Update, Delete...
    │   ├── operators.py      # filter operators (Equal, Contains, In...)
    │   ├── exceptions.py     # custom errors
    │   └── admin.py          # the /admin panel object
    │
    ├── products/         # products, brands, stock, tax...
    ├── orders/           # orders, items, discounts...
    ├── user/             # users, roles, shops, customers...
    │      each of them has: models.py  schemas.py  resources.py  views.py
    │
    └── admin_panel/      # admin_manager.py - registers the tables in /admin
```
---
## 2. The 4 layers of every module
`products`, `orders` and `user` all have the same 4 files:
| File | Layer | What it does |
|---|---|---|
| `models.py` | Model | The database tables (SQLAlchemy) |
| `schemas.py` | Schema | Converts JSON to python objects and back (marshmallow). Also validates the data |
| `resources.py` | Resource | The rules: which filters are allowed, which roles can use it, who can read / change / delete |
| `views.py` | View | Connects a URL to a resource using `@api.register()` |
---
## 3. How the app starts

1. `manager.py` loads `.env`, then does `from src import ...`.
2. That runs `src/__init__.py`, which imports all the models, schemas and views. While the views are imported, `@api.register()` runs and adds all the routes to the blueprint.
3. `create_app()` (in `factory.py`) makes the Flask app, loads the config, enables CORS, registers the blueprint and calls `init_app` on the extensions (api, db, ma, security, admin).
4. `flask run` starts the server.
---
## 4. Request flow
```text
HTTP request
   -> Blueprint (/api/v1)
   -> flask_restful Resource (a View class)
   -> auth_token_required
   -> roles_accepted / roles_required
   -> handler (get / post / put / patch / delete)
   -> marshmallow schema (validate / load / dump)
   -> SQLAlchemy (database)
   -> JSON response
```
Example: `GET /api/v1/product?__limit=10`

1. **Blueprint.** The blueprint `pos` has `url_prefix='/api/v1'`, so the url is matched here. `@api.register()` made the route `/product` from the model name.
2. **View.** Flask creates `ProductView`. In `BaseView.__init__` it creates the `ProductResource` and adds the decorators.
3. **Auth.** If the resource has `auth_required = True`, three decorators are added: `roles_required`, `roles_accepted`, `auth_token_required`. The last one added runs first, so the order is token check -> roles_accepted -> roles_required. A wrong token gives 401, a wrong role gives 403.
4. **Handler.** `get()` in `BaseView` runs. It reads the query string (`__limit`, `__page`, `__order_by`, `__only`, `__exclude`, `__include` and filters like `name__contains`) and builds the query through the resource.
5. **Permission.** `has_read_permission(qs)` in the resource limits the query, for example only the data of the user's own brand.
6. **Database.** SQLAlchemy runs the query with pagination.
7. **Schema.** `schema.dump(...)` turns the rows into JSON-friendly dicts.
8. **Response.** The view returns it with `jsonify`, for example `{"success": true, "data": [...], "total": 10}`.

For `POST` / `PUT` / `PATCH` it is the same, but the schema is used the other way: `schema.load(request.json)` validates the data and makes the model objects. If there are errors it returns 400. Then `has_add_permission` / `has_change_permission` is checked, and then `db.session.commit()`. A database error is rolled back and returned as a JSON error.
Login, logout and register are not written by us. They come from Flask-Security and are on `/api/v1/login/`, `/api/v1/register/` etc. (set in `config.py`).
---
## 7. Role of each file in `src/utils/`

| File | Role |
|---|---|
| `factory.py` | `create_app()` builds the Flask app, loads config, CORS, blueprint and extensions |
| `blue_prints.py` | Makes the blueprint `pos` with prefix `/api/v1` |
| `api.py` | `ApiFactory.register()` decorator makes the urls. `BaseView` and `AssociationView` have the get/post/put/patch/delete code |
| `resource.py` | The two resource base classes (see section 5) |
| `methods.py` | The list of allowed methods for a view (`List`, `Fetch`, `Create`, `Update`, `Delete`, `BulkUpdate`). A view can limit this with `api_methods = [List]` |
| `operators.py` | The filter operators used in `filters = {...}` of a resource |
| `models.py` | `db` and `BaseMixin` (uuid `id`, `created_on`, `updated_on`, table name from class name) |
| `schema.py` | `ma` and `BaseSchema` (attaches the db session to marshmallow) |
| `security.py` | The user datastore and the Flask-Security setup |
| `exceptions.py` | Custom errors that the views catch and turn into JSON |
| `admin.py` | The Flask-Admin object. `/admin` redirects to login if you are not logged in |

---

## 8. Adding a new feature
For example a new `Supplier` table:
1. Add `class Supplier(BaseMixin, db.Model)` in `models.py`.
2. Add `SupplierSchema` in `schemas.py`.
3. Add `SupplierResource(ModelResource)` in `resources.py`. Set `model`, `schema`, `filters`, `auth_required`, `roles_accepted` and the permission methods.
4. Add `SupplierView(BaseView)` in `views.py` with `@api.register()` and `get_resource()`.
5. Run `flask db migrate -m "add supplier"` and then `flask db upgrade`.

After this `/api/v1/supplier` will work with list, get, create, update and delete.
