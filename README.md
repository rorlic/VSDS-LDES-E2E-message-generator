# Test Message Generator
This small tool helps to generate (test) messages based on a template. It can act as a simple replacement for message source systems and allows to send on regular time intervals some data, based on a template that gets altered before sending based on a mapping.

## Docker
The generator can be run as a Docker container, after creating a Docker image for it. The Docker container will keep running until stopped.

To create a Docker image, run the following command:
```bash
docker build --tag vsds/test-message-generator .
```

To run the generator, you can use:
```bash
docker run -v $(pwd)/data:/tmp/data -e TEMPLATEFILE=/tmp/data/other.template.json -e MAPPINGFILE=/tmp/data/other.mapping.json vsds/test-message-generator
```
You can also pass the following arguments when running the container:
* `TARGETURL=<target-uri>` to POST the output to the target URI instead of to the console
* `SILENT=true` to display no logging to the console
* `MIMETYPE=<mime-type>` to use a different mime-type

Alternatively, you can also pass the template and mapping as string instead of as files, use `TEMPLATE` respectively `MAPPING`.

## Build the Generator
The generator is implemented as a [Node.js](https://nodejs.org/en/) application.
You need to run the following commands to build it:
```bash
npm i
npm run build
```

## Run the Generator
The generator works based on a JSON template, defining the structure to use for each generated item, and a JSON mapping file, defining the transformations that need to be performed on the template. It can send the generated JSON data to a target URL or simply send it to the console.

The generator takes the following command line arguments:
* `--silent=<true|false>` prevents any console debug output if true, defaults to false (not silent, logging all debug info)
* `--targetUrl` defines the target URL to where the generated JSON is POST'ed as `application/json`, no default (if not provided, sends output to console independant of `--silent`)
> **Note**: alternatively, you can provide the target URL as a plain text in a file named `TARGETURL` (located in the current working directory) allowing to change the target URL at runtime as the file is read at cron schedule time (see below), e.g.:
> ```bash
> echo http://example.org/my-ingest-endpoint > ./TARGETURL
> ```

> **Note**: for testing the target URL you can use a [webhook service](https://webhook.site/), e.g. using command line arguments `--targetUrl=https://webhook.site/f140204a-9514-4bfa-8d3e-fd18ba325ee3` or using the `TARGETURL` file:
> ```bash
> echo https://webhook.site/f140204a-9514-4bfa-8d3e-fd18ba325ee3 > ./TARGETURL
> ```
* `--mimeType=<mime-type>` mime-type of message send to target URL, defaults to `application/json`
* `--cron` defines the time schedule, defaults to `* * * * * * ` (every second)
* `--template='<json-content>'` allows to provide the JSON template on the command line, no default (if not provided, you MUST provide `--templateFile`)
* `--templateFile=<partial-or-full-pathname>` allows to provide the JSON template in a file, no default (if not provided, you MUST provide `--template`)
* `--mapping='<json-content>'` allows to provide the JSON mapping on the command line, no default (if not provided, you MUST provide `--mappingFile`)
* `--mappingFile=<partial-or-full-pathname>` allows to provide the JSON mapping in a file, no default (if not provided, you MUST provide `--mapping`)

The template or template file should simply contain a valid JSON structure (with one or more JSON objects). E.g.:
```json
[
    { "id": "my-id", "type": "Something", "modifiedAt": "2022-09-09T09:10:00.000Z" },
    { "id": "my-other-id", "type": "SomethingElse", "modifiedAt": "2022-09-09T09:10:00.000Z" }
]
```

The mapping file is also a JSON file but uses a key/value mapping where the key conforms the [JSON path specifications](https://datatracker.ietf.org/doc/id/draft-goessner-dispatch-jsonpath-00.html) and the value conforms a syntax allowing to change the value matched by the JSON path to a new value obtained by replacing the variables specified in the value part, e.g.:
```json
{ "$.id": "${@}-${nextCounter}", "$.modifiedAt": "${currentTimestamp}" }
```

The `${@}` will be replaced by the currently match value of the JSON path `$.id` (e.g. `my-id`) while any other `${<property>}` will use a property of the generator itself. Currently the only allowed properties are:
* `nextCounter`: increasing integer value, starting from 1
* `currentTimestamp`: current date and time formatted as [ISO 8601](https://en.wikipedia.org/wiki/ISO_8601) in UTC (e.g. `2007-04-05T14:30:00.000Z`)

You can run the generator after building it, e.g.:

Using this [template](./data/other.template.json) and this [mapping](./data//other.mapping.json) and with silent output to console:
```bash
node ./dist/index.js --templateFile ./data/other.template.json --mappingFile ./data/other.mapping.json --silent
```
This results in something like the following:
```
{"id":"my-id-1","type":"Something","modifiedAt":"2022-09-12T13:15:42.009Z"}
{"id":"my-id-2","type":"Something","modifiedAt":"2022-09-12T13:15:43.003Z"}
{"id":"my-id-3","type":"Something","modifiedAt":"2022-09-12T13:15:44.003Z"}
...
```
By specifying the template (containing multiple objects) and mapping on the command file:
```bash
node ./dist/index.js --template '[{"id": "my-id", "type": "Something", "modifiedAt": "2022-09-09T09:10:00.000Z" },{ "id": "my-other-id", "type": "SomethingElse", "modifiedAt": "2022-09-09T09:10:00.000Z" }]' --mapping '{ "$..id": "${@}-${nextCounter}", "$..modifiedAt": "${currentTimestamp}" }' --silent
```
This results in something like:
```json
[{"id":"my-id-1","type":"Something","modifiedAt":"2022-09-12T13:44:12.010Z"},{"id":"my-other-id-2","type":"SomethingElse","modifiedAt":"2022-09-12T13:44:12.010Z"}]
[{"id":"my-id-3","type":"Something","modifiedAt":"2022-09-12T13:44:13.005Z"},{"id":"my-other-id-4","type":"SomethingElse","modifiedAt":"2022-09-12T13:44:13.005Z"}]
[{"id":"my-id-5","type":"Something","modifiedAt":"2022-09-12T13:44:14.004Z"},{"id":"my-other-id-6","type":"SomethingElse","modifiedAt":"2022-09-12T13:44:14.004Z"}]
...
```

Alternatively you can generate the output using a different time schedule (e.g. every 2 seconds) to a [dummy HTTP server](https://docs.webhook.site/) (including debugging to the console):
```bash
node ./dist/index.js --templateFile ./data/other.template.json --mappingFile ./data/other.mapping.json --cron '*/2 * * * * *' --targetUrl https://webhook.site/28dba053-5bc2-4934-9cd8-0541012470a5
```
This results in:
```
data template:  { "id": "my-id", "type": "Something", "modifiedAt": "2022-09-09T09:10:00.000Z" }
Mapping:  {
  '$.id': '${@}-${nextCounter}',
  '$.modifiedAt': '${currentTimestamp}'
}
Runs at:  */2 * * * * *
Sending:  {"id":"my-id-1","type":"Something","modifiedAt":"2022-09-12T13:23:58.017Z"}
Next run at:  2022-09-12T15:24:00.000+02:00
Sending:  {"id":"my-id-2","type":"Something","modifiedAt":"2022-09-12T13:24:00.004Z"}
Next run at:  2022-09-12T15:24:02.000+02:00
Sending:  {"id":"my-id-3","type":"Something","modifiedAt":"2022-09-12T13:24:02.003Z"}
Next run at:  2022-09-12T15:24:04.000+02:00
...
```
