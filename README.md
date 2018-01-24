# PipeScript&reg;

This is a Draft Specification for PipeScript&reg;, a [DSL](https://en.wikipedia.org/wiki/Domain-specific_language) (domain specific language) for describing how to orchestrate the flow of data between systems. PipeScript&reg; is brought to you by the people at Actio Pty Ltd. We believe in an open and accessible evolution of this DSL and welcome your feedback. [DataPipes](https://github.com/ActioPtyLtd/datapipes-examples) is the official interpreter of PipeScript&reg; written in Scala and runs on the JVM, allowing for it to be deployed on Windows, Linux and macOS systems. 

## Introduction

This project was conceived by the authors out of the need to simplify and standardise the approach of coordinating the flow of data between systems and generally the dissatisfaction of the fragility, long term maintenance and limitations of common accessible ETL tools such as SSIS and Pentaho.





PipeScript is captured in the [HOCON](https://github.com/lightbend/config) format and is composed of the following sections:

1. [Tasks](#task-section) - defines operations to be performed using the incoming DOM
2. [DataSources](#datasource-section) - defines how to connect and communicate with a source
2. [Pipelines](#pipeline-section) - defines flow control of DOMs between Tasks
3. [Services](#services-section) - allows for pipeline operations to be exposed as RESTful endpoints
4. [Startup](#startup-section) - defines which pipeline to execute by default

The following BNF form of PipeScript&reg; is captured below:

```BNF
<script> ::= 'script {' <sections> '}'

<sections> ::= <task_section> [<pipelines_section>] [<services_section>] <startup>
```

## Task Section
Tasks can be identified by name and live under the tasks section of the configuration. Each task should have a task type defined which indicates what function it has. Generally, if a task supports different behaviors it will have a behavior defined. For Extractors and Loaders, tasks will require a Data Source to be defined.

The following BNF describes the supported syntax for tasks:
```BNF
<task_section> ::= 'tasks { ' <tasks> ' }'
<tasks> ::= <task> [<tasks>]
<task> ::= <name> ' { type = "' <text> '"' [<key_values>] [<datasource_section>] '}'
```

The following Task types supported are explained below:
* Extract - extract data from a DataSource
* Transform - transform data using an expression
* Assert - validate the data
* Load - create data in a DataSource
* Merge - create or update a DataSource using a key


### Task Extract
To extract data from a data source (using a task called *task_extract*), you will need the following definition:
```HOCON
task_extract {
  type = extract
  dataSource {
    ...
    query {
      read {
        ...
      }
    }
  }
}
```

The following properties are explained below:
* **type** - Always set to extract.
* **dataSource** - Contains the section that defines which [DataSource](#datasource-section) to connect to and extract data.

The Task will execute the *read* query against the defined DataSource for each item in the incoming DOMs (successful) DataSet. Each of the items (also DataSets) will be in scope in any [expressions](#expressions) used in the query part. The task produces DOMs extracted from the DataSource.

Example:

```HOCON
task_extract {
  type = extract
  dataSource {
    type = jdbc
    connect = "jdbc:postgresql://localhost/finance?user=fred&password=secret&ssl=true"
    query {
      read {
        sql = "select * from invoices where amount >= $min_amount"
      }
    }
  }
}
```
This example Task produces data by connecting to a postgresql database and querying the SQL table *invoices*, filtering by the supplied *min_amount* variable.

### Task Transform
To transform data using an expression (using a task called *task_transform*), you will need the following definition:
```
task_transform {
  type = transformTerm
  [behavior = ('batch' | 'expand')]
  term = '"' <expression> '"'
}
```

The following properties are explained below:
* **type** - Always set to transformTerm.
* **behavior** - when using *batch*, the transform is performed at a batch level, when using *expand*. The default is to transform at the DOM DataSet item level.
* **term** - the [expression](#expressions) to evaluate so the task will produce a new DOM with the DataSet result.

### Task Assert
To assert that the incoming DOM meets your criteria using an expression (using a task called *task_assert*), you will need the following definition:
```
task_assert {
  type = assert
  term = '"' <expression> '"'
  [abort = ('true' | 'false')]
  [statuscode = <number>]
  message = '"' <text> '"'
}
```

The following properties are explained below:
* **type** - Always set to assert.
* **term** - the [expression](#expressions) to evaluate to determine if the DataSet meets the correct criteria. This should return a Boolean.
* **abort** - the property specifies whether the application should abort if the assertion isn't met. Default value is false.
* **statuscode** - the property allows one to specifiy a code that may be used as an application exit code if the assertion isn't met. The default is 1.
* **message** - a descriptive text for the reason of failure.

Example:
```
task_assert {
  type = assert
  term = "this.amount.isDefined() && this.amount >= 50.00"
  abort = true
  message = "Premium amount shouldn't be null or less than $50."
}
```
This task will test for whether an amount is specified and greater than equal 50. If data can be found that fails to meet this criteria, abort immediately and return an exit code of 1.

### Task Load
To load data into a data source (using a task called *task_load*), you will need the following definition:
```
task_load {
  type = load
  dataSource {
    ...
    query {
      create {
        ...
      }
    }
  }
}
```

The following properties are explained below:
* **type** - Always set to load.
* **dataSource** - Contains the section that defines which [DataSource](#datasource-section) to connect to and load data.


### Task Merge
To merge and load data  into an entity using a set of keys (using a task called *task_merge*), you will need the following definition:
```
task_merge {
  type = mergeLoad
  entity = <entity_name>
  keys = '[' <properties> ']'
  [update = ('true' | 'false')]

  dataSource {
    ...
  }
}
```

The following properties are explained below:
* **type** - Always set to mergeLoad.
* **entity** - The name of the entity where data will be merged to.
* **keys** - a list of properties, comma seperated, that will identify whether the item in entity should be created or updated.
* **update** - Specifies whether the matched items shoud be updated if differences are detected. Default is true.
* **dataSource** - Contains the section that defines which [DataSource](#datasource-section) to connect to and merge data.

## DataSource Section
```BNF
<datasource_section> ::= 'dataSource { ' <datasource> ' }'
<datasource> ::= 'type = "' <text> '"' [<key_values>] [query_section]

<query_section> ::= 'query {' <datasource_queries> '}'
<datasource_queries> ::= <datasource_query> [<datasource_queries>]
<datasource_query> ::= <name> ' {' <templates> '}'

<templates> ::= <name> ' = "' <template> '"' [<templates>] 
<template> ::= <text> [('$'<expression> | '${'<expression>'}')] <template>

```

## Pipeline Section
```BNF
<pipelines_section> ::= 'pipelines { ' <pipelines> ' }'
<pipelines> ::= <pipeline> [<pipelines>]
<pipeline> ::= <name> ' { pipe = "' <pipeline_expression> '" }'
<pipeline_expression> ::= <name> ['(' <args> ')'] [<pipeline_operator> <pipeline_expression>]
<pipeline_operator> ::= ('|' | '&')
```

### The Pipe Operator (|)
To demonstrate the Pipe operator, take the following two tasks *T1* and *T2* as an example:
```
T1 | T2
```
![Pipe Operator](http://yuml.me/diagram/activity/(start)1->(T1)2->(T2))

For every DOM *T1* receives (1), it will execute its task on the DOM and potentially produce further DOMs which *T2* will consume in similar fashion (2).

### The Ampersand Operator (&)
To demonstrate the Ampersand operator, take the following two tasks *T1* and *T2* as an example:
```
T1 & T2
```

![Ampersand Operator](http://yuml.me/diagram/activity/(start)1->(T1)2->(start)3->(T2))

For every DOM *T1* consumes (1), allow for *T1* to produce **all** its DOM's and complete (2) its operation entirely before allowing *T2* to consume the same DOM (3).


## Services Section
The Services Section allows one to define API endpoints and the appropriate routing that needs to occur for different http methods. The BNF form is shown below:

```BNF
<services_section> ::= 'service {' [<port>] <routes> '}'
<port> ::= 'port = ' <number>
<routes> ::= ' routes = [ ' <routeList> ' ]'
<routeList> ::= <route> [',' <routeList>]
<route> ::= '{ path =  "' <url> '"' <route_methods> ' }'
<route_methods> ::= <route_method> [<route_methods>]
<route_method> ::= ('get' | 'put' | 'post' | 'patch' | 'delete') ' = ' <name>
```
Note: *name* here is the name of the pipeline to start if the spefific method is called.

Example:

```HOCON
service {
  port = 8080
  routes = [
    {
      path = "/api/v1/users"
      get = get_users
    },
    {
      path = "/api/v1/user/$userid"
      get = get_user
      put = update_user
    }
  ]
}
```

This configuration will provide access to two endpoints on port 8080, one to extract users by calling the get_users pipeline and another that allows for a specific user to be retrieved or updated. Note the *userid* parameter will be extracted and passed onto either pipeline. For the update_user pipeline, the http request as well as the *userid* parameter will be provided.

To test the users endpoint, you can use curl as follows:
```
$ curl http://localhost:8080/api/v1/users
```

## Startup Section
The Startup Section allows one to specify the default pipeline to execute if no specific pipeline is specified on the command line. The BNF is as follows:

```BNF
<startup> ::= 'startup { exec = ' <name> ' }'
```

Note: Here *name* is the default pipeline to execute.

## Common
```BNF

<startup> ::= 'startup { exec = ' <name> ' }'

<key_values> ::= <name>
<name> ::= ;alphanumeric text
<value> := ;text or number
<args> ::= <name>

<url> ::= ;text with optional variables

```

## Expressions
```BNF
<expression> ::= 
  <literal> | 
  <variable> | 
  <select> | 
  <apply> | 
  <user_function> | 
  <operator_expression> | 
  <if_statement>

<literal> ::= <number> | ('"' <text> '"') | 'false' | 'true'
<variable> ::= <name>
<apply> ::= <name> '()'
<user_function> ::= <name> '(' <args> ')'
<args> ::= <expression> [',' <expression>]
<select> ::= <expression> '.' (<variable> | <apply>)
<operator_expression> ::= <expression> <operator> <expression>
<operator> ::= '+' | '-' | '*' | '/' | '&&' | '||' | '<' | '>' | '<=' | '>=' | '->' | '==' | '!' | '!='
<if_statement> ::= 'if(' <expression> ')' ['elseif(' <expression> ')'] 'else' <expression>
```

All expressions evaluate to a DataSet. DataSets are hierarchical data structures. DataSets can be defined by the following data types: 

![DataSet](http://yuml.me/diagram/scruffy;dir:TB/class/[<<DataSet>>]^[Record],%20[<<DataSet>>]^[Array],%20[Record]++-%20%20%20%20%20%20fields%20*[<<DataSet>>],%20[Array]++-%20%20%20%20%20%20items%20*[<<DataSet>>],%20[<<DataSet>>]^[String;Date;Numeric;Boolean;Empty])

These data structures can be thought of as json structures, such as the following:

```json
{
  "person": {
    "firstName": "John",
    "lastName": "Smith",
    "address": {
      "addr1": "George St.",
      "addr2": "Sydney",
      "postcode": 2000
    },
    "phoneNumbers": [
      "98765432",
      "87654321",
      "76543212"
    ]
  }
}
```
We will use this example DataSet (call it *ds*) to demonstrate the different type of expressions.

### Literal
Literals can be of type Numeric, String, Boolean:

```javascript
> 2000
2000: Numeric
> "text"
"text": String
> false
false: Boolean
```

### Selection
Given a DataSet, dot notation allows one to access elements contained within the structureby name. 

To access the persons first name we can do the following:
```scala
> ds.person.firstName
"John": String
```

To access elements using an index we can do the following:
```scala
> ds.person.phoneNumbers(0)
"98765432": String
```

### Conditional Statement
One can use conditional statements to return different DataSets based on conditions:

```scala
> if(ds.person.address.postcode == 2000) "Sydney" else "Not Sydney"
"Sydney": String
```

### Functions
The functions that every DataSet has, independant of its type, are listed below:

Return the name of the DataSet (named *ds*):
```scala
> ds.person.address.label()
"address": String
```

Convert a DataSet to a String:
```scala
> ds.person.address.postcode.toString()
"2000": String
```

Convert a DataSet to a String in JSON format and vise versa:
```scala
> ds.person.address.toJson()
"{ \"addr1\": \"George St.\", \"addr2\": \"Sydney\", \"postcode\": 2000 }": String
> ds.person.address.toJson().parseJson().addr1
"George St.": String
```

Convert the DataSet to a String in XML format and vise versa:
```scala
> ds.person.address.toXml()
"<address><addr1>George St.</addr1><addr2>Sydney</addr2><postcode>2000</postcode></address>": String
> ds.person.address.toXml().parseXml().addr1
"George St.": String
```

Check whether a DataSet has a property value defined, returns a Boolean:
```scala
> ds.person.isDefined()
true: Boolean
> ds.email.isDefined()
false: Boolean
```

Check whether a DataSet has child elements, returns a Boolean:
```scala
> ds.person.phoneNumbers.isEmpty()
false: Boolean
```

Flatten the child elements of a DataSet, returns a Array:
```scala
ds.flatten()
```

### Higher Order Functions

Function *f*, generally specified as a lambda expression (x => f(x)) can be used as an input into the following higher order functions:

Map with function *f*:
```javascript
ds.map(a => f(a))
```

FlatMap with function *f*:
```javascript
ds.flatMap(a => f(a))
```

Filter with function *f*:
```javascript
ds.filter(a => f(a))
ds.filterNot(a => f(a))
```

Find an element with function *f*:
```javascript
ds.find(a => f(a))
```

Reduceleft with function *f*:
```javascript
ds.reduceLeft((a,b) => f(a,b))
```

### Utility Functions

```scala

// string functions
def toUpperCase(str: String): String
def toLowerCase(str: String): String
def trim(str: String): String
def substring(str: String, start: Int): String
def substring(str: String, start: Int, finish: Int): String
def contains(str: String, targetStr: String): Boolean
def replaceAll(str: String, find: String, replaceWith: String)
def split(str: String, s: String): Array[String]
def concat(strArray: Array[String], separator: String): String
def sha256(str: String): String

// numeric functions
def numeric(str: String, default: BigDecimal): BigDecimal
def numeric(str: String): BigDecimal
def numeric(str: String, format: String): String
def numeric(str: String, format: String, default: String): String

// date functions
def date(date: Date, format: String): String
def now(): Date
def dateParse(dateStr: String, formatIn: String, default: String): Date
def plusDays(date: Date, days: Int): Date

```



