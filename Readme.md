
# Portland Oregon Open Street Map Data Wrangling Project

### By Niharika Jeena Shah

### About Map Area:

#### Location: Portland, Oregon, United States 

https://www.nextzen.org/metro-extracts/index.html#portland_oregon

File Size: 1581.17 MB

### Why did I choose Portland, Oregon:

I have spent significant amount of time of my life in Portland Oregon and I think it's quite natural I was inclined to choose Portland as I can call this as home city.

### Steps:

1. Generate Sample Data
2. Check different Tags Types
3. Count Tags
4. Check Users
5. Check Tag addr type
6. Audit for potential problems in Street, City, and Postcode
7. Prepare, Update and Fix Problems
9. Push data into a SQL database
10. Data Anlaysis on cleaned data

### Step 1. Generate Smaple Data


```python
import xml.etree.ElementTree as ET  # Use cElementTree or lxml if too slow

OSM_FILE = "./Data/portland_oregon.osm/portland_oregon.osm" # Replace this with your osm file
SAMPLE_FILE = "./Data/sample1.osm"

k = 10 # Parameter: take every k-th top level element

def get_element(osm_file, tags=('node', 'way', 'relation')):
    
    context = iter(ET.iterparse(osm_file, events=('start', 'end')))
    _, root = next(context)
    for event, elem in context:
        if event == 'end' and elem.tag in tags:
            yield elem
            root.clear()


with open(SAMPLE_FILE, 'a') as output:
    output.write('<?xml version="1.0" encoding="UTF-8"?>\n')
    output.write("<osm>\n")

    # Write every kth top level element
    for i, element in enumerate(get_element(OSM_FILE)):
        if i % k == 0:
            output.write(ET.tostring(element).decode("utf-8"))

    output.write("</osm>")
```

### Step 2.  Check different Tags Types


```python
import xml.etree.cElementTree as ET
import pprint
import re

filename = "./Data/portland_oregon.osm/portland_oregon.osm"
lower = re.compile(r'^([a-z]|_)*$')
lower_colon = re.compile(r'^([a-z]|_)*:([a-z]|_)*$')
problemchars = re.compile(r'[=\+/&<>;\'"\?%#$@\,\. \t\r\n]')

def key_type(element, keys):
    if element.tag == "tag":
        v = element.attrib['k']
       
        if lower.search(v):
            keys["lower"] += 1
        elif lower_colon.search(v):
            keys["lower_colon"] += 1
        elif problemchars.search(v):
            keys["problemchars"] += 1
        else:
            keys["other"] += 1
    return keys

def process_map(filename):
    keys = {"lower": 0, "lower_colon": 0, "problemchars": 0, "other": 0}
    for _, element in ET.iterparse(filename):
        keys = key_type(element, keys)

    return keys

keys = process_map(filename)
pprint.pprint(keys)
```


```python
{'lower': 2578067, 'lower_colon': 2408619, 'other': 34957, 'problemchars': 0}
```

### Step 3. Count Tags


```python
import pprint
import xml.etree.cElementTree as ET
from itertools import islice
# Let's see what types of tags we have in OSM_FILE and count their numbers

OSM_FILE = "./Data/portland_oregon.osm/portland_oregon.osm"


def count_tags(filename):
    tags = {}
    for event, elem in ET.iterparse(filename):
        if elem.tag in tags:
            tags[elem.tag] += 1
        else:
            tags[elem.tag] = 1
    return tags


pprint.pprint(count_tags(OSM_FILE))
```


```python
{'bounds':1,
 'member':70206,
 'nd':7796501,
 'node':6765370,
 'osm': 1,
 'relation': 6636,
 'tag':5021643,
 'way': 877981}
```

### Step 4. Check Users


```python
import xml.etree.cElementTree as ET

OSM_FILE = "./Data/portland_oregon.osm/portland_oregon.osm"
def get_user(element):
    users = ""
    if element.tag == "node" or element.tag == "way" or element.tag == "relation":
        users = element.get('uid')
    return users
    

def process_map(filename):
    users = set()
   
    for _, element in ET.iterparse(filename):
         if(get_user(element)):
            users.add(element.attrib['uid'])
              
    return users

users = process_map(OSM_FILE)
users
```

### Step 5. Check Tag addr type


```python
import operator
import xml.etree.cElementTree as ET

#Now in the tag 'tag' let's see what types of addresses we have inside the 'k' attribute and count them

OSM_FILE = "./Data/portland_oregon.osm/portland_oregon.osm"


address_types = {}
for _, tag in ET.iterparse(OSM_FILE):
    if tag.tag == "tag" and tag.attrib['k'].startswith("addr:"):
        k = tag.attrib['k'][5:]
        address_types.setdefault(k, 0)
        address_types[k] += 1


for k,v in sorted(address_types.items(), key=operator.itemgetter(1), reverse=True):
    print(k, ':', v)
```


