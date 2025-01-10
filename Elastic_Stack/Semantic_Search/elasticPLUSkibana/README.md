# **Semantic Search Implementation in Elasticsearch**

## **Objective**
The task is to implement semantic search in Elasticsearch with the following goals:  
1. Process existing and incoming data to generate embeddings using the ELSER model.  
2. Ensure embeddings are optimized for semantic search.  
3. Enable scalability and performance optimization for 100M+ records.  
4. Provide flexibility to handle client requests such as adding new fields.  

---

## **Prerequisites**

### **Requirements**
1. Elasticsearch cluster with support for:  
   - `dense_vector` field type  
   - k-Nearest Neighbors (kNN) search  
   - Machine Learning features  

2. **ELSER Model Deployment**:  
   Ensure the `.elser_model_2_linux-x86_64` model is installed and deployed.  
   - Deploy via Kibana: Navigate to **Machine Learning > Trained Models** and deploy `.elser_model_2_linux-x86_64`.  

3. Ensure you have sufficient resources for large data updates (memory, CPU, and disk).  

---

## **Steps**

### **Step 1: Create Index Mapping**
Define the structure for storing embeddings as a `dense_vector` for the existing index.

#### **Code:**
```bash
PUT <(existing_index_name)>/_mapping
{
  "properties": {
    "text_expansion": {
      "type": "dense_vector",
      "dims": 512, # The dimension of the embedding vector
      "index": true, # Enable indexing for vector search
      "similarity": "cosine", # Use cosine similarity for kNN
      "index_options": {
        "type": "int8_hnsw", # Use HNSW for faster approximate search
        "m": 16, # Number of connections per node in HNSW graph
        "ef_construction": 100 # Size of dynamic candidate list during graph construction
      }
    }
  }
}
```
Define the structure for storing embeddings as a `dense_vector` for the new index.
So for that create the mapping first and then insert the embedded generated document in the new index

#### **Code:**
```bash
PUT <(new_index_name)>/_mapping
{
  "properties": {
    "level": {
          "type": "text",
          "fields": {
            "keyword": {
              "type": "keyword",
              "ignore_above": 256
            }
          }
        },
      "message": {
          "type": "text",
          "fields": {
            "keyword": {
              "type": "keyword",
              "ignore_above": 256
            }
          }
        },
        "methodName": {
          "type": "text",
          "fields": {
            "keyword": {
              "type": "keyword",
              "ignore_above": 256
            }
          }
        },
        "passengerNumber": {
          "type": "text",
          "fields": {
            "keyword": {
              "type": "keyword",
              "ignore_above": 256
            }
          }
        },
        "serviceName": {
          "type": "text",
          "fields": {
            "keyword": {
              "type": "keyword",
              "ignore_above": 256
            }
          }
        },
        "sourceFile": {
          "type": "text",
          "fields": {
            "keyword": {
              "type": "keyword",
              "ignore_above": 256
            }
          }
        },
        "tags": {
          "type": "text",
          "fields": {
            "keyword": {
              "type": "keyword",
              "ignore_above": 256
            }
          }
        },
      "text_expansion": {
        "type": "dense_vector",
        "dims": 512, # The dimension of the embedding vector
        "index": true, # Enable indexing for vector search
        "similarity": "cosine", # Use cosine similarity for kNN
        "index_options": {
          "type": "int8_hnsw", # Use HNSW for faster approximate search
          "m": 16, # Number of connections per node in HNSW graph
          "ef_construction": 100 # Size of dynamic candidate list during graph construction
        }
    }
  }
}
```
## **Step 2: Create Ingest Pipeline**

The pipeline processes incoming documents to:
1. Combine the `serviceName`, `level`, and `message` fields into a single text field.
2. Generate embeddings for the combined field using the ELSER model.
3. Adjust the embedding dimensions to a fixed size (e.g., 64) to maintain consistency.

### **Pipeline Definition**

Use the following code to define the pipeline:

