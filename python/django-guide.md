# Django Development Guide

**Generated:** 2024-11-09  
**Version:** 1.0  
**Technology:** Python + Django  
**Author:** SWO Team

## Overview
Comprehensive guide for Django web development following Django best practices and conventions.

## Project Setup

### Project Structure
```
myproject/
├── manage.py
├── myproject/
│   ├── __init__.py
│   ├── settings/
│   │   ├── __init__.py
│   │   ├── base.py
│   │   ├── development.py
│   │   └── production.py
│   ├── urls.py
│   ├── wsgi.py
│   └── asgi.py
├── apps/
│   ├── users/
│   │   ├── migrations/
│   │   ├── __init__.py
│   │   ├── models.py
│   │   ├── views.py
│   │   ├── serializers.py
│   │   ├── urls.py
│   │   ├── admin.py
│   │   └── tests.py
│   └── products/
├── static/
├── media/
├── templates/
└── requirements/
    ├── base.txt
    ├── development.txt
    └── production.txt
```

## Models Best Practices

### Model Definition
```python
from django.db import models
from django.core.validators import MinValueValidator, MaxValueValidator
from django.utils.translation import gettext_lazy as _
from django.utils import timezone

class TimeStampedModel(models.Model):
    """Abstract base model with created/updated timestamps."""
    created_at = models.DateTimeField(auto_now_add=True)
    updated_at = models.DateTimeField(auto_now=True)
    
    class Meta:
        abstract = True

class User(TimeStampedModel):
    """User model with custom fields."""
    email = models.EmailField(
        _('email address'),
        unique=True,
        db_index=True
    )
    first_name = models.CharField(_('first name'), max_length=150)
    last_name = models.CharField(_('last name'), max_length=150)
    is_active = models.BooleanField(_('active'), default=True)
    date_joined = models.DateTimeField(_('date joined'), default=timezone.now)
    
    class Meta:
        verbose_name = _('user')
        verbose_name_plural = _('users')
        ordering = ['-date_joined']
        indexes = [
            models.Index(fields=['email', 'is_active']),
        ]
    
    def __str__(self) -> str:
        return self.email
    
    def get_full_name(self) -> str:
        """Return the full name."""
        return f"{self.first_name} {self.last_name}".strip()

class Product(TimeStampedModel):
    """Product model."""
    name = models.CharField(_('name'), max_length=200)
    slug = models.SlugField(_('slug'), unique=True)
    description = models.TextField(_('description'), blank=True)
    price = models.DecimalField(
        _('price'),
        max_digits=10,
        decimal_places=2,
        validators=[MinValueValidator(0)]
    )
    stock = models.PositiveIntegerField(
        _('stock'),
        default=0
    )
    is_available = models.BooleanField(_('available'), default=True)
    category = models.ForeignKey(
        'Category',
        on_delete=models.PROTECT,
        related_name='products'
    )
    
    class Meta:
        verbose_name = _('product')
        verbose_name_plural = _('products')
        ordering = ['name']
    
    def __str__(self) -> str:
        return self.name
    
    def save(self, *args, **kwargs):
        """Override save to auto-generate slug."""
        if not self.slug:
            from django.utils.text import slugify
            self.slug = slugify(self.name)
        super().save(*args, **kwargs)
```

### Managers and QuerySets

```python
from django.db import models
from django.db.models import Q, F, Count

class ProductQuerySet(models.QuerySet):
    """Custom QuerySet for Product model."""
    
    def available(self):
        """Filter available products."""
        return self.filter(is_available=True, stock__gt=0)
    
    def by_category(self, category_slug: str):
        """Filter by category slug."""
        return self.filter(category__slug=category_slug)
    
    def search(self, query: str):
        """Search products by name or description."""
        return self.filter(
            Q(name__icontains=query) | Q(description__icontains=query)
        )

class ProductManager(models.Manager):
    """Custom manager for Product model."""
    
    def get_queryset(self):
        return ProductQuerySet(self.model, using=self._db)
    
    def available(self):
        return self.get_queryset().available()
    
    def by_category(self, category_slug: str):
        return self.get_queryset().by_category(category_slug)
    
    def search(self, query: str):
        return self.get_queryset().search(query)
    
    def with_stock_count(self):
        """Annotate with stock count."""
        return self.get_queryset().annotate(
            total_stock=F('stock')
        )

# Usage in model
class Product(TimeStampedModel):
    # ... fields ...
    
    objects = ProductManager()
    
    # Usage:
    # Product.objects.available()
    # Product.objects.search('django')
```

## Views