```python
street : 467957
housenumber : 467780
postcode : 467545
city : 467246
state : 37351
unit : 2842
postcode:right : 660
postcode:left : 659
country : 221
full : 71
housename : 51
place : 49
floor : 8
county : 8
flats : 4
city_1 : 2
interpolation : 2
housenumber_1 : 1
suite : 1
streetnumber : 1
street_1 : 1
suburb : 1
```

#### Step 6. Audit for potential problems in Street, City, and Postcode

### Audit Street and Audit City

After running code for Auditing city I found there were 
1. Over abbreviated street names such as "St." sould be "Street" and "Rd." should be "Road"

2. Inconsistent street,and city names and postcode numbers such as "97086" should be "Happy Valley",
   and "Portlan" should be "Portland"

You can find code auditing street in audit_street.ipynb and for auditing city in audit_city.ipynb

### Audit Postcode

After auditing postcode I decided to ignore postcode update as all of the postcode were 5 digits after auditing process. you can find auditing for pstcode in audit_postcode.ipynb file

### Step 7. Prepare and Update/ Fix problems in step 6.



```python
def update_street(street, mapping_street):
    m = street_type_re.search(street)
    if m.group() not in expected_street:
        if m.group() in mapping_street.keys():
            street = re.sub(m.group(), mapping_street[m.group()], street)
    return street

def update_city(city,mapping_city):
    m = street_type_re.search(city)
    if m.group() not in expected_city:
        if m.group() in mapping_city.keys():
            street = re.sub(m.group(), mapping_city[m.group()], city)
    return city
```

After finishing audit_street.ipynb and audit_city.ipynb files I prepared Preparing_Database.ipynb file which converts 
OSM file tags to .csv files to be imported in a SQL database.

update_street and update_city function were used in Prepare_Database.ipynb file to update/fix problems with street and city names.

### Step 8. Push Data into Database

I created Push_Data_Database.ipynb to push data from .csv to SQLITE3 database.
I created a database portland_or.db, and created tables nodes, nodes_tags, ways, ways_tags, and ways_nodes and populated data in these tables from respective .csv's

### Step 9. Data Analysis on cleaned Data

### Data Overview

#### File Sizes
portland_oregon.osm     1581.17 MB

nodes.csv               619.59 MB

nodes_tags.csv          11.005 MB

ways.csv                59.310 MB

ways_nodes.csv          182.193 MB

ways_tags.csv           154.036 MB

### Connecting to DataBase


```python
import sqlite3
db = sqlite3.connect('./Data/sample_portland_oregon.db')
cur = db.cursor()

```

### Number of nodes


```python
total_nodes = '''
SELECT COUNT(*) FROM nodes;
'''
cur.execute(total_nodes)
count_nodes = cur.fetchall()
print(count_nodes)
db.commit()
```

    [(6765370,)]
    

###  Number of Nodes Tags


```python
total_nodes_tags = '''
SELECT COUNT(*) FROM nodes_tags;
'''
cur.execute(total_nodes_tags)
count_nodes_tags = cur.fetchall()
print(count_nodes_tags)
db.commit()
```

    [(305568,)]
    

### Number of ways


```python
total_ways = '''
SELECT COUNT(*) FROM ways;
'''
cur.execute(total_ways)
count_ways = cur.fetchall()
print(count_ways)
db.commit()
```

    [(877981,)]
    

### Number of ways tags


```python
total_ways_tags = '''
SELECT COUNT(*) FROM ways_tags;
'''

cur.execute(total_ways_tags)
count_ways_tags = cur.fetchall()
print(count_ways_tags)
db.commit()
```

    [(4691621,)]
    

### Number of ways nodes


```python
total_ways_nodes = '''
SELECT COUNT(*) FROM ways_nodes;
'''

cur.execute(total_ways_nodes)
count_ways_nodes = cur.fetchall()
print(count_ways_nodes)
db.commit()
```

    [(7796501,)]
    

### Top user from nodes table


```python

top_user1 = '''
SELECT count(user), user FROM nodes GROUP BY user order by count(user) desc limit 10;
'''
cur.execute(top_user1)
user1 = cur.fetchall()
print(user1)
db.commit()
```

    [(1736048, 'Peter Dobratz_pdxbuildings'), (1648266, 'lyzidiamond_imports'), (550100, 'Mele Sax-Barnett'), (499507, 'baradam'), (373422, 'Darrell_pdxbuildings'), (330838, 'cowdog'), (279327, 'Grant Humphries'), (257126, 'Peter Dobratz'), (105946, 'amillar-osm-import'), (100677, 'justin_pdxbuildings')]
    

### Total number of users

The number of users looked skewed as ways tags also had users listed. Hence, I joined nodes and ways tables to get number of users. Here we can see "Peter Dobratz_pdxbuildings" is still the top contributor, however, number 9 and 10 contributor's position has been exchanged with new number counts for their contributions.


