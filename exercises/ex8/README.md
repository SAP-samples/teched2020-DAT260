# Exercise 8 - Calculate Shortest Paths with a more complex cost function

In the previous exercise we have use hop distance to calculate a shortest path. Now we will use a more meaningful cost function - we derive take the time it takes to traverse a street segment. The EDGES table contains a "length" and "maxspeed" column. "maxspeed" is a string column with values like '30 mph'. We will create a new numeric column "SPEED_MPH" and extract the number part of "maxspeed" into this column. We will then re-write our procedure to take the expression "length"/"SPEED_MPH" as cost function.

## Exercise 8.1 Generate a numeric column that contains the maximum speed allowed information <a name="subex1"></a>

---
**Add an integer column to the LONDON_BIKE_EDGES table. Extract the number part of "maxspeed".**

---

```sql
ALTER TABLE "LONDON_EDGES" ADD("SPEED_MPH" INT);
UPDATE "LONDON_EDGES"
	SET "SPEED_MPH" = TO_INT(REPLACE("maxspeed", ' mph', ''))
	WHERE REPLACE("maxspeed", ' mph', '') <> "maxspeed" ;
SELECT "SPEED_MPH", COUNT(*) AS C FROM "LONDON_EDGES" GROUP BY "SPEED_MPH" ORDER BY C DESC;
-- let's add a default value on the segments that do not have a speed information
UPDATE "LONDON_EDGES" SET "SPEED_MPH" = 30 WHERE "SPEED_MPH" IS NULL;
```

This is the SPEED_MPH distribution after updating with default values.

![](images/SPEED.png)

## Exercise 8.2 Calculate Shortest Paths, minimizing the time it takes to get from start to end <a name="subex2"></a>

Just like in the previous example, we define a table type and a procedure. This time, we are using "length"/SPEED_MPH" as cost function. Syntactically, the cost function is a lambda function like this:&nbsp;

`
(Edge e) => DOUBLE{ return :e."length"/DOUBLE(:e."SPEED_MPH"); }
`

```sql
CREATE TYPE "TT_SPOO_WEIGHTED_EDGES" AS TABLE (
    "ID" NVARCHAR(5000), "SOURCE" BIGINT, "TARGET" BIGINT, "EDGE_ORDER" BIGINT, "length" DOUBLE, "SPEED_MPH" INT
);
```
```sql
CREATE OR REPLACE PROCEDURE "GS_SPOO_WEIGHTED"(
	IN i_startVertex BIGINT, 		-- INPUT: the ID of the start vertex
	IN i_endVertex BIGINT, 			-- INPUT: the ID of the end vertex
	IN i_direction VARCHAR(10), 	-- INPUT: the direction of the edge traversal: OUTGOING (default), INCOMING, ANY
	OUT o_path_length BIGINT,		-- OUTPUT: the hop distance between start and end
	OUT o_path_weight DOUBLE,		-- OUTPUT: the path weight/cost
	OUT o_edges "TT_SPOO_WEIGHTED_EDGES"  -- OUTPUT: the edges that make up the path
	)
LANGUAGE GRAPH READS SQL DATA AS BEGIN
	-- Create an instance of the graph, referring to the graph workspace object
	GRAPH g = Graph("DAT260", "LONDON_GRAPH");
	-- Create an instance of the start/end vertex
	VERTEX v_start = Vertex(:g, :i_startVertex);
	VERTEX v_end = Vertex(:g, :i_endVertex);
	--WeightedPath<DOUBLE> p = Shortest_Path(:g, :v_start, :v_end, (Edge e) => DOUBLE{ return :e."length"; }, :i_direction);
	WeightedPath<DOUBLE> p = Shortest_Path(:g, :v_start, :v_end,
		(Edge e) => DOUBLE{
			return :e."length"/DOUBLE(:e."SPEED_MPH");
		}, :i_direction);
	o_path_length = LENGTH(:p);
	o_path_weight = WEIGHT(:p);
	o_edges = SELECT :e."ID", :e."SOURCE", :e."TARGET", :EDGE_ORDER, :e."length", :e."SPEED_MPH" FOREACH e IN Edges(:p) WITH ORDINALITY AS EDGE_ORDER;
END;
```

Call the procedure.

```sql
CALL "GS_SPOO_WEIGHTED"(1433737988, 1794145673, 'ANY', ?, ?, ?);
```

