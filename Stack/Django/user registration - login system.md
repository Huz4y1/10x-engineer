Your goal is to allow users to register and then log in

  

The first step is to define a table in models

```python
from django.db import models
from django.contrib.auth.models import User

class Profile(models.Model):

    ROLE_CHOICES = [
        ("admin", "Admin"),
        ("interviewer", "Interviewer"),
        ("interviewee", "Interviewee"),
    ]
		
    user = models.OneToOneField(
        User,
        on_delete=models.CASCADE
    )

    role = models.CharField(
        max_length=20,
        choices=ROLE_CHOICES
    )

    def __str__(self):
        return f"{self.user.username} - {self.role}"
```

second step is to define the urls so when one is hit it goes to the right place

```python
from django.urls import path
from . import views

urlpatterns = [

    path(
        "register/",
        views.register,
        name="register"
    ),

    path(
        "login/",
        views.login_view,
        name="login"
    ),

    path(
        "admin-dashboard/",
        views.admin_dashboard,
        name="admin_dashboard"
    ),

    path(
        "interviewer-dashboard/",
        views.interviewer_dashboard,
        name="interviewer_dashboard"
    ),

    path(
        "interviewee-dashboard/",
        views.interviewee_dashboard,
        name="interviewee_dashboard"
    ),
]
```

Create the frontend registration form for the user to complete

```html
<form method="POST">

    {% csrf_token %}

    <input
        type="text"
        name="username"
        placeholder="Username"
    >

    <input
        type="password"
        name="password"
        placeholder="Password"
    >

    <select name="role">

        <option value="admin">
            Admin
        </option>

        <option value="interviewer">
            Interviewer
        </option>

        <option value="interviewee">
            Interviewee
        </option>

    </select>

    <button type="submit">
        Register
    </button>

</form>
```

create the backend for the form so the post api request can be sent to the backend in the views file

```python
from django.shortcuts import render, redirect
from django.contrib.auth.models import User
from .models import Profile

def register(request):

    if request.method == "POST":

        username = request.POST.get("username")
        password = request.POST.get("password")
        role = request.POST.get("role")

  
        if User.objects.filter(username=username).exists():
            return render(
                request,
                "register.html",
                {"error": "Username already exists"}
            )

        user = User.objects.create_user(
            username=username,
            password=password
        )

        Profile.objects.create(
            user=user,
            role=role
        )

        return redirect("login")

    return render(request, "register.html")
```

create the frontend login form

```html
<form method="POST">

    {% csrf_token %}

    <input
        type="text"
        name="username"
        placeholder="Username"
    >

    <input
        type="password"
        name="password"
        placeholder="Password"
    >

    <button type="submit">
        Login
    </button>

</form>
```

create the backend view function for the login

```python
from django.contrib.auth import authenticate, login
from django.shortcuts import redirect, render
from django.contrib.auth import (
    authenticate,
    login
)

def login_view(request):

    print("LOGIN VIEW HIT", flush=True)

    if request.method == "POST":

        username = request.POST.get("username")
        password = request.POST.get("password")

        user = authenticate(request, username=username, password=password)

        print("AUTH:", user, flush=True)

        if user is None:
            print("AUTH FAILED", flush=True)
            return render(request, "login.html")

        login(request, user)

        profile = Profile.objects.get(user=user)

        print("ROLE:", profile.role, flush=True)

        if profile.role == "admin":
            return redirect("admin_dashboard")

        elif profile.role == "interviewer":
            return redirect("interviewer_dashboard")

        elif profile.role == "interviewee":
            return redirect("interviewee_dashboard")

        # ONLY runs if something is wrong
        print("NO ROLE MATCH - DEFAULT HOME", flush=True)
        return redirect("home")

    return render(request, "login.html")
```

create the dashboard view for each redirect

```python
from django.shortcuts import render
from django.contrib.auth.decorators import login_required

@login_required
def admin_dashboard(request):

    return render(
        request,
        "dashboard/admin-dashboard.html"
    )


@login_required
def interviewer_dashboard(request):

    return render(
        request,
        "dashboard/interviewer-dashboard.html"
    )


@login_required
def interviewee_dashboard(request):

    return render(
        request,
        "dashboard/interviewee-dashboard.html"
    )
```

now just run the migration commands