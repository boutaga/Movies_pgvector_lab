# Movies_pgvector_lab 

This repository is intended for educational purpose, by following the instructions you will be able to have a LAB and play with pgvector and similarity searches on PostgreSQL.

The aim of this LAB is to performa a similarity search to be able to recommend Netflix shows to DVDRental's users based on their renting profile. 



## future features

- Implementation of GraphDB
- Example of RAG search using GraphDB
- Example of RAG search using Graph + vectorDB



## Installation steps

Here are the steps to build your environment :

- Provision a Linux server that will host your PostgreSQL instance.
- Install PostgreSQL 17.2 and pgvector 0.8.0 extension with it.
- Import both dvdrental and netflix_shows databases into one database (this makes it easier to query both sources at the same time).
- Add the vector fields in the target tables.
- On your user environment create the two python scripts to create the embeddings and to run the query search for similarity.
- Setup your Python virtual environment.
- Get an API key for your AI model, in my case OpenAI, may even host locally your own LLM.
- Run the script that creates the embeddings.
- Run the script that queries your vectors and look for similarities.


---

## pgvector installation and environment preparation

**Clone the Repository:**  
    Clone the pgvector extension from GitHub:
    
```bash
    cd /tmp
    git clone https://github.com/pgvector/pgvector.git
    cd pgvector
```
    
**Build and Install:**  
    Use make to compile the extension and then install it:
    
```bash
    make
    sudo make install
```
    
**Verify Installation:**  
    In psql, connect to your database and run:
    
```sql
    CREATE EXTENSION IF NOT EXISTS vector;
    SELECT * FROM pg_extension WHERE extname = 'vector';
```
    


**Download the Sample Database:**  

