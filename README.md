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







