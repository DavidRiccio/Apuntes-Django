# Paso a paso para hacer proyectos de django


### Preparacion del entorno virtual
Creamos la carpeta del proyecto y hacemos ``` pypas get <nombre-del-ejercicio> ```, depsues una vez descargado este, hacemos ```j create-venv``` esto nos creara el entorno virtual y nos instalara los requerimientos necesarios. El siguiente paso es hacer ```j setup``` que  nos termina de configurar el proyecto. Despues haremos ``` j startapp <nombre> ``` esto nos añadira automaticamente las apps al installed apps del settings. 

Para no olvidarnos pondremos en el settings lo siguiente 
```python
DEFAULT_AUTO_FIELD = 'django.db.models.BigAutoField'
LOGIN_URL = 'login'
#Redireccion del login
LOGIN_REDIRECT_URL = 'echos:echo_list'
#Redireccion del logout
LOGOUT_URL = 'logout'
#Carpeta de guardado de las imagenes
MEDIA_ROOT = BASE_DIR / 'media'
```
Despues habria que agregar las palicaciones al admin 
```python 
from django.contrib import admin
from .models import Echo
class EchosAdmin(admin.ModelAdmin):
    pass


admin.site.register(Echo, EchosAdmin)
```
### Creacion de los modelos 

En la creacion del modelo definiremos los campos que queremos que tenga en la base de datos,definiendo tambien las FK que contenga ( en este caso el modelo users de django), aqui un ejemplo:
```python
class Echo(models.Model):
    content = models.TextField(max_length=250)
    created_at = models.DateTimeField(auto_now_add=True)
    updated_at = models.DateTimeField(auto_now=True)
    user = models.ForeignKey(settings.AUTH_USER_MODEL, on_delete=models.CASCADE)

    class Meta:
        ordering = ['-created_at']

    def __str__(self):
        return self.content

    def get_absolute_url(self):
        return reverse('Echo_detail', kwargs={'pk': self.pk})

```

El ordering se emplea para que este ordenado de manera predefinida al sacar los echos en alguna vista, aqui otro ejemplo con echos como FK:

```python
class Wave(models.Model):
    content = models.TextField(max_length=250)
    created_at = models.DateTimeField(auto_now_add=True)
    updated_at = models.DateTimeField(auto_now=True)
    user = models.ForeignKey(settings.AUTH_USER_MODEL, on_delete=models.CASCADE)
    echo = models.ForeignKey('echos.Echo', related_name='waves', on_delete=models.CASCADE)

    class Meta:
        ordering = ['-created_at']

    def __str__(self) -> str:
        return self.content
```


Una vez hechos todos los modelos se ejecutara el comando ```j load-data``` para cargar con informacion la base de datos

### Accounts

Para hacer el manejo de cuentas vamos a emplear un aplicacion llamada accounts, todas las templates van dentro de templates/registration las urls de accounts van a ir dentro del main.urls de la siguiente manera:
```python
from django.contrib import admin
from django.contrib.auth.views import LoginView
from django.shortcuts import redirect
from django.urls import include, path
import accounts.views

urlpatterns = [
    path('admin/', admin.site.urls),
    path('', lambda _: redirect('echos:echo_list')),
    path('login/', LoginView.as_view(), name='login'),
    path('logout/', accounts.views.logout_view, name='logout'),
    path('signup/', accounts.views.signup, name='signup'),
    path('echos/', include('echos.urls')),
    path('waves/', include('waves.urls')),
    path('users/', include('users.urls')),
]
```

Para el login vamos a emplear la vista que te proporciona django eso se hace como se ve arriba, importando LoginView y poniendo ```     path('login/', LoginView.as_view(), name='login'),``` esto es suficiente para que funcione el login.

Para la template haria falta esto:
```python
{% extends "shared/base.html" %}

{% block body %}
{% if user.is_authenticated %}
    <p>Ya estás logeado... <a  class="btn bg-custom-orange ms-auto me-auto mt-3" href="{% url "echos:echo_list" %}">Ir al inicio</a></p>
{% else %}
    {% if form.errors %}
        <p>Tu nombre de usuario y contraseña no coinciden. Por favor inténtalo de nuevo.</p>
    {% endif %}
    {% if next and next != '/' %}
        {% if user.is_authenticated %}
            <p>Tu cuenta no tiene acceso a esta página. Para proceder, por favor haz login con una cuenta que tenga acceso.</p>
        {% else %}
            <p>Por favor haz login para ver esta página.</p>
        {% endif %}
    {% endif %}
    <form method="post" action="{% url 'login' %}" novalidate>
        {% csrf_token %}
        {{ form }}
        <input type="hidden" name="next" value="{{ next }}">
        <input type="submit" value="Login">
    </form>
{% endif %}
{% endblock %}
```

En el caso del signup vamos a tener que hacerlo nosotros a mano y crear la vista y el formulario 
```python
from django.contrib.auth import login, logout
from django.shortcuts import redirect, render
from .forms import SignupForm
def signup(request):
    form = SignupForm(request.POST or None)
    if (form := SignupForm(request.POST)).is_valid():
        user = form.save(commit=False)
        user.set_password(form.cleaned_data['password'])
        user.save()
        login(request, user)
        return redirect('echos:echo_list')
    return render(request, 'registration/signup.html', dict(form=form))
```
Y para el formulario pondremos **Este fichero hay que crearlo**
```python 
from django import forms
from django.contrib.auth import get_user_model


class SignupForm(forms.ModelForm):
    class Meta:
        model = get_user_model()
        fields = ['username', 'first_name', 'last_name', 'email', 'password']
        widgets = dict(password=forms.PasswordInput)

    def __init__(self, *args, **kwargs):
        super().__init__(*args, **kwargs)
        for visible in self.visible_fields():
            visible.field.widget.attrs['class'] = 'form-control'
```

