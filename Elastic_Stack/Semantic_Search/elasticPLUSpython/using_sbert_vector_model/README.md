# Elasticsearch and Semantic Search Example

This repository contains a Jupyter Notebook demonstrating how to use Elasticsearch with semantic search to analyze product descriptions and perform similarity-based queries. The notebook leverages Elasticsearch, pandas, and Sentence Transformers for handling and processing data.

## Requirements

To run the code, make sure you have the following prerequisites:

### Dependencies
- Python 3.7+
- Elasticsearch 8.9.1+
- Required Python libraries:
  - `elasticsearch`
  - `pandas`
  - `sentence-transformers`

### Additional Files
- A CSV file named `myntra_products_catalog.csv` containing product data with at least the following columns:
  - `ProductID`
  - `Description`

- A valid Elasticsearch index mapping configuration provided in `indexMapping.py`.

## Setup

### Elasticsearch Configuration
1. Install and configure Elasticsearch.
2. Update the connection details in the notebook:
   - Host: `https://localhost:9200`
   - Basic Authentication: `elastic` (username) and your password.
   - CA Certificate Path: `elasticsearch-8.9.1/config/certs/http_ca.crt`

### Python Dependencies Installation
Install the required Python libraries using pip:
```bash
pip install elasticsearch pandas sentence-transformers
```

## Code Overview

### Steps

#### 1. Elasticsearch Connection
Establishes a connection to Elasticsearch using the provided credentials and CA certificate.

**Code:**
```python
from elasticsearch import Elasticsearch

es = Elasticsearch(
    "https://localhost:9200",
    basic_auth=("elastic", "<your-password>"),
    ca_certs="/path/to/http_ca.crt"
)
assert es.ping(), "Elasticsearch connection failed!"
```

#### 2. Data Preprocessing
Loads product data from `myntra_products_catalog.csv` and handles missing values.

**Code:**
```python
import pandas as pd

# Load data
df = pd.read_csv("myntra_products_catalog.csv").loc[:499]

# Handle missing values
df.fillna("None", inplace=True)

print(df.head())
```

#### 3. Semantic Vectorization
Uses the `SentenceTransformer` model (`all-mpnet-base-v2`) to generate embeddings for the `Description` column.

**Code:**
```python
from sentence_transformers import SentenceTransformer

# Load the model
model = SentenceTransformer('all-mpnet-base-v2')

# Generate embeddings
df["DescriptionVector"] = df["Description"].apply(lambda x: model.encode(x))

print(df.head())
```

#### 4. Index Creation
Creates an Elasticsearch index named `all_products` with a specified mapping.

**Code:**
```python
from indexMapping import indexMapping

# Create index
es.indices.create(index="all_products", mappings=indexMapping)
```

#### 5. Data Ingestion
Converts the DataFrame to a list of dictionaries and indexes the product records in Elasticsearch.

**Code:**
```python
# Convert DataFrame to a list of dictionaries
record_list = df.to_dict("records")

# Index records
for record in record_list:
    try:
        es.index(index="all_products", document=record, id=record["ProductID"])
    except Exception as e:
        print(f"Error indexing record: {e}")
```

#### 6. Semantic Search
Encodes an input keyword into a semantic vector and performs a KNN search on the `DescriptionVector` field.

**Code:**
```python
# Input keyword
input_keyword = "Blue Shoes"

# Encode keyword
vector_of_input_keyword = model.encode(input_keyword)

# Define KNN query
query = {
    "field": "DescriptionVector",
    "query_vector": vector_of_input_keyword,
    "k": 2,
    "num_candidates": 500
}

# Perform search
res = es.knn_search(index="all_products", knn=query, source=["ProductName", "Description"])

# Display results
print(res["hits"]["hits"])
```

### Outputs
- Displays the top search results with product names and descriptions.

## Running the Notebook
1. Clone the repository and navigate to the project directory.
2. Open the Jupyter Notebook:
   ```bash
   jupyter notebook
   ```
3. Run the notebook cells sequentially.

## Notes
- Ensure that Elasticsearch is running and accessible before executing the notebook.
- Adjust the Elasticsearch configuration and index mapping to suit your dataset if necessary.
- If any errors occur during indexing, they will be printed to the console.

