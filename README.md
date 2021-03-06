##


# BADAAS - DEVELOPER GUIDE

## Table of Contents

[About BADAAS](#about-badaas)

[Introduction](#introduction)

[Source](#source)&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;

[Wiki](#wiki)

[Prerequisites](#prerequisites)

[Apache Spark](#apache-spark)

[How to Build](#how-to-build)

[Data Storage Representation](#data-storage-representation)

[BADAAS CORE](#badaas-core)

[Using Apache Spark with Delta Lake](#using-apache-spark-with-delta-lake)

[Transformation or Lambda](#transformation-or-lambda)

[Guidelines for implementing new transformation](#guidelines-for-implementing-new-transformation)

[BADAAS API](#badaas-api)

[Steps to add an endpoint](#steps-to-add-an-endpoint)

[Routing BADAAS Requests ](#routing-badaas-requests)

[Using BADAASRouter](#using-badaasrouter)

[Using BADAAS Controller](#using-badaas-controller)

[BADAAS Actions](#badaas-actions)

##



## Version 1.0.0.2

####  

## About BADAAS

Buddi affiliated Data as a Service (BADAAS) is mainly for virtualizing the medical chart data to avoid having redundant copies of such data on various machines. Also focusing on providing role-based access, transformation, auditing, supporting version control for the data.

![](RackMultipart20210121-4-eim9eg_html_d7dab34902be7fb9.png)

**System Architecture - BADAAS**

## Introduction

BADAAS uses Play framework to build REST services.This document helps you to step through for understanding the internal implementation and adding new API endpoint, transformation in BADAAS project.

## Source

BADAAS source can be obtained using below git URLs. It is recommended to switch to &#39;develop&#39; branch.

Core - [http://gitlab.india.claritrics.com/Claritrics/BuddiHealth/badaas-core.git](http://gitlab.india.claritrics.com/Claritrics/BuddiHealth/badaas.git)

API - [http://gitlab.india.claritrics.com/Claritrics/BuddiHealth/badaas-api.git](http://gitlab.india.claritrics.com/Claritrics/BuddiHealth/badaas.git)

##

## Wiki

The full details about BADAAS can refer using the below wiki URL.

[http://gitlab.india.claritrics.com/Claritrics/BuddiHealth/badaas/-/wikis/Buddi-affiliated-Data-as-a-Service-BaDaaS](http://10.0.1.17/Claritrics/BuddiHealth/badaas/-/wikis/Buddi-affiliated-Data-as-a-Service-BaDaaS)

### **Prerequisites**

- SBT Version - 1.3.4
- Scala Version - 2.12.10
- Java Version - 1.8 and above
- Hadoop - 3.3.0 - (use this [link](https://phoenixnap.com/kb/install-hadoop-ubuntu) for installation and configuration)
- IntelliJ

### **Apache Spark**

For Development, no need to setup Spark environment and Spark session gets created automatically. For Production only we need to setup Spark separately when go for using Spark Cluster to run the map job. (use this [link](https://phoenixnap.com/kb/install-spark-on-ubuntu) to install and configure Spark)

### **How to Build**

1. _IntelliJ_

> - Add a Configuration of type sbt Task to run or debug
> - sbt Tool Window in the Sidebar
> - sbt shell at the Bottom Left

2. _Terminal_ - IDE integrated or native:

> - sbt - opens an sbt shell

### **Using sbt shell:**

Once the sbt shell is started, use below commands to appropriate actions.

- `clean`- deletes all the compiled classes and other generated resources in **target** directory
- `update`- resolves _libraryDependencies_ and updates any new/broken library
- `compile`- compiles all the files in the _current_ project
- `run`- to run application as a rest service
- `assembly`- packages all the **.class** files of the project in a jar file
- `publishLocal` - publish the package into local file system

### **Data Storage Representation**

The current design uses the tree based storage representation to store/fetch the data into/from Data Store using HDFS. The developer who wants to read/write the data from/into HDFS, please follow the hierarchy.

![](RackMultipart20210121-4-eim9eg_html_e24461d69bba02d8.png)

##

## BADAAS CORE

## Using Apache Spark with Delta Lake

Apache Spark is used as data execution engine for processing the large set of data from data store and applying the transformation tasks on the top of the data. Additionally we are using Delta Lake extension. Delta Lake is an open-source storage layer that brings ACID transactions to Apache Spark and big data workloads. These functionalities cover in TDEngine Class and having spark session with support of delta lake.
```
case object TDEngine { 

  private val logger = Logger(getClass) 

  private val sparkDeltaLake: SparkSession = SparkSession.builder() 

    .master(ConfigFactory.load().getString("spark.uri")) 

    .appName("badaas-buddi.ai-delta-lake") 

    .config("spark.sql.extensions", "io.delta.sql.DeltaSparkSessionExtension") 

    .config("spark.delta.logStore", "org.apache.spark.sql.delta.storage.HDFSLogStore") 

    .config("spark.sql.catalog.spark_catalog", "org.apache.spark.sql.delta.catalog.DeltaCatalog") 

    .getOrCreate() 

  sparkDeltaLake.sparkContext.setLogLevel("ERROR") 


  def get(numberOfFiles: Int, clientName: String, categoryName: String = "ed", 

          transformationName: TransformationEnumeration.Value = PHIMASK, nameStartsWith:String = ""): List[BadaasDataResource] = { 

    val dataRDD = getOrLoadData(clientName, categoryName, transformationName) 

    if (dataRDD.isEmpty) 

      List[BadaasDataResource]() 

    else 

      dataRDD.rdd.filter( x => x.getString(0).startsWith(nameStartsWith)).take(numberOfFiles).map(x => 		BadaasDataResource(generateUUID(x.getString(0).getBytes()), Array(x.getString(1)))).toList 

  } 
  
  def method() : Any = { 

    // your logic 

    // call transformation 

  } 
}
```

## Transformation or Lambda

Spark Transformation is a function that takes the inputs as RDD and map into another format without changing the input. The Lambdas are separated and implemented in ```Transformation.scala class```. Currently we are supporting minimal number of transformations. The supported transformations are below,

- Conversion
- Parsing
- PHI Mask
- Auto-Coding
- Custom - Not implemented yet

## Guidelines for implementing new transformation

  - Implement the lambda function in ```Transformation.scala```.
  - Add the transformation name as enumeration in ```TransformationEnumeration.scala```.
  - Make the necessary changes in apply() method at ```TDEngine.scala``` to call the transformation

##### badaas-core: .\src\main\scala\ai\buddi\core\Transformation.scala
```
case object Transformation { 

  private val configHDFS = new Configuration() 

  private val uriHDFS = new URI(ConfigFactory.load().getString("hdfs.uri")) 

  private val fileSystemHDFS: FileSystem = FileSystem.get(uriHDFS, configHDFS) 

  def parsingTransformation(docID: String, docContent: String): (String, String) = { 
    applyTransformation(docID, docContent, PARSING) 
  } 
  
  def applyTransformation(docID: String, docContent: String, transformationName: TransformationEnumeration.Value): (String, String) = { 
    try { 
      (docID, mlpOperation(docContent, transformationName.toString)) 
    } finally { 
    
    } 
  }

  def autoCodingTransformation(docID: String, docContent: String): (String, String) = { 
    applyTransformation(docID, docContent, AUTOCODING) 
  } 

  def yourTransformation(): Any = { 

    // transformation logic 

  } 
} 
```


##### badaas-core: .\src\main\scala\ai\buddi\core\TransformationEnumeration.scala
```
object TransformationEnumeration extends Enumeration { 

  type TransformationEnumeration = Value 
  
  val PHIMASK = Value("phimask") 

  val PARSING = Value("parsing") 

  val AUTOCODING = Value("autocoding") 

  val CONVERSION = Value("conversion") 

  // Add new lambda as enum here 

  // format 

  implicit val myFormat = new Format[TransformationEnumeration.TransformationEnumeration] { 

    def reads(json: JsValue): JsSuccess[Value] = JsSuccess(TransformationEnumeration.withName(json.as[String])) 

    def writes(myEnum: TransformationEnumeration.TransformationEnumeration) = JsString(myEnum.toString) 
  } 
} 
```

##

## BADAAS API

## Steps to add an endpoint

1. Add your HTTP endpoint for routing

> 1. Using Route file
> 2. Using Router class

2. Add action for your endpoint in Controller class

## Routing BADAAS Requests

As a first step, add your HTTP endpoint,we should add the HTTP request for routing. Play framework has two complimentary routing mechanisms. In the conf directory, there's a file called "routes: which contains entries for the HTTP method and a relative URL path, and points it at an action in a controller.

##### badaas-api: .\conf\routes

```GET    /yourRequest        controllers.badaas.yourController.yourAction()```

This is useful for situations where a front end service is rendering HTML or direct way to implement the action in controller.

However, Play framework also contains a more powerful routing DSL that we will use for the REST API.For every HTTP request start with / only, Play routes it to a dedicated BadaasRouter class to handle the BADAAS requests, through the conf/routes file:

```->     /                   controllers.badaas.BadaasRouter```

##

## Using BADAASRouter

The next step is to assign the action for the request using controller. BADAAS Router uses Play&#39;s routing DSLconcept to extract the parameter value from the URL string. For example, here &#39;query&#39; parameter can be extracted from Request automatically and pass it to controller.

The BadaasRouter has a BadaasController injected into it through standard  **dependency injection**

```
class BadaasRouter @Inject()(controller: BadaasController) extends SimpleRouter { 
  override def routes: Routes = { 

    case GET(p"/getSection/$files//$clientName/$tagName/$categoryName") => 

      controller.getSection(files.toInt, clientName, tagName, categoryName) 

    case GET(p"/getData/$files/$clientName/$categoryName/$transName") => 

      controller.getData(files.toInt, clientName, categoryName, transName) 

    case GET(p"/yourRequest/$query") => 

      controller.yourAction(query) 
```

## Using BADAAS Controller

The next step is to add the action for the request in controller. A controller handles the work of processing the HTTP request into an HTTP response in the context of an Action.

The methods in a controller consist of a method returning an Action. The Action provides the &quot;engine&quot; to Play.Using the action, the controller passes in a block of code that takes a [Request](https://www.playframework.com/documentation/latest/api/scala/index.html#play.api.mvc.Request) passed in as implicit. Then, the block must return [Future[Result]](http://www.scala-lang.org/api/current/index.html#scala.concurrent.Future)

```
class BadaasController @Inject()(val controllerComponents: ControllerComponents)( 
  implicit ec: ExecutionContext) extends BadaasBaseController { 
{ 

  def getData(fileCount: Int, clientName: String, categoryName: String, transformationName: String): Action[AnyContent] = BadaasAction.async { 

    implicit request => 
      logger.trace("Controller - Get Data") 
      
      BadaasServiceHandler.getData(fileCount, clientName, categoryName, transformationName).map { result => 
        Ok(result) 
      } 
  } 

 def yourAction(): Action[AnyContent] = BadaasAction.async { 

    implicit request => 
      BadaasServiceHandler.yourHandlerMethod().map { result => 
        Ok(result) 
      } 
  } 
} 
```

## BADAAS Actions

BadaasAction is involved in each action in the controller and mediates the paperwork involved with processing a request into a response, adding context to the request and enriching the response with headers and cookies. ActionBuilders are essential for handling authentication, authorization and monitoring functionality.

ActionBuilders work through a process called action composition. The ActionBuilder class has a method called invokeBlock that takes in a Request and a function (also known as lambda or closure) that accepts a Request of a given type, and produces a Future[Result].

The BadaassActionis a custom action builder that can handle BadaasRequest and does a couple of different things here. The first thing it does is to log the request as it comes in. Next, it checks authorizations for applying the lambda using Access Management Gateway by JWT verification. Next. it pulls out MessagesApi for the request, and adds that to a BadaasRequest , and runs the function, returning a Future[Result].

##
