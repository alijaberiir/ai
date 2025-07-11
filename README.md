بسیار عالی. درخواست شما برای یک "پرامپت کامل" به این معنی است که ما تمام دانش و کدهایی را که در مراحل قبل توسعه داده‌ایم، در یک راهنمای جامع و یکپارچه جمع‌آوری کنیم. این راهنما به شما اجازه می‌دهد تا کل سیستم را از ابتدا تا انتها، با تمام جزئیات و ویژگی‌های درخواست شده، بسازید.

در ادامه، پرامپت کامل برای ساخت صفحه‌ساز ماژولار `Jelemnt` با Django و جاوا اسکریپت خالص ارائه شده است.

---

### **پرامپت جامع برای ساخت یک صفحه‌ساز ماژولار با Django و جاوا اسکریپت خالص**

#### **هدف**
ایجاد یک سیستم صفحه‌ساز تحت وب، با الهام از Elementor، که به کاربر اجازه می‌دهد به صورت بصری و با کشیدن و رها کردن (Drag & Drop)، صفحات وب را بسازد. این سیستم باید کاملاً ماژولار باشد تا بتوان به راحتی ویجت‌های جدید به آن اضافه کرد. تمام داده‌ها در بک‌اند Django ذخیره شده و خروجی نهایی به صورت HTML خالص و بدون فریم‌ورک‌های سنگین به کاربر نمایش داده می‌شود.

#### **معماری سیستم**
1.  **Backend (Django):** مسئول ذخیره‌سازی ساختار صفحات به صورت JSON، مدیریت کاربران و رندر کردن قالب‌های اصلی.
2.  **Frontend (Vanilla JavaScript):** تمام منطق صفحه‌ساز، از جمله مدیریت ویجت‌ها، درگ-اند-دراپ، ویرایش زنده و ارسال داده به بک‌اند، با جاوا اسکریپت خالص پیاده‌سازی می‌شود.
3.  **Styling (Custom CSS):** تمام استایل‌ها به صورت سفارشی و بدون وابستگی به فریم‌ورک‌هایی مانند Tailwind یا Bootstrap نوشته می‌شوند.
4.  **Modular Widgets:** هر ویجت یک ماژول مستقل است که در پوشه خود قرار دارد و شامل تنظیمات، فرم ویرایش و ظاهر خود در نوار کناری است.

---

### **بخش اول: ساختار پروژه**

ابتدا، ساختار فایل‌ها و پوشه‌ها را به شکل زیر در پروژه جنگو خود ایجاد کنید (فرض می‌کنیم نام اپ شما `jelemnt` است):

```
/ (ریشه پروژه جنگو)
├── manage.py
├── your_project_name/
|   ├── settings.py
|   └── urls.py
├── jelemnt/
|   ├── migrations/
|   ├── admin.py
|   ├── models.py
|   ├── templates/
|   |   └── jelemnt/
|   |       ├── builder.html
|   |       └── render_page.html
|   ├── urls.py
|   └── views.py
└── static/
    ├── css/
    |   ├── builder.css
    |   └── iframe.css
    ├── js/
    |   └── builder.js
    ├── iframe.html
    └── widgets/
        ├── widgets.json
        ├── _common_form.html
        ├── text/
        |   ├── config.js
        |   ├── form.html
        |   └── widget.html
        └── image/
            ├── config.js
            ├── form.html
            └── widget.html
```

---

### **بخش دوم: پیاده‌سازی Backend (Django)**

#### ۱. مدل (`jelemnt/models.py`)
```python
from django.db import models

class Page(models.Model):
    title = models.CharField(max_length=200, verbose_name="عنوان صفحه")
    slug = models.SlugField(max_length=200, unique=True, verbose_name="اسلاگ (URL)")
    content_json = models.JSONField(default=list, blank=True, verbose_name="محتوای JSON صفحه")
    
    created_at = models.DateTimeField(auto_now_add=True)
    updated_at = models.DateTimeField(auto_now=True)

    def __str__(self):
        return self.title
```

#### ۲. ادمین (`jelemnt/admin.py`)
```python
from django.contrib import admin
from .models import Page

admin.site.register(Page)
```

#### ۳. ویوها (`jelemnt/views.py`)
```python
from django.shortcuts import render, get_object_or_404
from django.http import JsonResponse
from django.views.decorators.csrf import csrf_exempt
import json
from .models import Page

def page_builder_view(request, page_id):
    page = get_object_or_404(Page, id=page_id)
    return render(request, 'jelemnt/builder.html', {'page': page})

def page_render_view(request, slug):
    page = get_object_or_404(Page, slug=slug)
    return render(request, 'jelemnt/render_page.html', {'page': page})

@csrf_exempt
def api_save_page(request, page_id):
    if request.method == 'POST':
        try:
            page = get_object_or_404(Page, id=page_id)
            data = json.loads(request.body)
            page.content_json = data.get('content_json', [])
            page.save()
            return JsonResponse({'status': 'success', 'message': 'صفحه با موفقیت ذخیره شد.'})
        except Exception as e:
            return JsonResponse({'status': 'error', 'message': str(e)}, status=400)
    return JsonResponse({'status': 'error', 'message': 'درخواست نامعتبر است.'}, status=400)
```

#### ۴. آدرس‌ها (`jelemnt/urls.py`)
```python
from django.urls import path
from . import views

urlpatterns = [
    path('builder/<int:page_id>/', views.page_builder_view, name='page_builder'),
    path('api/save/<int:page_id>/', views.api_save_page, name='api_save_page'),
    path('<slug:slug>/', views.page_render_view, name='page_render'),
]
```

---

### **بخش سوم: پیاده‌سازی سیستم ویجت ماژولار**

#### ۱. `static/widgets/widgets.json`
```json
[
  "text",
  "image"
]
```

#### ۲. فرم تنظیمات عمومی (`static/widgets/_common_form.html`)
```html
<hr style="border-color: #444; margin: 20px 0;">
<h4 style="font-size: 14px; font-weight: 600; color: #ccc; margin-bottom: 10px;">تنظیمات عمومی استایل</h4>

<label class="form-label">فاصله خارجی (Margin - top, right, bottom, left)</label>
<div class="spacing-grid">
    <input type="text" name="marginTop" class="form-control" placeholder="top">
    <input type="text" name="marginRight" class="form-control" placeholder="right">
    <input type="text" name="marginBottom" class="form-control" placeholder="bottom">
    <input type="text" name="marginLeft" class="form-control" placeholder="left">
</div>

<label class="form-label">فاصله داخلی (Padding - top, right, bottom, left)</label>
<div class="spacing-grid">
    <input type="text" name="paddingTop" class="form-control" placeholder="top">
    <input type="text" name="paddingRight" class="form-control" placeholder="right">
    <input type="text" name="paddingBottom" class="form-control" placeholder="bottom">
    <input type="text" name="paddingLeft" class="form-control" placeholder="left">
</div>
```

#### ۳. ویجت متن (`static/widgets/text/`)
*   **`widget.html`**:
    ```html
    <span>📝</span>
    <span>متن ساده</span>
    ```
*   **`form.html`**:
    ```html
    <label class="form-label">محتوا:</label>
    <textarea name="content" class="form-control" rows="5"></textarea>
    <label class="form-label">رنگ متن:</label>
    <input type="color" name="color" class="form-control">
    <label class="form-label">اندازه فونت (مثال: 16px یا 1.2rem):</label>
    <input type="text" name="fontSize" class="form-control">
    ```
*   **`config.js`**:
    ```javascript
    export default {
        name: "متن",
        type: "text",
        defaultData: {
            content: "این یک متن نمونه است. برای ویرایش کلیک کنید.",
            color: "#333333",
            fontSize: "16px",
            styles: {
                marginTop: "0px", marginRight: "0px", marginBottom: "10px", marginLeft: "0px",
                paddingTop: "0px", paddingRight: "0px", paddingBottom: "0px", paddingLeft: "0px",
            }
        },
        render: (data) => {
            const styles = `
                color: ${data.color};
                font-size: ${data.fontSize};
                margin: ${data.styles.marginTop} ${data.styles.marginRight} ${data.styles.marginBottom} ${data.styles.marginLeft};
                padding: ${data.styles.paddingTop} ${data.styles.paddingRight} ${data.styles.paddingBottom} ${data.styles.paddingLeft};
            `;
            return `<div style="${styles}">${data.content}</div>`;
        }
    };
    ```