```python
top_user2 = '''
SELECT u.user,count(u.id) as contributor FROM (SELECT user , id FROM nodes UNION
ALL SELECT user, id from ways) as u GROUP BY u.user ORDER BY contributor DESC LIMIT 10;
'''
cur.execute(top_user2)
user2 = cur.fetchall()
print(user2)
db.commit()
```

    [('Peter Dobratz_pdxbuildings', 1948281), ('lyzidiamond_imports', 1894065), ('Mele Sax-Barnett', 558499), ('baradam', 540801), ('Darrell_pdxbuildings', 430472), ('cowdog', 366306), ('Peter Dobratz', 322630), ('Grant Humphries', 294672), ('justin_pdxbuildings', 116528), ('amillar-osm-import', 106831)]
    

### Top keys in nodes_tags table

Highway has the most number of count, however, I would like to know what amenities and natural keys hold which have been recorded in osm map data by their users.


```python
top_keys = '''
SELECT count(*),key FROM nodes_tags GROUP BY key ORDER BY COUNT(key) DESC LIMIT 10;
'''
cur.execute(top_keys)
keys = cur.fetchall()
print(keys)
db.commit()

```

    [(37031, 'highway'), (30538, 'street'), (30519, 'housenumber'), (30272, 'city'), (30252, 'postcode'), (19780, 'source'), (12426, 'name'), (11288, 'amenity'), (9513, 'natural'), (5685, 'railway')]
    

### Top Amenities
'bicycle_parking' is one of the top amenities recorded in osm map data for Portland, OR. I also noticed that there were 666 Restaurants were recorded. Would like to explore restuarants more.


```python
top_amenity = '''
SELECT value, count(*) FROM nodes_tags WHERE key = 'amenity' GROUP BY value ORDER BY COUNT(*) DESC LIMIT 10;
'''
cur.execute(top_amenity)
amenity = cur.fetchall()
print(amenity)
db.commit()
```

    [('bicycle_parking', 3355), ('bench', 922), ('waste_basket', 845), ('restaurant', 666), ('place_of_worship', 578), ('fast_food', 564), ('post_box', 495), ('cafe', 390), ('drinking_water', 251), ('recycling', 224)]
    

### Top Natural


```python
top_natural ='''
SELECT value, count(*) FROM nodes_tags WHERE key = 'natural' GROUP BY value ORDER BY COUNT(*) DESC ;
'''

cur.execute(top_natural)
natural = cur.fetchall()
print(natural)
db.commit()
```

    [('tree', 9105), ('peak', 303), ('spring', 66), ('bay', 9), ('cliff', 6), ('volcano', 6), ('waterfall', 6), ('beach', 3), ('wetland', 3), ('saddle', 2), ('water', 2), ('rock', 1), ('yes', 1)]
    


```python
db.close()
```

### Conclusions

Learning Data Wrangling with Open Street Map was challenging. However, most of the code was provided with the case study to ease the process of learning and working on project, it was still interesting to explore how each snippet of code is working and interacting with other functions. 
It was also interesting how SQL helped in processing data and reducing the execution time for analysing data, whereas running the same data with python script on an OSM file took so long.

The process of cleaning data using auditing and update points out the potential problems and fixes in the data. Again it is not guranteed that cleaning process with clean the a very big dataset but it helps to analyze to a certain point where we have have some meaningful insights with less corrupted data.

### References

Technical Problems:

https://discussions.udacity.com/t/help-cleaning-data/169833/4

https://stackoverflow.com/questions/19877306/nameerror-global-name-unicode-is-not-defined-in-python-3

https://discussions.udacity.com/t/exporting-to-csv-problem-extra-letter-b-in-output/223928/19

http://stackoverflow.com/questions/3095434/inserting-newlines-in-xml-file-generated-via-xml-etree-elementtree-in-python

https://discussions.udacity.com/t/xml-sample-error-version/353325

https://docs.python.org/3/howto/pyporting.html#text-versus-binary-data

https://stackoverflow.com/questions/42480442/elementtree-typeerror-write-argument-must-be-str-not-bytes-in-python3?utm_medium=organic&utm_source=google_rich_qa&utm_campaign=google_rich_qa

https://discussions.udacity.com/t/nodes-csv-1-insert-failed-datatype-mismatch/239638/13

https://stackoverflow.com/questions/9233027/unicodedecodeerror-charmap-codec-cant-decode-byte-x-in-position-y-character?utm_medium=organic&utm_source=google_rich_qa&utm_campaign=google_rich_qa

https://discussions.udacity.com/t/use-update-name-in-shape-element-function-for-project/284766


City's Zipcodes:
https://www.zip-codes.com/city/or-portland.asp


City Names in Greater Portland Oregon:

https://en.wikipedia.org/wiki/Washington_County,_Oregon

https://en.wikipedia.org/wiki/Clackamas_County,_Oregon

https://en.wikipedia.org/wiki/Multnomah_County,_Oregon

https://en.wikipedia.org/wiki/Clark_County,_Washington#Cities



```python

```
