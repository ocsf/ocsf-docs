# OCSF – Iceberg Storage Strategies
Paul Agbabian, April 2026; Updated May 2026

There are multiple ways that OCSF structured events can be stored for analysis in Parquet/Iceberg table format. The following list is not exhaustive but form the most common approaches seen so far in the industry. For example, a single partitioned Iceberg table is an option that has been used that can be considered a superset of the By Event Category approach.
1.	By event source
2.	By event category
3.	By event class
4.	By Observable and category

This short document will discuss the pros and cons of each strategy.
It should be noted that these tables will be sparsely populated in most cases with empty cells. This is not an issue with columnar storage as the missing cells don’t use any storage as they usually do with relational database storage extents. Tables may have many columns, but they are storage efficient.

However, minimizing the number of identical columns across tables may provide better compressibility, especially for row group columns of lower volume event types.

## By Event Source
Each event is emitted by an event source, usually associated with a single product with one or more features. A fully normalized or natively producing event source could span many different event classes. Each event class contains attributes which in turn can be of scalar or object type. Objects can have attributes that are scalar or object type, and so on.

Hence when creating tables by event source, there are two issues to be considered. First, do we know what classes the event source will emit? Second, how many event sources will be received by a given lakehouse implementation.

The assumption here is that each event source will have its own table, but each table will be different since each event source will likely emit different class information. In addition, the number of tables will expand based on the number of event sources, where new event source tables need to be built each time a new product is onboarded.

Lastly, there are over 50 OCSF event classes to date. Creating a single table structure for all products would be the cross-class union of all the attributes from each class, many of which are shared across classes and objects. If modeled as separate columns, advantages of a columnar table structure are reduced.

Nevertheless, several implementations have used this approach to be more product or event source oriented.

### Event Source Pros
-	Direct storage by source or product makes analyzing a single product or source easier
-	No cross-table joins for single source analysis across classes of events
-	1:1 ETL process from single source to single table: `metadata.product.name` or `metadata.source` directs the events to its destination

### Event Source Cons
-	New tables need to be created every time a new source is onboarded
-	Tables will all be different, with different but overlapping columns
-	Each table may have an unknown number of columns
-	As a source starts to emit new event classes, the table needs to be altered
-	Table management is more frequent

## By Event Category
OCSF event classes are organized by category, where each category contains classes that are related to each other by category domain. For example, operating system activities are found within the System Activity category, while network activities are found within the Network Activity category. (Host-based network activities are supported via the Host profile against network classes).

Further, classes in each category share common attributes, and all classes share common Base class attributes. (Note not all attributes are populated by every event source that uses the classes). Reuse of category-based tables across product event sources are by design. When new product event sources are onboarded, new tables are not generally required.

Because of the commonality across event categories, columns are not duplicated across the classes. Lastly, because the number of categories grows slowly across OCSF evolution, the number of tables will be relatively stable.

### Category Pros
-	A stable number of tables
-	Reuse of tables by multiple products
-	Minimize duplication of related columns within a table
-	Table holds all related classes in a domain
-	Reduce the number of joins within a domain’s category
-	ETL is straightforward as category_uid directs the events to its destination

### Category Cons
-	Searching for all events from a single product requires searching all tables if the product events span categories

## By Event Class
Each OCSF event is an instance of a single OCSF class. Events across different sources share the same classes, distinguished by their `metadata.source` or `metadata.product.name` attributes.

However, there are at this time of writing more than 50 event classes and new classes are developed more often than new categories of classes. Hence to cover a wide range of products that may share some but not all of the same classes, many tables need to be maintained and periodically added.

More joins are needed when analysis spans classes, as they map to separate tables. Given the commonality of many of the attributes across classes, columnar compression is not maximized. However, for targeted analysis, tables will be smaller (fewer rows).

### Class Pros
•	ETL is straightforward as `class_uid` directs the events to its destination
-	Reuse of tables by multiple products
-	Fewest number of columns per table
-	Best for targeted analysis across products emitting the same event types

### Class Cons
-	Searching for all events from a single product requires searching all tables if the product events span classes
-	Duplication of related columns across tables reducing compression
-	More joins within a single domain than with categories
-	Higher table growth velocity than with categories equating to more table maintenance

## By Observable and Category
This strategy is hybrid. It takes advantage of an OCSF schema feature called Observables.

Observables are an array of the most common attributes surfaced from any of the events that are emitted in a flat generic array. Each element of the array is a single Observable whose `value` attribute holds the element data.

The original intent of the observables was for threat intelligence matching, where the total fidelity and context of the event was not required, just the observable values that might match Indicators of Compromise (IoCs).

There is another use case for Observables: a table constructed for observables will be common across every event class, and therefore every event across all products. This table can have foreign keys to dimension tables much like a STAR schema in OLAP. The Observables array is flattened into about 40 columns, along with the most important Base event attribute columns that identify and classify the events.

In practice, many analytics can run very efficiently directly against a single table across all products and classes, while drill-down and detailed investigation is performed by a minimum number of joins to dimension tables (i.e. joins are not required for detection analytics, only for drill-down).

Based on the above analysis of strategies, either the By Event Class strategy or the By Event Category strategy could be employed for the dimension tables. Given that for most use cases, By Event Category is more efficient than By Event Class, using categories as the dimension tables is suggested here.

### Observable and Category Pros
-	Many cybersecurity detections use cases can run against a single table across all product event sources without joins
-	Single product searches are possible against a single table for all products
-	Observables table has a fixed number of non-duplicative columns with many rows for optimum compression
-	Ideal for threat intelligence matching and retrospective matching against IOCs
-	Drill-down to details is efficient
-	Retains all the Pros of By Category 

### Observable and Category Cons
-	Requires an ETL that can surface the Observables, or product native sources that send Observables
-	Duplication of data: an additional table is required with duplicate values from the category dimension tables (since dimension tables hold full fidelity events)
-	ETL is less straightforward as the Observables array needs to be retrieved and parsed before events are stored (and the array removed from the events to reduce but not eliminate duplication in dimension tables).

