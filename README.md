# long-file-reproducer

This reproducer was designed to prove that serverless workflows created with sonata-flow and quarkus-openapi-generator cannot be built if they are dependant on large yaml files that are needed at build time. 

## The Issue
This issue arose while we tried creating workflows that reference functions from large yaml files (here, openAPI spec files). After running `quarkus dev` (or `./mvnw compile quarkus:dev`) to build the workflows, we've encountered the following errors:

```
[ERROR] failed to read resource listing
com.fasterxml.jackson.core.JsonParseException: Unexpected character ('-' (code 45)) in numeric value: expected digit (0-9) to follow minus sign, for valid numeric value
 at [Source: REDACTED (`StreamReadFeature.INCLUDE_SOURCE_IN_LOCATION` disabled); line: 1, column: 3]
[...]
Caused by: org.yaml.snakeyaml.error.YAMLException: The incoming YAML document exceeds the limit: 3145728 code points.
[...]
[WARNING] Error snake-parsing yaml content
io.swagger.parser.util.DeserializationUtils$SnakeException: Exception safe-checking yaml content  (maxDepth 2000, maxYamlAliasesForCollections 2147483647)
```


## The Solution

We believe a change should be implemented [here:](https://github.com/quarkiverse/quarkus-openapi-generator/blob/f8bf842301fb5aff6c18c18a4b5f6bdfdcc1cdcc/server/deployment/src/main/java/io/quarkiverse/openapi/server/generator/deployment/codegen/ApicurioOpenApiServerCodegen.java) and spcifically in this function:
```
private File convertToJSON(Path yamlPath) throws CodeGenException {
        try {
            ObjectMapper yamlReader = new ObjectMapper(new YAMLFactory());
            Object obj = yamlReader.readValue(yamlPath.toFile(), Object.class);
            ObjectMapper jsonWriter = new ObjectMapper();
            File jsonFile = File.createTempFile(yamlPath.toFile().getName(), ".json");
            jsonFile.deleteOnExit();
            jsonWriter.writeValue(jsonFile, obj);
            return jsonFile;
        } catch (Exception e) {
            throw new CodeGenException("Error converting YAML to JSON", e);
        }
    }
```
We should change the way we create the YAMLFactory object to allow increasing the limit for the yaml file size.  
The way to do so is by configuring objects used by SnakeYAML in the following manner [here.](https://github.com/FasterXML/jackson-dataformats-text/tree/2.15/yaml#maximum-input-yaml-document-size-3-mb)
 


## Validate the issue

Clone this repository and cd into it:
```
git clone <add repo link>
cd repo name
```

Run maven / quarkus to build the project and reproduce the error:
```
quarkus dev (or)
./mvnw compile quarkus:dev
```

If you need help configuring quarkus or maven, see: https://quarkus.io/guides/config 