```bash
PUT _ingest/pipeline/concat_fields_pipeline
{
  "processors": [
    {
      "script": {
        "source": """
          // Combine fields into a single text input
          ctx.combined_field = ctx.serviceName + " " + ctx.level + " " + ctx.message;
        """
      }
    },
    {
      "inference": {
        "model_id": ".elser_model_2_linux-x86_64", # ELSER model for embedding generation
        "field_map": {
          "combined_field": "text_field" # Map the combined field to the model's input
        },
        "target_field": "text_expansion", # Output field for embeddings
        "inference_config": {
          "text_expansion": {} # Use text expansion configuration
        }
      }
    },
    {
      "script": {
        "source": """
          // Adjust embeddings to fixed dimensions (64 in this example)
          if (ctx.text_expansion != null && ctx.text_expansion.predicted_value != null) {
            ctx.text_expansion = new ArrayList(ctx.text_expansion.predicted_value.values());
            int targetDim = 64; // Target dimension
            while (ctx.text_expansion.size() < targetDim) {
              ctx.text_expansion.add(0.0); // Add padding if dimensions are smaller
            }
            if (ctx.text_expansion.size() > targetDim) {
              ctx.text_expansion = ctx.text_expansion.subList(0, targetDim); // Truncate if larger
            }
          }
        """
      }
    },
    {
      "remove": {
        "field": "combined_field" # Remove intermediate fields
      }
    }
  ]
}
```
## Explanation
- Script Processor: Combines fields serviceName, level, and message into a single field called combined_field.
- Inference Processor: Uses the ELSER model to generate embeddings for the combined_field.
- Script Processor: Adjusts the embedding vector to have a fixed dimension of 64.
- Remove Processor: Deletes the intermediate field combined_field to optimize storage.


## **Step 3: Test the Pipeline**

Test the pipeline using the _simulate API to ensure that:
1. The combined_field is correctly created.
2. The text_expansion field contains a 64-dimensional vector.

### **Pipeline Simulation**

```bash
POST _ingest/pipeline/concat_fields_pipeline/_simulate
{
  "docs": [
    {
      "_source": {
        "serviceName": "CAR-ENRICHMENT", # Example service name
        "level": "INFO", # Example log level
        "message": "CarEnrichmentService : enrichCarData start" # Example log message
      }
    }
  ]
}
```

### **Expected Output**

The output should look like this:

```bash
{
  "docs": [
    {
      "doc": {
        "_source": {
          "serviceName": "CAR-ENRICHMENT",
          "level": "INFO",
          "message": "CarEnrichmentService : enrichCarData start",
          "text_expansion": [0.123, 0.456, ..., 0.789] // 64-dimensional vector
        }
      }
    }
  ]
}

```
- text_expansion Field: Contains the generated 64-dimensional embedding.
- This ensures the pipeline is working as expected.

## **Step 4: Process Data**

Existing documents in your index need to be processed using the ingest pipeline to generate embeddings for semantic search.

### **Option 1: Reindex API**

The Reindex API allows you to copy documents from one index to another while applying the pipeline.

#### **Example Request**
```bash
POST /_reindex
{
  "source": {
    "index": "gbtqa-idms",
    "query": {
      "match_all": {}
    },
    "size": 1000
  },
  "dest": {
    "index": "gbtqa-idms-dev-test",
    "pipeline": "gbtqa_idms_fields_pipeline"
  }
}
```

### **Option 2: Update in the existing index**

Reprocess all documents in the existing index.
## For Existing Data: Use update_by_query to reprocess all documents in the index:

#### **Example Request**
```bash
POST gbtqa-idms/_update_by_query?pipeline=concat_fields_pipeline
{
  "query": {
    "match_all": {}
  }
}
```


### **Option 3: Automate the process of update the incoming data in existing index**

## For Incoming Data: Attach the pipeline to the index for automatic processing:

#### **Example Request**
```bash
PUT gbtqa-idms/_settings
{
  "index.default_pipeline": "concat_fields_pipeline"
}
```


## **Step 5: Perform Semantic Search**

