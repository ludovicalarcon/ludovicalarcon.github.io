---
layout: post
title: Validating Helm Charts Inputs with JSON Schema
img: helm.png
tags: [kubernetes, helm]
---

We will explore helm chart `JSON Schema` validation with a basic sample available [on github](https://github.com/ludovicalarcon/helm-json-schemas-sample).  
`JSON Schema` provides similar capabilities as the `required` or `fail` functions but  
can be used to provide more robust Helm input validation with type, pattern, min, max, etc...

According to [json-schema.org](https://json-schema.org/), JSON Schema is
> the vocabulary that enables JSON data consistency, validity, and interoperability at scale
<br>

## JSON Schema from value.yaml

First let's define our helm values and their requirements

```yaml
appName: "foo" # Mandatory not empty string
environment: "dev" # Mandatory and can be only be dev, tst or prd
replicasCount: 1 # Mandatory non negative integer 
image:
  repository: "nginx" # Mandatory not empty string
  tag: "1.27" # Mandatory not empty string
```

Now that we have our requirements, we need to write our JSON Schema.  
The JSON schema needs to be in a file named `values.schema.json`  
and it has to be located alongside values.yaml file.

```json
{
  "$schema": "https://json-schema.org/draft/2019-09/schema",
  "type": "object",
  "title": "Root Schema",
  "required": [
    "appName",
    "environment",
    "replicasCount",
    "image"
    ],
  "properties": {
    "appName": {
      "type": "string",
      "minLength": 1
    },
    "environment": {
      "type": "string",
      "enum": [
        "dev",
        "tst",
        "prd"
      ]
    },
    "replicasCount": {
      "type": "integer",
      "minimum": 0,
    },
    "image": {
      "type": "object",
      "required": [
        "repository",
        "tag"
      ],
      "properties": {
        "repository": {
          "type": "string",
          "minLength": 1
        },
        "tag": {
          "type": "string",
          "minLength": 1
        }
      }
    }
  }
}
```

It is possible to generate the schema structure from the values.yaml file and then adding the requirements.  

- Covert YAML to JSON, for example with [this tool](https://www.bairesdev.com/tools/json2yaml/)
- Generate Schema structure with [jsonschema.net](jsonschema.net)

## JSON Schema validation

The schema will be automatically validated when running one of the following commands:

- `helm install`
- `helm upgrade`
- `helm template`
- `helm lint`

### Output example

Using this values file containing several violations of the schema

```yaml
# values-ko.yaml
image:
  repository: "" # an empty string
  tag: 1.2 # int instead of string
appName: # no value provided
environment: foo # unknown environment
replicasCount: "2" # int instead of string
```

```
helm template . -f values-ko.yaml

Error: values don't meet the specifications of the schema(s) in the following chart(s):
json-schemas-sample:
- (root): appName is required
- environment: environment must be one of the following: "dev", "tst", "prd"
- replicasCount: Invalid type. Expected: integer, given: string
- image.repository: String length must be greater than or equal to 1
- image.tag: Invalid type. Expected: string, given: number
```

Another example by incorrectly overriding one value using set param

```
helm template . --set replicasCount=-1

Error: values don't meet the specifications of the schema(s) in the following chart(s):
json-schemas-sample:
- replicasCount: Must be greater than or equal to 0
```

We now have a simple and robust validation for our chart inputs
