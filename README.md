# ACS 7.3: Upgrading to Transform Core 3.0.0

This project includes the new features provided by [Transform Core 3.0.0](https://github.com/Alfresco/alfresco-transform-core/tree/3.0.0), released together with ACS Community 7.3.x.

For Enterprise Customers, there is an additional product named [Transform Service 2.0.0](https://docs.alfresco.com/transform-service/latest/). The main difference with the Community Transform Core is that Transform Service provides asynchronous transformations, to support [clustering and scaling up](https://github.com/aborroy/acs-transform-cluster) operations for large volumes.

## Source Code

All the Source Code from Transform Core is available in following Alfresco GitHub project:

https://github.com/Alfresco/alfresco-transform-core

The project includes a number of Transformation Engines that can be deployed together (AIO) or separately:

https://github.com/Alfresco/alfresco-transform-core/tree/master/engines

* [ImageMagick](https://imagemagick.org/index.php): images and thumbnails
* [LibreOffice](https://www.libreoffice.org): office documents (including Microsoft formats)
* [pdfrenderer](https://pdfium.googlesource.com/pdfium/+/master/README.md): PDF format
* [Apache Tika](https://tika.apache.org): metadata extraction
* [Misc](https://github.com/Alfresco/alfresco-transform-core/tree/master/engines/misc): HTML, iWorks, EML...


## Deployment

Sample Docker Compose configuration is described below, despite every service can be deployed manually installing dependencies (like ImageMagick or LibreOffice) and executing the engine as a regular Spring Boot app.

Deploying as a single Core AIO (All-In-One) T-Engine:

```
services:

  alfresco:
    image: alfresco/alfresco-content-repository-community:7.3.0
    environment:
	  JAVA_OPTS: >-
	    -DlocalTransform.core-aio.url=http://transform-core-aio:8090/

  transform-core-aio:
    image: alfresco/alfresco-transform-core-aio:3.0.0
```

Running configuration is available in [docker-compose-aio](docker-compose-aio/docker-compose.yml)


As a set of individual T-Engines

```
services:

  alfresco:
    image: alfresco/alfresco-content-repository-community:7.3.0
    environment:
      JAVA_OPTS: >-
        -DlocalTransform.core-aio.url=
        -DlocalTransform.pdfrenderer.url=http://alfresco-pdf-renderer:8090/
        -DlocalTransform.imagemagick.url=http://imagemagick:8090/
        -DlocalTransform.libreoffice.url=http://libreoffice:8090/
        -DlocalTransform.tika.url=http://tika:8090/
        -DlocalTransform.misc.url=http://transform-misc:8090/

  alfresco-pdf-renderer:
    image: alfresco/alfresco-pdf-renderer:3.0.0

  imagemagick:
    image: alfresco/alfresco-imagemagick:3.0.0

  libreoffice:
      image: alfresco/alfresco-libreoffice:3.0.0

  tika:
      image: alfresco/alfresco-tika:3.0.0

  transform-misc:
      image: alfresco/alfresco-transform-misc:3.0.0
```

Running configuration is available in [docker-compose-engines](docker-compose-engines/docker-compose.yml)


## Configuration

Once the service is running, registered transformations are available in Transform Configuration endpoint:

http://localhost:8090/transform/config

Additional endpoints for http://localhost:8090:

* `POST /transform` to perform a transform. Performs a transform on content uploaded as a Multipart File and provides the resulting content as a download. Transform options are extracted from the request properties. The following are not added as transform options, but are used to select the transformer: `sourceMimetype` & `targetMimetype`
* `GET /` provides an html test page to upload a source file, enter transform options and issue a synchronous transform request.
* `GET /log` provides a page with basic log information.
* `GET /error` provides an error page when testing.
* `GET /version` provides a String message to be included in client debug messages.


## Changing default configuration

Additional config files (which may be resources on the classpath or external files) are specified in Spring Boot properties or such as transform.config.file.<filename> or environment variables like TRANSFORM_CONFIG_FILE_<filename>.

**Configuration**

[docker-compose.yml](docker-compose-extension/docker-compose.yml)

```
  transform-core-aio:
    image: alfresco/alfresco-transform-core-aio:3.0.0
    environment:
      TRANSFORM_CONFIG_FILE_PDFOVERRIDESUPPORTED: "/modify-supported.json"
    volumes:
      - ./modify-supported.json:/modify-supported.json
```

**Overriding**

[modify-supported.json](docker-compose-extension/modify-supported.json)

```
{
  "addSupported": [
    {
      "transformerName": "imagemagick",
      "sourceMediaType": "image/tiff",
      "targetMediaType": "application/pdf",
      "maxSourceSizeBytes": -1
    }
  ],
  "overrideSupported": [
    {
      "transformerName": "PdfBox",
      "sourceMediaType": "application/pdf",
      "targetMediaType": "text/plain",
      "maxSourceSizeBytes": -1
    },
    {
      "transformerName": "TikaAuto",
      "sourceMediaType": "application/pdf",
      "targetMediaType": "text/plain",
      "maxSourceSizeBytes": -1
    }
  ]
}
```

The configuration above adds a new transform pipeline (from TIFF to PDF) and overrides existing transform pipelines (PDF to Text) to unlimit the size of transformed files.

Additional options to override transforms are available in https://github.com/Alfresco/alfresco-transform-core/blob/master/docs/transform-config.md#overriding-transforms


[override-pdfrenderer.json](docker-compose-extension/override-pdfrenderer.json)

```
{
  "transformers": [
    {
      "transformerName": "pdfrenderer",
      "supportedSourceAndTargetList": [
        {"sourceMediaType": "application/pdf", "targetMediaType": "image/png" }
      ],
      "transformOptions": [
        "pdfRendererOptions"
      ]
    }
  ]
}
```

The configuration above removes the [existing pipeline](https://github.com/Alfresco/alfresco-transform-core/blob/3.0.0/engines/pdfrenderer/src/main/resources/pdfrenderer_engine_config.json#L13) (from Illustrator to PNG) in `pdfrenderer` engine.

Additional options to override source and target pipelines are available in https://github.com/Alfresco/alfresco-transform-core/blob/master/docs/transform-config.md#overriding-the-supportedsourceandtargetlist


**Removing**

[remove-iworks.json](docker-compose-extension/remove-iworks.json)

```
{
  "removeTransformers" : [
    "appleIWorks"
   ]
}
```

The configuration above removes `appleIWorks` transformer.

Additional options to remove transformers are available in https://github.com/Alfresco/alfresco-transform-core/blob/master/docs/transform-config.md#removing-a-transformer


**Modifying defaults**


[default-max-priority.json](docker-compose-extension/default-max-priority.json)

```
{
  "supportedDefaults": [
    {
      "transformerName": "Office",             
      "sourceMediaType": "application/zip",
      "maxSourceSizeBytes": 18874368
    },
    {
      "sourceMediaType": "application/msword", 
      "maxSourceSizeBytes": 4194304,
      "priority": 45
    },
    {
      "priority": 60                           
    },
    {
      "maxSourceSizeBytes": -1                 
    }
  ]
}
```

Additional options to modify defaults are available in https://github.com/Alfresco/alfresco-transform-core/blob/master/docs/transform-config.md#default-maxsourcesizebytes-and-priority-values



# Creating new transform engine

To create a new Transform Engine, implementing two Java interfaces is required:

* `org.alfresco.transform.base.TransformEngine` to provide basic information about the Engine and the Transform pipelines
* `org.alfresco.transform.base.CustomTransformer` to implement the transformation operation

Sample implementation for a [Pandoc](https://pandoc.org) Transform Engine is provided in [PandocTransformEngine.java](pandoc-t-engine/src/main/java/org/alfresco/transform/PandocTransformEngine.java) and Custom Transformers in [MarkdownTransformer.java](pandoc-t-engine/src/main/java/org/alfresco/transform/MarkdownTransformer.java) and [LatexTransformer.java](pandoc-t-engine/src/main/java/org/alfresco/transform/LatexTransformer.java)

Pipeline for supported transformations should be defined in JSON format following [T-Engine configuration](https://github.com/Alfresco/alfresco-transform-core/blob/master/docs/transform-config.md#t-engine-configuration) guideline.

Sample pipeline for the Pandoc Transform Engine is provided in [engine_config.json](pandoc-t-engine/src/main/resources/engine_config.json), to transform Markdown and LaTeX to PDF mimetype.

Once the Docker Image has been built as `alfresco/pandoc-t-engine` (details available in [README.md](pandoc-t-engine/README.md)), deployment configuration for Docker Compose includes a URL environment variable in `alfresco` service and `transform-pandoc` service deployment:

```
services:
  alfresco:
    image: alfresco/alfresco-content-repository-community:7.3.1
    mem_limit: 3200m
    environment:
      JAVA_OPTS: >-
        -DlocalTransform.pandoc.url=http://transform-pandoc:8090/

  transform-pandoc:
    image: alfresco/pandoc-t-engine:latest
    mem_limit: 1g
    environment:
      JAVA_OPTS: " -XX:MinRAMPercentage=50 -XX:MaxRAMPercentage=80"
    ports:
        - 8096:8090    
```

Complete sample for this deployment is available in [docker-compose.yml](docker-compose-pandoc/docker-compose.yml)


# Additional deployment configurations

Limit the number of threads to be used for transformation operations (between 4 and 12 in the sample) and change the log level (to ERROR in the sample):

```
    transform-core-aio:
        image: alfresco/alfresco-transform-core-aio:3.0.0
        environment:
            JAVA_OPTS: "
              -Dserver.tomcat.threads.max=12
              -Dserver.tomcat.threads.min=4
              -Dlogging.level.org.alfresco.transform.common.TransformerDebug=ERROR
            "
```

Scaling up Transform Service samples are available in https://github.com/aborroy/acs-transform-cluster

SSL/TLS configuration sample is available in https://github.com/aborroy/alfresco-transform-ssl
