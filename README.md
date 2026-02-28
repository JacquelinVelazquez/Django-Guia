# Guía (Django + MariaDB con Laragon)

## Requisitos técnicos (lo mínimo para que funcione)

- Python instalado y pip  
- Django instalado dentro de un entorno virtual (venv)  
- Laragon levantando BD (MariaDB 11.8.6 por puerto 3306)  
- Editor (VS Code)
## Estructura:
sunlin/
├─ manage.py
├─ sunlin/
│  ├─ __init__.py
│  ├─ settings.py
│  ├─ urls.py
│  ├─ asgi.py
│  └─ wsgi.py
└─ members/
   ├─ migrations/
   │  ├─ 0001_initial.py
   │  ├─ 0002_...py
   │  └─ __init__.py
   ├─ templates/
   │  └─ members/
   │     ├─ article_list.html
   │     ├─ article_detail.html
   │     ├─ new_article_form.html
   │     └─ edit_article_form.html
   ├─ admin.py
   ├─ apps.py
   ├─ forms.py
   ├─ models.py
   ├─ urls.py
   ├─ views.py
   └─ tests.py
   
## 0) Prender MySQL/MariaDB (Laragon)
- Abrir Laragon
- Click Iniciar Todo
- Verifica que MySQL/MariaDB esté en verde y el puerto sea 3306
- Abre HeidiSQL y confirma:
  - SELECT VERSION();
      - sale: 11.x.x-MariaDB

## 1) Crear carpeta del proyecto
```python
mkdir Joe_Class_Django
cd Joe_Class_Django
```
## 2) Crear y activar entorno virtual (env)
```python
python -m venv env
.\env\Scripts\Activate.ps1
```
## 3) Instalar Django + driver para MariaDB
```python
pip install django
pip install mysqlclient
```
## 4) Crear el proyecto Django (sunlin)
```python
django-admin startproject sunlin
cd sunlin
```
- Prueba que corre:
  ```python
  python manage.py runserver
  ```
  y abre: http://127.0.0.1:8000/

## 5) Crear la base de datos (sunlin_database)
En HeidiSQL (Laragon > Base de Datos):
```sql
CREATE DATABASE sunlin_database;
```

## 6) Conectar Django a MariaDB (settings.py)
Abre: sunlin/sunlin/settings.py
Busca DATABASES y deja esto:
```python
DATABASES = {
  "default": {
    "ENGINE": "django.db.backends.mysql",
    "NAME": "sunlin_database",
    "USER": "root",
    "PASSWORD": "",
    "HOST": "127.0.0.1",
    "PORT": "3306",
  }
}
```

## 7) Aplicar migraciones base (tablas de Django)
```python
python manage.py migrate
```
Y en HeidiSQL ya deberías ver tablas como: auth_user, django_migrations, etc...

## 8) Crear una app (members)
```python
python manage.py startapp members
```
## 9) Activar la app (INSTALLED_APPS)
En sunlin/sunlin/settings.py agrega:
```python
INSTALLED_APPS = [
  # ...
  "members",
]
```
## 10) Crear la entidad (modelo) Article
En members/models.py:
```python
from django.db import models
class Article(models.Model):
    name = models.CharField(max_length=250, default="Sin nombre", null=True, blank=True)
    content = models.TextField(default="", null=True, blank=True)
    def __str__(self):
        return self.name
```

## 11) Crear tabla en la BD (migrations)
```python
python manage.py makemigrations
python manage.py migrate
```
Y en HeidiSQL debe aparecer: members_article

## 12) Crear formularios (forms.py) para “Nuevo” y “Editar”
Crea/edita: members/forms.py
```python
from django import forms
from .models import Article
class ArticleForm(forms.ModelForm):
    class Meta:
        model = Article
        fields = ["name", "content"]
```

## 13) Crear vistas (List + Detail + Create + Edit)
En members/views.py:
```python
from django.shortcuts import render, redirect, get_object_or_404
from django.views import View
from django.views.generic import ListView, DetailView
from .models import Article
from .forms import ArticleForm
class ArticleListView(ListView):
    model = Article
    template_name = "members/article_list.html"
    context_object_name = "article_list"
class ArticleDetailView(DetailView):
    model = Article
    template_name = "members/article_detail.html"
    context_object_name = "article"
class NewArticleForm(View):
    def get(self, request):
        form = ArticleForm()
        return render(request, "members/new_article_form.html", {"form": form})
    def post(self, request):
        form = ArticleForm(request.POST)
        if form.is_valid():
            form.save()
            return redirect("all_articles")
        return render(request, "members/new_article_form.html", {"form": form})
class EditArticleForm(View):
    def get(self, request, pk):
        article = get_object_or_404(Article, pk=pk)
        form = ArticleForm(instance=article)
        return render(request, "members/edit_article_form.html", {"form": form, "article": article})
    def post(self, request, pk):
        article = get_object_or_404(Article, pk=pk)
        form = ArticleForm(request.POST, instance=article)
        if form.is_valid():
            form.save()
            return redirect("article_detail", pk=article.pk)
        return render(request, "members/edit_article_form.html", {"form": form, "article": article})
```

## 14) Crear templates (HTML)
Crea carpetas:
members/templates/members/

### 14.1 Lista: article_list.html
```html
<h1>Artículos</h1>
<a href="{% url 'new_article' %}">➕ Nuevo</a>
{% if article_list %}
  <ul>
    {% for a in article_list %}
      <li>
        <a href="{% url 'article_detail' a.pk %}">{{ a.name }}</a>
        | <a href="{% url 'edit_article' a.pk %}">✏️ Editar</a>
      </li>
    {% endfor %}
  </ul>
{% else %}
  <p>No hay artículos todavía.</p>
{% endif %}

### 14.2 Nuevo: new_article_form.html
<h1>Nuevo artículo</h1>
<form method="post">
  {% csrf_token %}
  {{ form.as_p }}
  <button type="submit">Guardar</button>
</form>
<hr>
<a href="{% url 'all_articles' %}">⬅ Volver</a>
14.3 Detalle: article_detail.html
<h1>{{ article.name }}</h1>
<p>{{ article.content }}</p>
<a href="{% url 'edit_article' article.pk %}">✏️ Editar</a>
<br>
<a href="{% url 'all_articles' %}">⬅ Volver</a>

### 14.4 Editar: edit_article_form.html
<h1>Editar artículo</h1>
<form method="post">
  {% csrf_token %}
  {{ form.as_p }}
  <button type="submit">Guardar cambios</button>
</form>
<hr>
<a href="{% url 'article_detail' article.pk %}">⬅ Volver</a>
```


## 15) URLs de la app (members/urls.py)
Crea: members/urls.py
```python
from django.urls import path
from . import views
urlpatterns = [
    path("", views.ArticleListView.as_view(), name="all_articles"),
    path("new/", views.NewArticleForm.as_view(), name="new_article"),
    path("article/<int:pk>/", views.ArticleDetailView.as_view(), name="article_detail"),
    path("article/<int:pk>/edit/", views.EditArticleForm.as_view(), name="edit_article"),
]

## 16) Conectar las URLs en el proyecto (sunlin/urls.py)
En sunlin/sunlin/urls.py:
from django.contrib import admin
from django.urls import path, include
urlpatterns = [
    path("admin/", admin.site.urls),
    path("members/", include("members.urls")),
]
```

## 17) Probar todo (runserver)
```python
python manage.py runserver
```
