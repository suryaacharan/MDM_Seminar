# AsterixDB Query Workflow for MDM Seminar

This README documents the complete workflow used to analyze a filtered subset of the Yelp dataset, TIGER2018 Road network dataset using Apache AsterixDB.

---

## 📍Query 2 : Geospatial Queries - Find the businesses in the Waterfront area of Santa Barbara

```sql
USE MDM25;

SELECT *
FROM Businesses b
WHERE st_contains(
  st_geom_from_geojson({
    "type": "Polygon",
    "coordinates": [[
      [-119.698200, 34.414700],
      [-119.698800, 34.411900],
      [-119.695900, 34.409900],
      [-119.692400, 34.409100],
      [-119.688800, 34.409500],
      [-119.686800, 34.411700],
      [-119.687300, 34.413600],
      [-119.689500, 34.415200],
      [-119.692700, 34.415500],
      [-119.695500, 34.415300],
      [-119.698200, 34.414700]
    ]]
  }),
  b.g
);
```

---

## ⭐ Query 3 : Find the average stars for each category of the businesses that are located nearby the touristy "State Street" extending onto Stearn’s Wharf.

```sql
USE MDM25;

SELECT VALUE {
  "category": trim(c),
  "avg_stars": avg(b.stars)
}
FROM Businesses b
UNNEST split(b.categories, ",") AS c
WHERE st_distance(
  st_geom_from_geojson({
    "type": "LineString",
    "coordinates": [
      [-119.69549266696681, 34.41679198298692],
      [-119.68575084673559, 34.409785253730774],
      [-119.68504617192193, 34.40885709855257]
    ]
  }),
  b.g
) <= 0.001
GROUP BY trim(c);
```

---

## 🚦 Query 4: Finding the number of five-star reviews for each road.

```sql
USE MDM25;

SELECT r.FULLNAME, COUNT(*) AS c
FROM Roads r, Businesses b, Reviews a
WHERE st_distance(b.g, r.g) < 0.0003
  AND a.business_id = b.business_id
  AND a.stars = 5
GROUP BY r.LINEARID, r.FULLNAME
ORDER BY c DESC;
```

---

## 🌐 Query 5: Finding the top ten roads that have the most five-stars businesses relative to their length.

```sql
USE MDM25;

SELECT
  "FeatureCollection" AS `type`,
  {"name": r.FULLNAME} AS properties,
  (
    SELECT "Feature" AS `type`, {} AS properties, brg.y.g AS geometry
    FROM BRGroup AS brg
  ) AS features
FROM Roads r, Businesses y
WHERE st_distance(r.g, y.g) < 0.0003
  AND y.stars = 5
GROUP BY r.LINEARID, r.FULLNAME, r.g
GROUP AS BRGroup
ORDER BY (COUNT(1) / st_length(r.g)) DESC
LIMIT 10;
```

---

## 📝 Query 7: Find the five-star reviews that are "similar" to names of the business that have location inside a polygon.

```sql
USE MDM25;

SELECT {
  "business_name": b.name,
  "number_of_reviews": COUNT(*),
  "reviews": (
    SELECT r1.r.text
    FROM review_group AS r1
  )
}
FROM Businesses AS b, Reviews AS r
WHERE
  st_contains(
    st_geom_from_geojson({
      "type": "Polygon",
      "coordinates": [[
        [-124.5000, 42.0000],
        [-114.1000, 42.0000],
        [-114.1000, 32.5000],
        [-124.5000, 32.5000],
        [-124.5000, 42.0000]
      ]]
    }),
    b.g
  )
  AND r.business_id = b.business_id
  AND similarity_jaccard(word_tokens(r.text), word_tokens(b.name)) > 0.5
  AND r.stars = 5
GROUP BY b.name
GROUP AS review_group
ORDER BY COUNT(*) DESC;
```

---
