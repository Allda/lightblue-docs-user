# Query

Each service data layer request consists of a search criteria,
projection, and sort specification.

## Search Criteria
```
query_expression := logical_expression | comparison_expression
logical_expression := unary_logical_expression | nary_logical_expression
unary_logical_expression := { unary_logical_operator : query_expression }
nary_logical_expression := { nary_logical_operator : [ query_expression,...] }
unary_logical_operator := "$not"
nary_logical_operator := "$and" | "$or" | "$all" | "$any"

comparison_expression := relational_expression | array_comparison_expression
relational_expression := binary_relational_expression |
                         nary_relational_expression |
                         regex_match_expression

binary_relational_expression := field_comparison_expression |
                                value_comparison_expression
field_comparison_expression := { field: <field>,
                                 op: binary_comparison_operator,
                                 rfield: <field> }
value_comparison_expression := { field: <field>,
                                 op: binary_comparison_operator,
                                 rvalue: <value> }
binary_comparison_operator := "=" | "!=" | "<" | ">" | "<=" | ">=" |
                              "$eq" | "$neq" | "$lt" | "$gt" | "$lte" | "$gte"

nary_relational_expression := { field: <field>,
                                op: nary_comparison_operator,
                                values: value_list_array }
nary_comparison_operator := "$in" "$not_in" "$nin"

regex_match_expression := { field: <field>, regex: <pattern>,
                            case_insensitive: false,
                            extended: false,
                            multiline: false,
                            dotall: false }

array_comparison_expression := array_contains_expression |
                               array_match_expression
array_contains_expression := { array: <field>,
                               contains: "$any" | "$all" | "$none",
                               values: value_list_array }
array_match_expression := { array: <field>,
                            elemMatch: query_expression }
value_list_array := [ value1, value2, ... ]
```
Examples:

Search documents with login=someuser:
```
    {
        "field":"login",
        "op":"=",
        "rvalue":"someuser"
    }
```
Search documents whose firstname is not equal to lastname:
```
    {
        "field":"firstname",
        "op":"$ne",
        "rfield":"lastname"
    }
```
Logical expressions:
```
    {
        "$not":{
            "field":"firstname",
            "op":"$ne",
            "rfield":"lastname"
        }
    }
```
```
    {
        "$and":[
            {
                "field":"firstname",
                "op":"$ne",
                "rfield":"lastname"
            },
            {
                "field":"login",
                "op":"=",
                "rvalue":"someuser"
            }
        ]
    }
```
Can use "$all" instead of "$and".

```
    {
        "$or":[
            {
                "field":"firstname",
                "op":"$ne",
                "rfield":"lastname"
            },
            {
                "field":"login",
                "op":"=",
                "rvalue":"someuser"
            }
        ]
    }
```
Can use "$any" instead of "$or".

List of values:
```
    {
        "field":"city",
        "op":"$in",
        "values":[
            "Raleigh",
            "Cary"
        ]
    }
```
Search for a login name, starting with a prefix, case insensitive:
```
    {
        "field":"login",
        "regex":"prefix.*",
        "options":"i"
    }
 ```
Search for a document where an array field contains "value1" and "value2"
```
    {
        "array":"someArray",
        "contains":"$all",
        "values":[
            "value1",
            "value2"
        ]
    }
```
Search for a document that contains an array with an object
element with "item" field equals 1.
```
    {
        "array":"someArray",
        "elemMatch":{
            "field":"item",
            "op":"=",
            "rvalue":"1"
        }
    }

```

## Projection

Projection specification determines what fields will be returned from
a query. It is illegal to pass an empty projection specification. The
caller has to explicitly specify what fields are required. Caller can
specify patterns instead of listing fields individually.

Projection rules are executed in the order given. If an included field
is later excluded by a projection rule, the field remains excluded,
and vice versa.

```
projection := basic_projection | [ basic_projection, ... ]
basic_projection := field_projection | array_projection

field_projection := { field: <pattern>, include: boolean, recursive: boolean }
array_projection := { field: <pattern>, include: boolean,
                      match: query_expression, project : projection  } }  |
                    { field: <pattern>, include: boolean,
                      range: [ from, to ], project : projection }
```
Examples:

Return firstname and lastname:
```
 [ {  field: "firstname", include: true },
   { field: "lastname", include: true } ]
```
Return everything but firstname:
```
 [ { field: "*", include: true, recursive: true},
   { field: "firstname", include: false} ]
```
Return only those elements of the addresses array with
city="Raleigh", and only return the streetaddress field.
```
 [ { field: "address.*", include: true,
     match: { city="Raleigh" }, project: { "streetaddress": true} } ]
```
Return the first 5 addresses
```
 [ { field: "address.*", include: true, range: [ 0, 4 ],
     project: { "*", recursive: true} }]
```

