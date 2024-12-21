Для реализации книжного магазина на Django, в котором покупатели могут оформлять заявки на книги и получать уведомления о поступлении отсутствующих экземпляров, нужно выполнить следующие шаги:


---

1. Создание моделей

Добавьте модели для хранения информации о книгах, заявках и покупателях.

Файл: models.py в приложении shop

from django.db import models

class Book(models.Model):
    title = models.CharField(max_length=200)  # Название книги
    author = models.CharField(max_length=200)  # Автор книги
    publisher = models.CharField(max_length=200)  # Издательство
    publication_year = models.IntegerField()  # Год издания
    price = models.DecimalField(max_digits=10, decimal_places=2)  # Цена
    stock = models.PositiveIntegerField(default=0)  # Количество книг на складе

def __str__(self):
    return self.title


class Customer(models.Model):
    name = models.CharField(max_length=200)  # Имя покупателя
    email = models.EmailField(unique=True)  # Email покупателя

    def __str__(self):
        return self.name


class Order(models.Model):
    customer = models.ForeignKey(Customer, on_delete=models.CASCADE)  # Покупатель
    book = models.ForeignKey(Book, on_delete=models.CASCADE)  # Заказываемая книга
    quantity = models.PositiveIntegerField()  # Количество
    created_at = models.DateTimeField(auto_now_add=True)  # Дата оформления

    def __str__(self):
        return f"Order by {self.customer.name} for {self.book.title}"


class BookRequest(models.Model):
    customer = models.ForeignKey(Customer, on_delete=models.CASCADE)  # Покупатель
    book_title = models.CharField(max_length=200)  # Название книги
    created_at = models.DateTimeField(auto_now_add=True)  # Дата запроса
    is_notified = models.BooleanField(default=False)  # Был ли отправлен уведомление

    def __str__(self):
        return f"Request by {self.customer.name} for {self.book_title}"


---

2. Настройка админ-панели

Файл: admin.py

from django.contrib import admin
from .models import Book, Customer, Order, BookRequest

@admin.register(Book)
class BookAdmin(admin.ModelAdmin):
    list_display = ('title', 'author', 'publisher', 'publication_year', 'price', 'stock')

@admin.register(Customer)
class CustomerAdmin(admin.ModelAdmin):
    list_display = ('name', 'email')

@admin.register(Order)
class OrderAdmin(admin.ModelAdmin):
    list_display = ('customer', 'book', 'quantity', 'created_at')

@admin.register(BookRequest)
class BookRequestAdmin(admin.ModelAdmin):
    list_display = ('customer', 'book_title', 'created_at', 'is_notified')


---

3. Реализация представлений

Создайте представления для отображения списка книг, оформления заявок и создания запросов на книги.

Файл: views.py

from django.shortcuts import render, get_object_or_404, redirect
from django.contrib import messages
from .models import Book, Customer, Order, BookRequest
from .forms import OrderForm, BookRequestForm

def book_list(request):
    books = Book.objects.all()
    return render(request, 'shop/book_list.html', {'books': books})

def order_book(request, book_id):
    book = get_object_or_404(Book, id=book_id)
    if request.method == 'POST':
        form = OrderForm(request.POST)
        if form.is_valid():
            order = form.save(commit=False)
            order.book = book
            order.customer, _ = Customer.objects.get_or_create(
                email=form.cleaned_data['email'],
                defaults={'name': form.cleaned_data['name']}
            )
            order.save()
            book.stock -= order.quantity
            book.save()
            messages.success(request, 'Заказ оформлен успешно!')
            return redirect('book_list')
    else:
        form = OrderForm()
    return render(request, 'shop/order_book.html', {'form': form, 'book': book})

def request_book(request):
    if request.method == 'POST':
        form = BookRequestForm(request.POST)
        if form.is_valid():
            book_request = form.save(commit=False)
            book_request.customer, _ = Customer.objects.get_or_create(
                email=form.cleaned_data['email'],
                defaults={'name': form.cleaned_data['name']}
            )
            book_request.save()
            messages.success(request, 'Ваш запрос добавлен!')
            return redirect('book_list')
    else:
        form = BookRequestForm()
    return render(request, 'shop/request_book.html', {'form': form})


---

4. Формы

Файл: forms.py

from django import forms
from .models import Order, BookRequest

class OrderForm(forms.ModelForm):
    name = forms.CharField(max_length=200)
    email = forms.EmailField()

    class Meta:
        model = Order
        fields = ['quantity']