Visit [Neon’s PostgreSQL Sample Database](https://neon.tech/postgresql/postgresql-getting-started/postgresql-sample-database) and download the SQL dump file.

Visit [Kaggle’s Netflix Shows dataset](https://www.kaggle.com/datasets/shivamb/netflix-shows) and download the CSV file (e.g., `netflix_titles.csv`).



```sql

--psql 
CREATE DATABASE dvdrental;

--bash
unzip dvdrental.zip
  
pg_restore -U postgres -d dvdrental dvdrental.tar


--psql    
\c dvdrental  
    
CREATE TABLE netflix_shows (                                                                                        
    show_id TEXT PRIMARY KEY,
    type VARCHAR(20),
    title TEXT,
    director TEXT,
    "cast" TEXT,
    country TEXT,
    date_added TEXT,      -- stored as text; later you can convert to DATE with TO_DATE if desired
    release_year INTEGER,
    rating VARCHAR(10),
    duration VARCHAR(20),
    listed_in TEXT,
    description TEXT
);

--import from the csv file
COPY netflix_shows(show_id, type, title, director, "cast", country, date_added, release_year, rating, duration, listed_in, description)
FROM '/home/postgres/LAB_vector/netflix_titles.csv'
WITH (
    FORMAT csv,
    HEADER true,
    DELIMITER ',',
    NULL ''
);

    
ALTER TABLE film ADD COLUMN embedding vector(1536);
ALTER TABLE netflix_shows ADD COLUMN embedding vector(1536);

ALTER TABLE film ADD COLUMN sparse_embedding sparsevec(30522);
ALTER TABLE netflix_shows ADD COLUMN sparse_embedding sparsevec(30522);

    
CREATE INDEX IF NOT EXISTS film_embedding_idx ON film USING hnsw (embedding vector_l2_ops);   
CREATE INDEX IF NOT EXISTS netflix_shows_embedding_idx ON netflix_shows USING hnsw (embedding vector_l2_ops);
```

### **Python Environment:**  

Create a virtual environment and install the required packages:
    
```bash
    python3 -m venv pgvector_lab
    source pgvector_lab/bin/activate
    pip install psycopg2-binary openai pgvector transformers torch sentencepiece
 ```
    


---

## **create_emb.py Description**

This Python script, **`create_emb.py`**, is responsible for generating and storing **text embeddings** for rows in a PostgreSQL database. It is designed to work hand-in-hand with the previous recommendation script by populating the `embedding` column in specified tables (`film` and `netflix_shows`). Here’s a step-by-step breakdown of its key functionalities:



### 1. Prerequisites
- **Environment Variable**:  
  - `DATABASE_URL` must be set (e.g., `postgresql://postgres@localhost/dvdrental`) to connect to the PostgreSQL database.
  - `OPENAI_API_KEY` must be set to authenticate calls to the OpenAI API.
- **Database**:
  - The target tables (`film` and `netflix_shows`) are assumed to have a column named `embedding` of type `vector` (enabled via the `pgvector` extension).
  - The script expects text data (such as `description`) to be present in these tables.



### 2. Batching and Rate Limiting
- The script uses a **batch approach** to handle embedding requests in chunks:
  - `BATCH_SIZE` (default=30) controls how many text rows are sent to the OpenAI API in each request.
  - Batching helps manage throughput while staying under potential rate limits.

- **Exponential Backoff**:
  - If the script encounters rate-limit errors (HTTP 429) or quota issues from the API, it retries multiple times with an increasing delay (`delay *= 2`).



### 3. Retrieving and Updating Embeddings
Two main functions manage the core logic:

1. **`get_batch_embeddings(client, texts, model, max_retries=5)`**  
   - Makes a single API call to generate embeddings for a batch of text inputs.  
   - Implements retry logic with exponential backoff for rate-limit errors.  
   - On success, returns a list of embedding vectors (one per text input).  

2. **`update_table_embeddings(conn, table_name, id_column, text_column, model)`**  
   - Fetches all rows from a specified table, retrieving only two columns:
     - `id_column` (e.g., `film_id` or `show_id`)  
     - `text_column` (e.g., `description`)  
   - Groups these rows into batches of size `BATCH_SIZE`.  
   - For each batch:
     - Calls `get_batch_embeddings` to retrieve embeddings from the OpenAI API.  
     - Updates each row’s `embedding` column in the database with the corresponding vector.  
   - Commits changes to the database after each batch.

#### Example Database Update Query
```sql
UPDATE <table_name>
SET embedding = %s
WHERE <id_column> = %s;
```
(where `%s` in the Python script is replaced with the actual embedding vector and row ID)



### 4. Script Flow
1. **Open a Database Connection**:  
   Reads `DATABASE_URL` from the environment, then connects via `psycopg2`.  
   Registers `pgvector` with `register_vector(conn)`.

2. **Choose the Embedding Model**:  
   The script uses the OpenAI model `text-embedding-ada-002`, which outputs 1536-dimensional vectors.

3. **Update Embeddings for the `film` Table**:  
   - Retrieves the `film_id` and `description`.  
   - Generates embeddings in batches.  
   - Stores these embeddings in the `film.embedding` column.

4. **Update Embeddings for the `netflix_shows` Table**:  
   - Similarly retrieves `show_id` and `description`.  
   - Generates embeddings in batches.  
   - Stores these embeddings in the `netflix_shows.embedding` column.

5. **Close the Database Connection**:  
   Once all embeddings are updated, the script closes the connection.



### 5. Error Handling and Logging
- **Rate Limits**:  
  If a rate-limit or quota error occurs, the script prints a warning, waits a bit, then retries.  
- **Other Exceptions**:  
  Non-retryable errors (e.g., bad inputs, malformed requests) halt the process and raise an exception.  
- **Progress Logging**:  
  The script prints out batch IDs as they are processed and updated, so you can track progress and troubleshoot any issues.



### 6. Usage
1. **Set Environment Variables**:
   ```bash
   export DATABASE_URL="postgresql://postgres@localhost/dvdrental"
   export OPENAI_API_KEY="sk-..."  # Your OpenAI API key
   ```
2. **Run the Script**:
   ```bash
   python3 create_emb.py
   ```
3. **Check Results**:
   - Ensure the `embedding` columns in `film` and `netflix_shows` are populated with 1536-dimensional vectors.
   - If everything is successful, you can then use these embeddings in the recommendation script to provide content suggestions.




---


## **recommend_netflix.py Description**



### 1. Environment Setup
- **Database Connection**: The script uses the environment variable `DATABASE_URL` to connect to the database. If this variable is not set, it defaults to `postgresql://postgres@localhost/dvdrental`.
- **pgvector**: The script imports and registers the `pgvector` extension (`from pgvector.psycopg2 import register_vector`). This extension allows storing and querying vector columns (such as embeddings) in PostgreSQL.



### 2. Retrieving the Customer Profile (`get_customer_profile` function)
- **Purpose**: Takes a `customer_id` and a database cursor as parameters, then:
  1. **Fetches all films** rented by this customer.
  2. **Retrieves the embeddings** for each rented film.
  3. **Computes the customer’s “profile embedding”** by taking the average of all retrieved embeddings.

- **Database Query**:  
  ```sql
  SELECT f.film_id, f.embedding
  FROM rental r
  JOIN inventory i ON r.inventory_id = i.inventory_id
  JOIN film f ON i.film_id = f.film_id
  WHERE r.customer_id = %s;
  ```
  - Joins several tables (`rental`, `inventory`, `film`) to link a rental to its film, and retrieves the film’s embedding vector.

- **Edge Cases**:
  - If no rentals are found for the customer, the function returns `None`.
  - If a film has no embedding (`None`), that film is skipped.
  - If ultimately no valid embeddings are found, the function returns `None`.

- **Result**: Returns a single profile embedding (as a list of floats) that represents the user’s taste or preferences based on their past rentals.



### 3. Generating Netflix Recommendations (`recommend_netflix` function)
- **Customer Input**: Prompts the user to enter a customer ID. Validates that the input is a digit.
- **Database Connection**: Connects to the PostgreSQL database using the `psycopg2` library and registers the pgvector extension.

- **Profile Creation**: Calls `get_customer_profile` to calculate the user’s average embedding.
  - If `get_customer_profile` returns `None`, the script ends early (no recommendations can be made).

- **Fetching Recommendations**:
  - Queries the `netflix_shows` table for shows that have a non-NULL embedding.
  - Uses the `<->` operator (provided by pgvector) to compute the distance (similarity) between the show’s embedding and the user’s profile embedding.
  - Sorts results by ascending distance, meaning the most similar content appears first.
  - Limits the output to the top 5 results.

  ```sql
  SELECT title, description, embedding <-> (%s)::vector AS distance
  FROM netflix_shows
  WHERE embedding IS NOT NULL
  ORDER BY embedding <-> (%s)::vector
  LIMIT 5;
  ```

- **Output**: Prints the top 5 recommended titles, each accompanied by a similarity distance value. Lower distance implies higher similarity. If no shows are found, a fallback message appears.

- **Cleanup**: Closes the database cursor and connection before exiting.




- **Usage**: Simply run the script in a Python 3 environment where the `DATABASE_URL` environment variable is set (or defaults to the local database).  
  ```bash
  export DATABASE_URL="postgresql://postgres@localhost/dvdrental"
  python3 recommend_netflix.py
  ```


## Hybrid RAG Search with Sparse-dense vectors


### Example of query 


```sql
WITH dense_search AS (
    SELECT
        film_id,
        -- Calculate dense similarity score (cosine similarity: 1 - cosine distance)
        -- $1 should be the query's dense vector (e.g., from text-embedding-ada-002)
        1 - (embedding <=> $1::vector) AS dense_score,
        -- Rank results based on dense similarity
        ROW_NUMBER() OVER (ORDER BY embedding <=> $1::vector ASC) as dense_rank
    FROM
        film
    ORDER BY
        dense_score DESC
    LIMIT 100 -- Limit the number of results considered from dense search
),
sparse_search AS (
    SELECT
        film_id,
        -- Calculate sparse similarity score (inner product)
        -- $2 should be the query's sparse vector in '{idx:val,...}/dim' format
        sparse_embedding <#> $2::sparsevec AS sparse_score,
         -- Rank results based on sparse similarity
        ROW_NUMBER() OVER (ORDER BY sparse_embedding <#> $2::sparsevec DESC) as sparse_rank
    FROM
        film
    WHERE
        -- Pre-filter using inner product index if available and beneficial
        sparse_embedding <#> $2::sparsevec > 0 -- Example: only consider positive inner products
    ORDER BY
        sparse_score DESC
    LIMIT 100 -- Limit the number of results considered from sparse search
),
-- Reciprocal Rank Fusion (RRF)
rrf_ranked AS (
    SELECT
        film_id,
        -- Calculate RRF score (k is typically 60, adjust as needed)
        COALESCE(1.0 / (60 + dense_rank), 0.0) + COALESCE(1.0 / (60 + sparse_rank), 0.0) AS rrf_score
    FROM dense_search
    FULL OUTER JOIN sparse_search USING (film_id) -- Combine results using film_id
)
SELECT
    f.film_id,
    f.title,
    f.description,
    r.rrf_score
FROM
    rrf_ranked r
JOIN
    film f ON r.film_id = f.film_id
ORDER BY
    r.rrf_score DESC
LIMIT 10; -- Get the top N hybrid results

```

### Explanation of the Hybrid Search Query (RRF Method):

#### 1.dense_search CTE:
Performs a nearest neighbor search using the dense embedding column and the dense query vector ($1).
Calculates cosine similarity (1 - <=>).
Assigns a dense_rank based on similarity.
Limits the results (e.g., to 100) to manage performance.

#### 2.sparse_search CTE:

Performs a search using the sparse sparse_embedding column and the sparse query vector ($2).
Uses the inner product operator (<#>) which is suitable for SPLADE vectors (representing term importance). Higher inner product means better sparse match.
Assigns a sparse_rank based on the inner product score.
Includes an optional WHERE clause (sparse_embedding <#> $2::sparsevec > 0) which might leverage an index if created with sparse_inner_product_ops and helps filter out non-matches early.
Limits the results (e.g., to 100).

#### 3.rrf_ranked CTE:

Combines the results from dense_search and sparse_search using a FULL OUTER JOIN on the film_id to include items found in either search.
Calculates the Reciprocal Rank Fusion (RRF) score for each film_id. The formula is sum(1 / (k + rank)) for each list an item appears in. k is a constant (often 60) that dampens the impact of high ranks. COALESCE handles items not found in one of the searches.

#### 4.Final SELECT:

Joins the RRF scores back to the original film table to retrieve other details (like title, description).
Orders the final results by the calculated rrf_score in descending order.
Applies a final LIMIT to get the desired number of top hybrid results.

#### Parameters:

$1: Your query text embedded using the dense embedding model (e.g., OpenAI text-embedding-ada-002), passed as a vector.
$2: Your query text embedded using the sparse embedding model (SPLADE), passed as a sparsevec string in the format '{idx:val,...}/dim'.