Por ultimo tambien hay que hacer una template para este seria algo como esto:
```python
{% extends "shared/base.html" %}

{% block body %}
<form method="post" novalidate>
    {% csrf_token %}
    <div class="w-50 ms-auto me-auto">
    {{ form }}
    <input type="submit" value="Registro">
    <p>¿Ya tienes una cuenta? <a href="{% url 'login' %}">Haz login aquí</a></p>
    </div>
</form>
{% endblock %}
```

Para el logout seria mas de lo mismo la vista es esta:
```python
def logout_view(request):
    logout(request)
    return redirect('echos:echo_list')
```

Y la plantilla esta 
```python
{% extends "shared/base.html" %}

{% block body %}
<form action="{% url 'logout' %}" method="post">
    {% csrf_token %}
    <button type="submit">Cerrar sesión</button>
</form>
{% endblock %}
```

Una vez hecho esto todo el manejo de usuarios deberia funcionar correctamente

### Crear las urls y vistas 

Para esto nos es necesario mucho hay que analizar el proyecto y ver a que aplicacion pertenece cada urls, para despues crearla y en las vistas de cada uno poner pass **Este fichero hay que crearlo**

Un ejemplo 
```python
from django.urls import path

from waves.views import add_wave

from . import views

app_name = 'echos'


urlpatterns = [
    path('', views.echo_list, name='echo_list'),
    path('<int:echo_pk>/', views.echo_detail, name='echo_detail'),
    path('add/', views.add_echo, name='add_echo'),
    path('<int:echo_pk>/edit/', views.edit_echo, name='edit_echo'),
    path('<int:echo_pk>/delete/', views.delete_echo, name='delete_echo'),
    path('<int:echo_pk>/waves/', views.echo_waves, name='echo_waves'),
    path('<int:echo_pk>/waves/add/', add_wave, name='add_wave'),
]
```

### Borrado, edicion, y adicion
Le tienes que pasar la pk a cada uno de los formularios
**Vistas**
```python
from django.contrib.auth.decorators import login_required
from django.http import HttpResponseForbidden
from django.shortcuts import redirect, render

from .forms import AddEchoForm, EditEchoForm
from .models import Echo

@login_required
def add_echo(request):
    if request.method == 'GET':
        form = AddEchoForm()
    else:
        if (form := AddEchoForm(data=request.POST)).is_valid():
            echo = form.save(commit=False)
            echo.user = request.user
            echo.save()
            return redirect('echos:echo_list')
    return render(request, 'echos/add_echo.html', dict(form=form))
```
```python
@login_required
def edit_echo(request, echo_pk):
    echo = Echo.objects.get(pk=echo_pk)
    if request.user != echo.user:
        return HttpResponseForbidden('NO ERES PROPIETARIO DE ESTE ECHO')
    if request.method == 'GET':
        form = EditEchoForm(instance=echo)
    else:
        if (form := EditEchoForm(request.POST, instance=echo)).is_valid():
            echo = form.save(commit=False)
            echo.save()
            return redirect('echos:echo_list')
    return render(request, 'echos/add_echo.html', dict(echo=echo, form=form))
```
```python
@login_required
def delete_echo(request, echo_pk):
    echo = Echo.objects.get(pk=echo_pk)
    if request.user != echo.user:
        return HttpResponseForbidden(
            'ERROR 403: NO TIENES PERMITIDO EDITAR EL ECHO DE OTRO INTEGRANTE DE LA TRIBU'
        )
    echo = Echo.objects.get(pk=echo_pk)
    echo.delete()
    return render(request, 'echos/delete_echo.html', dict(echo=echo))
```
**Formularios**

```python
from django import forms

from .models import Echo


class AddEchoForm(forms.ModelForm):
    class Meta:
        model = Echo
        fields = ('content',)

    def __init__(self, *args, **kwargs):
        super().__init__(*args, **kwargs)
        for visible in self.visible_fields():
            visible.field.widget.attrs['class'] = 'form-control'


class EditEchoForm(forms.ModelForm):
    class Meta:
        model = Echo
        fields = ('content',)

    def __init__(self, *args, **kwargs):
        super().__init__(*args, **kwargs)
        for visible in self.visible_fields():
            visible.field.widget.attrs['class'] = 'form-control'

```

### Detalles de contenido

```python
@login_required
def wave_detail(request, wave_pk):
    wave = Wave.objects.get(pk=wave_pk)
    return render(request, 'waves/wave_detail.html', {'wave': wave})
```

```python
from django.urls import path

from . import views

app_name = 'waves'

urlpatterns = [
    path('<int:wave_pk>/', views.wave_detail, name='wave_detail'),
    path('<int:wave_pk>/edit/', views.edit_wave, name='edit_wave'),
    path('<int:wave_pk>/delete/', views.delete_wave, name='delete_wave'),
]

```

Esto se llama en la template asi 
```python
                    <a href="{% url 'waves:wave_detail' wave.pk %}" class="btn btn-primary">

```