#### ۴. ویجت تصویر (`static/widgets/image/`)
*   **`widget.html`**:
    ```html
    <span>🖼️</span>
    <span>تصویر</span>
    ```*   **`form.html`**:
    ```html
    <label class="form-label">آدرس تصویر (URL):</label>
    <input type="text" name="src" class="form-control">
    <label class="form-label">عرض (مثال: 100% یا 300px):</label>
    <input type="text" name="width" class="form-control">
    <label class="form-label">ارتفاع (مثال: auto یا 200px):</label>
    <input type="text" name="height" class="form-control">
    <label class="form-label">نحوه نمایش (Object Fit):</label>
    <select name="objectFit" class="form-control">
        <option value="fill">Fill (کشیده)</option>
        <option value="cover">Cover (پوشاندن)</option>
        <option value="contain">Contain (جا شدن)</option>
        <option value="none">None</option>
        <option value="scale-down">Scale Down</option>
    </select>
    ```
*   **`config.js`**:
    ```javascript
    export default {
        name: "تصویر",
        type: "image",
        defaultData: {
            src: "https://via.placeholder.com/400x200",
            width: "100%",
            height: "auto",
            objectFit: "cover",
            styles: {
                marginTop: "0px", marginRight: "0px", marginBottom: "10px", marginLeft: "0px",
                paddingTop: "0px", paddingRight: "0px", paddingBottom: "0px", paddingLeft: "0px",
            }
        },
        render: (data) => {
            const wrapperStyles = `
                margin: ${data.styles.marginTop} ${data.styles.marginRight} ${data.styles.marginBottom} ${data.styles.marginLeft};
                padding: ${data.styles.paddingTop} ${data.styles.paddingRight} ${data.styles.paddingBottom} ${data.styles.paddingLeft};
            `;
            const imgStyles = `
                width: ${data.width}; 
                height: ${data.height}; 
                object-fit: ${data.objectFit};
                display: block;
            `;
            return `<div style="${wrapperStyles}"><img src="${data.src}" style="${imgStyles}"></div>`;
        }
    };
    ```
---

### **بخش چهارم: پیاده‌سازی فایل‌های استاتیک اصلی**

#### ۱. `static/css/builder.css`
*کد این فایل را از پاسخ قبلی کپی کنید، کامل و بدون تغییر است.*

#### ۲. `static/css/iframe.css`
*کد این فایل را از پاسخ قبلی کپی کنید، کامل و بدون تغییر است.*

#### ۳. `static/iframe.html`
```html
<!DOCTYPE html>
<html lang="fa" dir="rtl">
<head>
    <meta charset="UTF-8">
    <title>Preview Canvas</title>
    <link rel="stylesheet" href="/static/css/iframe.css">
</head>
<body>
    <!-- محتوا توسط builder.js ساخته می‌شود -->
</body>
</html>
```

#### ۴. `static/js/builder.js`
*این فایل قلب برنامه است. کد کامل و نهایی آن را از پاسخ قبلی (پاسخ شماره ۲۱) کپی کنید. آن کد تمام منطق لازم برای درگ، حذف، ویرایش، و ذخیره را به درستی پیاده‌سازی کرده است.*

---

### **بخش پنجم: پیاده‌سازی قالب‌های Django**

#### ۱. `jelemnt/templates/jelemnt/builder.html`
*کد این فایل را از پاسخ قبلی (پاسخ شماره ۲۱) کپی کنید. این کد به درستی به تمام فایل‌های استاتیک متصل شده و داده‌ها را از جنگو دریافت می‌کند.*

#### ۲. `jelemnt/templates/jelemnt/render_page.html`
```html
{% load static %}
<!DOCTYPE html>
<html lang="fa" dir="rtl">
<head>
    <meta charset="UTF-8">
    <title>{{ page.title }}</title>
    <!-- شما می‌توانید در اینجا فایل CSS نهایی سایت خود را اضافه کنید -->
</head>
<body>
    <main id="page-content"></main>

    {{ page.content_json|json_script:"page-data" }}

    <script type="module">
        const elementsData = JSON.parse(document.getElementById('page-data').textContent);
        const WIDGET_REGISTRY = {};

        async function loadRenderers() {
            // این مسیر باید با ساختار فایل استاتیک شما مطابقت داشته باشد
            const response = await fetch("{% static 'widgets/widgets.json' %}");
            const widgetTypes = await response.json();
            for (const type of widgetTypes) {
                const configPath = `{% static 'widgets/' %}${type}/config.js`;
                const configModule = await import(configPath);
                WIDGET_REGISTRY[type] = configModule.default;
            }
        }

        function renderPage() {
            const container = document.getElementById('page-content');
            let finalHTML = '';
            for (const element of elementsData) {
                if (WIDGET_REGISTRY[element.type]) {
                    finalHTML += WIDGET_REGISTRY[element.type].render(element.data);
                }
            }
            container.innerHTML = finalHTML;
        }

        (async () => {
            await loadRenderers();
            renderPage();
        })();
    </script>
</body>
</html>
```
---

### **بخش ششم: تنظیمات نهایی و اجرا**

1.  **پیکربندی `settings.py`:** مطمئن شوید `jelemnt` در `INSTALLED_APPS` و تنظیمات `STATICFILES_DIRS` صحیح است.
2.  **پیکربندی `urls.py` پروژه اصلی:** مطمئن شوید که `jelemnt.urls` را `include` کرده‌اید.
3.  **اجرای `migrations`:** دستورات `python manage.py makemigrations jelemnt` و `python manage.py migrate` را اجرا کنید.
4.  **ساخت `superuser`:** با `python manage.py createsuperuser` یک کاربر ادمین بسازید.
5.  **ساخت یک صفحه:** از طریق پنل ادمین جنگو، یک آبجکت `Page` جدید بسازید و ID آن را یادداشت کنید.
6.  **اجرای سرور:** با `python manage.py runserver` پروژه را اجرا کنید.
7.  **مشاهده صفحه‌ساز:** به آدرس `http://127.0.0.1:8000/builder/YOUR_PAGE_ID/` بروید.





builder.html

