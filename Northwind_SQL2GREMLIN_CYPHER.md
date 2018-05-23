# Comparison of Gremlin to Cypher using Northwind

This document takes the examples from http://sql2gremlin.com which convert
SQL to Gremlin and adds a comparable Cypher statement.

I chose these examples because they're provided by DataStax - presumably with
tuning and performance considerations built in.  This eliminates any potential allegations
that I intentionally wrote inefficient Gremlin code.

# Select

## Select All

### SQL
```code
Select * from Categories;
```

### Gremlin
```code
gremlin> g.V().hasLabel("category").valueMap()
```
### Cypher
```code
Match (c:Category)
  return c;
```


## Select Individual Columns

### SQL
```code
Select CategoryName, Description from Categories;
```

### Gremlin
```code
gremlin> g.V().hasLabel("category").valueMap("CategoryName","Description")
```
### Cypher
```code
Match (c:Category)
  return c.categoryName, c.description;
```


## Select Calculated Columns
Show how to create a derived column

### SQL
```code
Select LENGTH(CategoryName) from Categories;
```

### Gremlin
```code
gremlin> g.V().hasLabel("category").values("name").
               map {it.get().length()}

```
### Cypher
```code
Match (c:Category)
  return length(c.categoryName);
```


## Select Distinct Values
Show how to obtain distinct values for a derived column

### SQL
```code
Select DISTINCT LENGTH(CategoryName) from Categories;
```

### Gremlin
```code
gremlin> g.V().hasLabel("category").values("name").
               map {it.get().length()}.dedup()

```
### Cypher
```code
Match (c:Category)
  return DISTINCT length(c.categoryName);
```


## Select Scalar Values
Show how to obtain distinct values for a derived column

### SQL
```code
Select MAX (LENGTH(CategoryName)) from Categories;
```

### Gremlin
```code
gremlin> g.V().hasLabel("category").values("name").
               map {it.get().length()}.max()

```
### Cypher
```code
Match (c:Category)
  return Max length(c.categoryName);
```

# Filtering
## Filter by Equality
Find all products with no units in stock

### SQL
```code
SELECT ProductName, UnitsInStock
  FROM Products
 WHERE UnitsInStock = 0
```

### Gremlin
```code
gremlin> g.V().has("product", "unitsInStock", 0).valueMap("name", "unitsInStock")

```
### Cypher
```code
MATCH (p:Product)
where p.unitsInStock=0
return p.productName
```
or using some syntactical sugar (for equality match only)

```code
MATCH (p:Product {unitsInStock:0})
return p.productName

```

## Filter by InEquality
Find all products with units in stock

### SQL
```code
SELECT ProductName, UnitsOnOrder
  FROM Products
 WHERE NOT(UnitsOnOrder = 0)
```

### Gremlin
```code
gremlin> g.V().has("product", "unitsOnOrder", new(0)).valueMap("name", "unitsOnOrder")

```
### Cypher
```code
MATCH (p:Product)
where p.unitsOnOrder<>0
return p.productName
```
we cannot use the syntactical sugar here because of the InEquality


## Filter by Value Range
Find all products within a given price range

### SQL
```code
SELECT ProductName, UnitPrice
  FROM Products
 WHERE UnitPrice >= 5 AND UnitPrice < 10
```

### Gremlin
```code
gremlin> g.V().has("product", "unitPrice", between(5f, 10f)).
               valueMap("name", "unitPrice")
```
### Cypher
```code
MATCH (p:Product)
where p.unitPrice >= 5 and p.unitPrice <10
return p.productName, p.unitPrice
```

## Multiple Filter conditions
Find all discontinued products with unit in stock

### SQL
```code
SELECT ProductName, UnitsInStock
  FROM Products
 WHERE Discontinued = 1
   AND UnitsInStock <> 0
```

### Gremlin
```code
gremlin> g.V().has("product", "discontinued", true).has("unitsInStock", neq(0)).
              valueMap("name", "unitsInStock")
```
### Cypher
```code
MATCH (p:Product)
where p.unitsInStock <> 0 and p.discontinued
return p.productName, p.unitsInStock
```



## Ordering By Value
Results can be ordered ASC or DESC

### SQL
```code
SELECT ProductName, UnitPrice
  FROM Products
ORDER BY UnitPrice DESC
```

### Gremlin
```code
gremlin> g.V().hasLabel("product").order().by("unitPrice", decr).
               valueMap("name", "unitPrice")
```
### Cypher
```code
MATCH (p:Product)
return p.productName, p.unitPrice
order by p.unitPrice DESC
```

# Paging
## Limiting Number of Results
Very useful when combined with offset