class BookRequestForm(forms.ModelForm):
    name = forms.CharField(max_length=200)
    email = forms.EmailField()

    class Meta:
        model = BookRequest
        fields = ['book_title']


---

5. Шаблоны

Файл: templates/shop/book_list.html

<h1>Книги</h1>
<ul>
    {% for book in books %}
    <li>
        {{ book.title }} - {{ book.author }} ({{ book.price }} руб.) 
        {% if book.stock > 0 %}
        <a href="{% url 'order_book' book.id %}">Заказать</a>
        {% else %}
        <a href="{% url 'request_book' %}">Запросить</a>
        {% endif %}
    </li>
    {% endfor %}
</ul>

Файл: templates/shop/order_book.html

<h1>Заказ книги: {{ book.title }}</h1>
<form method="post">
    {% csrf_token %}
    {{ form.as_p }}
    <button type="submit">Оформить заказ</button>
</form>

Файл: templates/shop/request_book.html

<h1>Запрос книги</h1>
<form method="post">
    {% csrf_token %}
    {{ form.as_p }}
    <button type="submit">Отправить запрос</button>
</form>


---

6. Настройка маршрутов

Файл: urls.py в приложении shop

from django.urls import path
from . import views

urlpatterns = [
    path('', views.book_list, name='book_list'),
    path('order/<int:book_id>/', views.order_book, name='order_book'),
    path('request/', views.request_book, name='request_book'),
]


---

Это базовая структура. Для отправки уведомлений можно добавить обработку запросов через Celery или Cron и использовать email-уведомления через встроенный в Django модуль send_mail.

Для реализации системы входа, регистрации и профиля в Django выполните следующие шаги:


---

1. Подготовка моделей

Используем встроенную модель пользователя User из django.contrib.auth.


---

2. Создание форм регистрации и редактирования профиля

Файл: forms.py

from django import forms
from django.contrib.auth.models import User
from django.contrib.auth.forms import UserCreationForm

class UserRegistrationForm(UserCreationForm):
    email = forms.EmailField(required=True)

    class Meta:
        model = User
        fields = ['username', 'email', 'password1', 'password2']

class UserEditForm(forms.ModelForm):
    email = forms.EmailField(required=True)

    class Meta:
        model = User
        fields = ['username', 'email']


---

3. Представления для регистрации, входа и профиля

Файл: views.py
Добавьте следующие функции:

from django.shortcuts import render, redirect
from django.contrib.auth import login, authenticate
from django.contrib.auth.decorators import login_required
from django.contrib import messages
from .forms import UserRegistrationForm, UserEditForm

def register(request):
    if request.method == 'POST':
        form = UserRegistrationForm(request.POST)
        if form.is_valid():
            user = form.save()
            login(request, user)
            messages.success(request, 'Регистрация прошла успешно!')
            return redirect('profile')
    else:
        form = UserRegistrationForm()
    return render(request, 'shop/register.html', {'form': form})

@login_required
def profile(request):
    if request.method == 'POST':
        form = UserEditForm(request.POST, instance=request.user)
        if form.is_valid():
            form.save()
            messages.success(request, 'Профиль обновлен!')
            return redirect('profile')
    else:
        form = UserEditForm(instance=request.user)
    return render(request, 'shop/profile.html', {'form': form})


---

4. Маршруты

Файл: urls.py
Добавьте маршруты:

from django.urls import path
from django.contrib.auth import views as auth_views
from . import views

urlpatterns = [
    path('register/', views.register, name='register'),
    path('login/', auth_views.LoginView.as_view(template_name='shop/login.html'), name='login'),
    path('logout/', auth_views.LogoutView.as_view(template_name='shop/logout.html'), name='logout'),
    path('profile/', views.profile, name='profile'),
]


---

5. Шаблоны

Файл: templates/shop/register.html

<h1>Регистрация</h1>
<form method="post">
    {% csrf_token %}
    {{ form.as_p }}
    <button type="submit">Зарегистрироваться</button>
</form>
<a href="{% url 'login' %}">Уже есть аккаунт? Войти</a>

Файл: templates/shop/login.html

<h1>Вход</h1>
<form method="post">
    {% csrf_token %}
    {{ form.as_p }}
    <button type="submit">Войти</button>
</form>
<a href="{% url 'register' %}">Нет аккаунта? Зарегистрироваться</a>

Файл: templates/shop/logout.html

<h1>Вы вышли из системы</h1>
<a href="{% url 'login' %}">Войти снова</a>

