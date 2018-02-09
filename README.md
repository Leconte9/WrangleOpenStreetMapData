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
In nodes_tags file, the key word of zip code is 'postcode'. However, in ways_tags file, the key word of zip code is either 'zip_left' or 'zip_right'.
First, regarding the table nodes_tags, there are a few data need to be clean, showing like "FL 33431-4403". The leading characters are always the abbrivation of Florida State, "FL", and the trailing characters are always zip code extensions. In this case, we will only keep 5 main digit code, which allows for more consistent queries.
Because of there is only 7 values have problems, I used below queries to clean the problem data.
```sql
Select * From nodes_tags Where value = 'FL 33431-4403';
UPDATE nodes_tags SET value = '33431' WHERE id = '626874613' and key = 'postcode' and type = 'addr';
```

After standardizing inconsistent postal codes, I queried zip data and get info like below:
```sql
Select value, COUNT(*) as count 
From nodes_tags 
Where key = 'postcode' 
GROUP BY value 
ORDER BY count DESC;
```
Here are the top ten results, beginning with the highest count:
```sql
value,count
33139,20
33316,13
33060,8
33460,8
33301,6
33496,6
33132,5
33133,5
33178,5
33062,4
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