```
{% load static %}
<!DOCTYPE html>
<html lang="fa" dir="rtl">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Website Builder - سازنده وب سایت</title>
    <link rel="stylesheet" href="{% static 'css/builder.css' %}">
    <link rel="stylesheet" href="https://cdnjs.cloudflare.com/ajax/libs/font-awesome/6.0.0/css/all.min.css">
    <link rel="preconnect" href="https://fonts.googleapis.com">
    <link rel="preconnect" href="https://fonts.gstatic.com" crossorigin>
    <link href="https://fonts.googleapis.com/css2?family=Vazirmatn:wght@300;400;500;600;700&display=swap" rel="stylesheet">
</head>
<body>
    <!-- Header -->
    <header class="header">
        <div class="header-left">
            <button class="btn btn-primary" id="save-btn">
                <i class="fas fa-save"></i>
                <span>ذخیره</span>
            </button>
            <button class="btn btn-secondary" id="preview-btn">
                <i class="fas fa-eye"></i>
                <span>پیش‌نمایش</span>
            </button>
        </div>
        
        <div class="header-center">
            <h1 class="logo">
                <i class="fas fa-cube"></i>
                Website Builder
            </h1>
        </div>
        
        <div class="header-right">
            <div class="device-selector">
                <button class="device-btn active" data-device="desktop">
                    <i class="fas fa-desktop"></i>
                </button>
                <button class="device-btn" data-device="tablet">
                    <i class="fas fa-tablet-alt"></i>
                </button>
                <button class="device-btn" data-device="mobile">
                    <i class="fas fa-mobile-alt"></i>
                </button>
            </div>
            
            <div class="theme-selector">
                <button class="theme-btn" data-theme="light">
                    <i class="fas fa-sun"></i>
                </button>
                <button class="theme-btn" data-theme="dark">
                    <i class="fas fa-moon"></i>
                </button>
            </div>
            
            <div class="actions">
                <button class="action-btn" id="undo-btn" title="بازگشت (Ctrl+Z)">
                    <i class="fas fa-undo"></i>
                </button>
                <button class="action-btn" id="redo-btn" title="تکرار (Ctrl+Y)">
                    <i class="fas fa-redo"></i>
                </button>
            </div>
        </div>
    </header>

    <!-- Main Content -->
    <main class="main-content">
        <!-- Sidebar -->
        <aside class="sidebar">
            <div class="sidebar-header">
                <div class="tabs">
                    <button class="tab active" data-tab="components">
                        <i class="fas fa-th-large"></i>
                        <span>کامپوننت‌ها</span>
                    </button>
                    <button class="tab" data-tab="properties">
                        <i class="fas fa-cog"></i>
                        <span>تنظیمات</span>
                    </button>
                    <button class="tab" data-tab="layers">
                        <i class="fas fa-layer-group"></i>
                        <span>لایه‌ها</span>
                    </button>
                </div>
            </div>

            <div class="sidebar-content">
                <!-- Components Tab -->
                <div class="tab-content active" id="components-tab">
                    <div class="search-box">
                        <input type="text" placeholder="جستجو کامپوننت..." id="component-search">
                        <i class="fas fa-search"></i>
                    </div>
                    
                    <div class="component-categories">
                        <div class="category">
                            <div class="category-header">
                                <i class="fas fa-file-text"></i>
                                <span>محتوا</span>
                                <i class="fas fa-chevron-down toggle-icon"></i>
                            </div>
                            <div class="category-content">
                                <div class="component-item" data-component="text">
                                    <div class="component-icon">
                                        <i class="fas fa-font"></i>
                                    </div>
                                    <div class="component-info">
                                        <span class="component-name">متن</span>
                                        <span class="component-desc">پاراگراف متن</span>
                                    </div>
                                </div>
                                
                                <div class="component-item" data-component="heading">
                                    <div class="component-icon">
                                        <i class="fas fa-heading"></i>
                                    </div>
                                    <div class="component-info">
                                        <span class="component-name">سرتیتر</span>
                                        <span class="component-desc">عنوان صفحه</span>
                                    </div>
                                </div>
                                
                                <div class="component-item" data-component="image">
                                    <div class="component-icon">
                                        <i class="fas fa-image"></i>
                                    </div>
                                    <div class="component-info">
                                        <span class="component-name">تصویر</span>
                                        <span class="component-desc">عکس و گرافیک</span>
                                    </div>
                                </div>
                                
                                <div class="component-item" data-component="video">
                                    <div class="component-icon">
                                        <i class="fas fa-video"></i>
                                    </div>
                                    <div class="component-info">
                                        <span class="component-name">ویدیو</span>
                                        <span class="component-desc">فیلم و انیمیشن</span>
                                    </div>
                                </div>
                            </div>
                        </div>
                        
                        <div class="category">
                            <div class="category-header">
                                <i class="fas fa-shapes"></i>
                                <span>ساختار</span>
                                <i class="fas fa-chevron-down toggle-icon"></i>
                            </div>
                            <div class="category-content">
                                <div class="component-item" data-component="container">
                                    <div class="component-icon">
                                        <i class="fas fa-square"></i>
                                    </div>
                                    <div class="component-info">
                                        <span class="component-name">کانتینر</span>
                                        <span class="component-desc">جعبه محتوا</span>
                                    </div>
                                </div>
                                
                                <div class="component-item" data-component="row">
                                    <div class="component-icon">
                                        <i class="fas fa-grip-lines"></i>
                                    </div>
                                    <div class="component-info">
                                        <span class="component-name">ردیف</span>
                                        <span class="component-desc">لایه افقی</span>
                                    </div>
                                </div>
                                
                                <div class="component-item" data-component="column">
                                    <div class="component-icon">
                                        <i class="fas fa-columns"></i>
                                    </div>
                                    <div class="component-info">
                                        <span class="component-name">ستون</span>
                                        <span class="component-desc">بخش عمودی</span>
                                    </div>
                                </div>
                                
                                <div class="component-item" data-component="section">
                                    <div class="component-icon">
                                        <i class="fas fa-square-full"></i>
                                    </div>
                                    <div class="component-info">
                                        <span class="component-name">بخش</span>
                                        <span class="component-desc">قسمت صفحه</span>
                                    </div>
                                </div>
                            </div>
                        </div>
                        
                        <div class="category">
                            <div class="category-header">
                                <i class="fas fa-mouse-pointer"></i>
                                <span>تعاملی</span>
                                <i class="fas fa-chevron-down toggle-icon"></i>
                            </div>
                            <div class="category-content">
                                <div class="component-item" data-component="button">
                                    <div class="component-icon">
                                        <i class="fas fa-mouse-pointer"></i>
                                    </div>
                                    <div class="component-info">
                                        <span class="component-name">دکمه</span>
                                        <span class="component-desc">کلیک و عمل</span>
                                    </div>
                                </div>
                                
                                <div class="component-item" data-component="form">
                                    <div class="component-icon">
                                        <i class="fas fa-wpforms"></i>
                                    </div>
                                    <div class="component-info">
                                        <span class="component-name">فرم</span>
                                        <span class="component-desc">جمع‌آوری اطلاعات</span>
                                    </div>
                                </div>
                                
                                <div class="component-item" data-component="link">
                                    <div class="component-icon">
                                        <i class="fas fa-link"></i>
                                    </div>
                                    <div class="component-info">
                                        <span class="component-name">لینک</span>
                                        <span class="component-desc">پیوند به صفحه</span>
                                    </div>
                                </div>
                            </div>
                        </div>
                    </div>
                </div>

                <!-- Properties Tab -->
                <div class="tab-content" id="properties-tab">
                    <div class="properties-panel">
                        <div class="no-selection">
                            <i class="fas fa-mouse-pointer"></i>
                            <p>برای ویرایش، یک عنصر را انتخاب کنید</p>
                        </div>
                        
                        <div class="properties-content" style="display: none;">
                            <div class="property-group">
                                <h3 class="property-group-title">عمومی</h3>
                                <div class="property-row">
                                    <label>شناسه</label>
                                    <input type="text" id="prop-id" placeholder="element-id">
                                </div>
                                <div class="property-row">
                                    <label>کلاس</label>
                                    <input type="text" id="prop-class" placeholder="css-class">
                                </div>
                            </div>
                            
                            <div class="property-group">
                                <h3 class="property-group-title">ابعاد</h3>
                                <div class="property-row">
                                    <label>عرض</label>
                                    <input type="text" id="prop-width" placeholder="auto">
                                </div>
                                <div class="property-row">
                                    <label>ارتفاع</label>
                                    <input type="text" id="prop-height" placeholder="auto">
                                </div>
                            </div>
                            
                            <div class="property-group">
                                <h3 class="property-group-title">فاصله</h3>
                                <div class="property-row">
                                    <label>حاشیه خارجی</label>
                                    <input type="text" id="prop-margin" placeholder="0">
                                </div>
                                <div class="property-row">
                                    <label>حاشیه داخلی</label>
                                    <input type="text" id="prop-padding" placeholder="0">
                                </div>
                            </div>
                            
                            <div class="property-group">
                                <h3 class="property-group-title">ظاهر</h3>
                                <div class="property-row">
                                    <label>رنگ پس‌زمینه</label>
                                    <input type="color" id="prop-background-color">
                                </div>
                                <div class="property-row">
                                    <label>رنگ متن</label>
                                    <input type="color" id="prop-color">
                                </div>
                                <div class="property-row">
                                    <label>حاشیه</label>
                                    <input type="text" id="prop-border" placeholder="none">
                                </div>
                                <div class="property-row">
                                    <label>شعاع گوشه</label>
                                    <input type="text" id="prop-border-radius" placeholder="0">
                                </div>
                            </div>
                            
                            <div class="property-group">
                                <h3 class="property-group-title">متن</h3>
                                <div class="property-row">
                                    <label>اندازه فونت</label>
                                    <input type="text" id="prop-font-size" placeholder="16px">
                                </div>
                                <div class="property-row">
                                    <label>وزن فونت</label>
                                    <select id="prop-font-weight">
                                        <option value="normal">معمولی</option>
                                        <option value="bold">ضخیم</option>
                                        <option value="lighter">نازک</option>
                                    </select>
                                </div>
                                <div class="property-row">
                                    <label>تراز متن</label>
                                    <select id="prop-text-align">
                                        <option value="right">راست</option>
                                        <option value="center">وسط</option>
                                        <option value="left">چپ</option>
                                        <option value="justify">دوطرفه</option>
                                    </select>
                                </div>
                            </div>
                        </div>
                    </div>
                </div>

                <!-- Layers Tab -->
                <div class="tab-content" id="layers-tab">
                    <div class="layers-panel">
                        <div class="layers-header">
                            <h3>لایه‌ها</h3>
                            <button class="btn btn-small" id="collapse-all">
                                <i class="fas fa-compress"></i>
                            </button>
                        </div>
                        
                        <div class="layers-tree" id="layers-tree">
                            <!-- Layers will be populated dynamically -->
                        </div>
                    </div>
                </div>
            </div>
        </aside>

        <!-- Canvas Area -->
        <div class="canvas-area">
            <div class="canvas-header">
                <div class="canvas-info">
                    <span class="page-title">صفحه اصلی</span>
                    <span class="page-status">در حال ویرایش</span>
                </div>
                
                <div class="canvas-controls">
                    <button class="control-btn" id="zoom-out">
                        <i class="fas fa-search-minus"></i>
                    </button>
                    <span class="zoom-level">100%</span>
                    <button class="control-btn" id="zoom-in">
                        <i class="fas fa-search-plus"></i>
                    </button>
                    <button class="control-btn" id="fit-screen">
                        <i class="fas fa-expand"></i>
                    </button>
                </div>
            </div>
            
            <div class="canvas-container">
                <div class="canvas-wrapper">
                    <div class="canvas" id="canvas">
                        <div class="drop-zone" id="drop-zone">
                            <div class="drop-zone-content">
                                <i class="fas fa-mouse-pointer"></i>
                                <h3>شروع طراحی</h3>
                                <p>کامپوننت‌ها را از سمت راست بکشید و اینجا رها کنید</p>
                            </div>
                        </div>
                    </div>
                </div>
            </div>
        </div>

        <!-- Properties Panel (Right) -->
        <div class="properties-sidebar">
            <div class="quick-actions">
                <button class="quick-action" id="duplicate-btn" title="کپی کردن">
                    <i class="fas fa-copy"></i>
                </button>
                <button class="quick-action" id="delete-btn" title="حذف">
                    <i class="fas fa-trash"></i>
                </button>
                <button class="quick-action" id="move-up-btn" title="بالا بردن">
                    <i class="fas fa-arrow-up"></i>
                </button>
                <button class="quick-action" id="move-down-btn" title="پایین بردن">
                    <i class="fas fa-arrow-down"></i>
                </button>
            </div>
        </div>
    </main>

    <!-- Context Menu -->
    <div class="context-menu" id="context-menu">
        <div class="context-item" data-action="copy">
            <i class="fas fa-copy"></i>
            <span>کپی</span>
        </div>
        <div class="context-item" data-action="paste">
            <i class="fas fa-paste"></i>
            <span>چسباندن</span>
        </div>
        <div class="context-item" data-action="duplicate">
            <i class="fas fa-clone"></i>
            <span>تکثیر</span>
        </div>
        <div class="context-divider"></div>
        <div class="context-item" data-action="delete">
            <i class="fas fa-trash"></i>
            <span>حذف</span>
        </div>
    </div>

    <!-- Modals -->
    <div class="modal" id="save-modal">
        <div class="modal-content">
            <div class="modal-header">
                <h3>ذخیره تغییرات</h3>
                <button class="modal-close">&times;</button>
            </div>
            <div class="modal-body">
                <p>آیا می‌خواهید تغییرات را ذخیره کنید؟</p>
            </div>
            <div class="modal-footer">
                <button class="btn btn-secondary" id="cancel-save">انصراف</button>
                <button class="btn btn-primary" id="confirm-save">ذخیره</button>
            </div>
        </div>
    </div>

    <!-- Loading Overlay -->
    <div class="loading-overlay" id="loading-overlay">
        <div class="loading-spinner">
            <i class="fas fa-spinner fa-spin"></i>
            <p>در حال ذخیره...</p>
        </div>
    </div>

    <!-- Toast Notifications -->
    <div class="toast-container" id="toast-container">
        <!-- Toasts will be populated dynamically -->
    </div>

    <!-- Scripts -->
    <script src="{% static 'js/builder.js' %}"></script>
    <script>
        // Initialize CSRF token
        const csrfToken = '{{ csrf_token }}';
        
        // Initialize page data
        const pageData = {
            id: {{ page.id|default:'null' }},
            title: '{{ page.title|default:"صفحه جدید" }}',
            slug: '{{ page.slug|default:"" }}',
            content: {{ page.content_json|default:'{}' }}
        };
        
        // Initialize builder
        document.addEventListener('DOMContentLoaded', function() {
            window.builder = new WebsiteBuilder(pageData);
        });
    </script>
</body>
</html>
```