Файл: templates/shop/profile.html

<h1>Профиль</h1>
<form method="post">
    {% csrf_token %}
    {{ form.as_p }}
    <button type="submit">Сохранить изменения</button>
</form>
<a href="{% url 'logout' %}">Выйти</a>


---

6. Настройка в settings.py

Добавьте конфигурацию перенаправления после входа и выхода:

LOGIN_REDIRECT_URL = 'profile'
LOGOUT_REDIRECT_URL = 'login'


---

Теперь пользователи смогут регистрироваться, входить в систему и редактировать свои профили.

Чтобы добавить в профиль пользователя отображение заявок на книги, оформленных этим пользователем, нужно обновить логику в представлении профиля, шаблон и модель заявки.


---

1. Обновление представления профиля

Мы будем загружать заявки текущего пользователя и передавать их в шаблон.

Файл: views.py

from .models import Order

@login_required
def profile(request):
    if request.method == 'POST':
        form = UserEditForm(request.POST, instance=request.user)
        if form.is_valid():
            form.save()
            messages.success(request, 'Профиль обновлен!')
            return redirect('profile')
    else:
        form = UserEditForm(instance=request.user)
    
    # Получение заявок текущего пользователя
    orders = Order.objects.filter(customer__email=request.user.email)

    return render(request, 'shop/profile.html', {'form': form, 'orders': orders})


---

2. Шаблон профиля

Обновим шаблон профиля, чтобы он отображал информацию о заявках.

Файл: templates/shop/profile.html

<h1>Профиль</h1>
<form method="post">
    {% csrf_token %}
    {{ form.as_p }}
    <button type="submit">Сохранить изменения</button>
</form>

<h2>Ваши заявки</h2>
{% if orders %}
    <ul>
        {% for order in orders %}
            <li>
                Книга: {{ order.book.title }} (Количество: {{ order.quantity }}) - 
                Дата оформления: {{ order.created_at|date:"d.m.Y H:i" }}
            </li>
        {% endfor %}
    </ul>
{% else %}
    <p>У вас пока нет оформленных заявок.</p>
{% endif %}

<a href="{% url 'logout' %}">Выйти</a>


---

3. Обновление модели Order (если потребуется)

Если у заявки нет привязки к конкретной книге (например, когда она недоступна), вы можете объединить с моделью BookRequest или добавить к заявке поле для отображения названия книги. Но текущая модель уже позволяет выводить данные о заказанных книгах.


---

4. Проверка связей

Убедитесь, что в таблице Customer email пользователя из User совпадает с email, используемым при оформлении заказов.


---

Теперь в профиле пользователи смогут видеть список всех оформленных ими заявок, включая название книги, количество экземпляров и дату заказа.

Для добавления Bootstrap стилей в ваш проект выполните следующие шаги:


---

1. Подключение Bootstrap

Добавьте ссылку на Bootstrap CSS и JavaScript в базовый шаблон.

Файл: templates/shop/base.html

<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>{% block title %}Книжный магазин{% endblock %}</title>
    <!-- Подключение Bootstrap -->
    <link href="https://cdn.jsdelivr.net/npm/bootstrap@5.3.0/dist/css/bootstrap.min.css" rel="stylesheet">
    <script src="https://cdn.jsdelivr.net/npm/bootstrap@5.3.0/dist/js/bootstrap.bundle.min.js"></script>
</head>
<body>
    <nav class="navbar navbar-expand-lg navbar-light bg-light">
        <div class="container-fluid">
            <a class="navbar-brand" href="{% url 'book_list' %}">Книжный магазин</a>
            <div class="collapse navbar-collapse">
                <ul class="navbar-nav me-auto mb-2 mb-lg-0">
                    {% if user.is_authenticated %}
                        <li class="nav-item">
                            <a class="nav-link" href="{% url 'profile' %}">Профиль</a>
                        </li>
                        <li class="nav-item">
                            <a class="nav-link" href="{% url 'logout' %}">Выйти</a>
                        </li>
                    {% else %}
                        <li class="nav-item">
                            <a class="nav-link" href="{% url 'login' %}">Вход</a>
                        </li>
                        <li class="nav-item">
                            <a class="nav-link" href="{% url 'register' %}">Регистрация</a>
                        </li>
                    {% endif %}
                </ul>
            </div>
        </div>
    </nav>
    <div class="container mt-4">
        {% block content %}
        {% endblock %}
    </div>
</body>
</html>


---

2. Использование Bootstrap в шаблонах

