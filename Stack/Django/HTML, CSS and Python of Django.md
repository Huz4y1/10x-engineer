  

CSS for Django and how it works

```bash
myproject/

app/

    static/

        css/

            style.css

    templates/

        home.html
```

Inside style.css

```css
body {
    background-color: black;
}

h1 {
    color: white;
}
```

then inside HTML template file

```html
{% load static %}

<link rel="stylesheet"
      href="{% static 'css/style.css' %}">
```

  

Card design with HTML and CSS

```html
<div class="card">

    <h2>Gaming Mouse</h2>

    <p>£59.99</p>

</div>
```

```css
.card {

    width: 300px;

    padding: 20px;

    border-radius: 10px;

    background: white;

    box-shadow: 0 0 10px grey;

}
```

  

Django doesn’t create the design it handles the data

Django syntax in HTML files

1. {{ }} = Print a Variable

```html
<h1>{{ username }}</h1>
```

If Django recieves

```python
{
    "username": "John"
}
```

The browser gets

```html
<h1>John</h1>
```

Object fields are similar

```python
product = {
    "name": "Gaming Mouse",
    "price": 59.99
}
```

```html
<h1>{{ product.name }}</h1>    // Gaming mouse

<p>{{ product.price }}</p>     // 59.99
```

1. {% %} = Instructions

```
{% for %}
```

```
{% if %}
```

```
{% include %}
```

```
{% extends %}
```

Example:

```html
{% for product in products %}

<p>{{ product.name }}</p>

{% endfor %}
```

This tells Django

```python
for product in products:
    print(product.name)
```

  

HTML user input

```html
<input type="text" name="username">
```

HTML Form

```html
<form method="POST">
    {% csrf_token %}

    <input type="text" name="username">

    <button type="submit">
        Send
    </button>
</form>
```

Drop down menu

```html
<select name="fruit">
    <option>Apple</option>
    <option>Banana</option>
    <option>Orange</option>
</select>
```

HTML button

```css
<a href="{% url 'dashboard' %}">
    <button>Dashboard</button>
</a>
```

Tables in HTML

```html
<table>
    <thead>
        <tr>
            <th>Name</th>
            <th>Age</th>
        </tr>
    </thead>

    <tbody>
        <tr>
            <td>John</td>
            <td>25</td>
        </tr>

        <tr>
            <td>Sarah</td>
            <td>30</td>
        </tr>
    </tbody>
</table>
```

This would render to the page like this

|Name|Age|
|---|---|
|John|25|
|Sarah|30|

The <table> is what everything goes inside

The <thread> usually contains the column names

the <tr> is for table row

  

example of table rendering data and its css

```html
<table class="accepted-applicant-table">
    
    <thead>
        <tr>
            <th>Name</th>
            <th>Status</th>
            <th>Pack</th>
            <th>Interview Date</th>
            <th>Start Interview</th>
        </tr>
    </thead>

    {% for applicant in applicants %}

    <tbody>



        <tr>
            <td>{{applicant.user.username}}</td>
            <td>{{applicant.status}}</td>
            <td>{{applicant.pack}}</td>
            <td>{{applicant.interview_date}}</td>
            <td>
                <a class="button is-small is-link"
                href="{% url 'help'%}">
                    Start
                </a>
            </td>
        </tr>
    </tbody>

    {% endfor %}

</table>

```

  

```css
.accepted-applicant-table {
    width: 100%;
    border-collapse: collapse;
    font-family: Arial, sans-serif;
    background: #ffffff;
    border-radius: 12px;
    overflow: hidden;
    box-shadow: 0 6px 20px rgba(0,0,0,0.08);
}

/* Header */
.accepted-applicant-table thead {
    background: #10069f;
    color: white;
}

.accepted-applicant-table th {
    padding: 14px 16px;
    text-align: left;
    font-weight: 600;
    font-size: 0.95rem;
}

/* Body rows */
.accepted-applicant-table td {
    padding: 14px 16px;
    border-bottom: 1px solid #eee;
    color: #333;
}

/* Hover effect */
.accepted-applicant-table tbody tr:hover {
    background: #f5f7ff;
}

/* Status styling */
.accepted-applicant-table td:nth-child(2) {
    font-weight: 600;
}

```

  

[[CSS]]