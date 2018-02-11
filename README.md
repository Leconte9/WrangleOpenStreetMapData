# Wrangle Open Street Map Data

### Map Area
Miami, FL, United States

- [https://www.openstreetmap.org/relation/1216769](https://www.openstreetmap.org/relation/1216769)
- [https://archive.is/o/ycQ4e/osm-extracted-metros.s3.amazonaws.com/miami.osm.bz2](https://archive.is/o/ycQ4e/osm-extracted-metros.s3.amazonaws.com/miami.osm.bz2)

This map is the place which I have been studing, working and living for 4 years. I would like an opportunity to know more details of this city and to contribute to its improvement on the Map.


## Problems Encountered in the Map

After initially downloading the Map Data of Miami area and running it against data.py file, it shows main problems with the data, which I will discuss in the following order:
- Over abbreviated street names ("NW 107th Ter")
- Inconsistent postal codes ("FL 33431-4403")

### To Make Street Names Uniform

```python
def update_name(name, mapping):   
    st_type = street_type_re.search(name).group()
    if st_type in mapping.keys():
        update_type = mapping[st_type]
        name = name.replace(st_type, update_type)
    return name
```

This updated all substrings in problematic address strings, like below:
*"SW 15th ST"* would be updated to *"SW 15th Street"*
*"Beeline Hwy"* would be updated to *"Beeline Highway"*
*"NW 107th Ter"* would be updated to *"NW 107th Terrace"*

### Postal Codes
There are 2 files include zip code information, nodes_tags and ways_tags.
The key word of zip code for both file is 'postcode'. However, in ways_tags file, the key word of zip code is also either 'zip_left' or 'zip_right'.
First, regarding post code data, there are a few data need to be clean, showing like "FL 33431-4403". The leading characters are always the abbrivation of Florida State, "FL", and the trailing characters are always zip code extensions. In this case, we will only keep 5 main digit code, which allows for more consistent queries.
Because of there is only a few values have problems, I used below queries to clean the problem data.
```sql
Select * From nodes_tags Where value = 'FL 33431-4403';
UPDATE nodes_tags SET value = '33431' WHERE id = '626874613' and key = 'postcode' and type = 'addr';
```

After standardizing inconsistent postal codes, I queried zip data and get info like below:
```sql
SELECT tags.value, COUNT(*) as count 
FROM (SELECT * FROM nodes_tags UNION ALL SELECT * FROM ways_tags) tags 
WHERE tags.key = 'postcode' 
GROUP BY tags.value 
ORDER BY count DESC;
```
Here are the top ten results, beginning with the highest count:
```sql
value|count
33319|21
33139|20
33313|14
33316|13
33166|12
33060|8
33178|8
33460|8
33496|8
33133|6
```
Then, regarding the table ways_tags, the zip code data are all looks make sence. All zip codes are 5 digit, and start with 330 to 334 which truly belongs to Miami city. Some zip codes are showing like below. The ways could be in multiple zip areas. In order to keep data complete, this part of data will not be edited.
```sql
value|count
"33487; 33483"|4
33004:33312|2
```
I queried zip code data and get info like below
```sql
SELECT value, COUNT(*) as count 
FROM ways_tags 
WHERE key = 'zip_left' or 'zip_right'
GROUP BY value 
ORDER BY count DESC;
```
Here are the top ten results, beginning with the highest count:
```sql
value|count
33157|2195
33176|1942
33064|1869
33186|1799
33175|1781
33311|1665
33312|1503
33024|1499
33029|1429
33063|1425
```

```sql
SELECT tags.value, COUNT(*) as count 
FROM (SELECT * FROM nodes_tags UNION ALL SELECT * FROM ways_tags) tags 
WHERE tags.key LIKE '%county' 
GROUP BY tags.value 
ORDER BY count DESC;
```
The results are showing as following:
```sql
Miami-Dade, FL|35297
Broward, FL|26985
Palm Beach, FL|23761
Miami-Dade|234
Monroe, FL|152
Broward|148
Palm Beach|110
Martin, FL:Palm Beach, FL|1
Monroe|1
```

Obviously, there are only 4 counties in total, but the results are showing several duplicates.
Let's use UPDATE to combine the info.
```sql
SELECT * FROM (SELECT * FROM nodes_tags UNION ALL SELECT * FROM ways_tags) tags 
WHERE tags.key LIKE '%county' and tags.value = 'Martin, FL:Palm Beach, FL';
```
```sql
11202755|county|Martin, FL:Palm Beach, FL|tiger
```

```sql
UPDATE ways_tags SET value = 'Palm Beach, FL' WHERE key LIKE '%county' and value = 'Martin, FL:Palm Beach, F';
```
After organizing counties, finally got a county report as following:
```sql
Miami-Dade, FL|35531
Broward, FL|27133
Palm Beach, FL|23872
Monroe, FL|153
```

## Data Overview and Additional Ideas

This section contains basic statistics about the dataset, the SQL queries used to gather them, and some additional ideas about the data in context.

### File sizes
```
miami.osm ................252.0 MB
miami.db .................161.0 MB
nodes.csv .................81.6 MB
nodes_tags.csv ............4.51 MB
ways.csv ..................7.53 MB
ways_tags.csv .............34.4 MB
ways_nodes.csv ............28.0 MB
```

### Number of nodes
```sql
SELECT COUNT(*) FROM nodes;
```
1048575

### Number of ways
```sql
SELECT COUNT(*) FROM ways;
```
137048

### Number of unique users
```sql
SELECT COUNT(DISTINCT(e.uid))          
FROM (SELECT uid FROM nodes UNION ALL SELECT uid FROM ways) e;
```
577

### Top 10 contributing users
```sql
SELECT e.user, COUNT(*) as num
FROM (SELECT user FROM nodes UNION ALL SELECT user FROM ways) e
GROUP BY e.user
ORDER BY num DESC
LIMIT 10;
```
```sql
grouper|274825
woodpeck_fixbot|251604
Latze|136628
bot-mode|71321
NE2|66231
westendguy|57578
dysteleologist|41434
Trex2001|36083
maxolasersquad|26627
OklaNHD|17629
```

### Number of users appearing only once (having 1 post)
```sql
SELECT COUNT(*) 
FROM (SELECT e.user, COUNT(*) as num
     FROM (SELECT user FROM nodes UNION ALL SELECT user FROM ways) e
     GROUP BY e.user
     HAVING num = 1)  u;
```
115



## Additional Ideas
### Contributor statistics
Here are some user percentage statistics:
- Top user contribution percentage ("grouper") 23.18%
- Combined top 3 users' contribution ("grouper", "woodpeck_fixbot" and "Latze") 55.92%
- Combined Top 10 users contribution 82.65%
- Combined number of users making up only 0.01% of posts 115 (19.93% of all users)

Thinking about these user percentages, the top users contribution is still much highter than others, but it looks reasonable. Data might be contributed by specific agency group or automated edited. More than 80% of users submit more than 1 edit. 

## Additional Data Exploration
### Number of Places where take bitcoin payment
```sql
SELECT COUNT(*) as count 
FROM (SELECT * FROM nodes_tags UNION ALL SELECT * FROM ways_tags) 
WHERE key = 'bitcoin' 
GROUP BY key;
```
18
### Name of Places where take bitcoin payment
```sql
SELECT tags.id, tags.key, tags.value 
FROM ways_tags as tags 
WHERE tags.id IN (SELECT id FROM (SELECT * FROM ways_tags WHERE key = 'bitcoin') as T) 
    and (tags.key = 'name');
```
```sql
id|key|value
95606130|name|Bitcountant
203436185|name|Liberty Extraction & Drying
```
```sql
SELECT tags.id, tags.key, tags.value 
FROM nodes_tags as tags 
WHERE tags.id IN (SELECT id FROM (SELECT * FROM nodes_tags WHERE key = 'bitcoin') as T) 
    and (tags.key = 'name');
```
```sql
id|key|value
2362164911|name|Planet Linux Caffe
2379324455|name|Audio Props
2517338154|name|Vanity Miami - Cosmetic Surgery
2525585171|name|Ecig Shop Miami
2543973344|name|Barnard Law Offices, L.P.
2543990516|name|OMNI Expert Consulting, Inc.
2544305329|name|Wellington Florist
2552834515|name|Xtreme Care Body Shop
2554422896|name|Vanity Cosmetic Surgery
2554429722|name|Vanity Cosmetic Surgery
2563336834|name|Ortega Cigars
2563943753|name|Verbena Products Online Beauty Store
2570898921|name|Therapy Alliance Inc
2573516704|name|Ballard Computer Solutions
2576132724|name|League Against Aids Inc
2578603017|name|Daddy Brews
```

### Top 10 appearing amenities
```sql
SELECT value, COUNT(*) as num
FROM nodes_tags
WHERE key='amenity'
GROUP BY value
ORDER BY num DESC
LIMIT 10;
```
```sql
school|1643
place_of_worship|556
kindergarten|528
fire_station|316
police|197
library|145
hospital|130
restaurant|129
post_office|82
parking|77
```

### Top 10 cuisines
```sql
SELECT nodes_tags.value, COUNT(*) as num
FROM nodes_tags 
    JOIN (SELECT DISTINCT(id) FROM nodes_tags WHERE value='restaurant') i
    ON nodes_tags.id=i.id
WHERE nodes_tags.key='cuisine'
GROUP BY nodes_tags.value
ORDER BY num DESC
LIMIT 10;
```
```sql
american|9
pizza|6
italian|5
burger|4
sandwich|4
seafood|4
chinese|3
Cuban|1
Irish,_etc.|1
asian|1
```