Обновите шаблоны, чтобы добавить стили.


---

Шаблон списка книг

Файл: templates/shop/book_list.html

{% extends 'shop/base.html' %}

{% block title %}Список книг{% endblock %}

{% block content %}
<h1 class="mb-4">Книги</h1>
<div class="row">
    {% for book in books %}
        <div class="col-md-4">
            <div class="card mb-4">
                <div class="card-body">
                    <h5 class="card-title">{{ book.title }}</h5>
                    <p class="card-text">
                        Автор: {{ book.author }}<br>
                        Издательство: {{ book.publisher }}<br>
                        Год издания: {{ book.publication_year }}<br>
                        Цена: {{ book.price }} руб.<br>
                        Количество: {{ book.stock }}
                    </p>
                    {% if book.stock > 0 %}
                        <a href="{% url 'order_book' book.id %}" class="btn btn-primary">Заказать</a>
                    {% else %}
                        <a href="{% url 'request_book' %}" class="btn btn-warning">Запросить</a>
                    {% endif %}
                </div>
            </div>
        </div>
    {% endfor %}
</div>
{% endblock %}


---

Шаблон профиля

Файл: templates/shop/profile.html

{% extends 'shop/base.html' %}

{% block title %}Профиль{% endblock %}

{% block content %}
<h1 class="mb-4">Профиль</h1>
<div class="card">
    <div class="card-body">
        <h5 class="card-title">Редактирование профиля</h5>
        <form method="post">
            {% csrf_token %}
            {{ form.as_p }}
            <button type="submit" class="btn btn-success">Сохранить изменения</button>
        </form>
    </div>
</div>

<h2 class="mt-5">Ваши заявки</h2>
{% if orders %}
    <ul class="list-group mt-3">
        {% for order in orders %}
            <li class="list-group-item">
                <strong>Книга:</strong> {{ order.book.title }}<br>
                <strong>Количество:</strong> {{ order.quantity }}<br>
                <strong>Дата оформления:</strong> {{ order.created_at|date:"d.m.Y H:i" }}
            </li>
        {% endfor %}
    </ul>
{% else %}
    <p class="text-muted mt-3">У вас пока нет оформленных заявок.</p>
{% endif %}
{% endblock %}


---

Шаблон регистрации

Файл: templates/shop/register.html

{% extends 'shop/base.html' %}

{% block title %}Регистрация{% endblock %}

{% block content %}
<h1 class="mb-4">Регистрация</h1>
<form method="post" class="border p-4 rounded">
    {% csrf_token %}
    {{ form.as_p }}
    <button type="submit" class="btn btn-primary">Зарегистрироваться</button>
</form>
<a href="{% url 'login' %}" class="btn btn-link mt-3">Уже есть аккаунт? Войти</a>
{% endblock %}









Для добавления футера на сайт и его стилизации с использованием Bootstrap выполните следующие шаги:


---

1. Обновление базового шаблона

Добавьте футер в base.html:

Файл: templates/shop/base.html

<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>{% block title %}Книжный магазин{% endblock %}</title>
    <!-- Подключение Bootstrap -->
    <link href="https://cdn.jsdelivr.net/npm/bootstrap@5.3.0/dist/css/bootstrap.min.css" rel="stylesheet">
    <script src="https://cdn.jsdelivr.net/npm/bootstrap@5.3.0/dist/js/bootstrap.bundle.min.js"></script>
</head>
<body>
    <nav class="navbar navbar-expand-lg navbar-light bg-light">
        <div class="container-fluid">
            <a class="navbar-brand" href="{% url 'book_list' %}">Книжный магазин</a>
            <div class="collapse navbar-collapse">
                <ul class="navbar-nav me-auto mb-2 mb-lg-0">
                    {% if user.is_authenticated %}
                        <li class="nav-item">
                            <a class="nav-link" href="{% url 'profile' %}">Профиль</a>
                        </li>
                        <li class="nav-item">
                            <a class="nav-link" href="{% url 'logout' %}">Выйти</a>
                        </li>
                    {% else %}
                        <li class="nav-item">
                            <a class="nav-link" href="{% url 'login' %}">Вход</a>
                        </li>
                        <li class="nav-item">
                            <a class="nav-link" href="{% url 'register' %}">Регистрация</a>
                        </li>
                    {% endif %}
                </ul>
            </div>
        </div>
    </nav>
    <div class="container mt-4">
        {% block content %}
        {% endblock %}
    </div>
    <!-- Футер -->
    <footer class="bg-dark text-white text-center py-4 mt-5">
        <div class="container">
            <p class="mb-1">© 2024 Книжный магазин. Все права защищены.</p>
            <p class="mb-0">Свяжитесь с нами: <a href="mailto:support@bookstore.com" class="text-white">support@bookstore.com</a></p>
        </div>
    </footer>
