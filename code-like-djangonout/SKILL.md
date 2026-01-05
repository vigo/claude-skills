---
name: code-like-djangonout
description: ...
---

## General Coding Approach

All the naming, commenting must be in **English**. We are writing `python`
code, using `django` web framework, we must follow python and django
conventions.

Virtual environment must be activated, all the python packages should be
installed to related environment.

Try to detect projects python version:

```bash
python --version
# or look for .python-version
```

---

## Formatting and Linting the Code

`ruff` linter should be available and ready to use:

```bash
python -m pip install ruff 
```

Here is minimal python-ruff config (`.ruff.toml`)

```toml
line-length = 119
indent-width = 4
target-version = "py314"
exclude = [
    "**/migrations",
    "**/manage.py",
]


[format]
quote-style = "single"
exclude = [
    "**/manage.py",
]

[lint]
allowed-confusables = ["ı","’"]
mccabe.max-complexity = 15
ignore = [
    "ANN",
    "D203",   # one-blank-line-before-class
    "D213",   # multi-line-summary-second-line
    "ISC001",
    "COM812",
    "ERA",
    "D",
    "RUF012", # mutable-class-default
    "FBT002", # boolean-default-value-positional-argument, django internals
    "TD003",  # missing-todo-link
    "FIX002", # line-contains-todo
    "PT009",  # pytest-unittest-assertion
    "PT019",  # pytest-fixture-param-without-value (unittest uses mock params)
    "PT027",  # pytest-unittest-raises-assertion
    "PGH004", # file-wide noqa
    "INP001", 

    # "COM812", # missing-trailing-comma
    # "ISC001", # single-line-implicit-string-concatenation
    # "D",
]
select = ["ALL"]

[lint.flake8-quotes]
inline-quotes = "single"
docstring-quotes = "double"

[lint.pylint]
max-statements = 100
max-returns = 20
max-args = 10
max-positional-args = 8
max-branches = 20

[lint.isort]
known-first-party = [
    "core",  # name of the django-app
]
section-order = [
    "future",
    "standard-library",
    "django",
    "third-party",
    "first-party",
    "local-folder",
]
[lint.isort.sections]
"django" = ["django"]
```

Ask user to add `pylint` if possible. `pylint` handles static checks, catches
more errors than `ruff`. If `pylint` or `.pylintrc` doesn’t exists, you can
create `.pylintrc` with:

```bash
pylint --generate-rcfile > .pylintrc
```

If you need to `noqa`, ask first. Some noqas are OK:

- # noqa: S324
- # noqa: SLF001 (Model._meta)
- # noqa: ARG002 (overriding django’s internal class methods)

---

## Coding Style

Always use single quotes. Double quotes only allowed in `docstring` or some
strings you can not avoid: `"vigo's number"`. Some examples:

```python
user_name = 'vigo'
page = request.GET.get('page')
```

Don’t use magic values:

```python
# wrong
def foo(user):
    if user.age > 10:
        pass

# good
USER_MAX_AGE = 10

def foo(user):
    if user.age > USER_MAX_AGE:
        pass
```

Always get key from `dict` must use `.get`:

```python
FOO['bar']       # is BAD!
FOO.get('bar')   # is GOOD! use this
```

Descriptive variable names, and concise `docstrings` that explain API usage choices.

Follow PEP 8 naming conventions.

Detect the Python version used in the project, or ask the user if it cannot be
determined. Generate code using **features** and **improvements** available in
that Python version, and prefer version-specific best practices.

Since **Django does not yet fully rely on** type annotations or static type
checking, **do not use type annotations** in Python code. If you really need
to use annotations, do not forget to import:

```python
from __future__ import annotations
```

Never ever make **blind-exceptions**. `except Exception:` is wrong, handle
required exception.

Always add proper error handling when implementing some feature, service or
refactoring the code. If you implement `NotificationService`, you must have
an exception `NotificationServiceError`:

```python
# exceptions.py example
class <Project>Error(Exception):
    def __init__(self, message, humans=False, **extras):
        if humans:
            message = message.title()
        super().__init__(message)
        self.humans = humans
        self.message = message
        self.extras = extras

class NotificationServiceError(<Project>Error): ...


# services/notification.py
import logging

logger = logging.getLogger('<project-ket>.NotificationService')
service = NotificationService()
try:
    response = SomeOtherServiceRequest()
except SomeOtherServiceRequestError as exc:
    msg = 'error description' # or str(exc)
    logger.exception(msg)
    raise NotificationServiceError from exc
```