builder.css

```
/* CSS Custom Properties for Theme System */
:root {
    /* Light Theme Colors */
    --primary-color: #3b82f6;
    --primary-hover: #2563eb;
    --secondary-color: #6b7280;
    --success-color: #10b981;
    --warning-color: #f59e0b;
    --error-color: #ef4444;
    
    /* Background Colors */
    --bg-primary: #ffffff;
    --bg-secondary: #f8fafc;
    --bg-tertiary: #f1f5f9;
    --bg-accent: #e2e8f0;
    
    /* Text Colors */
    --text-primary: #1e293b;
    --text-secondary: #64748b;
    --text-muted: #94a3b8;
    --text-inverse: #ffffff;
    
    /* Border Colors */
    --border-light: #e2e8f0;
    --border-medium: #cbd5e1;
    --border-dark: #94a3b8;
    
    /* Shadow */
    --shadow-sm: 0 1px 2px 0 rgba(0, 0, 0, 0.05);
    --shadow-md: 0 4px 6px -1px rgba(0, 0, 0, 0.1), 0 2px 4px -1px rgba(0, 0, 0, 0.06);
    --shadow-lg: 0 10px 15px -3px rgba(0, 0, 0, 0.1), 0 4px 6px -2px rgba(0, 0, 0, 0.05);
    --shadow-xl: 0 20px 25px -5px rgba(0, 0, 0, 0.1), 0 10px 10px -5px rgba(0, 0, 0, 0.04);
    
    /* Spacing */
    --spacing-xs: 4px;
    --spacing-sm: 8px;
    --spacing-md: 16px;
    --spacing-lg: 24px;
    --spacing-xl: 32px;
    --spacing-2xl: 48px;
    
    /* Border Radius */
    --radius-sm: 4px;
    --radius-md: 8px;
    --radius-lg: 12px;
    --radius-xl: 16px;
    --radius-full: 9999px;
    
    /* Transitions */
    --transition-fast: 0.15s ease-in-out;
    --transition-normal: 0.3s ease-in-out;
    --transition-slow: 0.5s ease-in-out;
    
    /* Z-Index */
    --z-dropdown: 1000;
    --z-modal: 1050;
    --z-tooltip: 1100;
    --z-toast: 1200;
}

/* Dark Theme */
[data-theme="dark"] {
    --bg-primary: #0f172a;
    --bg-secondary: #1e293b;
    --bg-tertiary: #334155;
    --bg-accent: #475569;
    
    --text-primary: #f1f5f9;
    --text-secondary: #cbd5e1;
    --text-muted: #94a3b8;
    --text-inverse: #1e293b;
    
    --border-light: #334155;
    --border-medium: #475569;
    --border-dark: #64748b;
}

/* Reset and Base Styles */
* {
    margin: 0;
    padding: 0;
    box-sizing: border-box;
}

body {
    font-family: 'Vazirmatn', -apple-system, BlinkMacSystemFont, 'Segoe UI', 'Roboto', 'Helvetica Neue', Arial, sans-serif;
    background-color: var(--bg-primary);
    color: var(--text-primary);
    line-height: 1.6;
    overflow: hidden;
    direction: rtl;
}

/* Typography */
h1, h2, h3, h4, h5, h6 {
    font-weight: 600;
    line-height: 1.4;
}

p {
    margin-bottom: var(--spacing-md);
}

/* Button Styles */
.btn {
    display: inline-flex;
    align-items: center;
    gap: var(--spacing-sm);
    padding: var(--spacing-sm) var(--spacing-md);
    border: 1px solid transparent;
    border-radius: var(--radius-md);
    font-size: 14px;
    font-weight: 500;
    text-decoration: none;
    cursor: pointer;
    transition: all var(--transition-fast);
    background: transparent;
    white-space: nowrap;
}

.btn:hover {
    transform: translateY(-1px);
    box-shadow: var(--shadow-md);
}

.btn:active {
    transform: translateY(0);
}

.btn-primary {
    background-color: var(--primary-color);
    color: var(--text-inverse);
}

.btn-primary:hover {
    background-color: var(--primary-hover);
}

.btn-secondary {
    background-color: var(--bg-secondary);
    color: var(--text-primary);
    border-color: var(--border-light);
}

.btn-secondary:hover {
    background-color: var(--bg-tertiary);
    border-color: var(--border-medium);
}

.btn-small {
    padding: var(--spacing-xs) var(--spacing-sm);
    font-size: 12px;
}

/* Header */
.header {
    display: flex;
    justify-content: space-between;
    align-items: center;
    padding: var(--spacing-md) var(--spacing-lg);
    background-color: var(--bg-primary);
    border-bottom: 1px solid var(--border-light);
    box-shadow: var(--shadow-sm);
    position: relative;
    z-index: 100;
}

.header-left,
.header-right {
    display: flex;
    align-items: center;
    gap: var(--spacing-md);
}

.header-center {
    position: absolute;
    left: 50%;
    transform: translateX(-50%);
}

.logo {
    display: flex;
    align-items: center;
    gap: var(--spacing-sm);
    font-size: 20px;
    font-weight: 700;
    color: var(--primary-color);
}

.device-selector {
    display: flex;
    background-color: var(--bg-secondary);
    border-radius: var(--radius-md);
    padding: var(--spacing-xs);
    gap: var(--spacing-xs);
}

.device-btn {
    padding: var(--spacing-sm);
    border: none;
    background: transparent;
    color: var(--text-secondary);
    border-radius: var(--radius-sm);
    cursor: pointer;
    transition: all var(--transition-fast);
}

.device-btn:hover {
    background-color: var(--bg-tertiary);
    color: var(--text-primary);
}

.device-btn.active {
    background-color: var(--primary-color);
    color: var(--text-inverse);
}

.theme-selector {
    display: flex;
    gap: var(--spacing-xs);
}

.theme-btn {
    padding: var(--spacing-sm);
    border: 1px solid var(--border-light);
    background-color: var(--bg-primary);
    color: var(--text-secondary);
    border-radius: var(--radius-sm);
    cursor: pointer;
    transition: all var(--transition-fast);
}

.theme-btn:hover {
    background-color: var(--bg-secondary);
    color: var(--text-primary);
}

.actions {
    display: flex;
    gap: var(--spacing-xs);
}

.action-btn {
    padding: var(--spacing-sm);
    border: 1px solid var(--border-light);
    background-color: var(--bg-primary);
    color: var(--text-secondary);
    border-radius: var(--radius-sm);
    cursor: pointer;
    transition: all var(--transition-fast);
}

.action-btn:hover {
    background-color: var(--bg-secondary);
    color: var(--text-primary);
}

.action-btn:disabled {
    opacity: 0.5;
    cursor: not-allowed;
}

/* Main Content Layout */
.main-content {
    display: flex;
    height: calc(100vh - 80px);
    overflow: hidden;
}

/* Sidebar */
.sidebar {
    width: 320px;
    background-color: var(--bg-secondary);
    border-left: 1px solid var(--border-light);
    display: flex;
    flex-direction: column;
    position: relative;
    z-index: 50;
}

.sidebar-header {
    padding: var(--spacing-md);
    border-bottom: 1px solid var(--border-light);
}

.tabs {
    display: flex;
    background-color: var(--bg-tertiary);
    border-radius: var(--radius-md);
    padding: var(--spacing-xs);
    gap: var(--spacing-xs);
}

.tab {
    flex: 1;
    display: flex;
    flex-direction: column;
    align-items: center;
    gap: var(--spacing-xs);
    padding: var(--spacing-sm);
    border: none;
    background: transparent;
    color: var(--text-secondary);
    border-radius: var(--radius-sm);
    cursor: pointer;
    transition: all var(--transition-fast);
    font-size: 12px;
}

.tab:hover {
    background-color: var(--bg-accent);
    color: var(--text-primary);
}

.tab.active {
    background-color: var(--primary-color);
    color: var(--text-inverse);
}

.tab i {
    font-size: 16px;
}

.sidebar-content {
    flex: 1;
    overflow-y: auto;
    padding: var(--spacing-md);
}

.tab-content {
    display: none;
}

.tab-content.active {
    display: block;
}

/* Search Box */
.search-box {
    position: relative;
    margin-bottom: var(--spacing-lg);
}

.search-box input {
    width: 100%;
    padding: var(--spacing-sm) var(--spacing-xl) var(--spacing-sm) var(--spacing-md);
    border: 1px solid var(--border-light);
    border-radius: var(--radius-md);
    background-color: var(--bg-primary);
    color: var(--text-primary);
    font-size: 14px;
    transition: all var(--transition-fast);
}

.search-box input:focus {
    outline: none;
    border-color: var(--primary-color);
    box-shadow: 0 0 0 3px rgba(59, 130, 246, 0.1);
}

.search-box i {
    position: absolute;
    right: var(--spacing-md);
    top: 50%;
    transform: translateY(-50%);
    color: var(--text-muted);
}

/* Component Categories */
.component-categories {
    display: flex;
    flex-direction: column;
    gap: var(--spacing-md);
}

.category {
    background-color: var(--bg-primary);
    border-radius: var(--radius-md);
    overflow: hidden;
    box-shadow: var(--shadow-sm);
}

.category-header {
    display: flex;
    align-items: center;
    gap: var(--spacing-sm);
    padding: var(--spacing-md);
    background-color: var(--bg-tertiary);
    cursor: pointer;
    transition: all var(--transition-fast);
}

.category-header:hover {
    background-color: var(--bg-accent);
}

.category-header i:first-child {
    color: var(--primary-color);
}

.category-header span {
    flex: 1;
    font-weight: 500;
}

.toggle-icon {
    transition: transform var(--transition-fast);
}

.category.collapsed .toggle-icon {
    transform: rotate(-90deg);
}

.category-content {
    padding: var(--spacing-sm);
    display: flex;
    flex-direction: column;
    gap: var(--spacing-sm);
}

.category.collapsed .category-content {
    display: none;
}

/* Component Items */
.component-item {
    display: flex;
    align-items: center;
    gap: var(--spacing-md);
    padding: var(--spacing-md);
    border-radius: var(--radius-md);
    cursor: grab;
    transition: all var(--transition-fast);
    user-select: none;
}

.component-item:hover {
    background-color: var(--bg-secondary);
    transform: translateX(-2px);
}

.component-item:active {
    cursor: grabbing;
}

.component-icon {
    display: flex;
    align-items: center;
    justify-content: center;
    width: 40px;
    height: 40px;
    background-color: var(--primary-color);
    color: var(--text-inverse);
    border-radius: var(--radius-md);
    font-size: 16px;
}

.component-info {
    flex: 1;
}

.component-name {
    display: block;
    font-weight: 500;
    color: var(--text-primary);
    margin-bottom: var(--spacing-xs);
}

.component-desc {
    font-size: 12px;
    color: var(--text-muted);
}

/* Properties Panel */
.properties-panel {
    height: 100%;
    display: flex;
    flex-direction: column;
}

.no-selection {
    display: flex;
    flex-direction: column;
    align-items: center;
    justify-content: center;
    height: 100%;
    text-align: center;
    color: var(--text-muted);
}

.no-selection i {
    font-size: 48px;
    margin-bottom: var(--spacing-md);
    opacity: 0.5;
}

.properties-content {
    flex: 1;
    overflow-y: auto;
}

.property-group {
    margin-bottom: var(--spacing-lg);
}

.property-group-title {
    font-size: 14px;
    font-weight: 600;
    color: var(--text-primary);
    margin-bottom: var(--spacing-md);
    padding-bottom: var(--spacing-sm);
    border-bottom: 1px solid var(--border-light);
}

.property-row {
    display: flex;
    flex-direction: column;
    gap: var(--spacing-sm);
    margin-bottom: var(--spacing-md);
}

.property-row label {
    font-size: 12px;
    font-weight: 500;
    color: var(--text-secondary);
}

.property-row input,
.property-row select {
    padding: var(--spacing-sm);
    border: 1px solid var(--border-light);
    border-radius: var(--radius-sm);
    background-color: var(--bg-primary);
    color: var(--text-primary);
    font-size: 14px;
    transition: all var(--transition-fast);
}

.property-row input:focus,
.property-row select:focus {
    outline: none;
    border-color: var(--primary-color);
    box-shadow: 0 0 0 3px rgba(59, 130, 246, 0.1);
}

.property-row input[type="color"] {
    height: 32px;
    padding: 2px;
    cursor: pointer;
}

/* Layers Panel */
.layers-panel {
    height: 100%;
    display: flex;
    flex-direction: column;
}

.layers-header {
    display: flex;
    justify-content: space-between;
    align-items: center;
    margin-bottom: var(--spacing-md);
}

.layers-header h3 {
    font-size: 16px;
    font-weight: 600;
}

.layers-tree {
    flex: 1;
    overflow-y: auto;
}

.layer-item {
    display: flex;
    align-items: center;
    gap: var(--spacing-sm);
    padding: var(--spacing-sm);
    margin-bottom: var(--spacing-xs);
    border-radius: var(--radius-sm);
    cursor: pointer;
    transition: all var(--transition-fast);
    user-select: none;
}

.layer-item:hover {
    background-color: var(--bg-primary);
}

.layer-item.selected {
    background-color: var(--primary-color);
    color: var(--text-inverse);
}

.layer-item.hidden {
    opacity: 0.5;
}

.layer-toggle {
    width: 16px;
    height: 16px;
    display: flex;
    align-items: center;
    justify-content: center;
    font-size: 10px;
    color: var(--text-muted);
    cursor: pointer;
}

.layer-icon {
    width: 20px;
    height: 20px;
    display: flex;
    align-items: center;
    justify-content: center;
    font-size: 12px;
}

.layer-name {
    flex: 1;
    font-size: 14px;
    font-weight: 500;
}

.layer-actions {
    display: flex;
    gap: var(--spacing-xs);
    opacity: 0;
    transition: opacity var(--transition-fast);
}

.layer-item:hover .layer-actions {
    opacity: 1;
}

.layer-action {
    width: 20px;
    height: 20px;
    display: flex;
    align-items: center;
    justify-content: center;
    font-size: 10px;
    color: var(--text-muted);
    cursor: pointer;
    border-radius: var(--radius-sm);
    transition: all var(--transition-fast);
}

.layer-action:hover {
    background-color: var(--bg-accent);
    color: var(--text-primary);
}

/* Canvas Area */
.canvas-area {
    flex: 1;
    display: flex;
    flex-direction: column;
    background-color: var(--bg-tertiary);
    overflow: hidden;
}

.canvas-header {
    display: flex;
    justify-content: space-between;
    align-items: center;
    padding: var(--spacing-md);
    background-color: var(--bg-primary);
    border-bottom: 1px solid var(--border-light);
}

.canvas-info {
    display: flex;
    flex-direction: column;
    gap: var(--spacing-xs);
}

.page-title {
    font-size: 16px;
    font-weight: 600;
    color: var(--text-primary);
}

.page-status {
    font-size: 12px;
    color: var(--text-muted);
}

.canvas-controls {
    display: flex;
    align-items: center;
    gap: var(--spacing-md);
}

.control-btn {
    padding: var(--spacing-sm);
    border: 1px solid var(--border-light);
    background-color: var(--bg-primary);
    color: var(--text-secondary);
    border-radius: var(--radius-sm);
    cursor: pointer;
    transition: all var(--transition-fast);
}

.control-btn:hover {
    background-color: var(--bg-secondary);
    color: var(--text-primary);
}

.zoom-level {
    font-size: 14px;
    font-weight: 500;
    color: var(--text-primary);
    min-width: 50px;
    text-align: center;
}

.canvas-container {
    flex: 1;
    padding: var(--spacing-xl);
    overflow: auto;
    display: flex;
    justify-content: center;
    align-items: flex-start;
}

.canvas-wrapper {
    position: relative;
    transition: all var(--transition-normal);
}

.canvas {
    background-color: var(--bg-primary);
    border-radius: var(--radius-lg);
    box-shadow: var(--shadow-xl);
    overflow: hidden;
    position: relative;
    min-height: 600px;
    transition: all var(--transition-normal);
}

/* Device Responsive Canvas */
.canvas[data-device="desktop"] {
    width: 1200px;
}

.canvas[data-device="tablet"] {
    width: 768px;
}

.canvas[data-device="mobile"] {
    width: 375px;
}

/* Drop Zone */
.drop-zone {
    width: 100%;
    height: 100%;
    display: flex;
    align-items: center;
    justify-content: center;
    border: 2px dashed var(--border-light);
    border-radius: var(--radius-md);
    transition: all var(--transition-fast);
    min-height: 600px;
}

.drop-zone.drag-over {
    border-color: var(--primary-color);
    background-color: rgba(59, 130, 246, 0.05);
}

.drop-zone.has-content {
    border: none;
    background: none;
}

.drop-zone-content {
    text-align: center;
    color: var(--text-muted);
}

.drop-zone-content i {
    font-size: 48px;
    margin-bottom: var(--spacing-md);
    opacity: 0.5;
}

.drop-zone-content h3 {
    font-size: 18px;
    font-weight: 600;
    margin-bottom: var(--spacing-sm);
}

.drop-zone-content p {
    font-size: 14px;
    margin: 0;
}

/* Canvas Elements */
.canvas-element {
    position: relative;
    transition: all var(--transition-fast);
    cursor: pointer;
}

.canvas-element:hover {
    outline: 2px solid var(--primary-color);
    outline-offset: 2px;
}

.canvas-element.selected {
    outline: 2px solid var(--primary-color);
    outline-offset: 2px;
}

.canvas-element.dragging {
    opacity: 0.5;
    z-index: 1000;
}

/* Element Controls */
.element-controls {
    position: absolute;
    top: -36px;
    right: 0;
    display: flex;
    gap: var(--spacing-xs);
    background-color: var(--primary-color);
    border-radius: var(--radius-sm);
    padding: var(--spacing-xs);
    opacity: 0;
    transform: translateY(-4px);
    transition: all var(--transition-fast);
    z-index: 100;
}

.canvas-element:hover .element-controls,
.canvas-element.selected .element-controls {
    opacity: 1;
    transform: translateY(0);
}

.element-control {
    width: 24px;
    height: 24px;
    display: flex;
    align-items: center;
    justify-content: center;
    background-color: rgba(255, 255, 255, 0.2);
    color: var(--text-inverse);
    border-radius: var(--radius-sm);
    cursor: pointer;
    transition: all var(--transition-fast);
    font-size: 12px;
}

.element-control:hover {
    background-color: rgba(255, 255, 255, 0.3);
}

/* Drag Indicators */
.drag-indicator {
    position: absolute;
    background-color: var(--primary-color);
    border-radius: var(--radius-sm);
    opacity: 0;
    transition: all var(--transition-fast);
    pointer-events: none;
}

.drag-indicator.show {
    opacity: 1;
}

.drag-indicator.horizontal {
    height: 2px;
    width: 100%;
    left: 0;
}

.drag-indicator.vertical {
    width: 2px;
    height: 100%;
    top: 0;
}

/* Properties Sidebar (Right) */
.properties-sidebar {
    width: 60px;
    background-color: var(--bg-secondary);
    border-right: 1px solid var(--border-light);
    display: flex;
    flex-direction: column;
    align-items: center;
    padding: var(--spacing-md);
    gap: var(--spacing-md);
}

.quick-actions {
    display: flex;
    flex-direction: column;
    gap: var(--spacing-sm);
}

.quick-action {
    width: 36px;
    height: 36px;
    display: flex;
    align-items: center;
    justify-content: center;
    background-color: var(--bg-primary);
    color: var(--text-secondary);
    border: 1px solid var(--border-light);
    border-radius: var(--radius-md);
    cursor: pointer;
    transition: all var(--transition-fast);
}

.quick-action:hover {
    background-color: var(--primary-color);
    color: var(--text-inverse);
    transform: scale(1.05);
}

.quick-action:disabled {
    opacity: 0.5;
    cursor: not-allowed;
}

.quick-action:disabled:hover {
    background-color: var(--bg-primary);
    color: var(--text-secondary);
    transform: none;
}

/* Context Menu */
.context-menu {
    position: fixed;
    background-color: var(--bg-primary);
    border: 1px solid var(--border-light);
    border-radius: var(--radius-md);
    box-shadow: var(--shadow-lg);
    padding: var(--spacing-sm);
    z-index: var(--z-dropdown);
    opacity: 0;
    transform: scale(0.95);
    transition: all var(--transition-fast);
    pointer-events: none;
}

.context-menu.show {
    opacity: 1;
    transform: scale(1);
    pointer-events: auto;
}

.context-item {
    display: flex;
    align-items: center;
    gap: var(--spacing-sm);
    padding: var(--spacing-sm) var(--spacing-md);
    border-radius: var(--radius-sm);
    cursor: pointer;
    transition: all var(--transition-fast);
    font-size: 14px;
    white-space: nowrap;
}

.context-item:hover {
    background-color: var(--bg-secondary);
}

.context-item i {
    width: 16px;
    color: var(--text-secondary);
}

.context-divider {
    height: 1px;
    background-color: var(--border-light);
    margin: var(--spacing-sm) 0;
}

/* Modals */
.modal {
    position: fixed;
    top: 0;
    left: 0;
    right: 0;
    bottom: 0;
    background-color: rgba(0, 0, 0, 0.5);
    display: flex;
    align-items: center;
    justify-content: center;
    z-index: var(--z-modal);
    opacity: 0;
    pointer-events: none;
    transition: opacity var(--transition-normal);
}

.modal.show {
    opacity: 1;
    pointer-events: auto;
}

.modal-content {
    background-color: var(--bg-primary);
    border-radius: var(--radius-lg);
    box-shadow: var(--shadow-xl);
    width: 90%;
    max-width: 500px;
    max-height: 80vh;
    overflow-y: auto;
    transform: scale(0.95);
    transition: transform var(--transition-normal);
}

.modal.show .modal-content {
    transform: scale(1);
}

.modal-header {
    display: flex;
    justify-content: space-between;
    align-items: center;
    padding: var(--spacing-lg);
    border-bottom: 1px solid var(--border-light);
}

.modal-header h3 {
    font-size: 18px;
    font-weight: 600;
}

.modal-close {
    background: none;
    border: none;
    font-size: 24px;
    color: var(--text-muted);
    cursor: pointer;
    padding: 0;
    width: 32px;
    height: 32px;
    display: flex;
    align-items: center;
    justify-content: center;
    border-radius: var(--radius-sm);
    transition: all var(--transition-fast);
}

.modal-close:hover {
    background-color: var(--bg-secondary);
    color: var(--text-primary);
}

.modal-body {
    padding: var(--spacing-lg);
}

.modal-footer {
    display: flex;
    justify-content: flex-end;
    gap: var(--spacing-md);
    padding: var(--spacing-lg);
    border-top: 1px solid var(--border-light);
}

/* Loading Overlay */
.loading-overlay {
    position: fixed;
    top: 0;
    left: 0;
    right: 0;
    bottom: 0;
    background-color: rgba(0, 0, 0, 0.5);
    display: flex;
    align-items: center;
    justify-content: center;
    z-index: var(--z-modal);
    opacity: 0;
    pointer-events: none;
    transition: opacity var(--transition-normal);
}

.loading-overlay.show {
    opacity: 1;
    pointer-events: auto;
}

.loading-spinner {
    background-color: var(--bg-primary);
    border-radius: var(--radius-lg);
    padding: var(--spacing-xl);
    text-align: center;
    color: var(--text-primary);
}

.loading-spinner i {
    font-size: 32px;
    color: var(--primary-color);
    margin-bottom: var(--spacing-md);
}

/* Toast Notifications */
.toast-container {
    position: fixed;
    top: var(--spacing-lg);
    left: var(--spacing-lg);
    z-index: var(--z-toast);
    display: flex;
    flex-direction: column;
    gap: var(--spacing-sm);
    pointer-events: none;
}

.toast {
    background-color: var(--bg-primary);
    border: 1px solid var(--border-light);
    border-radius: var(--radius-md);
    padding: var(--spacing-md);
    box-shadow: var(--shadow-lg);
    display: flex;
    align-items: center;
    gap: var(--spacing-md);
    max-width: 350px;
    opacity: 0;
    transform: translateX(-100%);
    transition: all var(--transition-normal);
    pointer-events: auto;
}

.toast.show {
    opacity: 1;
    transform: translateX(0);
}

.toast.success {
    border-color: var(--success-color);
}

.toast.error {
    border-color: var(--error-color);
}

.toast.warning {
    border-color: var(--warning-color);
}

.toast-icon {
    width: 24px;
    height: 24px;
    display: flex;
    align-items: center;
    justify-content: center;
    border-radius: var(--radius-full);
    color: var(--text-inverse);
    font-size: 12px;
}

.toast.success .toast-icon {
    background-color: var(--success-color);
}

.toast.error .toast-icon {
    background-color: var(--error-color);
}

.toast.warning .toast-icon {
    background-color: var(--warning-color);
}

.toast-content {
    flex: 1;
}

.toast-title {
    font-weight: 600;
    margin-bottom: var(--spacing-xs);
}

.toast-message {
    font-size: 14px;
    color: var(--text-secondary);
    margin: 0;
}

.toast-close {
    background: none;
    border: none;
    color: var(--text-muted);
    cursor: pointer;
    padding: 0;
    width: 20px;
    height: 20px;
    display: flex;
    align-items: center;
    justify-content: center;
    border-radius: var(--radius-sm);
    transition: all var(--transition-fast);
}

.toast-close:hover {
    background-color: var(--bg-secondary);
    color: var(--text-primary);
}

/* Animations */
@keyframes fadeIn {
    from {
        opacity: 0;
        transform: translateY(10px);
    }
    to {
        opacity: 1;
        transform: translateY(0);
    }
}

@keyframes fadeOut {
    from {
        opacity: 1;
        transform: translateY(0);
    }
    to {
        opacity: 0;
        transform: translateY(-10px);
    }
}

@keyframes slideIn {
    from {
        transform: translateX(-100%);
    }
    to {
        transform: translateX(0);
    }
}

@keyframes pulse {
    0%, 100% {
        transform: scale(1);
    }
    50% {
        transform: scale(1.05);
    }
}

/* Utility Classes */
.fade-in {
    animation: fadeIn 0.3s ease-out;
}

.fade-out {
    animation: fadeOut 0.3s ease-out;
}

.slide-in {
    animation: slideIn 0.3s ease-out;
}

.pulse {
    animation: pulse 2s infinite;
}

.hidden {
    display: none !important;
}

.invisible {
    visibility: hidden !important;
}

.sr-only {
    position: absolute;
    width: 1px;
    height: 1px;
    padding: 0;
    margin: -1px;
    overflow: hidden;
    clip: rect(0, 0, 0, 0);
    white-space: nowrap;
    border: 0;
}

/* Responsive Design */
@media (max-width: 1024px) {
    .sidebar {
        width: 280px;
    }
    
    .canvas[data-device="desktop"] {
        width: 900px;
    }
}

@media (max-width: 768px) {
    .header-center {
        display: none;
    }
    
    .sidebar {
        position: fixed;
        top: 80px;
        right: -320px;
        height: calc(100vh - 80px);
        z-index: 1000;
        transition: right var(--transition-normal);
    }
    
    .sidebar.open {
        right: 0;
    }
    
    .main-content {
        flex-direction: column;
    }
    
    .canvas-area {
        flex: 1;
    }
    
    .properties-sidebar {
        width: 100%;
        height: auto;
        flex-direction: row;
        padding: var(--spacing-sm);
    }
    
    .quick-actions {
        flex-direction: row;
    }
}

/* Print Styles */
@media print {
    .header,
    .sidebar,
    .properties-sidebar,
    .canvas-header,
    .element-controls,
    .context-menu,
    .modal,
    .loading-overlay,
    .toast-container {
        display: none !important;
    }
    
    .canvas-area {
        flex: none;
    }
    
    .canvas-container {
        padding: 0;
    }
    
    .canvas {
        box-shadow: none;
        border: 1px solid #000;
    }
}

```


 widgets/
        ├── widgets.json
        ├── _common_form.html
        ├── text/
        |   ├── config.js
        |   ├── form.html
        |   └── widget.html
        └── image/
            ├── config.js
            ├── form.html
            └── widget.html