1. ### **Create a Pipeline to Convert Embeddings into Desired Vectors**  
  This step ensures that the embeddings generated by the model are adjusted to the required vector size (e.g., padding or truncating to match the target   
   dimension). The pipeline uses a script processor for this transformation.

   ```bash
PUT _ingest/pipeline/test_fields_pipeline
{
  "processors": [
    {
      "script": {
        "source": """
          // Extract embeddings as a list
          def embeddings = ctx.predicted_value.values();
          
          // Ensure embeddings is a list
          if (!(embeddings instanceof List)) {
            embeddings = new ArrayList(embeddings);
          }
          
          // Check dimensions
          int targetDim = 64; // Expected dimensions
          if (embeddings.size() < targetDim) {
            // Add zero-padding if smaller
            for (int i = embeddings.size(); i < targetDim; i++) {
              embeddings.add(0.0);
            }
          } else if (embeddings.size() > targetDim) {
            // Truncate if larger
            embeddings = embeddings.subList(0, targetDim);
          }

          // Replace the original embeddings with the adjusted one
          ctx.predicted_value = embeddings;
        """
      }
    }
  ]
}

```

2. ### **Generate Query Embedding**  
   Use the ELSER model to create an embedding for the input query text. Replace "Search query text here" with your input text.

   ```bash
   POST _ml/trained_models/.elser_model_2_linux-x86_64/deployment/_infer
   {
     "docs": [
       { "text_field": "Search query text here" }
     ]
   }
   ```

3. ### **Simulate the pipeline**  
   Pass the generated embedding of the input string through the pipeline to ensure it is transformed into the desired vector format. Replace the sample       
   predicted_value object with your generated embeddings.

   ```bash
    POST _ingest/pipeline/test_fields_pipeline/_simulate
    {
      "docs": [
        {
          "_source": {
            "predicted_value": {
                "##ty": 2.7752354,
                "port": 2.7327287,
                "net": 2.341957,
                "started": 1.6374772,
                "start": 1.3053229,
                "network": 1.1802303,
                "on": 1.0774305,
                "serial": 0.98220545,
                "ports": 0.93985283,
                "onboard": 0.731506,
                "remote": 0.72532815,
                "ship": 0.7197743,
                "pilot": 0.7171623,
                "episode": 0.65427387,
                "##ny": 0.5955155,
                "production": 0.56072825,
                "computer": 0.54560906,
                "transmission": 0.5030203,
                "wireless": 0.4962213,
                "television": 0.08160914,
                "emergency": 0.056852635,
                "server": 0.051834513,
                "programming": 0.048186623,
                "project": 0.018895505,
                "audio": 0.0136941485,
                "dock": 0.012689207,
                "maintenance": 0.007436333
              }
          }
        }
      ]
    }
    ```
   
4. ### **Perform k-Nearest Neighbors (kNN) Search**  
  After processing the embeddings, perform a kNN search to retrieve the most relevant results from the index.

4.1 #### **Retrieve All Fields**
    Use the kNN query to retrieve all fields for the top k results. Replace query_vector with your processed vector.

   ```bash
   POST gbtqa-idms-test/_knn_search
  {
    "knn": {
      "field": "text_expansion",
      "query_vector": [ \* query vector here *\ ],
       "k": 20,
      "num_candidates": 1000
    }
  }
   ```

4.2 #### **Retrieve Selected Fields**
    Specify the fields you want to retrieve in the _source section.

   ```bash
   POST gbtqa-idms-dev-test/_search
  {
    "knn": {
      "field": "text_expansion",
      "query_vector":[
              0.6063633,
              0.12875347,
              0.1593622,
              0.07886452,
              2.7162697,
              0.79152536,
              0.50561506,
              0.3401676,
              0.17824368,
              0.4983237,
              0.18582378,
              ...
            ],
      "k": 20,
      "num_candidates": 1000
    },
    "_source": ["_index", "_id", "_score", "level", "message", "serviceName" ]
  }

   ```

## **Step 6: Handle Client Requests for Adding Fields**

