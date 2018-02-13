# Open Street Map Data Processing
I will discuss the data analysis process in following order:
Step 1: Download miami.osm file.
Step 2: Read, analyze and fix data.
Step 3: Write data into .csv files.
Step 4: Import data into sqlite database, and analyze useful data.

## First of all, import following packages:
```python
import csv
import codecs
import pprint
import re
import xml.etree.cElementTree as ET
from collections import defaultdict
import cerberus
import schema
```

## Number of all tags by iterative parsing
```python
def count_tags(filename):
    count_tags = {}
    tree = ET.parse(filename)
    root = tree.getroot()
    for tags in root.iter():
        if tags.tag not in count_tags.keys():
            count_tags[tags.tag] = 1
        else:
            count_tags[tags.tag] += 1
    return count_tags
```
```python
tags = count_tags('Miami.osm')
pprint.pprint(tags)
```
The result shows as following:
```python
{'bounds': 1,
 'member': 59001,
 'nd': 1280433,
 'node': 1058001,
 'osm': 1,
 'relation': 1014,
 'tag': 1156695,
 'way': 137048}
```
## Checking data with regular expressions
```python
lower = re.compile(r'^([a-z]|_)*$')
lower_colon = re.compile(r'^([a-z]|_)*:([a-z]|_)*$')
problemchars = re.compile(r'[=\+/&<>;\'"\?%#$@\,\. \t\r\n]')

def key_type(element, keys):
    if element.tag == "tag":
        tag_k = element.attrib['k']
        if lower.search(tag_k):
            keys['lower'] += 1
        elif lower_colon.search(tag_k):
            keys['lower_colon'] += 1
            
        elif problemchars.search(tag_k):
            keys['problemchars'] += 1
        else:
            keys['other'] += 1        
        pass        
    return keys

def process_map(filename):
    keys = {"lower": 0, "lower_colon": 0, "problemchars": 0, "other": 0}
    for _, element in ET.iterparse(filename):
        keys = key_type(element, keys)
    return keys
```
```python
keys = process_map('miami.osm')
pprint.pprint(keys)
```
The result shows as following:
```python
{'lower': 444676, 'lower_colon': 667786, 'other': 44216, 'problemchars': 17}
```
## Number of unique users
```python
def get_user(element):
    return

def process_map(filename):
    users = set()
    for _, element in ET.iterparse(filename):
        if element.tag == "node" or element.tag == "way" or element.tag == "relation":
            uid = element.attrib['uid']
            set_uid = {uid}
            users = users ^ set_uid

    return users
```
```python
users = process_map('miami.osm')
pprint.pprint(len(users))
```
The result shows as following:
```
329
```
## Improving street names
```python
OSMFILE = "miami.osm"
street_type_re = re.compile(r'\b\S+\.?$', re.IGNORECASE)

expected = ["Street", "Avenue", "Boulevard", "Drive", "Court", "Place", "Square", "Lane", "Road", 
            "Trail", "Parkway", "Commons"]

mapping = { "ST": "Street",
            "St": "Street",
            "Sr": "Street",
            "St.": "Street",
            "st": "Street",
            "street": "Street",
            "ave": "Avenue",
            "avenue": "Avenue",
            "Ave.": "Avenue",
            "AVE": "Avenue",
            "Ave": "Avenue",
            "BLVD": "Boulevard",
            "Blvd": "Boulevard",
            "Blvd.": "Boulevard",
            "Rd.": "Road",
            "Broadwalk": "Boardwalk",
            "Cir": "Circle",
            "Cres": "Crescent",
            "Ct": "Court",
            "ct": "Court",
            "Cv": "Cove",
            "DRIVE": "Drive",
            "Dr": "Drive",
            "Dr.": "Drive",
            "HWY": "Highway",
            "Hwy": "Highway",
            "Ln": "Lane",
            "Mnr": "Manor",
            "PL": "Place",
            "Pl": "Place",
            "Pkwy": "Parkway",
            "Pt": "Point",
            "RD": "Road",
            "Rd": "Road",
            "Rd.": "Road",
            "rd": "Road",
            "road": "Road",
            "Ter": "Terrace",
            "Trce": "Trace",
            "Trl": "Trail"           
          }

def audit_street_type(street_types, street_name):
    m = street_type_re.search(street_name)
    if m:
        street_type = m.group()
        if street_type not in expected:
            street_types[street_type].add(street_name)

def is_street_name(elem):
    return (elem.attrib['k'] == "addr:street")

def audit(osmfile):
    osm_file = open(osmfile, "r")
    street_types = defaultdict(set)
    for event, elem in ET.iterparse(osm_file, events=("start",)):

        if elem.tag == "node" or elem.tag == "way":
            for tag in elem.iter("tag"):
                if is_street_name(tag):
                    audit_street_type(street_types, tag.attrib['v'])
    osm_file.close()
    return street_types

def update_name(name, mapping):
    
    st_type = street_type_re.search(name).group()
    if st_type in mapping.keys():
        update_type = mapping[st_type]
        name = name.replace(st_type, update_type)

    return name
```