widgets.json

```
[
  "text",
  "image",
]
```

_common_form.html

```
<hr class="my-4">
<h4 class="text-sm font-bold text-gray-400 mb-2">تنظیمات عمومی</h4>

<label class="form-label">فاصله داخلی (Padding):</label>
<select name="padding" class="form-control">
    <option value="p-0">هیچ</option>
    <option value="p-2">کم</option>
    <option value="p-4" selected>متوسط</option>
    <option value="p-8">زیاد</option>
</select>

<label class="form-label">فاصله خارجی (Margin):</label>
<select name="margin" class="form-control">
    <option value="m-0">هیچ</option>
    <option value="m-2" selected>کم</option>
    <option value="m-4">متوسط</option>
    <option value="m-8">زیاد</option>
</select>
```



text/config.js

```
export default {
    name: "متن",
    type: "text",
    defaultData: {
        // داده‌های محتوایی
        content: "این یک متن برای نمونه است.",
        // داده‌های استایل (با مقادیر پیش‌فرض از کلاس‌های Tailwind)
        styles: {
            fontSize: 'text-base',
            textColor: 'text-gray-800',
            padding: 'p-2',
            margin: 'm-2',
        }
    },
    // تابع render اکنون کلاس‌های Tailwind را ترکیب و اعمال می‌کند
    render: (data) => {
        const classes = Object.values(data.styles).join(' ');
        return `<p class="${classes}">${data.content}</p>`;
    }
};
```