In this step, we address client requests to add new fields and update the pipeline to reflect these changes. This ensures that new and existing data can accommodate any additional fields that clients may require.

---

### 1. **Update the Pipeline**

When a client requests new fields, you need to update the ingest pipeline to include these fields. The pipeline will combine the new fields into a single field (`combined_field` in this example) before sending the data to the model for embedding generation. 

#### Example: Adding a New Field (`app`)

Suppose the client wants to add a new field called `app` to the `combined_field`. This field, along with existing fields (`serviceName`, `level`, `message`), will be concatenated and sent to the model for embedding generation.

Here is how you would update the pipeline:

```bash
PUT _ingest/pipeline/concat_fields_pipeline
{
  "processors": [
    {
      "script": {
        "source": """
          // Combine the app field also
          ctx.combined_field = ctx.serviceName + " " + ctx.level + " " + ctx.message + " " + ctx.app;
        """
      }
    },
    {
      "inference": {
        "model_id": ".elser_model_2_linux-x86_64",
        "field_map": {
          "combined_field": "text_field"
        },
        "target_field": "text_expansion", 
        "inference_config": {
          "text_expansion": {}
        }
      }
    },
    {
      "script": {
        "source": """
          if (ctx.text_expansion != null && ctx.text_expansion.predicted_value != null) {
            ctx.text_expansion = new ArrayList(ctx.text_expansion.predicted_value.values());
            int targetDim = 64;
            while (ctx.text_expansion.size() < targetDim) {
              ctx.text_expansion.add(0.0);
            }
            if (ctx.text_expansion.size() > targetDim) {
              ctx.text_expansion = ctx.text_expansion.subList(0, targetDim);
            }
          }
        """
      }
    },
    {
      "remove": {
        "field": "combined_field" # Remove intermediate fields
      }
    }
  ]
}
```

### 2. **Regenerate Embeddings for Existing Data**
Once the pipeline is updated, you need to apply the changes to the existing data that was indexed before the new field (app) was added.
#### Repeat from step 4 again
Since the updated pipeline is now attached to the index, any new data coming into the index will automatically use the updated pipeline logic. This means that any new documents indexed with the app field will automatically be processed using the new pipeline, which includes combining the new app field with existing fields and generating embeddings.
You don't need to manually reprocess new data; it will be handled automatically by Elasticsearch as long as the data is indexed with the correct pipeline.


## **Step 7: Optimized Approach for Large Data (100M Documents)**

### **Option 1: Use Scroll or Slice Queries**
Instead of processing all documents in one go, process them in smaller batches.

#### **Example Using Slices:**
```bash
POST gbtqa-idms/_update_by_query?pipeline=concat_fields_pipeline&scroll_size=1000&slices=10
{
  "query": {
    "match_all": {}
  }
}
```
- scroll_size: Controls the batch size of documents processed in memory (default: 1000).
- slices: Divides the workload into slices (default: 1). Adjust slices to match the number of available CPUs or nodes.

### **Option 2: Run During Off-Peak Hours**
- Schedule the update_by_query task during periods of low traffic to reduce impact on production workloads.
- Monitor cluster performance metrics (CPU, memory, I/O).

### **Option 3: Use Task Management API**
update_by_query runs as a background task in Elasticsearch. You can monitor and manage this process using the Task Management API.

#### **Start the Update Task**
```bash
POST gbtqa-idms/_update_by_query?pipeline=concat_fields_pipeline&wait_for_completion=false
{
  "query": {
    "match_all": {}
  }
}
```
- wait_for_completion=false: Allows the process to run asynchronously.


## **Step 8: Visualization**

1. Kibana Dashboards:
- Create visualizations for kNN results.
2. Monitor Pipeline Performance:
- Use Elasticsearch monitoring.


## **Step 8: Summary**

1. Designed for Scalability:
- Handles 100M+ records efficiently.
2. Adaptable to Changes:
- Supports adding fields dynamically.
3. Optimized for Performance:
- Efficient embeddings and kNN searches.









