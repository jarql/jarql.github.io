---
title: JARQL: SPARQL for JSON
---

# JARQL: SPARQL for JSON
Jarql is a tool, inspired by [Tarql](https://tarql.github.io/), for converting JSON to RDF using SPARQL 1.1 syntax, mostly using constract queries.

|||||
| --- | --- | --- | --- |
| [Code](https://github.com/jarql/jarql) | [Download](https://github.com/jarql/jarql/releases/latest) | [Examples](#examples) | [About](about) |

## Introduction
JARQL reads a JSON file from input and converts it in a *raw graph* using the Jarql ontology 
(```jarql: <http://jarql.com/>```). The idea is to convert the JSON in a graph on which you can ```CONSTRUCT``` SPARQL queries 
to create the RDF graph that you want.

### How it works
The root of the JSON file is identified by ```jarql:root``` and each field becomes an attribute of the same ontology. 
For example the field ```name``` becomes ```jarql:name```. So a minimal example like:
```
{ "fieldName": "fieldValue"}
```
produces:

```jarql:root jarql:filedName "fieldValue"```

A nested object becomes a blank node. So if you have:
```
{
    "name": "Paolino",
    "surname": "Paperino",
    "uncle: {
        "name": "Paperone",
        "surname": "De Paperoni"
    }
}
```
you can access to ```"name": "Paperone"``` using a variable ```?uncle``` that identifies the blank node:
```
...
WHERE {
    jarql:root jarql:uncle ?uncle .
    ?uncle jarql:name ?uncleName .
}
...
```
Instead, if you have an array:
```
{
    "name": "Paperino",
    "nephew": ["Qui", "Quo", "Qua"],
}
```
every element inside it becomes one object of array's name property:
```
@prefix jarql: <http://jarql.com/>.
jarql:root jarql:name "Paperino";
           jarql:nephew "Qui", "Quo", "Qua"
```
that can be queried by:
```
...
WHERE {
    jarql:root jarql:nephew ?nephew .
}
...
```

## Usage
```java -jar jarql-<version>.jar <JSON-File> <Query-File>```

So if your JSON is called ```foobar.json``` and you setup a query file called ```foobar.query``` and 
the latest software version is ```1.0-pre1``` you can run:

```java -jar jarql-1.0-pre1.jar foobar.json foobar.query```

## Examples
### Example 1 (Tenders in Paperopoli)
#### JSON input file
```
{
    "cig": "XXXX4A36A7",
    "strutturaProponente": {
        "codiceFiscaleProp": "01111111111",
        "denominazione": "Comune di Paperopoli"
    },
    "oggetto": "Costruzione di un grande deposito su una collina.",
    "partecipanti": [
        {
            "ragioneSociale": "OCOPOLI COSTRUZIONI S.R.L.",
            "vatID": "42424242424"
        },
        {
            "ragioneSociale": "DE PAPERONI S.P.A.",
            "vatID": "23232323232"
        }
    ]
}
```
#### RAW graph automagically created
```
@prefix jarql: <http://jarql.com/>.
jarql:root jarql:cig "XXXX4A36A7";
           jarql:strutturaProponente [
               jarql:codiceFiscaleProp "01111111111";
               jarql:denominazione "Comune di Paperopoli"
           ];
           jarql:oggetto "Costruzione di un grande deposito su una collina.";
           jarql:partecipanti [
               jarql:ragioneSociale "OCOPOLI COSTRUZIONI S.R.L.";
               jarql:vatID "42424242424"
           ];
           jarql:partecipanti [
               jarql:ragioneSociale "DE PAPERONI S.P.A.";
               jarql:vatID "23232323232"
].
```
#### SPARQL input query
```
prefix rdfs: <http://www.w3.org/2000/01/rdf-schema#>
prefix dcterm: <http://purl.org/dc/terms/>
prefix pc: <http://contrattipubblici.org/>
prefix jarql: <http://jarql.com/> 

CONSTRUCT { 
  ?URI a "Contract";
       dcterm:identifier ?cig;
       pc:contractingAuth ?URIpa;
       rdfs:label ?oggetto.
  ?URIpa a "PubAdmin";
       rdfs:label ?denominazione;
       dcterm:identifier ?codiceFiscaleProp.
  ?URIpart a "Participant";
       rdfs:label ?ragioneSocialePart;
       dcterm:identifier ?vatIDPart.
}
WHERE {
  jarql:root jarql:cig ?cig.
  BIND(URI(CONCAT("http://test.yo/", MD5(?cig))) as ?URI).
  jarql:root jarql:strutturaProponente ?proponente.
  ?proponente jarql:codiceFiscaleProp ?codiceFiscaleProp.
  ?proponente jarql:denominazione ?denominazione.
  BIND(URI(CONCAT("http://test.yo/", MD5(?codiceFiscaleProp))) as ?URIpa).
  jarql:root jarql:oggetto ?oggetto.
  jarql:root jarql:partecipanti ?partecipanti.
  ?partecipanti jarql:ragioneSociale ?ragioneSocialePart.
  ?partecipanti jarql:vatID ?vatIDPart.
  BIND(URI(CONCAT("http://test.yo/", MD5(?vatIDPart))) as ?URIpart).
}
```
#### RDF in output
```
<http://test.yo/1f839a6f81727fc54c06d4e4be5bc51e>
        a       "Participant"^^<http://www.w3.org/2001/XMLSchema#string> ;
        <http://www.w3.org/2000/01/rdf-schema#label>
                "DE PAPERONI S.P.A."^^<http://www.w3.org/2001/XMLSchema#string> ;
        <http://purl.org/dc/terms/identifier>
                "23232323232"^^<http://www.w3.org/2001/XMLSchema#string> .

<http://test.yo/0e2d5004f3fdb311f2832aa0e08031a2>
        a       "PubAdmin"^^<http://www.w3.org/2001/XMLSchema#string> ;
        <http://www.w3.org/2000/01/rdf-schema#label>
                "Comune di Paperopoli"^^<http://www.w3.org/2001/XMLSchema#string> ;
        <http://purl.org/dc/terms/identifier>
                "01111111111"^^<http://www.w3.org/2001/XMLSchema#string> .

<http://test.yo/a80ebd2e303fa3ca5eb53f2dfb689186>
        a       "Participant"^^<http://www.w3.org/2001/XMLSchema#string> ;
        <http://www.w3.org/2000/01/rdf-schema#label>
                "OCOPOLI COSTRUZIONI S.R.L."^^<http://www.w3.org/2001/XMLSchema#string> ;
        <http://purl.org/dc/terms/identifier>
                "42424242424"^^<http://www.w3.org/2001/XMLSchema#string> .

<http://test.yo/cc44b6b5f7889c004a399e90679996a0>
        a       "Contract"^^<http://www.w3.org/2001/XMLSchema#string> ;
        <http://www.w3.org/2000/01/rdf-schema#label>
                "Costruzione di un grande deposito su una collina."^^<http://www.w3.org/2001/XMLSchema#string> ;
        <http://contrattipubblici.org/contractingAuth>
                <http://test.yo/0e2d5004f3fdb311f2832aa0e08031a2> ;
        <http://purl.org/dc/terms/identifier>
"XXXX4A36A7"^^<http://www.w3.org/2001/XMLSchema#string> .
```

## Limitations
+ Nested arrays in JSON are not supported.
+ The order of JSON elements in an array is irrelevant (so you can't use their order as a way to construct the RDF 
with enumerative properties).

Last update: 2017-01-30