text/form.html

```
<label class="form-label">محتوای متن:</label>
<textarea name="content" class="form-control" rows="5"></textarea>

<label class="form-label">اندازه فونت:</label>
<select name="fontSize" class="form-control">
    <option value="text-sm">کوچک</option>
    <option value="text-base" selected>معمولی</option>
    <option value="text-lg">بزرگ</option>
    <option value="text-2xl">خیلی بزرگ</option>
</select>

<label class="form-label">رنگ متن:</label>
<select name="textColor" class="form-control">
    <option value="text-gray-800" selected>تیره</option>
    <option value="text-white">سفید</option>
    <option value="text-blue-600">آبی</option>
    <option value="text-red-600">قرمز</option>
</select>
```

widget.html

```
<span>📝</span> متن ساده
```




# پرامپت کامل برای ادامه پروژه Website Builder

## وضعیت فعلی پروژه
من یک پروژه Website Builder با Django دارم که شامل موارد زیر است:

### ساختار Backend (Django):
- **مدل Page**: برای ذخیره صفحات با فیلدهای title، slug، content_json
- **Views**: شامل page_builder_view، page_render_view، api_save_page
- **URLs**: مسیرهای builder، api/save، و render
- **سیستم ویجت ماژولار**: ویجت‌های text و image با فایل‌های config.js، form.html، widget.html