### SQL
```code
SELECT TOP (5) ProductName, UnitPrice
    FROM Products
ORDER BY UnitPrice
```

### Gremlin
```code

gremlin> g.V().hasLabel("product").order().by("unitPrice", incr).limit(5).
               valueMap("name", "unitPrice")
```
### Cypher
```code
MATCH (p:Product)
return p.productName, p.unitPrice
order by p.unitPrice ASC
limit 5
```
## Paged Result Sets
Adding a starting offset adds complexity to the SQL code

### SQL
```code
SELECT Products.ProductName, Products.UnitPrice
     FROM (SELECT ROW_NUMBER()
                    OVER (
                      ORDER BY UnitPrice) AS [ROW_NUMBER],
                  ProductID
             FROM Products) AS SortedProducts
       INNER JOIN Products
               ON Products.ProductID = SortedProducts.ProductID
    WHERE [ROW_NUMBER] BETWEEN 6 AND 10
 ORDER BY [ROW_NUMBER]
```

### Gremlin
```code

gremlin> g.V().hasLabel("product").order().by("unitPrice", incr).range(5, 10).
               valueMap("name", "unitPrice")
```
### Cypher
```code
MATCH (p:Product)
return p.productName, p.unitPrice
order by p.unitPrice ASC
skip 5
limit 5
```
# Grouping
## Group by Value
Find the most frequently used UnitPrice

### SQL
```code
SELECT TOP(1) UnitPrice
  FROM (SELECT Products.UnitPrice,
               COUNT(*) AS [Count]
          FROM Products
      GROUP BY Products.UnitPrice) AS T
ORDER BY [Count] DESC
```
### Gremlin
```code

gremlin> g.V().hasLabel("product").groupCount().by("unitPrice").
               order(local).by(values, decr).select(keys).limit(local, 1)
```
### Cypher

```code
MATCH (p:Product)
return p.productName, p.unitPrice
order by p.unitPrice ASC
skip 5
limit 5
```

# Grouping
## Group by Value
Find the most frequently used UnitPrice

### SQL
```code
SELECT TOP(1) UnitPrice
  FROM (SELECT Products.UnitPrice,
               COUNT(*) AS [Count]
          FROM Products
      GROUP BY Products.UnitPrice) AS T
ORDER BY [Count] DESC
```
### Gremlin
```code

gremlin> g.V().hasLabel("product").groupCount().by("unitPrice").
               order(local).by(values, decr).select(keys).limit(local, 1)
```
### Cypher

```code
// most frequently used unit price
MATCH (p:Product)
return  p.unitPrice as price, count(p.unitPrice) as count
order by count DESC
limit 1
```

# Joining
## Inner JOIN
Find all products from a given category

### SQL
```code
SELECT Products.ProductName
      FROM Products
INNER JOIN Categories
        ON Categories.CategoryID = Products.CategoryID
     WHERE Categories.CategoryName = 'Beverages'
```
### Gremlin
```code
gremlin> g.V().has("name","Beverages").in("inCategory").values("name")
```
### Cypher

```code
// join on category
match (p:Product)-[:PART_OF]->(:Category {categoryName:'Produce'})
 return p
```

## Left JOIN
Count all orders for each customer

### SQL
```code
SELECT Customers.CustomerID, COUNT(Orders.OrderID)
  FROM Customers
LEFT JOIN Orders
    ON Orders.CustomerID = Customers.CustomerID
GROUP BY Customers.CustomerID
```
### Gremlin

```code
gremlin> g.V().hasLabel("customer").match(
           __.as("c").values("customerId").as("customerId"),
           __.as("c").out("ordered").count().as("orders")
         ).select("customerId", "orders")```
### Cypher

```code
// Count orders
match (c:Customer)
optional match (c)-[]-(o:Order)
return c.companyName, count(o) as orders
```

# Miscellaneous
## Unions
Concatenate two result sets

### SQL
```code
SELECT [customer].[CompanyName]
  FROM [Customers] AS [customer]
 WHERE [customer].[CompanyName] LIKE 'A%'
 UNION ALL
SELECT [customer].[CompanyName]
  FROM [Customers] AS [customer]
 WHERE [customer].[CompanyName] LIKE 'E%'
```
### Gremlin

```code
gremlin> g.V().hasLabel("customer").union(
           filter {it.get().value("company")[0] == "A"},
           filter {it.get().value("company")[0] == "E"}).values("company")
         ```
### Cypher

```code
match (c:Customer)
	where c.companyName starts with('A')
    return c.companyName
UNION
match (c:Customer)
	where c.companyName starts with('E')
	return c.companyName
```


