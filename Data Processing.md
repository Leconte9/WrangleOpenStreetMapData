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
Using below for loop to show all name updating
```python
st_types = audit(OSMFILE)
for st_type, ways in st_types.items():
    for name in ways:
        better_name = update_name(name, mapping)
        print(name, "=>", better_name)
```
The changing shows as following:
```
Seminole Blvd. => Seminole Boulevard
Cordova Rd => Cordova Road
NW 58th St. => NW 58th Street
NW 87th Ave => NW 87th Avenue
Royal Fern Dr => Royal Fern Drive
Beeline Hwy => Beeline Highway
S. Military Trl => S. Military Trail
E Melrose Cir => E Melrose Circle
SW 15th Ct => SW 15th Court
```

## Preparing for database miami.db
```python
OSM_PATH = "miami.osm"

NODES_PATH = "nodes.csv"
NODE_TAGS_PATH = "nodes_tags.csv"
WAYS_PATH = "ways.csv"
WAY_NODES_PATH = "ways_nodes.csv"
WAY_TAGS_PATH = "ways_tags.csv"

LOWER_COLON = re.compile(r'^([a-z]|_)+:([a-z]|_)+')
PROBLEMCHARS = re.compile(r'[=\+/&<>;\'"\?%#$@\,\. \t\r\n]')

SCHEMA = schema.schema

# Make sure the fields order in the csvs matches the column order in the sql table schema
NODE_FIELDS = ['id', 'lat', 'lon', 'user', 'uid', 'version', 'changeset', 'timestamp']
NODE_TAGS_FIELDS = ['id', 'key', 'value', 'type']
WAY_FIELDS = ['id', 'user', 'uid', 'version', 'changeset', 'timestamp']
WAY_TAGS_FIELDS = ['id', 'key', 'value', 'type']
WAY_NODES_FIELDS = ['id', 'node_id', 'position']

def shape_element(element, node_attr_fields=NODE_FIELDS, way_attr_fields=WAY_FIELDS,
                  problem_chars=PROBLEMCHARS, default_tag_type='regular'):
    """Clean and shape node or way XML element to Python dict"""
    node_attribs = {}
    way_attribs = {}
    way_nodes = []
    tags = []  # Handle secondary tags the same way for both node and way elements
    
    LOWER_COLON_self = re.compile(r'^([a-z]|_)+')

    if element.tag == 'node':
        for nodeFields in node_attr_fields:
            if nodeFields == 'id' or nodeFields == 'uid' or nodeFields == 'changeset':
                node_attribs[nodeFields] = int(element.attrib[nodeFields])
            elif nodeFields in ['lat', 'lon']:
                node_attribs[nodeFields] = float(element.attrib[nodeFields])
            elif nodeFields in ['user', 'version', 'timestamp']:
                node_attribs[nodeFields] = fix_b(str(element.attrib[nodeFields]))
        for tag in element.iter('tag'):
            tag_attribs = {}
            tag_k = tag.attrib['k']

            if ":" not in tag_k:
                tag_type = default_tag_type
                type_key = tag_k
            else:
                tag_type_colon = re.split(":", tag_k, 1)
                tag_type = tag_type_colon[0]
                type_key = tag_type_colon[1]
            
            tag_attribs[NODE_TAGS_FIELDS[0]] = int(element.attrib['id'])
            tag_attribs[NODE_TAGS_FIELDS[1]] = fix_b(str(type_key))
            tag_attribs[NODE_TAGS_FIELDS[2]] = fix_b(str(tag.attrib['v']))
            tag_attribs[NODE_TAGS_FIELDS[3]] = fix_b(str(tag_type))
            
            tags.append(tag_attribs)       
        return {'node': node_attribs, 'node_tags': tags}
        
    elif element.tag == 'way':
        for wayFields in way_attr_fields:
            if wayFields in ['id', 'uid', 'changeset']:
                way_attribs[wayFields] = int(element.attrib[wayFields])
            elif wayFields in ['user', 'version', 'timestamp']:
                way_attribs[wayFields] = fix_b(str(element.attrib[wayFields]))
            
        for tag in element.iter('tag'):
            tag_attribs = {}
            tag_k = tag.attrib['k']
            if ":" not in tag_k:
                tag_type = default_tag_type
                type_key = tag_k
            else:
                tag_type_colon = re.split(":", tag_k, 1)
                tag_type = tag_type_colon[0]
                type_key = tag_type_colon[1]           
            tag_attribs[WAY_TAGS_FIELDS[0]] = int(element.attrib['id'])
            tag_attribs[WAY_TAGS_FIELDS[1]] = fix_b(str(type_key))
            tag_attribs[WAY_TAGS_FIELDS[2]] = fix_b(str(tag.attrib['v']))
            tag_attribs[WAY_TAGS_FIELDS[3]] = fix_b(str(tag_type))            
            tags.append(tag_attribs)       
        position = 0
        for wayNodes in element.iter('nd'):
            nd_attribs = {}
            nd_attribs[WAY_NODES_FIELDS[0]] = int(element.attrib['id'])
            nd_attribs[WAY_NODES_FIELDS[1]] = int(wayNodes.attrib['ref'])
            nd_attribs[WAY_NODES_FIELDS[2]] = int(position)
            position += 1            
            way_nodes.append(nd_attribs)        
        return {'way': way_attribs, 'way_nodes': way_nodes, 'way_tags': tags}

def fix_b(string):
    if "b'" in string:
        newStr = string.split("'")[1]
        return newStr
    else:
        return string
        
# ================================================== #
#               Helper Functions                     #
# ================================================== #
def get_element(osm_file, tags=('node', 'way', 'relation')):
    """Yield element if it is the right type of tag"""

    context = ET.iterparse(osm_file, events=('start', 'end'))
    _, root = next(context)
    for event, elem in context:
        if event == 'end' and elem.tag in tags:
            yield elem
            root.clear()

def validate_element(element, validator, schema=SCHEMA):
    """Raise ValidationError if element does not match schema"""
    if validator.validate(element, schema) is not True:
        field, errors = next(validator.errors.iteritems())
        message_string = "\nElement of type '{0}' has the following errors:\n{1}"
        error_string = pprint.pformat(errors)
        
        raise Exception(message_string.format(field, error_string))

class UnicodeDictWriter(csv.DictWriter, object):
    """Extend csv.DictWriter to handle Unicode input"""

    def writerows(self, rows):
        for row in rows:
            self.writerow(row)

# ================================================== #
#               Main Function                        #
# ================================================== #
def process_map(file_in, validate):
    """Iteratively process each XML element and write to csv(s)"""

    with codecs.open(NODES_PATH, 'w') as nodes_file, \
         codecs.open(NODE_TAGS_PATH, 'w') as nodes_tags_file, \
         codecs.open(WAYS_PATH, 'w') as ways_file, \
         codecs.open(WAY_NODES_PATH, 'w') as way_nodes_file, \
         codecs.open(WAY_TAGS_PATH, 'w') as way_tags_file:

        nodes_writer = UnicodeDictWriter(nodes_file, NODE_FIELDS)
        node_tags_writer = UnicodeDictWriter(nodes_tags_file, NODE_TAGS_FIELDS)
        ways_writer = UnicodeDictWriter(ways_file, WAY_FIELDS)
        way_nodes_writer = UnicodeDictWriter(way_nodes_file, WAY_NODES_FIELDS)
        way_tags_writer = UnicodeDictWriter(way_tags_file, WAY_TAGS_FIELDS)

        nodes_writer.writeheader()
        node_tags_writer.writeheader()
        ways_writer.writeheader()
        way_nodes_writer.writeheader()
        way_tags_writer.writeheader()

        validator = cerberus.Validator()

        for element in get_element(file_in, tags=('node', 'way')):
            el = shape_element(element)
            if el:
                if validate is True:
                    validate_element(el, validator)

                if element.tag == 'node':
                    nodes_writer.writerow(el['node'])
                    node_tags_writer.writerows(el['node_tags'])
                elif element.tag == 'way':
                    ways_writer.writerow(el['way'])
                    way_nodes_writer.writerows(el['way_nodes'])
                    way_tags_writer.writerows(el['way_tags'])
```

Above function will create 5 .csv files, which will be imported into miami.db database.
