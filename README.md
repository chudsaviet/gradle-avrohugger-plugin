# gradle-avrohugger-plugin

[![Build Status](https://travis-ci.org/chudsaviet/gradle-avrohugger-plugin.svg?branch=master)](https://travis-ci.org/chudsaviet/gradle-avrohugger-plugin)
[![Build Status](https://img.shields.io/maven-metadata/v/https/plugins.gradle.org/m2/com/chudsaviet/gradle/avrohugger/com.chudsaviet.gradle.avrohugger.gradle.plugin/maven-metadata.xml.svg?label=gradle%20plugin%20portal)](https://plugins.gradle.org/plugin/com.chudsaviet.gradle.avrohugger)


[Gradle](https://gradle.org) plugin for generating scala case classes from [Apache Avro](https://avro.apache.org/) schemas, datafiles and protocols. 
It is based on [avrohugger](https://github.com/julianpeeters/avrohugger) scala library.

## Requirements

Plugin requires **Gradle** in version **4.3** or higher. 

## Usage

For minimal usecase it is enough to add following to your `build.gradle` file:
                                    
```groovy
plugins {
    id 'com.chudsaviet.gradle.avrohugger' version '0.2.4'
}
```

But usually `scala` plugin and `scala-library` dependency should be added to your `build.gradle` file 
to enable compilation of generated scala sources:

```groovy
plugins {
    id 'scala'
    id 'com.chudsaviet.gradle.avrohugger' version '0.2.4'
}

repositories {
   mavenCentral()
}

dependencies {
    compile 'org.scala-lang:scala-library:2.12.8'
}
```

Then by default scala classes generation will be triggered during build before `compileScala` task. 
It will look into directory `src/main/avro` for all files ending with `.avsc`, `.avdl`, `.avro` or `.avpr`.
Classes will be generated under `${buildDir}/generated-src/avro` and they will be added to `main` scala source set.

> You can always call `generateAvroScala` task to perform classes generation only

## Configuration

You can change default configuration in `avrohugger` block like in following example:

```groovy
avrohugger {
    sourceDirectories = files('src-avro')
    destinationDirectory = file('src-scala')
    namespaceMapping = [ 'com.example' : 'com.chudsaviet' ]
    typeMapping {
        protocolType = ScalaADT
    }
    sourceFormat = Standard
}
```

### Description of configuration values

| Name                       | Default                               | Description                              |
| ---------------------------|---------------------------------------| -----------------------------------------|
| sourceDirectories          | src/main/avro                         | Directories (can be multiple) used to look for `.avsc`, `.avdl`, `.avro` or `.avpr` files as an input for classes generation |
| destinationDirectory       | ${buildDir}/generated-src/avro        | Directory where classes will be generated |
| namespaceMapping           | empty map                             | Map where key is name of namespace and value is it's replacement which will be used as package for resulting classes |
| typeMapping                | default type mapping                  | Defines how avro types are represented as scala types |
  
### Type mapping values

| Name (avro type)           | Default (scala type)                           | PossibleValues                    |
| ---------------------------| -----------------------------------------------| ----------------------------------| 
| intType                    | ScalaInt                                       | ScalaInt, ScalaLong, ScalaFloat, ScalaDouble |
| longType                   | ScalaLong                                      | ScalaInt, ScalaLong, ScalaFloat, ScalaDouble |   
| floatType                  | ScalaFloat                                     | ScalaInt, ScalaLong, ScalaFloat, ScalaDouble |   
| doubleType                 | ScalaDouble                                    | ScalaInt, ScalaLong, ScalaFloat, ScalaDouble |   
| recordType                 | ScalaCaseClass                                 | ScalaCaseClass, ScalaCaseClassWithSchema |
| enumType                   | ScalaEnumeration (JavaEnum for SpecificRecord) | ScalaEnumeration, JavaEnum, ScalaCaseObjectEnum, EnumAsScalaString |   
| unionType                  | OptionEitherShapelessCoproduct                 | OptionEitherShapelessCoproduct, OptionalShapelessCoproduct, OptionShapelessCoproduct |   
| arrayType                  | ScalaSeq (ScalaArray for Scavro)               | ScalaSeq, ScalaArray, ScalaList, ScalaVector |   
| protocolType               | NoTypeGenerated                                | NoTypeGenerated, ScalaADT |
| dateType                   | JavaTimeLocalDate                              | JavaTimeLocalDate, JavaSqlDate |
| timestampMillisType        | JavaTimeInstant                                | JavaTimeInstant, JavaSqlTimestamp |

> See [avrohugger](https://github.com/julianpeeters/avrohugger/blob/master/README.md#supports-generating-case-classes-with-arbitrary-fields-of-the-following-datatypes) library for all details about types

### Source formats

1. **Standard** - vanilla case classes

2. **SpecificRecord** - case classes implementing `SpecificRecordBase` with mutable fields

   There is need additional dependency on [Apache Avro](https://avro.apache.org/) to compile generated classes with `SpecificRecord` source format.
   
   ```groovy
    dependencies {
       // ... other dependencies here 
       compile 'org.apache.avro:avro:1.8.2'
    }
    ``` 

3. **Scavro** - case classes intended to wrap java classes 

    There is need additional dependency on [Scavro](https://github.com/oedura/scavro#scavro-reader-and-writer) to compile generated classes with `Scavro` source format.
    ```groovy
    dependencies {
       // ... other dependencies here 
       compile 'org.oedura:scavro_2.12:1.0.3'
    }
    ```
    
    Java classes are not generated by `gradle-avrohugger-plugin`. 
    You can use [gradle-avro-plugin](https://github.com/commercehub-oss/gradle-avro-plugin) to generate java classes.
    
    See following example of combining `gradle-avrohugger-plugin` with `gradle-avro-plugin`: 
    
    ```groovy
    plugins {
        id 'scala'
        id 'com.commercehub.gradle.plugin.avro' version '0.9.1'
        id 'com.chudsaviet.gradle.avrohugger' version '0.2.4'
    }
    
    repositories {
       mavenCentral()
    }
    
    dependencies {
        compile 'org.scala-lang:scala-library:2.12.8'
        compile 'org.apache.avro:avro:1.8.2'
        compile 'org.oedura:scavro_2.12:1.0.3'
    }
    
    avro {
        stringType = "CharSequence"
    }
    
    avrohugger {
        sourceFormat = Scavro
    }
    ```

> See [avrohugger](https://github.com/julianpeeters/avrohugger/blob/master/README.md#generates-scala-case-classes-in-various-formats) library for all details about different formats

### How to change default source set

If you do not want generated scala classes be a part of `main` source set you can change it as in following example to be a part of `test` source set:

```groovy
sourceSets {
    test {
        scala {
            srcDir avrohugger.destinationDirectory
        }
    }
}
```

### How to use schema files from dependencies

If you have schema files provided as part of one or more `jar` files you can include them as dependencies and configure `avrohugger` to use them:

1. Define new configuration

    ```groovy
    configurations {
        avroSchema
    }
    ```

2. Declare dependencies

    ```groovy
    dependencies {
        avroSchema 'sample:schema-one:0.1.0'
        avroSchema 'sample:schema-two:0.1.0'
    }
    ```

3. Configure avrohugger

    ```groovy
    avrohugger {
        sourceDirectories {
            from configurations.avroSchema.files.collect { zipTree(it) }
        }
    }
    ```

## Scala 2.10 support

There is a restriction about maximum fields count for case classes in scala 2.10 and older.
If some of your records has more then 22 fields and scala 2.10 is used 
simple serializable class will be generated instead of case class.

>Plugin will get your scala version from `scala-library` version in compile dependencies. 
If it is not able to found the dependency it will act as the version is 2.11.  

## Build and Test

Plugin can be built and tested using:

    ./gradlew clean build

## Credits

Core generation logic is implemented by [Julian Peeters](https://github.com/julianpeeters) 
in scala library called [avrohugger](https://github.com/julianpeeters/avrohugger).
Plugin was inspired by it's sbt version [sbt-avrohugger](https://github.com/julianpeeters/sbt-avrohugger). 
