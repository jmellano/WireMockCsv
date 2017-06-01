﻿
# Wiremock CSV Extension

*Read this documentation in other language: [Français](README.fr.md).*

## Introduction

The WireMock extension will allow you to read data from CSV with SQL queries, in order to automatically generate response JSON from this data.

## Build the application:

Maven 3 and a JDK 8 are necessary to build the archive. Then, the following command has to be run:

    mvn clean install

The "target" directory will then contain a standalone jar file with all necessary binaries.

## Launch in a standalone mode with console:

    java -jar ".\WiremockCsv-1.0-standalone.jar" -Dfile.encoding=UTF-8 --port 8181 --root-dir "###MY_PROJECT_PATH###\src\test\resources\mock"

## Launch in a standalone mode with Eclipse:

Use the provided launch configuration, eventually changing the root directory.

## Usage:

### Create your DB

It's a good idea to start from the provided test data. It's a good example.
To create you CSV database, follow these steps:
* Take the Database.properties and Database.script files from the examples.
* Create your CSV (one file per table, without header)
* Create the lines "CREATE TEXT TABLE PUBLIC.XXX" et "SET TABLE PUBLIC.XXX SOURCE 'xxx.csv;encoding=UTF-8;fs=\semi'" in Database.script, for all CSV. Columns names must be surrounded by double quotes, this is the key element for the field mappings to be correct.

That's it ! :-)

### Create your CSV mapping

*Note: it remains possible to use standard WireMock mappings and files in addition to specific CSV transformer files.*

The concept is simple: interrogate the CSV DB in SQL and map results to JSON response using SQL column names as JSON field names.

The standard "jsonBody" is replaced with 2 values:
* "transformers": ["wiremock-csv"]
* "transformerParameters": { Structure, Request }

The structure allows defining expected result structure and where will be integrated the request result. The result place is represented by a parameter named "${WireMockCsv}".
Default structure is `${WireMockCsv}`, to simply output the JSONified result.
Structure change example:

    "structure": {
      "main": "${WireMockCsv}"
    },


The request allows:
* Listing a table lines, with one or more filters from HTTP parameters.
* Performing joins, subselects.
* Obtaining a complex Object hierarchy as a result, with sub-objects and sub-lists.

### Building the request

A request is composed by several components:
* "query": A SQL request, potentially parameterized with HTTP request parameters or "mother" query results (for sub-queries):
    * A parameter is declared as follows: ${myParameter} or $[myParameter] to escape quotes in SQL strings.
    * Main query: It will be replaced with the value of the value of the custom parameter with the same name, or if not found with the value of the HTTP parameter with the same name.
    * Sub-query: It will be replaced with the value of the column with the same name, or if not found with the value of the value of the custom parameter with the same name, or if not found with the value of the HTTP parameter with the same name.
    * A parameter with no replacement values will be replaced by an empty String.
* "mask": A list of column names of the query results which will not appear in the generated JSON. This allows retrieving values to use as parameters for sub-requests. Other trick, use "select*" and mask one column instead of listing all needed columns, the request will be sorter.
* "aliases": Alternative to parameter sub-objects and fields names, instead of using columns names.
* "subqueries": It's simply a list of Map [String ; Request]. The key is the JSON field name of the future sub-list or sub-object to retrieve and the request is again all of the components "query", "subqueries", etc ... (except "noLines" and "customParameters"). It's hence possible to interlock an infinite number of sub-requests.
* "resultType": Tells if the expected result is a value ("value"), a unique object ("object") or a list ("list"). List by default. If "object" and several objects returned by the query, then only the first one is taken into account, the others are ignored. If "value" and several columns, then only the first one is taken into account, the others are ignored.
* "noLines": If no lines are returned by the SQL query the HTTP status, status message and response can be overriden with this parameter.
* "customParameters": Allows creating new parameters, eventually derived from existing ones. See dedicated chapter.

The SQL columns names or the SQL alias given while querying provide directly the JSON field name.
Example: `select "myField" from my_table`
Result:

    {
      "data":[ {
        "myField" : "Value 1"
      }, {
        "myField" : "Value 2"
      } ]
    }

Or: `select "myField" as "myAlias" from my_table`
Result:

    {
      "data":[ {
        "myAlias" : "Value 1"
      }, {
        "myAlias" : "Value 2"
      } ]
    }

It is possible to obtain sub-objects by using a double underscore ("__") to separate the names of the JSON fields. There are no limits on the number of levels in the hierarchy.
Example: `"query": "select "myField", "myOtherField1" as "other__myField1", "myOtherField2" as "other__myField2" from ma_table"`
Result:

    {
      "data":[ {
        "myField" : "valeur 1",
        "other" : {
          "myField1" : "Value 1 - 1"
          "myField2" : "Value 1 - 2"
        }
      }, {
        "myField" : "valeur 2",
        "other" : {
          "myField1" : "Value 2 - 1"
          "myField2" : "Value 2 - 2"
        }
      } ]
    }

