# hand-me-down 

#1. Create a Django Project:

django-admin startproject handmedown
cd handmedown


#2. Create an App:

python manage.py startapp exchange


#3. Add exchange to INSTALLED_APPS in settings.py.




---

models.py (Exchange App)

from django.db import models
from django.contrib.auth.models import User

class Item(models.Model):
    owner = models.ForeignKey(User, on_delete=models.CASCADE, related_name="items")
    name = models.CharField(max_length=100)
    description = models.TextField()
    image = models.ImageField(upload_to="item_images/")
    available = models.BooleanField(default=True)

    def __str__(self):
        return f"{self.name} - {self.owner.username}"

class ExchangeRequest(models.Model):
    from_user = models.ForeignKey(User, on_delete=models.CASCADE, related_name="sent_requests")
    to_user = models.ForeignKey(User, on_delete=models.CASCADE, related_name="received_requests")
    item = models.ForeignKey(Item, on_delete=models.CASCADE)
    status = models.CharField(
        max_length=20,
        choices=[('pending', 'Pending'), ('accepted', 'Accepted'), ('rejected', 'Rejected')],
        default='pending'
    )
    created_at = models.DateTimeField(auto_now_add=True)

    def __str__(self):
        return f"Request: {self.from_user.username} -> {self.to_user.username} for {self.item.name}"




forms.py (Exchange App)

from django import forms
from .models import Item, ExchangeRequest

class ItemForm(forms.ModelForm):
    class Meta:
        model = Item
        fields = ['name', 'description', 'image', 'available']

class ExchangeRequestForm(forms.ModelForm):
    class Meta:
        model = ExchangeRequest
        fields = ['item']




views.py (Exchange App)

from django.shortcuts import render, redirect, get_object_or_404
from django.contrib.auth.decorators import login_required
from .models import Item, ExchangeRequest
from .forms import ItemForm, ExchangeRequestForm

@login_required
def item_list(request):
    items = Item.objects.filter(available=True).exclude(owner=request.user)
    return render(request, 'exchange/item_list.html', {'items': items})

@login_required
def my_items(request):
    items = Item.objects.filter(owner=request.user)
    return render(request, 'exchange/my_items.html', {'items': items})

@login_required
def add_item(request):
    if request.method == 'POST':
        form = ItemForm(request.POST, request.FILES)
        if form.is_valid():
            item = form.save(commit=False)
            item.owner = request.user
            item.save()
            return redirect('my_items')
    else:
        form = ItemForm()
    return render(request, 'exchange/add_item.html', {'form': form})

@login_required
def request_exchange(request, item_id):
    item = get_object_or_404(Item, id=item_id)
    if request.method == 'POST':
        form = ExchangeRequestForm(request.POST)
        if form.is_valid():
            exchange_request = form.save(commit=False)
            exchange_request.from_user = request.user
            exchange_request.to_user = item.owner
            exchange_request.item = item
            exchange_request.save()
            return redirect('item_list')
    else:
        form = ExchangeRequestForm()
    return render(request, 'exchange/request_exchange.html', {'form': form, 'item': item})




urls.py (Exchange App)

from django.urls import path
from . import views

urlpatterns = [
    path('', views.item_list, name='item_list'),
    path('my-items/', views.my_items, name='my_items'),
    path('add-item/', views.add_item, name='add_item'),
    path('request-exchange/<int:item_id>/', views.request_exchange, name='request_exchange'),
]




urls.py (Main Project)

from django.contrib import admin
from django.urls import path, include

urlpatterns = [
    path('admin/', admin.site.urls),
    path('', include('exchange.urls')),  # Include exchange app URLs
]




Templates (HTML)

templates/exchange/item_list.html

<h1>Available Items</h1>
<ul>
    {% for item in items %}
    <li>
        <h3>{{ item.name }}</h3>
        <p>{{ item.description }}</p>
        <img src="{{ item.image.url }}" alt="{{ item.name }}" style="max-width: 200px;">
        <p>Owned by: {{ item.owner.username }}</p>
        <a href="{% url 'request_exchange' item.id %}">Request Exchange</a>
    </li>
    {% endfor %}
</ul>

templates/exchange/my_items.html

<h1>My Items</h1>
<ul>
    {% for item in items %}
    <li>
        <h3>{{ item.name }}</h3>
        <p>{{ item.description }}</p>
        <img src="{{ item.image.url }}" alt="{{ item.name }}" style="max-width: 200px;">
    </li>
    {% endfor %}
</ul>
<a href="{% url 'add_item' %}">Add New Item</a>

templates/exchange/add_item.html

<h1>Add Item</h1>
<form method="post" enctype="multipart/form-data">
    {% csrf_token %}
    {{ form.as_p }}
    <button type="submit">Add Item</button>
</form>

templates/exchange/request_exchange.html

<h1>Request Exchange</h1>
<p>Item: {{ item.name }}</p>
<p>Description: {{ item.description }}</p>
<p>Owner: {{ item.owner.username }}</p>

<form method="post">
    {% csrf_token %}
    {{ form.as_p }}
    <button type="submit">Send Request</button>
</form>



#Migrations and Running the Server

#1. Run Migrations:

python manage.py makemigrations
python manage.py migrate


#2. Run the Development Server:

python manage.py runserver


#3. Access the Application:
Open http://127.0.0.1:8000 in your browser.
