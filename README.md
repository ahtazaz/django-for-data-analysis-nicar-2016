# Django for Data Analysis #

## Who are we, and what are we doing here? ##
So you know some SQL, or maybe done data analysis with R, but would prefer a more integrated way to explore and publish data. 

In addition to making front-facing web apps, Django can also be used internally as a reporting and research tool to explore a dataset. 

To do this, we'll use a publicly available dataset from the [City of L.A.'s data portal](https://data.lacity.org/) on [response times to complaints](https://data.lacity.org/browse?q=building%20and%20safety%20customer%20service%20request&sortBy=relevance&utf8=%E2%9C%93) filed to the Department of Building and Safety. We used this data to publish a [story on the varying response times](http://www.latimes.com/local/cityhall/la-me-building-safety-delay-20141219-story.html) to DBS complaints throughout the city, including especially slow response times on the Eastside.  

We'll learn how to:
- Load data from a CSV into Django models,  
- Create queries and views to help us analyze the data,
- Use these views to create tables and vizualizations that we can use for publication.  

## Setup ##

### Installation ###
```bash
$ pip install numpy==1.8.1 before requirements.txt ( we have to install numpy first due to some bugs with the numpy install )
$ pip install -r requirements.txt
```

### Database setup ###
```bash
$ createdb django_data_analysis
$ python manage.py syncdb
$ python manage.py schemamigration building_and_safety --init
$ python manage.py migrate 
```

## Getting started: The Models ##
- First, let's look at our models. There are a couple of fields we're concerned with or need more explanation.
  - **csr** - this is a complaint ID, and can serve as our primary key
  - **ladbs_inspection_district** - How LADBS divides their inspectors, one per district. After reporting this out further we found that these actually change quite a bit from year-to-year.
  - **date_received** - date the complaint was received.
  - **date_closed** - "Closed" is a funny term here. This is actually the date that an inspector visited the location of a complaint. In many cases this is when an inspector determines whether a complaint is valid or not. So really "closed" just means an inspector has visited a complaint.
  - **date_due** - arbitrary measure of time that the LADBS would like to address a complaint by. They apparently don't place a whole lot of meaning on this, and not all complaints have due dates.
  - **csr_priority** - This is the priority level that LADBS responds to the complaints. Level 1 is often life-threatening, level 2 is quality of life issues, and level 3 is more like a nuisance.
  - **area_planning_commission** - one of the seven different planning department areas the complaint is located in. We used this and the neighborhood level for most of our analysis. 
  - **response_days** - good reason to vet your data. We realized this was calculated differently for open and closed days, so was essentially useless. Instead we calculated this on our own. 


- There are also a few fields that we're calculating on load 
  - **full_address** - caching the address by concatenating the seven different types of address field in the data
  - **is_closed** - easier way to find whether a case is "closed" or not. 
  - **days_since_complaint** - basically a useful version of the **response_days** field above. Calculated by finding the difference between the date received and the date closed. 
  - **gt_30_days**, **gt_90_days**, **gt_180_days**, **more_than_one_year** - different booleans to make it easy to access cases of a certain age level. 


- load Complaints data

```bash
$ python manage.py load_complaints
```

This command creates a Complaint record for every row in our two csvs. Note that instead of saving every individual record as it's loaded, we use Django's bulk_create method to create them in batches of 500. This saves a ton of time as we're not hitting the database for row in the CSV.

## What are we looking at here? Exploring the data ##
We can use basic Django commands to get a feel the data we're looking at. 

```bash
$ python manage.py shell
```

Let's use this to answer a couple of basic questions about the data:
- How many complaints do we have total?
- How many are still unaddressed?
- How many complaints went more than a year before being addressed?

```bash
>>> from building_and_safety.models import *
>>> complaints = Complaint.objects.all()
>>> complaints.count()
71120
>>> complaints.filter(is_closed=False).count()
4060
>>> complaints.filter(more_than_one_year=True).count()
1231
```

Whats happening behind the scenes? Django's converting all of our queries to SQL, and executing them on the database. 

We can see these, and it's helpful to see exactly what Django's trying to do when you're having trouble with a query.

```bash
>>> from django.db import connection
>>> print connection.queries[-1]
{u'time': u'0.028', u'sql': u'SELECT COUNT(*) FROM "building_and_safety_complaint" WHERE "building_and_safety_complaint"."more_than_one_year" = true '}
```

We can use this for more complicated questions too. 
- How are the complaints older than one year distributed throughout the planning commissions? Are some worse than others?

```bash
>>> from django.db.models import Count
>>> complaints.filter(more_than_one_year=True).values('area_planning_commission').annotate(count=Count("csr")).order_by('-count')
[{'count': 618, 'area_planning_commission': u'East Los Angeles'}, {'count': 394, 'area_planning_commission': u'Central'}, {'count': 86, 'area_planning_commission': u'West Los Angeles'}, {'count': 60, 'area_planning_commission': u'South Valley'}, {'count': 56, 'area_planning_commission': u'South Los Angeles'}, {'count': 8, 'area_planning_commission': u'North Valley'}, {'count': 7, 'area_planning_commission': u''}, {'count': 2, 'area_planning_commission': u'Harbor'}]
```

Now we're on to something.

## Views: Documenting and replicating your work ##
One advantage of using Django is that all of our data manipulation can be stored in the views, and the output displayed in HTML templates. 

We've already created a few views that start to explore the data, so let's start exploring. 

Exit the python interpreter, if you still have it running, by pressing `CTRL-D`, and start the Django server.

```bash
$ fab rs
Removing .pyc files
[localhost] local: python manage.py runserver 0.0.0.0:8000
Validating models...

0 errors found
February 20, 2015 - 16:40:36
Django version 1.6.5, using settings 'project.settings'
Starting development server at http://0.0.0.0:8000/
Quit the server with CONTROL-C.
```

Open up the views.py file in your text editor, and look at the [ComplaintAnalysis](https://github.com/datadesk/django-for-data-analysis-nicar-2015/blob/master/building_and_safety/views.py#L71) view.

There's a lot going on here. We've already seen that, for some reason, there were more complaints open after a year in East L.A. But is this because they receive more complaints in general? Are some types of complaints addressed more quickly than others?

To find the median time to address a complaint, we used a fancy statistical method called a survival analaysis, employing a Python library called [Lifelines](http://lifelines.readthedocs.org/en/latest/index.html). This takes into account the closure rate for complaints that are still open and haven't been closed yet. 




## Visualizing the data: Let's make a map ##
Take a look at the view `ComplaintsMap`.

What's happening here? It doesn't look like anything's in the view, right? Let's take a look at the template. Open up `templates/complaints_map.html`

This template calls two other views through a `$.getJSON()` call - `open_complaints_json` and `closed_complaints_json`.

These two views are essentially the same, pulling in all geocoded complaints that waited more than a year for a response, except one pulls in closed and one pulls in open complaints. 

Both use a method on the Complaint model to export relevant fields of the model to a GeoJSON object. 

```
features = [complaint.as_geojson_dict() for complaint in complaints]
```

This refers back the `as_geojson_dict()` method on the Complaint model. This method takes properties from the model and returns them in a geojson dict format:

```
def as_geojson_dict(self):
  """
  Method to return each feature in the DB as a geojson object.
  """
  as_dict = {
      "type": "Feature",
      "geometry": {
          "type": "Point",
          "coordinates": [
              float(self.lon),
              float(self.lat)
          ]
      },
      "properties": {
          "address": self.full_address,
          "csr": self.csr,
          "date": dateformat.format(self.date_received, 'F j, Y'),
          "closed": self.get_closed_date(),
          "type": self.csr_problem_type,
          "priority": self.csr_priority
      }
  }
  return as_dict
```

We get this dict for each item in the queryset, and then form them into a geojson object, that is returned when we request the view. 

```
objects = {
    'type': "FeatureCollection",
    'features': features
}
response = json.dumps(objects)
return HttpResponse(response, content_type='text/json')
```

Load the URL [http://localhost:8000/complaints](http://localhost:8000/complaints). It should load a map in your browser. 

What is this map loading in? What's it showing us? How does this demonstrate the flexibility of using a Django backend instead of exporting our results from SQL or another database format? 

Take a look at the views for `open_complaints_json` and `closed_complaints_json` - we're filtering for all complaints older than a year, and excluding any without geocoded lat/lon points. 

Say, later in the editing process, we want to export only complaints older than 180 days, instead of a year. In Django, this is easy:

```
complaints = Complaint.objects\
        .filter(is_closed=True, gt_180_days=True).exclude(lat=None, lon=None)
```

Go ahead and change that in the `open_complaints_json` and `closed_complaints_json` views now, and reload the map. You'll see the changes instantly. Instead of having to go back to the database, re-enter our SQL, and re-export our files, Django takes care of all of that for you. 

Later, if we decide not to include the closed complaints, for example, we can simply remove that call in the [template](https://github.com/datadesk/django-for-data-analysis-nicar-2015/blob/master/templates/complaints_map.html#L207). 


## Pitfalls of Django ##
Like anything, there are a few disadvantages to using Django as a reporting framework.

- It's a lot of up-front work. 
  - There's no doubt that setting up a Django project takes a lot of time to get started and running. Models and loader scripts take planning, and we're all on a constant time crunch. And not every project merits this, sometimes you can get the results you need with a few quick pivot tables or R scripts. But the saved time in the analysis and publication phases make it well worth the input for me. 
- With huge databases (and a slow computer) the queries can take forever
  - This can be a problem with large datasets, but there are a few solutions. Caching is one. Django has a number of [built-in caching options](https://docs.djangoproject.com/en/1.7/topics/cache/) to help out with this. Another possible option is to bake out any visualizations that will be used for publication. [Django-bakery](https://github.com/datadesk/django-bakery) can help with this. 


## What can we do from here? ##
We skipped it for this lesson, but in the original story, we used [GeoDjango](https://docs.djangoproject.com/en/1.7/ref/contrib/gis/) and a GIS-enabled database to allow for geospatial queries on the data - This allowed us to relate complaints to [L.A. Times Neighborhoods](http://boundaries.latimes.com/set/la-county-neighborhoods-current/) shapes, and to census tracts and LADBS inspection districts. 

Using GeoDjango, you can also set a buffer around a point, which is what LAT Alum Ken Schwencke did for [this piece](http://homicide.latimes.com/post/westmont-homicides/) analyzing homicides in the unincorporated neighborhood of Westmont and along a two-mile stretch of Vermont Ave. called "Death Alley."