# CRUD
Create, Read, Update and Delete values
## Unions
Concatenate two result sets

### SQL
```code

INSERT INTO [Categories] ([CategoryName], [Description])
     VALUES (N'Merchandising', N'Cool products to promote Gremlin')

INSERT INTO [Products] ([ProductName], [CategoryID])
     SELECT TOP (1) N'Red Gremlin Jacket', [CategoryID]
       FROM [Categories]
      WHERE [CategoryName] = N'Merchandising'

UPDATE [Products]
   SET [Products].[ProductName] = N'Green Gremlin Jacket'
 WHERE [Products].[ProductName] = N'Red Gremlin Jacket'

DELETE FROM [Products]
 WHERE [Products].[ProductName] = N'Green Gremlin Jacket'

DELETE FROM [Categories]
 WHERE [Categories].[CategoryName] = N'Merchandising'

```
### Gremlin

```code

gremlin> c = graph.addVertex(label, "category",
                   "name", "Merchandising",
                   "description", "Cool products to promote Gremlin")

gremlin> p = graph.addVertex(label, "product",
                   "ame", "Red Gremlin Jacket")


gremlin> p.addEdge("inCategory", c)

gremlin> g.V().has("product", "name", "Red Gremlin Jacket").
               property("name", "Green Gremlin Jacket").iterate()

gremlin> p.remove()

gremlin> g.V().has("category", "name", "Merchandising").drop()
         ```
### Cypher

```code
CREATE (p:Product {productName:'Red Gremlin Jacket'} )
CREATE (c:Category {categoryName:'Merchandising',description:'Cool Stuff'} )
CREATE (p)-[:PART_OF]->(c)

MATCH (p:Product {productName:'Red Gremlin Jacket'} )
   SET p.productName='Green Cypher Jacket'

MATCH (p:Product {productName:'Green Cypher Jacket'} )
    Detach Delete p

MATCH (c:Category {categoryName:'Merchandising'} )
    Detach Delete c
```


## Recursion
Query all employees, supervisors and hierarchy

### SQL
```code

WITH EmployeeHierarchy (EmployeeID,
                        LastName,
                        FirstName,
                        ReportsTo,
                        HierarchyLevel) AS
(
    SELECT EmployeeID
         , LastName
         , FirstName
         , ReportsTo
         , 1 as HierarchyLevel
      FROM Employees
     WHERE ReportsTo IS NULL

     UNION ALL

    SELECT e.EmployeeID
         , e.LastName
         , e.FirstName
         , e.ReportsTo
         , eh.HierarchyLevel + 1 AS HierarchyLevel
      FROM Employees e
INNER JOIN EmployeeHierarchy eh
        ON e.ReportsTo = eh.EmployeeID
)
  SELECT *
    FROM EmployeeHierarchy
ORDER BY HierarchyLevel, LastName, FirstName
```
### Gremlin

```code

gremlin> g.V().hasLabel("employee").where(__.not(out("reportsTo"))).
               repeat(__.as("reportsTo").in("reportsTo").as("employee")).emit().
               select(last, "reportsTo", "employee").by(map {
                 def employee = it.get()
                 employee.value("firstName") + " " + employee.value("lastName")
               })
         ```
### Cypher

```code
// Show HierarchyLevel
MATCH (e:Employee)<-[:REPORTS_TO]-(sub)
  RETURN (e.firstName + " " + e.lastName)  AS reportsTo,
  (sub.firstName + " " + sub.lastName) AS employee;

```



## Pivot Table
Query all employees, supervisors and hierarchy

### SQL
```code

SELECT Customers.CompanyName,
           COALESCE([1], 0)  AS [Jan],
           COALESCE([2], 0)  AS [Feb],
           COALESCE([3], 0)  AS [Mar],
           COALESCE([4], 0)  AS [Apr],
           COALESCE([5], 0)  AS [May],
           COALESCE([6], 0)  AS [Jun],
           COALESCE([7], 0)  AS [Jul],
           COALESCE([8], 0)  AS [Aug],
           COALESCE([9], 0)  AS [Sep],
           COALESCE([10], 0) AS [Oct],
           COALESCE([11], 0) AS [Nov],
           COALESCE([12], 0) AS [Dec]
      FROM (SELECT Orders.CustomerID,
                   MONTH(Orders.OrderDate)                                   AS [Month],
                   SUM([Order Details].UnitPrice * [Order Details].Quantity) AS Total
              FROM Orders
        INNER JOIN [Order Details]
                ON [Order Details].OrderID = Orders.OrderID
          GROUP BY Orders.CustomerID,
                   MONTH(Orders.OrderDate)) o
     PIVOT (AVG(Total) FOR [Month] IN ([1],
                                       [2],
                                       [3],
                                       [4],
                                       [5],
                                       [6],
                                       [7],
                                       [8],
                                       [9],
                                       [10],
                                       [11],
                                       [12])) AS [Pivot]
INNER JOIN Customers
        ON Customers.CustomerID = [Pivot].CustomerID
  ORDER BY Customers.CompanyName

```
### Gremlin