### Django Conventions

Django app structure should look like this:

    core/
        admin/
            __init__.py
            user.py
            :
            :
        fixture/
            core.user.json
            :
        forms/
            __init__.py
            user.py
            :
        management/
            __init__.py
            commands/
                create_foo.py
        migrations/
        models/
            __init__.py
            user.py
            :
        services/
            __init__.py
            notification_service.py
            :
        signals/
            __init__.py
            user.py
        tasks/
            __init__.py
            notification_task.py
        templates/
            auth/
                signin.html
            :
            :
        views/
            __init__.py
            auth/
                __init__.py
                login.py
                :
                :
        checks.py
        storage.py

Example <App>Config:

```python
# pylint: disable=W0611,C0415
# ruff: noqa: F401,PLC0415
from django.apps import AppConfig
from django.conf import settings


class CoreConfig(AppConfig):
    default_auto_field = 'django.db.models.BigAutoField'
    name = 'core'

    def ready(self):
        from .celery_signals import task_failure_handler
        from .tasks.interview_expiration_tasks import check_interview_expiration_task
        from .tasks.periodic_tasks import disable_completed_state_trackers_task

        if settings.DEBUG:
            from .checks import check_environment_variables, check_models
```

`checks.py`

```python
# pylint: disable=W0613
# ruff: noqa: ARG001,SLF001
import ast
import inspect
import os

import django.apps
from django.core import checks
from django.core.exceptions import FieldDoesNotExist

DEVELOPMENT_ENVIRONMENT_VARIABLES = [
    'DJANGO_SECRET_KEY',
    'DATABASE_URL',
    # other required environment variable name
]

FIELD_VERBOSE_NAME_WHITE_LIST = ['slug']
# MODEL_NAME_WHITE_LIST = [foomodel']


@checks.register()
def check_environment_variables(app_configs, **kwargs):
    errors = []
    for var_name in DEVELOPMENT_ENVIRONMENT_VARIABLES:
        if not os.environ.get(var_name):
            errors = [
                *errors,
                checks.Error(
                    f'Missing environment variable for development: {var_name}',
                    hint=f'Set the "{var_name}" environment variable in your environment.',
                    id='core.ENV001',
                ),
            ]
    return errors


def check_model_get_argument(node, arg):
    for kw in node.value.keywords:
        if kw.arg == arg:
            return kw
    return None


def check_model_is_gettext_node(node):
    if not isinstance(node, ast.Call):
        return False

    return node.func.id == '_'


def check_model_get_field(model, node):
    if not isinstance(node, ast.Assign):
        return None
    if len(node.targets) != 1:
        return None
    if not isinstance(node.targets[0], ast.Name):
        return None
    try:
        return model._meta.get_field(node.targets[0].id)
    except FieldDoesNotExist:
        return None


def check_model_fields_verbose_name(field, node):
    verbose_name = check_model_get_argument(node, 'verbose_name')
    if field.name not in FIELD_VERBOSE_NAME_WHITE_LIST:
        if verbose_name is None:
            yield checks.Warning(
                'Field has no verbose name',
                hint='Set verbose name on the field.',
                obj=field,
                id='BLT001',
            )
        elif not check_model_is_gettext_node(verbose_name.value):
            yield checks.Warning(
                'Verbose name should use gettext _() style',
                hint='Use gettext on the verbose name.',
                obj=field,
                id='BLT002',
            )


def check_model_class_meta(class_meta, model):
    if class_meta is None:
        yield checks.Warning(
            f'Model "{model._meta.model_name}" must define class Meta',
            hint=f'Add class Meta to model "{model._meta.model_name}".',
            obj=model,
            id='BLT003',
        )
    else:
        verbose_name = None
        verbose_name_plural = None

        for node in ast.iter_child_nodes(class_meta):
            if not isinstance(node, ast.Assign):
                continue

            if not isinstance(node.targets[0], ast.Name):
                continue

            attr = node.targets[0].id

            if attr == 'verbose_name':
                verbose_name = node

            if attr == 'verbose_name_plural':
                verbose_name_plural = node

        if verbose_name is None:
            yield checks.Warning(
                'Model has no verbose name',
                hint='Add verbose_name to class Meta.',
                obj=model,
                id='BLT004',
            )

        elif not check_model_is_gettext_node(verbose_name.value):
            yield checks.Warning(
                'Verbose name in class Meta should use gettext',
                hint=f'Use gettext on the verbose_name of class Meta "{model._meta.model_name}".',
                obj=model,
                id='BLT002',
            )

        if verbose_name_plural is None:
            yield checks.Warning(
                'Model has no verbose name plural',
                hint='Add verbose_name_plural to class Meta.',
                obj=model,
                id='BLT005',
            )

        elif not check_model_is_gettext_node(verbose_name_plural.value):
            yield checks.Warning(
                'Verbose name plural in class Meta should use gettext',
                hint=f'Use gettext on the verbose_name_plural of class Meta "{model._meta.model_name}".',
                obj=model,
                id='BLT002',
            )


def check_model(model):
    """

    BLT001: Field has no verbose name.
    BLT002: Verbose name should use gettext.
    BLT003: Model must define class Meta.
    BLT004: Model has no verbose name.
    BLT005: Model has no verbose name plural.

    """
    if model._meta.model_name not in MODEL_NAME_WHITE_LIST:
        model_source = inspect.getsource(model)
        model_node = ast.parse(model_source)
        class_meta = None
        for node in model_node.body[0].body:
            if isinstance(node, ast.ClassDef) and node.name == 'Meta':
                class_meta = node

            field = check_model_get_field(model, node)
            if field is None:
                continue

            yield from check_model_fields_verbose_name(field, node)

        yield from check_model_class_meta(class_meta, model)


@checks.register(checks.Tags.models)
def check_models(app_configs, **kwargs):
    errors = []
    for app in django.apps.apps.get_app_configs():
        if app.path.find('site-packages') > -1:
            continue

        for model in app.get_models():
            for check_message in check_model(model):
                errors = [*errors, check_message]
    return errors
```

