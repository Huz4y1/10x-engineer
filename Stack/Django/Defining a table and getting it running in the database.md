  

This imports Django’s database tool. This is everything Django needs to create tables, columns, relationships and database queries

```python
from django.db import models
```

This creates a python class. This means create product database table. Django sees this and thinks this class represents a table

```python
from django.db import models

class Product(models.Model):
```

This creates a column. In Django a column for ID is automatically created as well which uniquely identifies each row. You have only created columns in a table no data is actually in these fields yet

```python
from django.db import models

name = models.CharField(max_length=100)
```

Example of defining a table in Django

```python
from django.db import models

class People(models.Model):
	name = models.CharField(max_field=100)
	
	#This part of the function makes it better to view all the data once the table is running in the db 
	 def __str__(self):
        return self.name      
```

Types of data in a DataBase:

CharField(max_legnth=100)

Like string just for small things like names and titles, this tells Django that 100 characters are allowed to be stored in each field

```python
from django.db import models

name = models.CharField(max_length=100)
```

TextField

this is for large amounts of texts like a description

```python
from django.db import models

description = models.TextField()
```

IntergerField()

For whole numbers (ints)

```python
from django.db import models

age = models.IntegerField()
```

FloatField()

For decimal numebers like prices and temperature

```python
from django.db import models

price = models.FloatField()
```

DecimalField

Preferred for prices allows more precision

```python
from django.db import models

stock = models.PositiveIntegerField()
```

BooleanField()

for true or false

```python
from django.db import models

is_active = models.BooleanField()
```

  

Putting data into the table you just created

making the migrations

```bash
python manage.py makemigrations
```

applying the migration

```bash
python manage.py migrate
```

You now have schema migrations

  

Now you can add data to the table but this is optional.

This command creates an empty migration file which you can edit to add the data you want.

```bash
python manage.py makemigrations app_src --empty
```

In this new empty migration file add the missing parts and alter them to the file

```bash
from django.db import migrations

def example_data(apps, schema_editor):
    Example_name_of_Table_name = apps.get_model("app_src", "Example_name_of_Table_name")

    Example_name_of_Table_name.objects.create(variable="data")

class Migration(migrations.Migration):

    dependencies = [
        ('app_src', '000X_Example_name_of_Table_name'),
    ]

    operations = [
        migrations.RunPython(example_data),
    ]
```

Then run this command

```bash
python manage.py migrate
```

To view the data in this new table just run these commands

```bash
python manage.py shell
```

```bash
Example_name_of_table_name.objects.all() 
```