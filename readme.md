# Django OAuth 2.0 Example

This repository contains an example implementation of registration, login, application login, and application registration processes using Django and OAuth 2.0.

Writed by: Osvaldo Calombe

Why: I still love and want more than a friendship with Will

## Índice

1. [Introdução](#introdução)
2. [Instalação](#instalação)
3. [Configuração](#configuração)
4. [Estrutura do Projeto](#estrutura-do-projeto)
5. [Processo de Registro](#processo-de-registro)
6. [Processo de Login](#processo-de-login)
7. [Login de Aplicação](#login-de-aplicação)
8. [Registro de Aplicação](#registro-de-aplicação)
9. [Testando o Código](#testando-o-código)
10. [Contribuição](#contribuição)

## Introdução

Este projeto demonstra como configurar um sistema de autenticação e autorização usando Django e OAuth 2.0. Ele cobre os processos de registro e login de usuário, bem como login e registro de aplicação.

## Configuração

1. Adicione `oauth2_provider` ao seu `INSTALLED_APPS` no `settings.py`.

    ```python
    # settings.py
    INSTALLED_APPS = [
        # Outros apps
        'oauth2_provider',
    ]
    ```

2. Configure as URLs do OAuth 2.0 no seu projeto:

    ```python
    # urls.py
    from django.urls import path, include

    urlpatterns = [
        # Outras URLs
        path('o/', include('oauth2_provider.urls', namespace='oauth2_provider')),
    ]
    ```

## Estrutura do Projeto

Estrutura sugerida para o projeto Django:

```
myproject/
    manage.py
    myproject/
        __init__.py
        settings.py
        urls.py
        wsgi.py
    myapp/
        __init__.py
        admin.py
        apps.py
        forms.py
        models.py
        tests.py
        views.py
        templates/
            register.html
            login.html
            verification_failed.html
            register_application.html
            application_detail.html
        utils.py
```

## Processo de Registro

1. **Criação do Formulário de Registro** - `myapp/forms.py`:

    ```python
    from django import forms
    from django.contrib.auth.models import User

    class RegistrationForm(forms.ModelForm):
        password = forms.CharField(widget=forms.PasswordInput)
        confirm_password = forms.CharField(widget=forms.PasswordInput)

        class Meta:
            model = User
            fields = ['username', 'email', 'password']

        def clean(self):
            cleaned_data = super().clean()
            password = cleaned_data.get("password")
            confirm_password = cleaned_data.get("confirm_password")

            if password != confirm_password:
                raise forms.ValidationError("As senhas não correspondem.")
    ```

2. **View para Registro** - `myapp/views.py`:

    ```python
    from django.shortcuts import render, redirect
    from django.contrib.auth.models import User
    from django.contrib.auth.hashers import make_password
    from .forms import RegistrationForm
    from .models import EmailVerificationToken
    from .utils import send_verification_email

    def register(request):
        if request.method == 'POST':
            form = RegistrationForm(request.POST)
            if form.is_valid():
                user = form.save(commit=False)
                user.password = make_password(form.cleaned_data['password'])
                user.is_active = False  # Desativa a conta até que o e-mail seja verificado
                user.save()

                # Gera um token de verificação e envia e-mail
                token = EmailVerificationToken.objects.create(user=user)
                send_verification_email(user.email, token.token)

                return redirect('registration_success')
        else:
            form = RegistrationForm()
        return render(request, 'register.html', {'form': form})
    ```

3. **Envio de E-mail de Verificação** - `myapp/utils.py`:

    ```python
    from django.core.mail import send_mail
    from django.conf import settings

    def send_verification_email(email, token):
        subject = "Verifique seu e-mail"
        message = f"Por favor, clique no link para verificar seu e-mail: http://seusite.com/verify-email/?token={token}"
        from_email = settings.DEFAULT_FROM_EMAIL
        recipient_list = [email]

        send_mail(subject, message, from_email, recipient_list)
    ```

4. **Modelo para o Token de Verificação de E-mail** - `myapp/models.py`:

    ```python
    from django.db import models
    from django.contrib.auth.models import User

    class EmailVerificationToken(models.Model):
        user = models.OneToOneField(User, on_delete=models.CASCADE)
        token = models.CharField(max_length=50, unique=True)
        created_at = models.DateTimeField(auto_now_add=True)
    ```

5. **View para Verificação de E-mail** - `myapp/views.py`:

    ```python
    from django.shortcuts import render, redirect
    from .models import EmailVerificationToken
    from django.contrib.auth.models import User

    def verify_email(request):
        token = request.GET.get('token')
        try:
            verification_token = EmailVerificationToken.objects.get(token=token)
            user = verification_token.user
            user.is_active = True
            user.save()
            verification_token.delete()
            return redirect('login')
        except EmailVerificationToken.DoesNotExist:
            return render(request, 'verification_failed.html')
    ```

6. **Templates**:

    - `myapp/templates/register.html`
    - `myapp/templates/verification_failed.html`

## Processo de Login

1. **Criação do Formulário de Login** - `myapp/forms.py`:

    ```python
    from django import forms

    class LoginForm(forms.Form):
        username = forms.CharField()
        password = forms.CharField(widget=forms.PasswordInput)
    ```

2. **View para Login** - `myapp/views.py`:

    ```python
    from django.shortcuts import render, redirect
    from django.contrib.auth import authenticate, login
    from .forms import LoginForm

    def login_view(request):
        if request.method == 'POST':
            form = LoginForm(request.POST)
            if form.is_valid():
                username = form.cleaned_data['username']
                password = form.cleaned_data['password']
                user = authenticate(request, username=username, password=password)
                if user is not None:
                    if user.is_active:
                        login(request, user)
                        return redirect('home')
                    else:
                        return render(request, 'login.html', {'form': form, 'error': 'Conta não verificada.'})
                else:
                    return render(request, 'login.html', {'form': form, 'error': 'Credenciais inválidas.'})
        else:
            form = LoginForm()
        return render(request, 'login.html', {'form': form})
    ```

3. **Template**:

    - `myapp/templates/login.html`

## Login de Aplicação

1. **Configuração do Cliente OAuth 2.0** - `settings.py`:

    ```python
    INSTALLED_APPS = [
        # Outros apps
        'oauth2_provider',
    ]
    ```

2. **Configuração das URLs** - `urls.py`:

    ```python
    from django.urls import path, include

    urlpatterns = [
        # Outras URLs
        path('o/', include('oauth2_provider.urls', namespace='oauth2_provider')),
    ]
    ```

3. **View para Iniciar o Login de Aplicação** - `myapp/views.py`:

    ```python
    from django.shortcuts import redirect

    def initiate_app_login(request):
        client_id = 'SEU_CLIENT_ID'
        redirect_uri = 'SUA_REDIRECT_URI'
        response_type = 'code'
        scope = 'read write'
        state = 'algum_estado_unico'

        authorization_url = f"https://idp.com/o/authorize/?client_id={client_id}&redirect_uri={redirect_uri}&response_type={response_type}&scope={scope}&state={state}"
        return redirect(authorization_url)
    ```

4. **View para Lidar com o Callback do IDP** - `myapp/views.py`:

    ```python
    import requests
    from django.conf import settings

    def handle_idp_callback(request):
        authorization_code = request.GET.get('code')
        state = request.GET.get('state')

        token_url = 'https://idp.com/o/token/'
        client_id = 'SEU_CLIENT_ID'
        client_secret = 'SEU_CLIENT_SECRET'
        redirect_uri = 'SUA_REDIRECT_URI'
        grant_type = 'authorization_code'

        response = requests.post(token_url, data={
            'code': authorization_code,
            'client_id': client

_id,
            'client_secret': client_secret,
            'redirect_uri': redirect_uri,
            'grant_type': grant_type
        })

        if response.status_code == 200:
            tokens = response.json()
            access_token = tokens['access_token']
            refresh_token = tokens['refresh_token']
            # Armazene os tokens de forma segura
            store_tokens(access_token, refresh_token)
            return redirect('home')
        else:
            return render(request, 'callback_failed.html')
    ```

5. **Função para Armazenar Tokens** - `myapp/utils.py`:

    ```python
    def store_tokens(access_token, refresh_token):
        # Implementar armazenamento seguro de tokens
        pass
    ```

6. **Template**:

    - `myapp/templates/callback_failed.html`

## Registro de Aplicação

1. **Criação do Modelo de Aplicação** - `myapp/models.py`:

    ```python
    from django.db import models
    from django.contrib.auth.models import User

    class Application(models.Model):
        user = models.ForeignKey(User, on_delete=models.CASCADE)
        name = models.CharField(max_length=255)
        client_id = models.CharField(max_length=255, unique=True)
        client_secret = models.CharField(max_length=255, unique=True)
        redirect_uri = models.URLField()
        created_at = models.DateTimeField(auto_now_add=True)
    ```

2. **Criação do Formulário de Registro de Aplicação** - `myapp/forms.py`:

    ```python
    from django import forms
    from .models import Application

    class ApplicationRegistrationForm(forms.ModelForm):
        class Meta:
            model = Application
            fields = ['name', 'redirect_uri']
    ```

3. **View para Registrar a Aplicação** - `myapp/views.py`:

    ```python
    from django.shortcuts import render, redirect
    from .forms import ApplicationRegistrationForm
    from .models import Application
    import secrets

    def register_application(request):
        if request.method == 'POST':
            form = ApplicationRegistrationForm(request.POST)
            if form.is_valid():
                application = form.save(commit=False)
                application.user = request.user
                application.client_id = secrets.token_urlsafe(16)
                application.client_secret = secrets.token_urlsafe(32)
                application.save()
                return redirect('application_detail', application.id)
        else:
            form = ApplicationRegistrationForm()
        return render(request, 'register_application.html', {'form': form})
    ```

4. **View para Detalhar a Aplicação** - `myapp/views.py`:

    ```python
    from django.shortcuts import render, get_object_or_404
    from .models import Application

    def application_detail(request, application_id):
        application = get_object_or_404(Application, id=application_id)
        return render(request, 'application_detail.html', {'application': application})
    ```

5. **Templates**:

    - `myapp/templates/register_application.html`
    - `myapp/templates/application_detail.html`

## Testando o Código

1. Crie um superusuário para acessar o admin do Django:

    ```bash
    python manage.py createsuperuser
    ```

2. Execute o servidor de desenvolvimento:

    ```bash
    python manage.py runserver
    ```

3. Abra seu navegador e vá para `http://localhost:8000/admin` para acessar o admin do Django.

4. Teste o registro de usuário:
    - Vá para a URL de registro de usuário e preencha o formulário.
    - Verifique se o e-mail de verificação foi enviado e clique no link de verificação.

5. Teste o login de usuário:
    - Vá para a URL de login de usuário e faça login com as credenciais registradas.

6. Teste o registro de aplicação:
    - Vá para a URL de registro de aplicação e preencha o formulário.
    - Verifique se a aplicação foi registrada corretamente.

7. Teste o login de aplicação:
    - Inicie o login de aplicação e autorize a aplicação na IDP.

## Contribuição

1. Faça um fork do projeto
2. Crie uma branch para sua feature (`git checkout -b feature/fooBar`)
3. Faça commit das suas mudanças (`git commit -am 'Add some fooBar'`)
4. Faça push para a branch (`git push origin feature/fooBar`)
5. Crie um novo Pull Request