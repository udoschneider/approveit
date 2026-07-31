# approveit

A small staffing-approval application: a requester asks for a specific person to be
assigned to a project, and that person's manager approves, rejects, or asks for more
information. The repository contains a Django 1.7 / Django REST Framework backend plus a
**pre-built** Cappuccino (Objective-J) single-page frontend checked in under `static/`.

Written in January 2015 as a university coursework project (see the `ABGABE_VERSION` tag
and the FH Brandenburg logout URL baked into the frontend bundle). It has not been touched
since; see [Status](#status).

## Domain model

Defined in `rest/models.py`:

- `Project` — `title`, `notes`.
- `UserProfile` — one-to-one with `django.contrib.auth.User`, adds a nullable `manager`
  foreign key to another `User` (reverse accessor `subordinates`). Created automatically
  by a `post_save` signal whenever a `User` is created.
- `PersonRequest` — `title`, `notes`, `project`, `requester` (User), `requestee` (User),
  and a `django-fsm` `status` field.

A DRF auth `Token` is also auto-created for every new user by a `post_save` signal.

### Request state machine

States: `pending` (initial), `approved`, `rejected`, `waiting`, `finished`.

| Transition | From | To | Allowed for |
| --- | --- | --- | --- |
| `approve` | `pending` | `approved` | the requestee's manager |
| `reject` | `pending` | `rejected` | the requestee's manager |
| `request` | `pending` | `waiting` | the requestee's manager |
| `provide` | `waiting` | `pending` | requester or requestee |
| `finish` | `approved` | `finished` | requester or requestee |
| `reopen` | `rejected`, `finished` | `pending` | requester or requestee |

The state graph itself is enforced by `django-fsm`; the "allowed for" column is enforced
by `PersonRequest.can_transition()` and checked in `rest/viewset_actions_mixin.py` before
a transition is executed over the API.

## HTTP API

Routes come from `approveit/urls.py` (DRF `DefaultRouter`) and `rest/views.py`.

| Path | Methods | Notes |
| --- | --- | --- |
| `/` | GET | Router API root |
| `/users/` , `/users/{pk}/` | full CRUD | `UserSerializer`; `manager` is writable, `subordinates` read-only, `password` write-only-ish (always rendered as a fixed placeholder hash) |
| `/users/current/` | GET | The authenticated user |
| `/projects/` , `/projects/{pk}/` | full CRUD | |
| `/requests/` , `/requests/{pk}/` | full CRUD | `status` is read-only; serializer also exposes `possible_actions` and `allowed_actions` for the calling user |
| `/requests/{pk}/{approve,reject,request,provide,finish,reopen}/` | POST | Transition endpoints, generated dynamically from the FSM transitions |
| `/api-token-auth/` | POST | DRF `obtain_auth_token` (username/password → token) |
| `/api-token-logout/` | GET | Stub — returns `{"response": "success"}` and does **not** delete the token |
| `/api-auth/…` | | DRF browsable-API login/logout |
| `/docs/` | GET | `django-rest-swagger` UI |
| `/admin/` | | Django admin; the `User` admin is extended with an inline listing of subordinates |

Authentication is `TokenAuthentication` + `SessionAuthentication`, and every viewset
requires an authenticated user (`AuthMixin` in `rest/views.py`). `BasicAuthentication` is
present but commented out.

## Frontend

`static/` is the **build output** of a separate Cappuccino application,
[approveitCapp](https://github.com/udoschneider/approveitCapp). The Objective-J sources are
*not* in this repository; the `Makefile` copies the compiled bundle in from a sibling
checkout:

```
make copy_capp     # cp -vfR ../approveitCapp/Build/Release/approveitCapp/* static/
```

To rebuild the frontend, clone `approveitCapp` next to this repository, build it there with
`jake release`, then run `make copy_capp`.

**The bundle committed here is newer than the `approveitCapp` sources.** This build contains
`LoginController` and `UserSessionManager` classes and reads an `AuthLoginURL` key from
`Info.plist`; none of that exists in the `approveitCapp` repository, whose app has no login
screen and authenticates via the Django session cookie instead. The sources for the login
work described below survive only as compiled output in this directory. Rebuilding from
`approveitCapp` will therefore give you a *different*, earlier application.

From the compiled bundle (`static/*.environment/approveitCapp.sj`) the app consists of
`AppController`, `LoginController`, `PasswordChangeController`, `UserSessionManager`,
`ProjectController`, `UserController`, and remote models `Project` / `Request` / `User`.
It bundles Ratatosk (`WLRemoteLink`, `WLRemoteObject`, plus its `OJTestCase` suite) for
REST object mapping against `/`, and zxcvbn for password-strength
feedback in the password-change panel. A `RequestGraphView` draws the state-machine
diagram with the current state highlighted.

Login posts to the `AuthLoginURL` from `static/Info.plist` (`/api-token-auth/`) and uses
the returned DRF token as an `Authorization` header for subsequent calls. `Info.plist`
also hardcodes `LogoutURL` = `http://osmi.fh-brandenburg.de/`, i.e. where the browser is
sent after logging out.

The app is served as static files at **`/static/index.html`** (`/` is the REST API root,
not the UI). Note that `approveit/urls.py` wires static files via
`staticfiles_urlpatterns()`, which only works while `DEBUG` is on.

## Project layout

First-party:

```
manage.py                     Django entry point
Makefile                      copy_capp / pip_freeze helpers
Procfile, runtime.txt         Heroku process + Python version
requirements.txt
approveit/settings.py         Django settings
approveit/urls.py             URL routing
approveit/wsgi.py
rest/models.py                Project, UserProfile, PersonRequest + FSM
rest/serializers.py           DRF serializers
rest/views.py                 viewsets
rest/viewset_actions_mixin.py generates POST endpoints from FSM transitions
rest/admin.py, rest/tests.py, rest/migrations/
static/index.html             Cappuccino bootstrap page (build artifact)
static/Info.plist             Cappuccino bundle config (build artifact)
static/Resources/             .cib nib files and icons (build artifacts)
static/*.environment/         compiled Objective-J bundle (build artifacts)
```

Vendored / third-party (~35 MB, do not edit):

```
static/Frameworks/            Cappuccino: Objective-J, Foundation, AppKit (+ Debug builds)
```

## Prerequisites

- Python 3.4 (`runtime.txt` pins `python-3.4.2`)
- The pinned dependencies in `requirements.txt`: Django 1.7.3, djangorestframework 3.0.3,
  django-fsm 2.2.0, django-rest-swagger 0.2.8, dj-database-url, psycopg2,
  django-postgrespool, SQLAlchemy, gunicorn, PyYAML

These versions are from 2014/2015 and will not install or run on a current Python. Expect
to need an old interpreter (or a container) to get this running at all.

## Running locally

```
pip install -r requirements.txt
python manage.py migrate
python manage.py createsuperuser
python manage.py runserver
```

Then open `http://127.0.0.1:8000/static/index.html` for the UI, `/` for the browsable API,
`/docs/` for Swagger, `/admin/` to create users and assign managers.

Run `python manage.py runserver` **from the repository root** — `STATICFILES_DIRS` is
computed relative to the current working directory, not to the project directory.

The default database is SQLite (`db.sqlite3` in the repo root). If `DATABASE_URL` is set,
settings switch to `dj_database_url` with the `django_postgrespool` engine.

There is no fixture or seed data. To exercise the workflow you need at least three users
(a requester, a requestee, and a manager), with the requestee's `UserProfile.manager` set
to the manager — do this in the Django admin or via `/users/`.

## Deployment

`Procfile` runs `gunicorn approveit.wsgi`, `runtime.txt` pins the Python version, and
`DATABASE_URL` handling is present — the project was set up for Heroku. Note that
`approveit/settings.py` hardcodes `DEBUG = True`, hardcodes a `SECRET_KEY`, leaves
`ALLOWED_HOSTS` empty, and defines no `STATIC_ROOT`, so as committed it is not safe or
correct for a production deployment.

## Tests

```
python manage.py test rest
```

Be aware that `rest/tests.py` asserts that model transitions raise `PermissionDenied` for
the wrong user (e.g. `assertRaises(PermissionDenied, self.request.approve, self.sales_rep)`).
The `@transition` decorators in `rest/models.py` carry no `permission=` argument, and
permission checking lives in the viewset mixin instead, so these tests do not match the
current model code.

## Status

Unmaintained. The last commit is 2015-01-25; tags are `0.1`, `0.2`, and `ABGABE_VERSION`
("submission version"). The only later activity on the remote is a set of automated
`snyk-fix-*` branches. Treat this as a 2015 prototype/coursework snapshot targeting an
obsolete toolchain (Django 1.7, Python 3.4, Cappuccino), not as usable software.

## License

The project itself ships no license file. The only `LICENSE` files in the tree belong to
the vendored Cappuccino frameworks under `static/Frameworks/` (LGPL 2.1). The Cappuccino
bundle metadata carries the copyright line "Copyright © 2015, Krodelin Software Solutions
All rights reserved."