Never use hardcoded un-localized texts; this is not ok: `return Http('Error')` 
always use `gettext`, for views, use:

`from django.utils.translation import gettext_lazy as _` and `return Http(_('Error'))`

for html, use: `translate` or `blocktranslate` tags.

Models should live in `<app>/models/<model>.py`. Check project’s django model
approach/style/structure, implement new models according to it.

Basic django model structure:

```python
from django.db import models
from django.contrib.auth import get_user_model
from django.utils.translation import gettext_lazy as _


class PostManager(models.Manager):
    def get_by_natural_key(self, author_nk, title):
        author = get_user_model().objects.get_by_natural_key(*author_nk)
        return self.get(author=author, title=title)


class Post(models.Model):
    created_at = models.DateTimeField(
        auto_now_add=True,
        verbose_name=_('Created At'),
    )
    updated_at = models.DateTimeField(
        auto_now=True,
        verbose_name=_('Updated At'),
    )
    title = models.CharField(
        max_length=255,
        verbose_name=_('Title'),
    )
    body = models.TextField(
        blank=True,
        verbose_name=_('Body'),
    )
    author = models.ForeignKey(
        to=get_user_model(),
        related_name='posts',
        related_query_name='post',
        on_delete=models.CASCADE,
        verbose_name=_('Author'),
    )

    objects = PostManager()

    class Meta:
        app_label = 'core'
        verbose_name = _('Post')
        verbose_name_plural = _('Posts')
        db_table = 'post'

    def __str__(self):
        return f'{self.title}'

    def natural_key(self):
        return (self.author.email, self.title)
    natural_key.dependencies = ['auth.user']
```

The project may use a `BaseModel` for Django models. If a `BaseModel` is
present, follow the existing example and inherit from it when creating new
models. Possible example:

```python
from django.conf import settings
from django.db import models
from django.urls import reverse
from django.utils.translation import gettext_lazy as _

from shared.models.base import BaseModel
from shared.models.managers import BaseModelManager


class PageManager(BaseModelManager):
    def get_by_natural_key(self, slug):
        return self.get(slug=slug)


def get_page_language_choices():
    return settings.LANGUAGES


class Page(BaseModel):
    ADMIN_INDEX_ORDER = 1000

    language = models.CharField(
        max_length=2,
        choices=get_page_language_choices,
        verbose_name=_('language'),
    )
    login_required = models.BooleanField(
        default=False,
        verbose_name=_('login required'),
    )
    title = models.CharField(
        max_length=255,
        verbose_name=_('title'),
    )
    slug = models.SlugField(
        max_length=255,
        unique=True,
    )
    content = models.TextField(
        verbose_name=_('content'),
    )
    enable_markdown = models.BooleanField(
        default=False,
        verbose_name=_('enable markdown'),
    )
    objects = PageManager()

    class Meta:
        app_label = 'core'
        verbose_name = _('Page')
        verbose_name_plural = _('Pages')
        db_table = 'page'

        constraints = [
            models.UniqueConstraint(
                fields=['language', 'slug'],
                name='uc_page_lang_slug',
            ),
        ]

    def __str__(self):
        return f'{self.title}'

    def natural_key(self):
        return (self.slug,)

    def get_absolute_url(self):
        return reverse('core:page-detail', kwargs={'language': self.language, 'slug': self.slug})
```