![](images/SPOO_WEIGHTED.png)

If you bring this on a map it looks like this.

![](images/SPOO_WEIGHTED_MAP_COMBI.png)

## Exercise 8.3 Finding Pubs and Bikelanes <a name="subex3"></a>

Finding the fastest route is easy. Let's find two more interesting paths. First, we want find paths suitable for bikes. We can do so by boosting street segments which are "cycleways". Note that in most cases you cannot take cycleways only. The path algorithm will choose cycleways unless they are 10x longer than a normal road. For this logic we will use an `IF` statement within the cost function.
Second, we would like to find "attractive" paths. We will calculate a new measure for the edges - "PUBINESS" - which is derived from the number of pubs nearby.

First, let's calculate PUBINESS by counting pubs within 100m distance and add this to our `LONDON_EDGES` table. We are using the spatial ST_WithinDistance predicate as join condition:<br>
`ON pubs."SHAPE".ST_WithinDistance(e."EDGESHAPE", 100) = 1`

```SQL
ALTER TABLE "LONDON_EDGES" ADD ("PUBINESS" DOUBLE DEFAULT 0);

MERGE INTO "LONDON_EDGES"
	USING (
		SELECT e."ID", COUNT(*) AS "PUBINESS" FROM
			(SELECT * FROM "LONDON_POI" WHERE "amenity" ='pub') AS pubs
			LEFT JOIN
			(SELECT "ID", "SHAPE" AS "EDGESHAPE" FROM "LONDON_EDGES") AS e
			ON pubs."SHAPE".ST_WithinDistance(e."EDGESHAPE", 100) = 1
			GROUP BY e."ID" ORDER BY "PUBINESS" DESC)	AS U
	ON "LONDON_EDGES"."ID" = U."ID"
WHEN MATCHED THEN UPDATE SET "LONDON_EDGES"."PUBINESS" = U."PUBINESS";
```
Let's take a look at the distribution of our PUBINESS property.
```SQL

SELECT "PUBINESS", COUNT(*) AS C FROM "LONDON_EDGES" GROUP BY "PUBINESS" ORDER BY "PUBINESS" ASC;
```
![](images/PUBINESS_DISTR.png)

Now, we can use the new measure as part of the cost function for path finding with mode "pub".

`Shortest_Path(:g, :v_start, :v_end, (Edge e) => DOUBLE {`<br>`
RETURN :e."length"/(5.0*:e."PUBINESS"+1.0); `<br>`
}, :i_direction);`

For pathfinding with mode "bike", we'll use a conditional cost function. Street segments which are of type "cycleway" are boosted by dividing the length by 10.

`Shortest_Path(:g, :v_start, :v_end, (EDGE e)=> DOUBLE {`<br>`
IF(:e."highway" == 'cycleway') { RETURN :e."length"/10.0; }`<br>`
ELSE { RETURN :e."length"; } `<br>`
}, :i_direction);`

