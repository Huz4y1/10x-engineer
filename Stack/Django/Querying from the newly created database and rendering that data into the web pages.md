  

Here is the main flow of how it works:

Database → Query Data → View → Template → HTML Page

  

[[Database object tools]]

Assuming you now have data in your table, you want to query it.

  

Open the Views file and import the table you have created. In the function that renders the page where you want to pull data from, make variables for all columns and use the objects all function to pull each row in the column. make a dictionary with a variable and key (column name) and return that in the render function.

```python
from django.shortcuts import render
from .models import QuestionTable

def interview(request):

    questions = QuestionTable.objects.all()

    return render(
        request,
        "interview.html",
        {
            "questions": questions
        }
    )
```

pass this dictionary of pulled data into the html of the page where you want the data to be rendered

```html
{% for question in questions %}

    <p>
        {{ question.question }}
    </p>

{% endfor %}
```

  

Process flow of querying and pulling data

**The thinking process first**  
Every "I want to pull X" question answers three questions in order. Get these three and the query writes itself.  

**Question 1: Which rows do I want?** → this becomes your `filter()`**Question 2: What shape is the answer?** → one row, many rows, a single number, or specific columns**Question 3: Do I need to reach into a related table?** → this becomes your `__` (double underscore)  
Let me show the toolkit, then walk real requests through those three questions.**The filtering toolkit**  
Everything starts with `InterviewResult.objects` then you chain methods. Here's the full vocabulary.**Picking rows —** `**filter**`**,** `**exclude**`**,** `**get**`

```python
.filter(score=5)              # all rows where score is exactly 5
.exclude(score=5)             # all rows where score is NOT 5 (the opposite)
.get(id=7)                    # THE one row with id 7 — errors if 0 or 2+ match
.all()                        # every row
```

`filter` and `exclude` are opposites and both return a _collection_ (zero or more rows). `get` returns a _single object_ and is only safe when you're certain exactly one matches.  
  

#### Conditions — field lookups (the `__` suffix)

The condition inside `filter()` uses `field__lookup=value`:

```python
.filter(score=5)              # equals
.filter(score__gt=4)          # greater than       (> 4)
.filter(score__gte=4)         # greater or equal   (>= 4)
.filter(score__lt=3)          # less than          (< 3)
.filter(score__lte=3)         # less or equal      (<= 3)
.filter(score__isnull=True)   # score is empty / not yet scored
.filter(score__in=[4, 5, 6])  # score is any of these values
```

  

For text fields:

```python
.filter(notes__icontains="strong")   # notes contain "strong" (i = case-insensitive)
.filter(notes__exact="")              # notes are exactly empty
.filter(feedback__istartswith="good") # feedback starts with "good"
```

  

#### Combining conditions

Multiple conditions in one `filter` = AND (all must be true):

```python
.filter(score__gte=4, application_id=3)   # score >= 4 AND belongs to application 3
```

  
For OR you need `Q` objects (bring these in when you need them):

```python
from django.db.models import Q
.filter(Q(score=1) | Q(score=6))          # score is 1 OR 6   ( | means OR )
```