#### Model Rules:

- Must define model manager with `get_by_natural_key` method
- Never hard-code `choices`, use with callable: `choices=get_page_language_choices`
  `get_page_language_choices` should return list of tuples.

```python
class LanguageChoices(models.TextChoices):
    ENGLISH = 'en', _('english')
    TURKISH = 'tr', _('turkish')
```

- Enums are OK for choices:

```python
from enum import Enum, StrEnum

class CandidateInterviewStatuses(Enum):
    STARTED = 'started'
    IN_PROGRESS = 'in_progress'
    COMPLETED = 'completed'
    REFRESH_SKIPPED = 'refresh_skipped'
    INCOMPLETE = 'incomplete'

class CandidateTechSkillLevel(StrEnum):
    BEGINNER = 'BEGINNER'
    INTERMEDIATE = 'INTERMEDIATE'
    ADVANCED = 'ADVANCED'
    EXPERT = 'EXPERT'

```

- Model should use model manager such as `objects = PageManager()`
- Model must have `class Meta` with `app_label`, `db_table`, `verbose_name`
  `verbose_name_plural`. `constraints` and `indexes` are optional.
- Model must have `natural_key` method (must match with manager’s `get_by_natural_key`)
  `natural_key.dependencies` if it’s needed.
- Model meta constraint name should follow `uc_<short-idefentifier>_field`
- Model meta indexe nname should follow `idx_<short-idefentifier>_field`
- UniqueConstraint should work for non null fields:

```python
constraints = [
    models.UniqueConstraint(
        fields=['name', 'owner'],
        condition=Q(owner__isnull=False),
        name='uc_org_name_owner',
    ),
]
```

- Index non null fields:

```python
indexes = [
    models.Index(
        fields=['candidate_full_name'],
        name='idx_rsm_cf_name',
        condition=~models.Q(candidate_full_name=''),
    ),
]
```

- Relational fields must use `to='<model>'` style, `related_name`, `related_query_name`
  and `on_delete` kwargs:

```python
interview = models.ForeignKey(
    to='Interview',
    related_name='interview_questions',
    related_query_name='interview_question',
    on_delete=models.CASCADE,
    verbose_name=_('interview'),
)
```

- Upload related field should use `upload_to=CALLABLE`:

```python
from django.core.files.storage import FileSystemStorage

def dynamic_file_storage():    # possible to extend this in the future w/o migrating
    return FileSystemStorage()

def save_uploaded_file(instance, _filename, media_type=None, extension='webm'):
    ...

def save_video_mp4_file(instance, filename):
    return save_uploaded_file(instance, filename, 'video', 'mp4')

video_mp4_file = models.FileField(
    null=True,
    blank=True,
    max_length=255,
    verbose_name=_('video MP4 file'),
    upload_to=save_video_mp4_file,
    storage=dynamic_file_storage(),
)
```

Model method order is important;

1. field declarations
1. custom managers
1. `class Meta`
1. `__str__`
1. `save`
1. `natural_key`
1. `get_absolute_url`

All the models of this project follows these rules.

#### Model Admin Rules

- Model Admin files should live `<app>/admin/<model>.py`
- Check project’s admin approach/style/structure, implement new model admins 
  according to it.

The project may use a `BaseModelAdmin` for Django admins. If a `BaseModelAdmin` is
present, follow the existing example and inherit from it when creating new
admins. Possible example:

```python
from django.contrib import admin

from core.models import Candidate
from core.admin.base import BaseModelAdmin

@admin.register(Candidate)
class CandidateAdmin(BaseModelAdmin):
    ...
```

- `list_display`, `list_display_links`, `search_fields`, `ordering` are minimum 
  admin properties.
- If model has ForeignKey fields; use `autocomplete_fields` and `list_select_related`
  for admin query performance.

#### View Rules

- View files should live `<app>/views/`
- We only use `Class-Based-Views`, function based views are not allowed

Try to separate business logic in views, try to put the heavy work on
**service layer**. I don’t want to see lots of logic in view’s `get` or `post`
method if possible... Here is a good example from
`CandidateInterviewView.post`:

```python
def post(self, request, *_args, **_kwargs):
    form = self.get_form()
    if not form.is_valid():
        return self.form_invalid(form)

    candidate_interview_view_service = CandidateInterviewViewService(
        request=request,
        kwargs=self.kwargs,
        logger=self.logger,
        form=form,
    )
    return HttpResponseRedirect(candidate_interview_view_service.handle_post())
```

All the heavy work done in `CandidateInterviewViewService`...

### Celery Tasks

- Celery tasks live under `<app>/tasks/`
- `<APP>Config.ready` is the place for tasks and signals.

If you need to add retry mechanism, feel free to do it. If you have any
doubts, either consult with me or attempt to resolve the issue by reviewing
similar implementations in the project.

### Test Conventions

Tests should live under: `tests/`, general convention for test files:

    test_(controllers|forms|services|signals|views|tasks)_name.py
    test_<app>_(controllers|forms|services|signals|views)_name.py

Try to use stdlib and Django’s test suites.

---

## Pre-Commit Hooks

If pre-commit config file `.pre-commit-config.yaml` doesn’t exists, ask user to
install pre-commit:

```bash
brew install pre-commit
```

and install hooks:

```bash
pre-commit install
```

Use this minimum config:

```yaml
exclude: core/migrations/
fail_fast: true
repos:
  - repo: local
    hooks:
      - id: django-check
        name: "Run django checks"
        entry: scripts/pre-commit/django-check.bash
        language: script
        always_run: true
        pass_filenames: false

      - id: ruff
        name: "Run ruff linter checks"
        entry: ruff
        language: system
        types: [python]
        args: ["check", "."]

      - id: pylint
        name: "Run pylint check"
        entry: pylint
        language: system
        types: [python]
        args:
          [
            "-rn", # Only display messages
            "-sn", # Don't display the score
            "-d R0401",
            "config",
            "core",
          ]

      - id: django-test
        name: "Run tests"
        entry: scripts/pre-commit/run-tests.bash
        language: system
        types: [python]
        pass_filenames: false
```

`django-check.bash`:

```bash
#!/usr/bin/env bash
set -e
set -o pipefail
set -o errexit
set -o nounset

DJANGO_ENV=production python manage.py check --deploy || exit 0
```

`run-tests.bash`:

```bash
#!/usr/bin/env bash

set -e
set -o pipefail
set -o errexit
set -o nounset

coverage run manage.py test --failfast   # pip install coverage required
```

---

## Commit Message Guidelines

- Prefix: `[claude-opus]:` or `[claude-sonnet]:` based on model
- Short summary in lowercase (max 50 chars)
- Blank line, then bullet points with details
- Include Claude Code footer
- Use **present tense** in commit messages.
- Always start the message with a **lowercase letter**.
- Start with a **verb**, followed by a brief and clear description.

Examples:

- `fix login redirect issue`
- `implement user profile page`
- `remove unused dependencies`

**If related to a GitHub issue**:

- Add `Fixes #ISSUE-NUMBER` or `Closes #ISSUE-NUMBER` at the end of the commit
  message. This auto closes issue on GitHub.
- Include a direct link to the related GitHub issue.

Also this commit-template is helpful:

    #
    # 3456789x123456789x123456789x123456789x123456789x
    # Short description (subject) : 50 chars

    # 3456789x123456789x123456789x123456789x123456789x123456789x123456789x12
    # Long description : 72 chars
    #
    # - Why was this change necessary?
    # - How does it address the problem?
    # - Are there any side effects?
    #
    # Fixes #ticket
    # Closes #ticket, #ticket, #ticket
    #
    # Include a link to the ticket, if any.
    #

Example:

    [claude-opus]: add TXT and DOCX resume upload support
    
    - Convert TXT/DOCX files to PDF on upload using fpdf2 and python-docx
    - Skip text extraction for pre-extracted content
    - Add Unicode font support for Turkish characters
    
    🤖 Generated with [Claude Code](https://claude.com/claude-code)
    
    Co-Authored-By: Claude Opus 4.5 <noreply@anthropic.com>

---

## Resources

- https://docs.djangoproject.com/en/5.2/contents/
- https://docs.djangoproject.com/en/6.0/contents/
- https://docs.astral.sh/ruff/
- https://pylint.readthedocs.io/en/latest/
- https://github.com/astral-sh/ruff
- https://github.com/pylint-dev/pylint