### ساختار Frontend موجود:
- **builder.js**: منطق اصلی درگ-اند-دراپ، ویرایش، ذخیره
- **builder.css**: استایل‌های اصلی
- **iframe.html**: پیش‌نمایش
- **سیستم ویجت**: text و image widgets

## ظاهر جدید درخواستی
من یک ظاهر جدید با ویژگی‌های زیر طراحی کردم:
- **RTL Support**: پشتیبانی کامل از راست به چپ
- **Sidebar**: شامل 3 تب (Components، Properties، Layers)
- **Component Library**: دسته‌بندی شده (Content، Structure، Decorations)
- **Responsive Preview**: نمایش Desktop، Tablet، Mobile
- **Theme Customization**: تغییر رنگ با CSS root variables
- **Modern UI**: طراحی مدرن با انیمیشن‌ها

## درخواست‌های من:

### 1. ترکیب کدهای موجود با ظاهر جدید
- کدهای backend Django را حفظ کن
- منطق builder.js را با UI جدید ترکیب کن
- ویژگی‌های جدید را اضافه کن

### 2. ویژگی‌های جدید مورد نیاز:
- **سیستم Layers**: مدیریت لایه‌های elements
- **Responsive Design Tools**: ابزارهای طراحی ریسپانسیو
- **Theme System**: سیستم تم‌بندی با CSS variables
- **Component Categories**: دسته‌بندی کامپوننت‌ها
- **Advanced Properties Panel**: پنل تنظیمات پیشرفته

