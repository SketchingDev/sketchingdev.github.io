---
layout: blog-post
title:  "Testing Pig scripts that use Avro"
date:   2016-08-29 00:00:00
categories: testing pig avro java
---

Writing unit tests for [Apache Pig](https://pig.apache.org/) is easy with [PigUnit](https://pig.apache.org/docs/r0.8.1/pigunit.html), however writing tests for scripts that load data from an
Avro file requires a little more work. Those impatient to see a working example can check out my
[demonstration project](https://github.com/SketchingDev/PigScript-Avro-Test).

## Script under test

The script we want to test simply filters out people who are older than 24 from an Avro file and then saves the
results back to an Avro file. Nothing too extraneous.

```
people = LOAD '$DATA_INPUT' USING AvroStorage();

people_over_24 = FILTER people BY age > 24;

STORE people_over_24 INTO '$DATA_OUTPUT' USING AvroStorage();
```

## The test

The not-so-obvious way of providing the test data is to instantiate the `PigTest` class with the location of your Avro
data-file (the data contained within it is ignored). At runtime PigUnit will use the schema defined in the Avro file to
override the `LOAD` statement with the schema, defined using the
[`AS` keyword](https://pig.apache.org/docs/r0.17.0/basic.html#load). You can see this in action by examining
the console's output when running the test.

```
STORE people_over_24 INTO 'output' USING AvroStorage();
--> none
people: {name: chararray,age: int}
people = LOAD '/var/folders/9n/cj1d204d6559wrszmp760xc80000gn/T/pig-avro-test8589720558697077822.avro' USING AvroStorage();
--> people = LOAD 'file:/tmp/temp-1468326526/tmp-822087986input_data.1471440361913.1' USING PigStorage(',') AS (
    name: chararray,
    age: int
);
STORE people_over_24 INTO 'output' USING AvroStorage();
--> none
```

The test data can then be passed into the `PigTest` class' `assertOutput`, as the code segment from my
[demonstration project](https://github.com/SketchingDev/PigScript-Avro-Test) shows...

```java
public class PigAvroTest {

    private static final String PIG_SCRIPT = PigAvroTest.class.getResource("filter_people_over_24.pig").getPath();

    // ...

    @Before
    public void setUp() throws IOException {
        File dataFile = File.createTempFile("pig-avro-test", ".avro");
        writeAvroSchema(dataFile, Person.getClassSchema());

        final String[] arguments = {"DATA_INPUT=" + dataFile.toURI(), "DATA_OUTPUT=output"};
        pigTest = new PigTest(PIG_SCRIPT, arguments);
    }

    @Test
    public void return_25_year_old_when_provided_24_and_25_year_old() throws Throwable {
        final String[] input = {
            "Athos,24",
            "Porthos,25"
        };

        final String[] output = {
            "(Porthos,25)"
        };

        pigTest.assertOutput(INPUT_DATA, input, FILTERED_DATA, output, DELIMITER);
    }

    private static void writeAvroSchema(File file, Schema schema) throws IOException {
        DatumWriter<GenericRecord> datumWriter = new GenericDatumWriter<GenericRecord>(schema);
        DataFileWriter<GenericRecord> dataFileWriter = new DataFileWriter<GenericRecord>(datumWriter);
        dataFileWriter.create(schema, file);
        dataFileWriter.close();
    }
}

```