### Class-Based Views (CBV)
```python
from django.views.generic import ListView, DetailView, CreateView, UpdateView
from django.contrib.auth.mixins import LoginRequiredMixin, PermissionRequiredMixin
from django.urls import reverse_lazy
from django.db.models import Q

class ProductListView(ListView):
    """List all available products."""
    model = Product
    template_name = 'products/product_list.html'
    context_object_name = 'products'
    paginate_by = 20
    
    def get_queryset(self):
        queryset = Product.objects.available()
        
        # Search
        query = self.request.GET.get('q')
        if query:
            queryset = queryset.search(query)
        
        # Category filter
        category = self.request.GET.get('category')
        if category:
            queryset = queryset.by_category(category)
        
        return queryset.select_related('category')
    
    def get_context_data(self, **kwargs):
        context = super().get_context_data(**kwargs)
        context['categories'] = Category.objects.all()
        return context

class ProductDetailView(DetailView):
    """Product detail view."""
    model = Product
    template_name = 'products/product_detail.html'
    context_object_name = 'product'
    
    def get_queryset(self):
        return super().get_queryset().select_related('category')

class ProductCreateView(LoginRequiredMixin, PermissionRequiredMixin, CreateView):
    """Create new product."""
    model = Product
    template_name = 'products/product_form.html'
    fields = ['name', 'description', 'price', 'stock', 'category']
    permission_required = 'products.add_product'
    success_url = reverse_lazy('products:list')
    
    def form_valid(self, form):
        form.instance.created_by = self.request.user
        return super().form_valid(form)
```

### Function-Based Views (FBV)
```python
from django.shortcuts import render, get_object_or_404, redirect
from django.contrib.auth.decorators import login_required, permission_required
from django.contrib import messages
from django.http import JsonResponse

@login_required
def product_list(request):
    """List all products."""
    products = Product.objects.available().select_related('category')
    
    # Search
    query = request.GET.get('q')
    if query:
        products = products.search(query)
    
    context = {
        'products': products,
        'query': query,
    }
    return render(request, 'products/product_list.html', context)

@login_required
@permission_required('products.change_product', raise_exception=True)
def product_update(request, pk):
    """Update product."""
    product = get_object_or_404(Product, pk=pk)
    
    if request.method == 'POST':
        form = ProductForm(request.POST, instance=product)
        if form.is_valid():
            form.save()
            messages.success(request, 'Product updated successfully!')
            return redirect('products:detail', pk=product.pk)
    else:
        form = ProductForm(instance=product)
    
    return render(request, 'products/product_form.html', {
        'form': form,
        'product': product,
    })

# API endpoint
from django.views.decorators.http import require_http_methods
import json

@require_http_methods(["GET", "POST"])
def api_products(request):
    """API endpoint for products."""
    if request.method == 'GET':
        products = Product.objects.available().values(
            'id', 'name', 'price', 'stock'
        )
        return JsonResponse(list(products), safe=False)
    
    elif request.method == 'POST':
        data = json.loads(request.body)
        product = Product.objects.create(**data)
        return JsonResponse({'id': product.id, 'name': product.name}, status=201)
```

## Forms

```python
from django import forms
from django.core.exceptions import ValidationError

class ProductForm(forms.ModelForm):
    """Form for creating/updating products."""
    
    class Meta:
        model = Product
        fields = ['name', 'description', 'price', 'stock', 'category', 'is_available']
        widgets = {
            'description': forms.Textarea(attrs={'rows': 4}),
            'price': forms.NumberInput(attrs={'step': '0.01'}),
        }
    
    def clean_price(self):
        """Validate price is positive."""
        price = self.cleaned_data.get('price')
        if price and price <= 0:
            raise ValidationError('Price must be greater than zero.')
        return price
    
    def clean(self):
        """Cross-field validation."""
        cleaned_data = super().clean()
        stock = cleaned_data.get('stock')
        is_available = cleaned_data.get('is_available')
        
        if is_available and stock == 0:
            raise ValidationError(
                'Product cannot be available with zero stock.'
            )
        
        return cleaned_data
```

## Django REST Framework