### 3. فایل‌های مورد نیاز:
لطفاً فایل‌های زیر را به صورت مجزا و کامل ارائه دهید:

#### Backend Files:
- `models.py` (بهبود یافته)
- `views.py` (با API های جدید)
- `urls.py` (مسیرهای جدید)

#### Frontend Files:
- `builder.html` (با UI جدید)
- `builder.css` (استایل کامل با theme system)
- `builder.js` (منطق کامل با ویژگی‌های جدید)
- `iframe.css` (استایل پیش‌نمایش)

#### Widget System:
- `widgets.json` (لیست ویجت‌های جدید)
- فایل‌های ویجت‌های جدید (button، form، section، etc.)

### 4. الزامات فنی:
- **Vanilla JavaScript**: بدون استفاده از فریمورک
- **CSS Custom Properties**: برای theme system
- **Modular Architecture**: قابلیت اضافه کردن ویجت جدید
- **RTL Support**: پشتیبانی کامل از راست به چپ
- **Responsive**: سازگار با تمام سایزهای صفحه

### 5. ویژگی‌های اضافی درخواستی:
- **Drag & Drop**: بهبود یافته برای تمام elements
- **Live Preview**: پیش‌نمایش زنده تغییرات
- **Undo/Redo**: قابلیت بازگشت/تکرار عملیات
- **Copy/Paste**: کپی و پیست elements
- **Keyboard Shortcuts**: میانبرهای کیبورد
- **Auto Save**: ذخیره خودکار

## نحوه ارائه پاسخ:
لطفاً پاسخ را به صورت زیر ارائه دهید:

1. **توضیح کلی** از تغییرات و بهبودها
2. **فایل‌های Backend** (models.py، views.py، urls.py)
3. **فایل‌های Frontend** (HTML، CSS، JS)
4. **سیستم ویجت** (ویجت‌های جدید)
5. **راهنمای نصب** و تنظیمات

## هدف نهایی:
یک Website Builder کامل و حرفه‌ای با UI مدرن که:
- قابلیت گسترش آسان داشته باشد
- تجربه کاربری عالی ارائه دهد
- کدهای تمیز و منظم داشته باشد
- تمام ویژگی‌های درخواست شده را پشتیبانی کند

---

**نکته**: لطفاً تمام کدها را کامل و بدون خلاصه‌سازی ارائه دهید تا بتوانم مستقیماً در پروژه استفاده کنم.

