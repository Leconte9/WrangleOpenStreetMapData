# Wrangle Open Street Map Data

### Map Area
Miami, FL, United States

- [https://www.openstreetmap.org/relation/1216769](https://www.openstreetmap.org/relation/1216769)
- [https://archive.is/o/ycQ4e/osm-extracted-metros.s3.amazonaws.com/miami.osm.bz2](https://archive.is/o/ycQ4e/osm-extracted-metros.s3.amazonaws.com/miami.osm.bz2)

This map is the place which I have been living for 4 years. I would like an opportunity to know more details of this city and to contribute to its improvement on the Map.


## Problems Encounteren in the Map

.........

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
value,count
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
value,count
"33487; 33483",4
33004:33312,2
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
value,count
33157,2195
33176,1942
33064,1869
33186,1799
33175,1781
33311,1665
33312,1503
33024,1499
33029,1429
33063,1425
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
11202755|county|Martin, FL:Palm Beach, FL|tiger

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