Other possibility, using aliases:
Example:

    "query": "select * from table1 t1 JOIN table2 t2 ON t1.\"code\" = t2.\"externalCode\"",
    "aliases":  {
      "table2": {
        "prefix": "subObjectTable2",
        "columns": {
          "code": "codeTable2"
        }
      }
    }

Result:

    {
      "data":[ {
        "code" : "Code Table 1",
        "otherField1" : "Other field table 1",
        "subObjectTable2" : {
          "codeTable2" : "Code Table 2",
          "externalCode" : "Code Table 1",
          "otherField2" : "Other field table 1"
        }
      } ]
    }
   
Finally, a SQL query can be long and difficult to read if on one line. JSon does not support line feeds, so the possibility to declare the query as an array of Strings has been given. All blocks will be concatenated with a space separator to execute the whole query.

### Global configuration

The extention allows providing global parameters via a configuration file. This file is named "WireMockCsv.json.conf" and is placed in the "csv" directory (i.e. same level than the database).
It contains JSON Data and its structure is the same than the "transformerParameters" field, knowing that in the real life, only few parameters will be useful to configure here  (minly "structure" and "no-line").
Global parameters may be overriden individually for each mapping.

Configuration file example:

    {
      "structure": {
        "data": "$WireMockCsv"
      },
      "noLines": {
        "status": 404,
        "statusMessage": "No data.",
        "response": {
          "message": "No data found."
        }
      }
    }

### Custom parameters

In some cases, the available parameters can't be directly integrated in the SQL queries. For example, if you have an array of values or a String containing several separated values to integrated in a SQL IN clause.

To achieve this, you may have to use the custom parameters features, which will allow you transforming the original parameter to obtain another one you will be able to use. Several operations can be chained to achieve the desired transformation.

Syntax:

    "transformerParameters":{
      "customParameters": {
        "first_parameter_name": {
          "action": "action-name",
          "param1": "value",
          "param2": "value"
        },
        "second_parameter_name": {
          "action": "action-name",
          "param1": "value",
          "param2": "value"
        }
      }
    }

The following operations are available:
* "split": split a string using a regular expression. Parameters:
    * "action": "split".
    * "sourceParam": Name of the parameter containing the value to split.
    * "regexp": The regular expression to use for split.
* "replace": replace all occurences identified by a regular expression matches in a String with a replacement. Parameters:
    * "action": "replace",
    * "sourceParam": Name of the parameter containing the value to apply replacement on.
    * "regexp": The regular expression to use for replace.
    * "replacement": The value used for replacement. Supports regexp groups ($0, $1, ...)
* "concatenate": concatenates one or more strings with a prefix, a separator and a suffix. Parameters:
    * "action": "concatenate",
    * "sourceParam": Name of the parameter containing the values to concatenate.
    * "prefix"
    * "suffix"
    * "separator"
* "fromQuery": Use the result of an SQL query and put it in a parameter. If the query returns multiple columns or multiple lines, the parameter will hold a list. Multiple columns and lines are not handled. Parameters:
    * "action": "fromQuery",
    * "query": The query to execute. All previously available or defined parameters can be used in it. 
* "escapeSql": Escapes quotes in one or more strings to use them in SQL strings. Parameters:
    * "action": "escapeSql",
    * "sourceParam": Name of the parameter containing the values to escape.


Example :

    "transformerParameters":{
      "customParameters": {
        "array_invoice_codes": {
          "action": "fromQuery",
          "query": "select code from invoice where year = ${yearToSearch}"
        },
        "string_invoice_codes": {
          "action": "concatenate",
          "sourceParam": "array_invoice_codes",
          "prefix": "all invoice codes: [",
          "separator": ", "
          "suffix": "]",
        }
      },
      "query": "values('${string_invoice_codes}')"
    }

