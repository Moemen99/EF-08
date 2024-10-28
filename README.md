# Entity Framework Core Loading Patterns

## Table of Contents
- [Introduction](#introduction)
- [Default Behavior](#default-behavior)
- [Basic Query Structure](#basic-query-structure)
- [Query Return Types](#query-return-types)
- [Null Propagation](#null-propagation)
- [Types of Loading](#types-of-loading)

## Introduction

When working with Entity Framework Core, understanding how related data is loaded is crucial for building efficient applications. This documentation covers the default behavior, querying patterns, and different loading strategies.

## Default Behavior

When querying an entity with navigation properties:
- Only basic properties are loaded (Id, Name, Age, etc.)
- Navigation properties (related entities) are not loaded by default
- This happens because related data requires additional table queries

Example properties loaded by default for Employee:
- Id
- Name
- Age
- Salary
- Address
- DepartmentId
- Department (navigation property) → NULL

```mermaid
flowchart TD
    A["Query Employee"] --> B["Load Basic Properties"]
    A --> C["Navigation Properties"]
    B --> D["Id"]
    B --> E["Name"]
    B --> F["Age"]
    B --> G["Salary"]
    B --> H["Address"]
    B --> I["DepartmentId"]
    C -->|Not Loaded by Default| J["Department = null"]
```

## Basic Query Structure

Here's how to query an employee using LINQ:

```csharp
using AppDbContext dbContext = new AppDbContext();

var employee = from E in dbContext.Employees
               where E.Id == 1
               select E;
```

## Query Return Types

### IQueryable vs IEnumerable

```csharp
// This returns IQueryable, not IEnumerable
var employee = from E in dbContext.Employees
               where E.Id == 1
               select E;
```

Important distinctions:
- Returns `IQueryable` instead of `IEnumerable`
- `IQueryable`: Filtering occurs in database
- `IEnumerable`: Would retrieve all records then filter in memory
- Use `SingleOrDefault()` to ensure single record return

| Return Type | Filter Location | Performance Impact |
|------------|-----------------|-------------------|
| IQueryable | Database | Better - Only matching records transferred |
| IEnumerable | Memory | Worse - All records transferred first |

## Null Propagation

When accessing navigation properties, proper null checking is important:

```csharp
Console.WriteLine($"Employee: {employee?.Name}, Department:{employee?.Department?.Name ?? "NA"}");
```

Null propagation breakdown:
1. `employee?` - Check if employee exists
2. `Department?` - Check if Department is loaded
3. `?? "NA"` - Provide default value if null

```mermaid
flowchart LR
    A["Check Employee"] -->|is null| B["Return null"]
    A -->|not null| C["Get Name"]
    A -->|not null| D["Check Department"]
    D -->|is null| E["Return 'NA'"]
    D -->|not null| F["Get Department Name"]
```

## Loading Strategies Overview

There are three ways to load related data:
1. Explicit Loading
2. Eager Loading
3. Lazy Loading

Each strategy has its use cases and performance implications.

| Strategy | When to Load | Best For |
|----------|-------------|-----------|
| Explicit Loading | On demand after main query | Optional related data |
| Eager Loading | With main query | Known required related data |
| Lazy Loading | When accessed | Uncertain data needs |

---



# Entity Framework Core Loading Patterns

## Table of Contents
- [Default Behavior Review](#default-behavior-review)
- [Explicit Loading](#explicit-loading)
  - [Single Reference Loading](#single-reference-loading)
  - [Collection Loading](#collection-loading)
  - [Generated SQL Queries](#generated-sql-queries)
  - [Pros and Cons](#pros-and-cons)

## Default Behavior Review

When querying entities, EF Core generates simple SELECT statements without JOINs:

```sql
-- For Employee query
SELECT TOP(1) [e].[Id], [e].[Address], [e].[Age], [e].[DepartmentId], [e].[Name], [e].[Salary]
FROM [Employees] AS [e]
WHERE [e].[Id] = 1

-- For Department query
SELECT TOP(1) [d].[Id], [d].[DateOfCreation], [d].[Name]
FROM [Departments] AS [d]
WHERE [d].[Id] = 10
```

```csharp
var employee = (from E in dbContext.Employees
                where E.Id == 1
                select E).FirstOrDefault();

// Department navigation property will be null
Console.WriteLine($"Employee: {employee?.Name}, Department:{employee?.Department?.Name ?? "NA"}");
```

## Explicit Loading

Explicit loading allows you to load related data on demand after loading the main entity.

### Single Reference Loading

Loading a single reference (Employee → Department):

```csharp
// First load employee
var employee = (from E in dbContext.Employees
                where E.Id == 1
                select E).FirstOrDefault();

// Explicitly load department - Two syntax options
dbContext.Entry(employee).Reference(E => E.Department).Load();
// OR
dbContext.Entry(employee).Reference(nameof(Employee.Department)).Load();

// Now department data is available
Console.WriteLine($"Employee: {employee?.Name}, Department: {employee?.Department?.Name ?? "NA"}");
```

### Collection Loading

Loading a collection (Department → Employees):

```csharp
// First load department
var department = (from D in dbContext.Departments
                 where D.Id == 10
                 select D).FirstOrDefault();

Console.WriteLine($"Department: Id = {department?.Id}, Name = {department?.Name}");

// Explicitly load employees collection
dbContext.Entry(department).Collection(D => D.Employees).Load();

// Now we can iterate through employees
foreach(var emp in department.Employees)
{
    Console.WriteLine($"Employee: Id = {emp.Id}, Name = {emp?.Name}");
}
```

### Generated SQL Queries

```mermaid
sequenceDiagram
    Application->>Database: Request 1 (Main Entity)
    Note over Application,Database: SELECT FROM Employees/Departments
    Database-->>Application: Return Main Entity
    Application->>Database: Request 2 (Related Data)
    Note over Application,Database: SELECT Related Data
    Database-->>Application: Return Related Data
```

For employee with department:
```sql
-- First Request
SELECT TOP(1) [e].[Id], [e].[Address], [e].[Age], [e].[DepartmentId], [e].[Name], [e].[Salary]
FROM [Employees] AS [e]
WHERE [e].[Id] = 1

-- Second Request (after explicit load)
SELECT [d].[Id], [d].[DateOfCreation], [d].[Name]
FROM [Departments] AS [d]
WHERE [d].[Id] = @DepartmentId
```

### Pros and Cons

| Aspect | Description |
|--------|-------------|
| Pros | - Load data only when needed<br>- Better memory usage for unused relations<br>- Fine-grained control over loading |
| Cons | - Multiple database roundtrips<br>- Need to explicitly request each related entity<br>- Can lead to N+1 query problem if not careful |

### Best Use Cases
1. When related data is optional
2. When you need fine-grained control over loading
3. When you want to reduce initial query complexity
4. When memory optimization is important

### Performance Considerations
- Each `.Load()` call generates a separate database query
- Consider using eager loading if you know you'll need the related data
- Be careful with loops that load related data (N+1 query problem)

---



# Entity Framework Core Loading Patterns - Eager Loading

## Table of Contents
- [Introduction to Eager Loading](#introduction-to-eager-loading)
- [Implementation](#implementation)
- [Generated SQL](#generated-sql)
- [Multiple Related Entities](#multiple-related-entities)
- [Pros and Cons](#pros-and-cons)
- [Advanced Scenarios](#advanced-scenarios)

## Introduction to Eager Loading

Eager loading is a pattern where related data is loaded along with the main entity in the same database query. This is achieved using the `Include` method, which generates a JOIN in the resulting SQL query.

```mermaid
graph TD
    A[Single Database Query] --> B[Main Entity Data]
    A --> C[Related Entity Data]
    B --> D[Employee Properties]
    C --> E[Department Properties]
```

## Implementation

Basic eager loading syntax:

```csharp
var employee = (from E in dbContext.Employees
                        .Include(E => E.Department)
               where E.Id == 1
               select E).FirstOrDefault();

Console.WriteLine($"Employee: {employee?.Name}, Department: {employee?.Department?.Name ?? "NA"}");
```

Generated SQL:
```sql
SELECT TOP(1) [e].[Id], [e].[Address], [e].[Age], [e].[DepartmentId], 
              [e].[Name], [e].[Salary], [d].[Id], [d].[Name]
FROM [Employees] AS [e]
LEFT JOIN [Departments] AS [d] ON [e].[DepartmentId] = [d].[Id]
WHERE [e].[Id] = 1
```

## Multiple Related Entities

You can chain multiple includes for nested relationships:

```csharp
var employee = dbContext.Employees
    .Include(E => E.Department)
    .Include(E => E.Project)
    .FirstOrDefault(e => e.Id == 1);
```

For nested related data, use ThenInclude:
```csharp
dbContext.Employees
    .Include(e => e.Department)
    .ThenInclude(d => d.Test)
```

## Pros and Cons

| Aspect | Description |
|--------|-------------|
| Pros | - Single database query<br>- Reduces round trips to database<br>- Simpler code than explicit loading |
| Cons | - Always loads related data<br>- Can retrieve unnecessary data<br>- May impact memory usage<br>- Less granular control |

## Advanced Scenarios

### Specification Pattern Integration

Eager loading works well with the Specification pattern for dynamic query building:

```csharp
public class Specification<T>
{
    private List<Expression<Func<T, object>>> includes = new();
    
    public void AddInclude(Expression<Func<T, object>> includeExpression)
    {
        includes.Add(includeExpression);
    }
    
    public IQueryable<T> ApplyIncludes(IQueryable<T> query)
    {
        return includes.Aggregate(query, (current, include) => 
            current.Include(include));
    }
}
```

### Best Practices

1. **Consider Usage Patterns**
   - Use eager loading when you know you'll need the related data
   - Avoid when related data is only occasionally needed

2. **Performance Optimization**
   - Be mindful of the amount of data being loaded
   - Consider projection (SELECT) when you need only specific properties

3. **Query Design**
   ```csharp
   // Good: When you always need department data
   public Employee GetEmployeeWithDepartment(int id)
   {
       return dbContext.Employees
           .Include(e => e.Department)
           .FirstOrDefault(e => e.Id == id);
   }

   // Good: When department data is optional
   public Employee GetEmployee(int id, bool includeDepartment = false)
   {
       var query = dbContext.Employees.AsQueryable();
       if (includeDepartment)
           query = query.Include(e => e.Department);
       return query.FirstOrDefault(e => e.Id == id);
   }
   ```

## Key Points
- Eager loading loads related data in a single query
- Uses LEFT JOIN to include related entities
- Suitable for predictable data access patterns
- Can be chained for multiple relationships
- Works well with specification pattern for dynamic queries

---
*Next section will cover Lazy Loading patterns and their implementation.*
