# **Vertex AI Extensions**

In the digital age, leveraging natural language for database queries represents a leap towards more intuitive data management. This tutorialblog introduces a novel approach that combines Google Gemini's advanced natural language processing with MongoDB, facilitated by Vertex AI Extensions. These extensions address key limitations of Large Language Models (LLMs) by enabling real-time data querying and modification, which traditional LLMs cannot do due to their static knowledge base post-training. By integrating MongoDB Atlas with Vertex AI Extensions, we offer a solution that not only enhances the accessibility and usability of database interactions through natural language but also ensures up-to-date, dynamic data access and manipulation. This symbiosis of MongoDB Atlas's robust data management with the cutting-edge capabilities of Google Gemini and Vertex AI Extensions embodies the fusion of generative AI with database technology, setting a new standard for data interaction experiences.

MongoDB seamlessly integrates with Google Vertex AI Extensions, enabling users to perform operations on their MongoDB database using natural language queries through Google Gemini, follow these detailed instructions and prerequisites. This guide will ensure users of all levels can successfully set up their environment and utilize the notebook effectively.

# **Prerequisites**

Before you start, make sure you have:

* **A Google Cloud Platform (GCP) Account**: Necessary to access Google Cloud services, including Vertex AI and Secrets Manager. Following the link for [documentation](https://www.mongodb.com/docs/atlas/app-services/data-api/generated-endpoints/) setup.

* **A MongoDB Atlas Account:** For hosting your MongoDB database. If you're new to MongoDB, sign up and create a cluster in MongoDB Atlas following MongoDB's [documentation](https://www.mongodb.com/docs/guides/atlas/cluster/)

* **Google Cloud SDK Installed:** For interacting with GCP services through the command line.

* **Vertex AI Extensions enabled:** Signup for Extension Trusted Tester program following the [documentation](https://cloud.google.com/vertex-ai/generative-ai/docs/extensions/overview)

# **Step-by-Step Guide**

1. **Enable MongoDB Data API**: Navigate to the Atlas console, find the "Data API" section under "App Services", and enable the Data API. Configure permissions and note down the API URL and key. Follow the link for [documentation](https://www.mongodb.com/docs/atlas/app-services/data-api/generated-endpoints/)

2. **Store MongoDB API Key in Google Secrets Manager**
      * **Create a Secret for Your API Key**: Go to the Google Cloud Console, navigate to Secrets Manager, and create a new secret. Store your MongoDB API key here for secure access. Follow the link for [documentation](https://cloud.google.com/secret-manager/docs/creating-and-accessing-secrets#:~:text=Go%20to%20the%20Secret%20Manager%20page%20in%20the%20Google%20Cloud%20console.&text=On%20the%20Secret%20Manager%20page,example%2C%20my%2Dsecret%20).

3. **Prepare Your OpenAPI 3 Specification**

      * **Develop API Specification:** Define the interactions between Google Vertex AI and your MongoDB using OpenAPI 3.0. This specification outlines how natural language queries will be translated into MongoDB operations.      

# **Vertex Extensions SDK: Connecting Models to APIs**

Vertex AI Extensions is a platform for creating and managing extensions that connect large language models to external systems via APIs. These external systems can provide LLMs with real-time data and perform data processing actions on their behalf.
This tutorial uses the mongodb default dataset from sample_mflix database , movies collection.  We will run all the below code on the Enterprise Colab [notebook](https://github.com/mongodb-partners/MongoDB-VertexAI-extensions/blob/main/notebook/Mongodb%20vertex%20AI%20integration.ipynb).
# Connect to project

```{python}
from google.colab import auth
auth.authenticate_user(<set your gcp project id here>)

!gcloud config set project <set your gcp project id here>
```


# To install required dependency

```{python}
!gsutil cp gs://vertex_sdk_private_releases/llm_extension/google_cloud_aiplatform-1.44.dev20240315+llm.extension-py2.py3-none-any.whl .
!pip install --force-reinstall --quiet google_cloud_aiplatform-1.44.dev20240315+llm.extension-py2.py3-none-any.whl[extension]
# This is for printing the Vertex AI service account.
!pip install --upgrade --quiet google-cloud-resource-manager
# This is for the section on Langchain using ReasoningEngine.
!pip install --force-reinstall --quiet langchain==0.0.298
# This is for the section on Videos using ReasoningEngine.
!pip install pytube

!pip install --upgrade google-auth
!pip install bigframes==0.26.0
```

# Set up  environmental variables
```{python}
## This is just a sample values please replace accordingly to your project
import os

# Setting up the GCP project
os.environ['PROJECT_ID'] = 'gcp-pov'  # GCP Project ID
os.environ['REGION'] =  "us-central1" # Project Region
## GCS Bucket location
os.environ['STAGING_BUCKET'] =  "gs://vertexai_extensions"
## Extension Config
os.environ['EXTENSION_DISPLAY_HOME'] =  "MongoDb Vertex API Interpreter"
os.environ['EXTENSION_DESCRIPTION'] =  "This extension makes api call to mongodb to do all crud operations"

## OPEN API SPec config
os.environ['MANIFEST_NAME'] =  "mdb_crud_interpreter"
os.environ['MANIFEST_DESCRIPTION'] =  "This extension makes api call to mongodb to do all crud operations"
os.environ['OPENAPI_GCS_URI'] =  "gs://vertexai_extensions/openapispec.yaml"

## API KEY secret location
os.environ['API_SECRET_LOCATION'] = "projects/787220387490/secrets/mdbapi/versions/1"

##LLM config
os.environ['LLM_MODEL'] = "gemini-1.0-pro"



```


# Initializing AI platform
```{python}

from google.cloud import aiplatform
from google.cloud.aiplatform.private_preview import llm_extension

PROJECT_ID = os.environ['PROJECT_ID']
REGION = os.environ['REGION']
STAGING_BUCKET = os.environ['STAGING_BUCKET']

aiplatform.init(
    project=PROJECT_ID,
    location=REGION,
    staging_bucket=STAGING_BUCKET,
)
```
# To create extension

An extension acts as a bridge between large language models (LLMs) and external systems, allowing for enriched interactions beyond the model's pre-trained knowledge. When creating an extension, you primarily define its behavior and interaction patterns through a manifest file. This manifest file details the API calls the extension can make, how it authenticates those calls, and other metadata about the extension.

**Manifest Parameters**

The manifest is a structured JSON object containing several key components:

**display_name:** A human-readable name for the extension.
description: (Optional) A brief description of what the extension does.
manifest:

**name:** A unique identifier for the extension.
description: A detailed description of the extension's functionality.
api_spec: Specifies the API interactions using an OpenAPI specification.

**open_api_gcs_uri:** The URI to the OpenAPI specification file stored in Google Cloud Storage (GCS).
auth_config: Configuration details for authenticating API calls.

**apiKeyConfig:**

**name:** The name of the API key (a reference name used in the extension).
apiKeySecret: The location of the actual API key stored in Google Secret Manager.
httpElementLocation: Specifies how the API key is included in API calls, typically "HTTP_IN_HEADER".

**Authentication Configuration**

Extensions often need to interact with secured external services, requiring authentication. One common method is using an API key, passed as a header in HTTP requests. For security, the actual API key should not be hardcoded in the extension's manifest or code. Instead, it's stored securely in Google Secret Manager, and the extension's manifest references its location:

apiKeySecret: The path to the secret in Google Secret Manager, allowing the extension to retrieve the API key securely at runtime.

This approach ensures that sensitive information, like API keys, remains secure and isn't exposed in codebases or configuration files

**OpenAPI Specification**

The OpenAPI Specification (OAS) defines a standard, language-agnostic interface to RESTful APIs, allowing both humans and computers to understand the capabilities of a service without accessing its source code or documentation. An OpenAPI spec outlines the available endpoints in an API, how to access them, the expected request/response formats, and authentication methods.

When creating an extension, you must provide an OpenAPI spec that describes how the extension interacts with the external service. This specification is typically hosted in a GCS bucket and referenced in the extension's manifest (open_api_gcs_uri).
Complete file is available [here](https://github.com/mongodb-partners/MongoDB-VertexAI-extensions/blob/main/open-api-spec/mdb-data-api.yaml)


```{python}
mdb_crud = llm_extension.Extension.create(
         display_name = os.environ['EXTENSION_DISPLAY_HOME'],
         # Optional.
         description = os.environ['EXTENSION_DESCRIPTION'],  # Optional.
         manifest = {
             "name": os.environ['MANIFEST_NAME'],
             "description": os.environ['MANIFEST_DESCRIPTION'],
             "api_spec": {
                 "open_api_gcs_uri": (
                     os.environ['OPENAPI_GCS_URI']
                 ),
             },
             "auth_config": {
                 # GOOGLE_SERVICE_ACCOUNT_AUTH is only for 1P supported extensions.
                 "apiKeyConfig":{
             "name":"api-key",
             "apiKeySecret":os.environ['API_SECRET_LOCATION'],
             "httpElementLocation": "HTTP_IN_HEADER"
             },
             "authType":"API_KEY_AUTH"
             },
         },
     )
mdb_crud
```


# Validate the Created Extension
```{python}
print("Name:", mdb_crud.gca_resource.name)
print("Display Name:", mdb_crud.gca_resource.display_name)
print("Description:", mdb_crud.gca_resource.description)
```

# Operation Schema and Parameters

For an Extension, we have the schema for its operations (it came from the OpenAPI Spec in the manifest when we created it):
```{python}
import pprint

pprint.pprint(mdb_crud.operation_schemas())
```


# MongoDB CRUD Operations

MongoDB is a powerful NoSQL database that offers flexibility and scalability for working with data in JSON-like documents. It supports a rich set of CRUD (Create, Read, Update, Delete) operations, which are essential for managing data stored in your databases. Below is a simple guide to performing basic CRUD operations in MongoDB, including an aggregation operation for summarizing data.

Incorporating MongoDB's CRUD operations into a Vertex AI extension offers a seamless way to query and manipulate data within the sample_mflix database, specifically within the movies collection, through natural language processing. This setup leverages the power of Generative AI to understand and execute database operations based on human-like queries. Here's how these operations can be transformed into a Generative AI use case, enhancing user interaction with the database:

## FindOne

*Retrieve data through Vertex Extension*

Imagine needing to quickly find out when a classic film was released without navigating through the database manually. By asking Vertex AI, "Find the release year of the movie 'A Corner in Wheat' from VertexAI-POC cluster, sample_mflix, movies," you get the specific release year instantly, as the system performs a findOne() operation to retrieve this detail.

### Environment variables for find one operations

```{python}
## Operation Ids
os.environ['FIND_ONE_OP_ID'] = "findone_mdb"


## NL Queries
os.environ['FIND_ONE_NL_QUERY'] = "Find the release year of the movie 'A Corner in Wheat' from VertexAI-POC cluster, sample_mflix, movies"

## Mongodb Config
os.environ['DATA_SOURCE'] = "VertexAI-POC"
os.environ['DB_NAME'] = "sample_mflix"
os.environ['COLLECTION_NAME'] = "movies"

### Test data setup
os.environ['TITLE_FILTER_CLAUSE'] = "A Corner in Wheat"


```

### Gemini integration for natural language to extension schema conversion
```{python}
from vertexai.preview.generative_models import GenerativeModel, Tool

fc_chat = GenerativeModel(os.environ['LLM_MODEL']).start_chat()
findOneResponse = fc_chat.send_message(os.environ['FIND_ONE_NL_QUERY'],
    tools=[Tool.from_dict({
        "function_declarations": mdb_crud.operation_schemas()
    })],
)

findOneResponse
```

### From the conveted schema query the extension

```{python}

response = mdb_crud.execute(
    operation_id = findOneResponse.candidates[0].content.parts[0].function_call.name,
    operation_params = findOneResponse.candidates[0].content.parts[0].function_call.args
)

response
```

## Find Many

*Retrieve multiple documents through Vertex Extension*

A film historian wants a list of all movies released in a specific year, say 1924, to study the cinematic trends of that era. They could ask, "Give me movies released in the year 1924 from VertexAI-POC cluster, sample_mflix, movies," and the system would use the find() method to list all movies from 1924, providing a comprehensive snapshot of that year's cinematic output.

### Environment variables for find many operations

```{python}
## Operation Ids
os.environ['FIND_MANY_OP_ID'] = "findmany_mdb"

## NL Queries
os.environ['FIND_MANY_NL_QUERY'] = "give me movies released in year 1924 from VertexAI-POC cluster, sample_mflix, movies"


## Mongodb Config
os.environ['DATA_SOURCE'] = "VertexAI-POC"
os.environ['DB_NAME'] = "sample_mflix"
os.environ['COLLECTION_NAME'] = "movies"
os.environ['YEAR'] = "1924"

```

### Gemini integration for natural language to extension schema conversion

```{python}
from vertexai.preview.generative_models import GenerativeModel, Tool

fc_chat = GenerativeModel(os.environ['LLM_MODEL']).start_chat()
findmanyResponse = fc_chat.send_message(os.environ['FIND_MANY_NL_QUERY'],
    tools=[Tool.from_dict({
        "function_declarations": mdb_crud.operation_schemas()
    })],
)

findmanyResponse
```

### Execute the extension to get response

```{python}

response = mdb_crud.execute(
    operation_id = findmanyResponse.candidates[0].content.parts[0].function_call.name,
    operation_params = findmanyResponse.candidates[0].content.parts[0].function_call.args
)

response
```

## Insert

*Create a new document*

A filmmaker is cataloging their new project in a database of films. They request, "Create a movie named 'My first movie' which is released in the year 2024 to VertexAI-POC cluster, sample_mflix, movies." The system uses insertOne() to add this new movie to the database, ensuring it's part of the historical record for future queries.

### Environment variables for insert operations
```{python}
## Operation Ids
os.environ['INSERT_ONE_OP_ID'] = "insertone_mdb"
## NL Queries
os.environ['INSERT_NL_QUERY'] = "create a movie named 'My first movie' which is released in the year 2024 to VertexAI-POC cluster, sample_mflix, movies"

## Mongodb Config
os.environ['DATA_SOURCE'] = "VertexAI-POC"
os.environ['DB_NAME'] = "sample_mflix"
os.environ['COLLECTION_NAME'] = "movies"

## Test data setup

os.environ['TITLE'] = "My first movie"
os.environ['YEAR'] = "2024"




```


### Gemini integration for natural language to extension schema conversion

```{python}
from vertexai.preview.generative_models import GenerativeModel, Tool

fc_chat = GenerativeModel(os.environ['LLM_MODEL']).start_chat()
insertresponse = fc_chat.send_message(os.environ['INSERT_NL_QUERY'],
    tools=[Tool.from_dict({
        "function_declarations": mdb_crud.operation_schemas()
    })],
)

insertresponse
```


### From the conveted schema query the extension

```{python}

response = mdb_crud.execute(
    operation_id = insertresponse.candidates[0].content.parts[0].function_call.name,
    operation_params = insertresponse.candidates[0].content.parts[0].function_call.args
)

response
```

## Update

*Modify a data entry*

After deciding to delay the release of their film, the filmmaker needs to update the database. They say, "Update the release year of the movie titled 'My first movie' to 2025 from VertexAI-POC cluster, sample_mflix, movies." The system then updates the release year of the movie to 2025 using the updateOne() operation, keeping the database current.

### Environment variables for update operations

```{python}
## Operation Ids
os.environ['UPDATE_OP_ID'] = "uppdateone_mdb"

## NL Queries
os.environ['UPDATE_NL_QUERY'] = "Update the release year of movie titled 'My first movie' to 2025 from VertexAI-POC cluster, sample_mflix, movies"

## Mongodb Config
os.environ['DATA_SOURCE'] = "VertexAI-POC"
os.environ['DB_NAME'] = "sample_mflix"
os.environ['COLLECTION_NAME'] = "movies"

## Test data setup

os.environ['TITLE'] = "My first movie"
os.environ['YEAR'] = "2025"


```

### Gemini integration for natural language to extension schema conversion

```{python}
from vertexai.preview.generative_models import GenerativeModel, Tool

fc_chat = GenerativeModel(os.environ['LLM_MODEL']).start_chat()
updateresponse = fc_chat.send_message(os.environ['UPDATE_NL_QUERY'],
    tools=[Tool.from_dict({
        "function_declarations": mdb_crud.operation_schemas()
    })],
)

updateresponse
```

### From the conveted schema query the extension

```{python}

response = mdb_crud.execute(
    operation_id = updateresponse.candidates[0].content.parts[0].function_call.name,
    operation_params = updateresponse.candidates[0].content.parts[0].function_call.args
)

response
```
## Delete

*Delete a data entry*

A database manager is cleaning up entries and decides to remove outdated or irrelevant records. By stating, "Delete the movie titled 'Gertie the Dinosaur' from VertexAI-POC cluster, sample_mflix, movies," the system finds and deletes this specific movie using deleteOne(), streamlining the database content.

### Environment variables for delete operations
```{python}

## Operation Ids
os.environ['DELETE_OP_ID'] = "deleteone_mdb"

## NL Queries
os.environ['DELETE_NL_QUERY'] =  "Delete the movie titled 'Gertie the Dinosaur' from VertexAI-POC cluster, sample_mflix, movies "

## Mongodb Config
os.environ['DATA_SOURCE'] = "VertexAI-POC"
os.environ['DB_NAME'] = "sample_mflix"
os.environ['COLLECTION_NAME'] = "movies"

## Test data setup


os.environ['TITLE'] = "A Corner in Wheat"

```

### Gemini integration for natural language to extension schema conversion
```{python}
from vertexai.preview.generative_models import GenerativeModel, Tool

fc_chat = GenerativeModel(os.environ['LLM_MODEL']).start_chat()
deleteresponse = fc_chat.send_message(os.environ['DELETE_NL_QUERY'],
    tools=[Tool.from_dict({
        "function_declarations": mdb_crud.operation_schemas()
    })],
)

deleteresponse
```

### From the conveted schema query the extension
```{python}

response = mdb_crud.execute(
    operation_id = deleteresponse.candidates[0].content.parts[0].function_call.name,
    operation_params = deleteresponse.candidates[0].content.parts[0].function_call.args
)

response
```
## Aggregate

*Retrieve data through Vertex Extension*

A researcher is analyzing the volume of films produced over the years and asks, "Get the count of the movies released in the year 1984 from VertexAI-POC cluster, sample_mflix, movies." The system employs the aggregate() method to count and return the number of movies released in 1984, providing valuable insights into the production rate of that year.

### Environment variables for aggregate operations

```{python}

## Operation Ids

os.environ['AGG_OP_ID'] = "aggregate_mdb"

## NL Queries
os.environ['AGGREGATE_NL_QUERY'] = 'Get the count of the movies released on the year 1984 from VertexAI-POC cluster, sample_mflix, movies'

## Mongodb Config
os.environ['DATA_SOURCE'] = "VertexAI-POC"
os.environ['DB_NAME'] = "sample_mflix"
os.environ['COLLECTION_NAME'] = "movies"
os.environ['YEAR'] = "1984"

```

### Gemini integration for natural language to extension schema conversion

```{python}
from vertexai.preview.generative_models import GenerativeModel, Tool

fc_chat = GenerativeModel(os.environ['LLM_MODEL']).start_chat()
aggregateresponse = fc_chat.send_message(os.environ['AGGREGATE_NL_QUERY'],
    tools=[Tool.from_dict({
        "function_declarations": mdb_crud.operation_schemas()
    })],
)

aggregateresponse


```

### From the conveted schema query the extension

```{python}

response = mdb_crud.execute(
    operation_id = aggregateresponse.candidates[0].content.parts[0].function_call.name,
    operation_params = aggregateresponse.candidates[0].content.parts[0].function_call.args
)

response
```


# Conclusion
In conclusion, integrating MongoDB with natural language querying capabilities revolutionizes data interaction, enhancing accessibility and intuitiveness for database queries. Leveraging the Google Gemini Foundation Model alongside a custom Vertex AI Extension not only enriches the data retrieval experience but also upholds data security and integrity. We are closely working with the GCP team to add additional query patterns to Vertex AI extensions. Watch out for more in this space. 