```python
from rest_framework import serializers, viewsets, filters
from rest_framework.decorators import action
from rest_framework.response import Response
from rest_framework.permissions import IsAuthenticated
from django_filters.rest_framework import DjangoFilterBackend

# Serializers
class ProductSerializer(serializers.ModelSerializer):
    """Serializer for Product model."""
    category_name = serializers.CharField(source='category.name', read_only=True)
    
    class Meta:
        model = Product
        fields = [
            'id', 'name', 'slug', 'description', 'price',
            'stock', 'is_available', 'category', 'category_name',
            'created_at', 'updated_at'
        ]
        read_only_fields = ['slug', 'created_at', 'updated_at']
    
    def validate_price(self, value):
        """Validate price."""
        if value <= 0:
            raise serializers.ValidationError("Price must be greater than zero.")
        return value

# ViewSets
class ProductViewSet(viewsets.ModelViewSet):
    """ViewSet for Product model."""
    queryset = Product.objects.all()
    serializer_class = ProductSerializer
    permission_classes = [IsAuthenticated]
    filter_backends = [DjangoFilterBackend, filters.SearchFilter, filters.OrderingFilter]
    filterset_fields = ['category', 'is_available']
    search_fields = ['name', 'description']
    ordering_fields = ['price', 'created_at']
    ordering = ['-created_at']
    
    def get_queryset(self):
        """Filter queryset based on user."""
        queryset = super().get_queryset()
        if not self.request.user.is_staff:
            queryset = queryset.filter(is_available=True)
        return queryset.select_related('category')
    
    @action(detail=False, methods=['get'])
    def available(self, request):
        """Get available products."""
        products = self.get_queryset().filter(is_available=True, stock__gt=0)
        serializer = self.get_serializer(products, many=True)
        return Response(serializer.data)
    
    @action(detail=True, methods=['post'])
    def update_stock(self, request, pk=None):
        """Update product stock."""
        product = self.get_object()
        quantity = request.data.get('quantity', 0)
        product.stock += quantity
        product.save()
        return Response({'stock': product.stock})
```

## Admin

```python
from django.contrib import admin
from django.utils.html import format_html

@admin.register(Product)
class ProductAdmin(admin.ModelAdmin):
    """Admin interface for Product model."""
    list_display = ['name', 'category', 'price_display', 'stock', 'is_available', 'created_at']
    list_filter = ['is_available', 'category', 'created_at']
    search_fields = ['name', 'description']
    prepopulated_fields = {'slug': ('name',)}
    readonly_fields = ['created_at', 'updated_at']
    date_hierarchy = 'created_at'
    list_per_page = 50
    
    fieldsets = (
        ('Basic Information', {
            'fields': ('name', 'slug', 'description', 'category')
        }),
        ('Pricing & Stock', {
            'fields': ('price', 'stock', 'is_available')
        }),
        ('Timestamps', {
            'fields': ('created_at', 'updated_at'),
            'classes': ('collapse',)
        }),
    )
    
    def price_display(self, obj):
        """Display formatted price."""
        return format_html('<strong>${}</strong>', obj.price)
    price_display.short_description = 'Price'
    
    actions = ['make_available', 'make_unavailable']
    
    def make_available(self, request, queryset):
        """Mark products as available."""
        updated = queryset.update(is_available=True)
        self.message_user(request, f'{updated} products marked as available.')
    make_available.short_description = 'Mark selected as available'
    
    def make_unavailable(self, request, queryset):
        """Mark products as unavailable."""
        updated = queryset.update(is_available=False)
        self.message_user(request, f'{updated} products marked as unavailable.')
    make_unavailable.short_description = 'Mark selected as unavailable'
```

## Testing

```python
from django.test import TestCase, Client
from django.urls import reverse

class ProductModelTest(TestCase):
    """Test Product model."""
    
    def setUp(self):
        self.category = Category.objects.create(name='Electronics', slug='electronics')
        self.product = Product.objects.create(
            name='Test Product',
            price=99.99,
            stock=10,
            category=self.category
        )
    
    def test_product_creation(self):
        """Test product is created correctly."""
        self.assertEqual(self.product.name, 'Test Product')
        self.assertEqual(str(self.product), 'Test Product')
    
    def test_slug_generation(self):
        """Test slug is auto-generated."""
        self.assertEqual(self.product.slug, 'test-product')

class ProductViewTest(TestCase):
    """Test Product views."""
    
    def setUp(self):
        self.client = Client()
        self.category = Category.objects.create(name='Electronics')
        self.product = Product.objects.create(
            name='Test Product',
            price=99.99,
            category=self.category
        )
    
    def test_product_list_view(self):
        """Test product list view."""
        response = self.client.get(reverse('products:list'))
        self.assertEqual(response.status_code, 200)
        self.assertContains(response, 'Test Product')
    
    def test_product_detail_view(self):
        """Test product detail view."""
        response = self.client.get(
            reverse('products:detail', args=[self.product.pk])
        )
        self.assertEqual(response.status_code, 200)
        self.assertContains(response, self.product.name)
```

---

*This guide is part of SWO OneClick Skills - Created by SoftwareOne Team*