### Full example

    {
      "request": {
        "method": "GET",
        "urlPath": "/url",
        "queryParameters" : {
          "parametreHttp" : {
            "matches" : "..*"
          }
        }
      },
      "response": {
        "status": 200,
        "headers": { "Content-Type": "application/json; charset=utf-8" },
        "transformerParameters":{
          "query":"SELECT t.* FROM my_table t WHERE t.\"code\" = '${httpParameter}'",
          "subqueries": {
            "subListe1": {
              "query":"SELECT st.* FROM my_sub_table1 st WHERE st.\"externalCode\" = '${code}'",
              "mask": ["fieldToHide"]
            },
            "subListe2": {
              "query":[
                "SELECT st.*, ast.\"myField\" AS \"subObjet__myField\", ast.\"myOtherField\" AS \"sousObjet__myOtherField\"",
                "FROM my_sub_table2 st JOIN other_sub_table ast ON st.\"otherExternalCode\" = ast.\"code\"",
                "WHERE st.\"externalCode\" = '${code}'"],
              "subqueries": {
                "subSubListe": {
                  "query": "SELECT sst.* FROM my_sub_sub_table sst WHERE sst.\"externalCode\" = '${code}'"
                }
              },
              "mask": ["externalCode", "otherFieldToHide"]
            },
            "noLines": {
              "status": 404,
              "statusMessage": "No data.",
              "response": "No data."
            }
          }
        },
        "transformers": ["wiremock-csv"]
      }
    }


## Test JSON URL:

Following URL are valid if standalone extension has been launched locally via one of the two way exposed above.

### Invoicing data

Following examples goes on from the simplest to the most complex. The last one uses all extension features, by mixing requests types.
The different SQL queries present some examples allowing filtering, count, validate, etc...
In addition, this example uses a global configuration file allowing to change the structure of the responses and the "no-line" behavior.

* Search clients with or without filtering
    * http://localhost:8181/rechercherClients
    * http://localhost:8181/rechercherClients?filtreNom=Jo
    * http://localhost:8181/rechercherClients?filtreNom=Jo&filtrePrenom=Do
   
* Retrieve a client, with data aggregation
    * http://localhost:8181/recupererClient?clientCode=CLI01
    * http://localhost:8181/recupererClient?clientCode=CLI02
    * http://localhost:8181/recupererClient?clientCode=CLI03
    * http://localhost:8181/recupererClient?clientCode=CLI04

* Validate client code
    * http://localhost:8181/validerClient?clientCode=CLI01
    * http://localhost:8181/validerClient?clientCode=CODE_BIDON

* Search articles with or without filtering
    * http://localhost:8181/rechercherArticles
    * http://localhost:8181/rechercherArticles?filtreLibelle=Cla

* Search invoices with or without filtering, with data aggregation and result structure changes. Two different syntaxes demonstrated
    * http://localhost:8181/rechercherFactures
    * http://localhost:8181/rechercherFactures?clientCode=CLI01
    * http://localhost:8181/rechercherFactures?clientCode=CLI02
    * http://localhost:8181/rechercherFactures2
    * http://localhost:8181/rechercherFactures2?clientCode=CLI01
    * http://localhost:8181/rechercherFactures2?clientCode=CLI02

* Retrieve an invoice, with client sub-object. Two different syntaxes demonstrated
    * http://localhost:8181/recupererFacture?factureCode=FAC01
    * http://localhost:8181/recupererFacture2?factureCode=FAC01

* Retrieve an invoice line, with article and invoice sub-objects and with client sub-sub-object for the invoice
    * http://localhost:8181/recupererLigneFacture?ligneFactureCode=L01_01

* Retrieve an invoice, with client sub-object and sub-list of lines, and handling the absence of data
    * http://localhost:8181/recupererFactureAvecLignes?factureCode=FAC01
    * http://localhost:8181/recupererFactureAvecLignes?factureCode=NOT_EXISTING

* Search invoices containing all articles in a list. Shows different syntaxes to use custom parameters
    * http://localhost:8181/rechercherFacturesSelonListeArticles1?articleCodes=ART01;ART03
    * http://localhost:8181/rechercherFacturesSelonListeArticles2?articleCodes=ART01;ART03
    * http://localhost:8181/rechercherFacturesSelonListeArticles3?articleCode=ART01&articleCode=ART03

* Retrieve a client with sub-lists of addresses and invoices, each invoice with a sub-list of lines, each line with an article sub-object. With or without filtering. Two different syntax demonstrated
    * http://localhost:8181/recupererClientAvecAdressesEtFacturesAvecLignesAvecArticle?clientCode=CLI01
    * http://localhost:8181/recupererClientAvecAdressesEtFacturesAvecLignesAvecArticle?clientCode=CLI01&dateCommandeMin=2017-03-01
    * http://localhost:8181/recupererClientAvecAdressesEtFacturesAvecLignesAvecArticle2?clientCode=CLI01
    * http://localhost:8181/recupererClientAvecAdressesEtFacturesAvecLignesAvecArticle2?clientCode=CLI01&dateCommandeMin=2017-03-01

* Extract data, by specifying only sub-queries
    * http://localhost:8181/extraction

* Dumb examples to show other possibilities on custom parameters
    * http://localhost:8181/testCustomParamFromQuery1
	* http://localhost:8181/testCustomParamFromQuery2
	* http://localhost:8181/testCustomParamFromQuery3