Create a `TABLE TYPE` first.
```SQL
CREATE TYPE "TT_SPOO_MULTI_MODE" AS TABLE (
		"ID" NVARCHAR(5000), "SOURCE" BIGINT, "TARGET" BIGINT, "EDGE_ORDER" BIGINT, "length" DOUBLE, "SPEED_MPH" INT, "highway" NVARCHAR(5000)
);
```
Then the procedure.
```SQL
CREATE OR REPLACE PROCEDURE "GS_SPOO_MULTI_MODE"(
	IN i_startVertex BIGINT, 		-- the ID of the start vertex
	IN i_endVertex BIGINT, 			-- the ID of the end vertex
	IN i_direction VARCHAR(10), 	-- the the direction of the edge traversal: OUTGOING (default), INCOMING, ANY
	IN i_mode VARCHAR(10), 		-- hop, time, bike
	OUT o_path_length BIGINT,		-- the hop distance between start and end
	OUT o_path_weight DOUBLE,		-- the path weight/cost based on the WEIGHT attribute
	OUT o_edges "TT_SPOO_MULTI_MODE"
	)
LANGUAGE GRAPH READS SQL DATA AS BEGIN
	GRAPH g = Graph("DAT260", "LONDON_GRAPH");
	VERTEX v_start = Vertex(:g, :i_startVertex);
	VERTEX v_end = Vertex(:g, :i_endVertex);
	-- mode=bike means cycleway preferred
	IF (:i_mode == 'bike') {
		WeightedPath<DOUBLE> p = Shortest_Path(:g, :v_start, :v_end,
		(EDGE e, DOUBLE current_path_weight)=> DOUBLE{
  			IF(:e."highway" == 'cycleway') { RETURN :e."length"/10.0; }
        ELSE { RETURN :e."length"; }
  	}, :i_direction);
		o_path_length = LENGTH(:p);
		o_path_weight = DOUBLE(WEIGHT(:p));
		o_edges = SELECT :e."ID", :e."SOURCE", :e."TARGET", :EDGE_ORDER, :e."length", :e."SPEED_MPH", :e."highway" FOREACH e IN Edges(:p) WITH ORDINALITY AS EDGE_ORDER;
	}
	-- mode=pub means street with pubs around preferred
	IF (:i_mode == 'pub') {
		WeightedPath<DOUBLE> p = Shortest_Path(:g, :v_start, :v_end, (Edge e) => DOUBLE{
			RETURN :e."length"/(5.0*:e."PUBINESS"+1.0);
		}, :i_direction);
		o_path_length = LENGTH(:p);
		o_path_weight = DOUBLE(WEIGHT(:p));
		o_edges = SELECT :e."ID", :e."SOURCE", :e."TARGET", :EDGE_ORDER, :e."length", :e."SPEED_MPH", :e."highway" FOREACH e IN Edges(:p) WITH ORDINALITY AS EDGE_ORDER;
	}
END;
```
```SQL
CALL "GS_SPOO_MULTI_MODE"(1433737988, 1794145673, 'ANY', 'pub', ?, ?, ?);
CALL "GS_SPOO_MULTI_MODE"(1433737988, 1794145673, 'ANY', 'bike', ?, ?, ?);
```
## Exercise 8.4 Wrapping a Procedure in a Table Function <a name="subex4"></a>

The procedure above returns more than one output - the path's length, weight, and a table with the edges. Sometimes it is convenient to wrap a GRAPH procedure in a table function, returning only the tabular output. Table functions are called via SELECT and are a convenient way to post-process graph results - you can use the full power of SQL on your graph results. This is how you do it.

```SQL
CREATE TYPE "TT_EDGES_SPOO_F" AS TABLE (
		"ID" NVARCHAR(5000), "SOURCE" BIGINT, "TARGET" BIGINT, "EDGE_ORDER" BIGINT, "length" DOUBLE, "SHAPE" ST_GEOMETRY(32630)
);
```
```SQL
CREATE OR REPLACE FUNCTION "F_SPOO_EDGES"(
	IN i_startVertex BIGINT,
	IN i_endVertex BIGINT,
	IN i_direction VARCHAR(10),
	IN i_mode VARCHAR(10)
	)
  RETURNS "LONDON_EDGES"
LANGUAGE SQLSCRIPT READS SQL DATA AS
BEGIN
	DECLARE o_path_length DOUBLE;
	DECLARE o_path_weight DOUBLE;
  CALL "GS_SPOO_MULTI_MODE"(:i_startVertex, :i_endVertex, :i_direction, :i_mode, o_path_length, o_path_weight, o_edges);
  RETURN SELECT lbe.* FROM :o_edges AS P LEFT JOIN "LONDON_EDGES" lbe ON P."ID" = lbe."ID";
END;
```

Now we can simply calculate the average PUBINESS of a path (whatever this means), or UNION two paths to compare.

```SQL
SELECT AVG("PUBINESS")
	FROM "F_SPOO_EDGES"(1433737988, 1794145673, 'ANY', 'pub');

-- Compare two paths
SELECT "ID", "SHAPE" FROM "F_SPOO_EDGES"(1433737988, 1794145673, 'ANY', 'pub')
UNION
SELECT "ID", "SHAPE" FROM "F_SPOO_EDGES"(1433737988, 1794145673, 'ANY', 'bike');
```
![](images/TWO_PATHS.png)

## Summary

We have used two more cost functions for path finding. We have wrapped the database procedure into a table function which can be called in a SQL SELECT statement. This is a nice way of mixing graph and relational processing.

Continue to - [Exercise 9 - Calculate Isochrones and Closeness Centrality](../ex9/README.md)
