Starting a project

```bash
django-admin startproject mysite djangotutorial
```

How to run the project

```bash
python manage.py runserver
```

Each project has a global settings file

```text
#These are app folders for each feature of the project

accounts/
transactions/
products/
inventory/
blog/
templates/ #This is where the html files are stored 
```

inside these folders contain the code needed to make the features work in the project

```text
accounts/
│
├── models.py
├── views.py
├── urls.py
├── services.py
├── validators.py
```

- Models represents the data tables, uses python classes
- Views handles the incoming and outgoing requests (sorts the data so that the html file can pull from the views file ). Views is like the traffic controller
- Urls map the urls to the views
- services is where the business logic is done
- Validators check if the data is valid

  

Here is what a template folder would look like this is where all the html is stored

```bash
accounts/
│
└── templates/
    │
    └── accounts/
        │
        └── dashboard.html
```

[[HTML, CSS and Python of Django]]

[[Database of Django and how it works]]

[[user registration - login system]]
---

[[Streamlit vs Django]] — when to reach for Django over Streamlit for a data dashboard, and the N+1 query trap