## Sort
```
sort := sort_key | [ sort_key, ... ]
sort_key := { field : "$asc" | "$desc" }
```
Examples:

Sort by login ascending:
```
    { "login":"asc" }
```
Sort by last update date descending, then login ascending
```
    [ { "lastUpdateDate":"desc" }, { "login": "asc" }]
```

## Update
```
update_expression := partial_update_expression |
                    [ partial_update_expression,...]
partial_update_expression := primitive_update_expression |
                             array_update_expression
primitive_update_expression := { $set : { path : rvalue_expression , ...} } |
                               { $unset : path } |
                               { $unset :[ path, ... ] }
                               { $add : { path : rvalue_expression, ... } }
rvalue_expression := value | { $valueof : path } | {}

array_update_expression := { $append : { path : rvalue_expression } } |
                           { $append : { path : [ rvalue_expression, ... ] }} |
                           { $insert : { path : rvalue_expression } } |
                           { $insert : { path : [ rvalue_expression,...] }} |
                           { $foreach : { path : update_query_expression,
                                         $update : foreach_update_expression } }
update_query_expression := $all | query_expression
foreach_update_expression := $remove | update_expression
```

Modifications are executed in the order they're given, and effects are
visible to subsequent operations immediately. For instance, to remove
the first two elements of an array, use:
```
    { "$unset" : [ "arr.0","arr.0" ] }
```

### Primitive updates
```
     { "$set" : { path : value } }
     { "$set" : { path : { "$valueof" : field } }
     { "$unset" : path }  (array index is supported, can be used to
                           remove elements of array)
     { "$add" : { path : number } }
     { "$add" : { path : { "$valueof" : pathToNumericField } }
```

### Array updates:
```
     { "$append" : { pathToArray : [ values ] } } (values can be empty
                      objects (extend array with a  new element)

     { "$append" : { pathToArray : value } }

     { "$insert" : { pathToArray.n : [ values ] } }
     { "$insert" : { pathToArray.n : value } } (index (n) can be negative)
```

Assuming that value(s) is inserted at the given index and the item at
index and items after are shifted down


### Updating array elements:
```
  { "$foreach" : { pathToArray : query_expression,
                   "$update": update_expression } }
```

The query_expression determines the elements that will be
updated. query_expression can be $all. update_expression can be
$remove.

Examples updating simple fields:
```
{ $set: { x.y.z : newValue } } : Update field x.y.z. x and y are objects.
                                  z is a value.

{ $unset : x.y.z } : Remove x.y.z from doc. x and y are objects.
                     z can be a value, object, or array.  x and y
                     are not removed from the document, only z is removed

{ $add: { x.y.z : number } } : Similar to $set
```

Examples of $foreach:

```
{ "$foreach" : { "x.y.*.z" : "$all", "$update": ... }  }
```
Select all elements of the array x.y.*.z. Here, y is also an array.

```
{ "$foreach" : { "x.y.*.z" : { "field" : "x.y.*.z", "op":"$gt","rvalue":5 },
                 "$update":...  }}
```
Select all elements of "x.y.*.z" that are greater than 5.

```
{ "$foreach" : { "x.y.*.z" : { "field" : "x.y.*.z.w", "op":"$eq","rfield":"k" },
                               "$update":... } }
{ "$foreach" : { "x.y.*.z" : { "field" : "$this.w", "op":"$eq","rfield":"k" },
                               "$update":... } }
```
Select all elements of "x.y.*.z" where the field 'w' is
equal to the top-level field 'k'.

```$update``` specifies how each matching array element will be updated.
The update expression itself may contain a $foreach.

Example:
```
 { "$foreach" : { "x.y.*.z" : "$all",
                  "$update": { "$set" : { "$this" : 1 } } }
```
Set all elements to 1

```
{ "$foreach" : { "x.y.*.z" : ...,
                 "$update" : { "$set" : { "$this.k" : "blah" } } }
```
Set field 'k' in matching elements to 'blah' (".k" means
"k relative to current context" )

```
{ "$foreach" : { "x.y.*.z" : ..... "$update" : {
           "$foreach" : { "$this.arr" : { "field" : "$this.p", "op":"$eq","rvalue":1},

           "$update" : { "$set" : { "$this.v" : { "$valueof":"k"}}}}}}}
```
For each element of ```x.y.*.z``` where ```x.y.*.z.arr.p=1``` set ```x.y.*.z.arr.v```
to the value of ```k``` (k is from root of doc) .