```code

gremlin> g.V().hasLabel("employee").where(__.not(out("reportsTo"))).
               repeat(__.as("reportsTo").in("reportsTo").as("employee")).emit().
               select(last, "reportsTo", "employee").by(map {
                 def employee = it.get()
                 employee.value("firstName") + " " + employee.value("lastName")
               })
         ```
### Cypher

```code
WITH ['jan','feb','mar','apr','may','jun','jul','aug','sep','oct','nov','dec'] AS months

match (c:Customer) --(o:Order)-[pu:PRODUCT]-()
return c.companyName as customer,
	months[toInt(substring(o.orderDate,5,2))-1] as month,
    avg( pu.quantity * pu.unitPrice) as totals
order by customer,month

```


##  Recommendations
This sample shows how to recommend 5 products for a specific customer. The products are chosen as follows:

* determine what the customer has already ordered
* determine what others also ordered
* determine products which were not already ordered by the initial customer, but ordered by the others
* rank products by occurence in other orders



### SQL
```code

SELECT TOP (5) [t14].[ProductName]
    FROM (SELECT COUNT(*) AS [value],
                 [t13].[ProductName]
            FROM [customers] AS [t0]
     CROSS APPLY (SELECT [t9].[ProductName]
                    FROM [orders] AS [t1]
              CROSS JOIN [order details] AS [t2]
              INNER JOIN [products] AS [t3]
                      ON [t3].[ProductID] = [t2].[ProductID]
              CROSS JOIN [order details] AS [t4]
              INNER JOIN [orders] AS [t5]
                      ON [t5].[OrderID] = [t4].[OrderID]
               LEFT JOIN [customers] AS [t6]
                      ON [t6].[CustomerID] = [t5].[CustomerID]
              CROSS JOIN ([orders] AS [t7]
                          CROSS JOIN [order details] AS [t8]
                          INNER JOIN [products] AS [t9]
                                  ON [t9].[ProductID] = [t8].[ProductID])
                   WHERE NOT EXISTS(SELECT NULL AS [EMPTY]
                                      FROM [orders] AS [t10]
                                CROSS JOIN [order details] AS [t11]
                                INNER JOIN [products] AS [t12]
                                        ON [t12].[ProductID] = [t11].[ProductID]
                                     WHERE [t9].[ProductID] = [t12].[ProductID]
                                       AND [t10].[CustomerID] = [t0].[CustomerID]
                                       AND [t11].[OrderID] = [t10].[OrderID])
                     AND [t6].[CustomerID] <> [t0].[CustomerID]
                     AND [t1].[CustomerID] = [t0].[CustomerID]
                     AND [t2].[OrderID] = [t1].[OrderID]
                     AND [t4].[ProductID] = [t3].[ProductID]
                     AND [t7].[CustomerID] = [t6].[CustomerID]
                     AND [t8].[OrderID] = [t7].[OrderID]) AS [t13]
           WHERE [t0].[CustomerID] = N'ALFKI'
        GROUP BY [t13].[ProductName]) AS [t14]
ORDER BY [t14].[value] DESC

```
### Gremlin

```code

gremlin> g.V().has("customer", "customerId", "ALFKI").as("customer").
               out("ordered").out("contains").out("is").aggregate("products").
               in("is").in("contains").in("ordered").where(neq("customer")).
               out("ordered").out("contains").out("is").where(without("products")).
               groupCount().order(local).by(values, decr).select(keys).limit(local, 5).
               unfold().values("name")

         ```
### Cypher

```code
// Recommendations
// Product Recommendation     
MATCH (cust1:Customer {companyName:'AlfredsFutterkiste'})-[:PURCHASED]->(:Order)-[:PRODUCT]->(p1:Product), (p1)<-[:PRODUCT]-(o2:Order)<-[:PURCHASED]-(othCust:Customer), (othCust)-[:PURCHASED]->(o3:Order)-[:PRODUCT]->(othProd:Product)  
where id(othCust) <> id(cust1)
AND  NOT (cust1)-[:PURCHASED]->(:Order)-[:PRODUCT]->(othProd)
return cust1.companyName, othCust.companyName, othProd.productName
```