</body>
</html>


---

2. Стилизация футера с Bootstrap

Мы используем стандартные классы Bootstrap для футера:

bg-dark: Задает темный фон.

text-white: Белый цвет текста.

py-4: Верхний и нижний отступы (padding).

text-center: Центрирование текста.



---

3. Пример футера на сайте

Теперь ваш футер будет отображаться на каждой странице сайта и выглядеть современно благодаря использованию стилей Bootstrap.

Если хотите добавить дополнительные ссылки в футер, например, социальные сети:

<div class="d-flex justify-content-center mt-3">
    <a href="https://facebook.com" class="text-white me-3">Facebook</a>
    <a href="https://twitter.com" class="text-white me-3">Twitter</a>
    <a href="https://instagram.com" class="text-white">Instagram</a>
</div>


---

Футер будет всегда внизу страницы и подойдет для всех экранов благодаря адаптивности Bootstrap.












---

3. Проверка дизайна

Теперь ваш проект будет выглядеть более современно и удобно благодаря Bootstrap. Для кастомизации вы можете добавлять дополнительные классы или создавать собственные стили.









Чтобы добавить поиск по книгам в ваш Django-проект, выполните следующие шаги:


---

1. Обновление представления

Добавьте обработку поисковых запросов в представление списка книг.

Файл: views.py

from django.db.models import Q
from django.shortcuts import render
from .models import Book

def book_list(request):
    query = request.GET.get('q')  # Получаем значение из поля поиска
    if query:
        books = Book.objects.filter(
            Q(title__icontains=query) | 
            Q(author__icontains=query) | 
            Q(publisher__icontains=query)
        )
    else:
        books = Book.objects.all()

    return render(request, 'shop/book_list.html', {'books': books, 'query': query})


---

2. Обновление маршрутов

Убедитесь, что маршрут для списка книг (book_list) настроен.

Файл: urls.py

from django.urls import path
from . import views

urlpatterns = [
    path('', views.book_list, name='book_list'),
]


---

3. Обновление шаблона

Добавьте форму поиска в шаблон списка книг.

Файл: templates/shop/book_list.html

{% extends 'shop/base.html' %}

{% block title %}Список книг{% endblock %}

{% block content %}
<h1 class="mb-4">Книги</h1>

<!-- Форма поиска -->
<form method="get" class="d-flex mb-4">
    <input 
        type="text" 
        name="q" 
        value="{{ query|default:'' }}" 
        class="form-control me-2" 
        placeholder="Введите название, автора или издательство">
    <button type="submit" class="btn btn-outline-primary">Поиск</button>
</form>

<div class="row">
    {% if books %}
        {% for book in books %}
            <div class="col-md-4">
                <div class="card mb-4">
                    <div class="card-body">
                        <h5 class="card-title">{{ book.title }}</h5>
                        <p class="card-text">
                            Автор: {{ book.author }}<br>
                            Издательство: {{ book.publisher }}<br>
                            Год издания: {{ book.publication_year }}<br>
                            Цена: {{ book.price }} руб.<br>
                            Количество: {{ book.stock }}
                        </p>
                        {% if book.stock > 0 %}
                            <a href="{% url 'order_book' book.id %}" class="btn btn-primary">Заказать</a>
                        {% else %}
                            <a href="{% url 'request_book' %}" class="btn btn-warning">Запросить</a>
                        {% endif %}
                    </div>
                </div>
            </div>
        {% endfor %}
    {% else %}
        <p class="text-muted">Книги по вашему запросу не найдены.</p>
    {% endif %}
</div>
{% endblock %}


---

4. Пример использования

Теперь вы можете использовать строку запроса q для поиска книг. Например:

Перейдите на главную страницу / и введите запрос в поле поиска.

Результаты будут отфильтрованы по названию, автору или издательству.



---

5. Пример поиска в интерфейсе

1. Форма поиска:

Поле ввода позволяет вводить текст для поиска.

Кнопка отправляет запрос.



2. Отображение результатов:

Если книги найдены, они отображаются в карточках.

Если ничего не найдено, выводится сообщение "Книги по вашему запросу не найдены."




Теперь ваш магазин поддерживает поиск!













