---
title: "🚀 S3 Just Killed the Vector Database: How Amazon S3 Vectors Changes Everything for AI Data Storage 💾 "
draft: false
date: 2025-08-10T15:06:41+02:00
tags:
  - vectors
  - search
  - data-engineering
  - database
  - AI
cover:
  image: ""
  alt: 
  caption: 
---


What if I told you that you could run vector searches directly on S3 without spinning up a single database or compute cluster? For years, we've been stuck with a painful pipeline: extract data from S3, chunk it, generate embeddings, load everything into **OpenSearch or Pinecone**, and manage all that infrastructure. 

Amazon just changed the game with S3 Vectors – it's S3 that can do vector math natively, no compute engine required. This means up to **90% cost savings and zero infrastructure management**. Let me show you exactly how this works and why it might replace your vector database entirely.


##  The Old Way: Why Vector Storage Was a Pain 😓

Picture the traditional vector search architecture: Your data sits in S3, then you need an embedding application to chunk and vectorize it, followed by a vector database like OpenSearch or Pinecone, and finally your application queries that database.

// some nice picture with flow for embeding and so on...

Each step is a potential failure point – managing chunking strategies, handling batch processing, dealing with failed embeddings, and orchestrating everything with Lambda or ECS.

The costs are brutal. OpenSearch runs you **$700+/month minimum** for a production cluster, plus you're managing index allocation, shard management, and capacity planning. You're essentially paying twice – once for S3 storage and again for vector database storage and compute – all just to run similarity searches on data that was already sitting in S3.

## Enter S3 Vectors: Storage That Actually Thinks 🧠
S3 Vectors isn't just another storage option – it's **S3 with built-in vector search capabilities.** No separate compute engine, no database cluster, just native vector operations right where your data lives. You create a vector bucket, define vector indexes with dimensions and distance metrics (cosine or euclidean), and start querying with subsecond response times.

_The numbers are impressive: each vector bucket supports up to **10,000 indexes**, and each index can hold **tens of millions of vectors.** You can attach metadata to vectors for filtered searches, and S3 automatically optimizes the data structure as your dataset grows. Best part? You only pay for storage and queries – no compute clusters burning money 24/7._

## How It Works Under the Hood? ⚙️

Vector buckets are a completely new S3 bucket type with dedicated APIs for vector operations. Unlike traditional buckets that store objects as blobs, vector buckets organize data into vector indexes – think of them as purpose-built structures optimized for similarity search. When you create an index, you specify the vector dimensions (must match your embedding model) and the distance metric.

The real magic happens during queries. S3 Vectors uses **approximate nearest neighbor (ANN) algorithms** to find similar vectors without scanning the entire dataset. You send a query vector, specify how many results you want (topK), and optionally add metadata filters. S3 handles all the indexing, optimization, and query execution internally – no FAISS libraries, no managing HNSW parameters, just simple API calls.

As you add, update, or delete vectors, S3 automatically rebalances and optimizes the index structure. This means consistent performance whether you have **1,000 or 10 million vectors**, without any manual tuning or re-indexing operations.


## Practical Implementation: From Zero to Vector Search 💻

First let's create a bucket.

![s3-buckets](/posts/s3-vector-bucket/s3-create-bucket.png)

Once created we can then create a vector index and query our data.

Let's build a simple vector search system. First, create a vector bucket and index:

```python
import boto3

# Create S3 Vectors client
s3vectors = boto3.client('s3vectors', region_name='us-west-2')


```

Generate embeddings using Bedrock and insert them into your index:

```python

# Generate embeddings with Bedrock
bedrock = boto3.client("bedrock-runtime", region_name="us-west-2")

text = "Star Wars: A farm boy joins rebels to fight an evil empire"
response = bedrock.invoke_model(
    modelId='amazon.titan-embed-text-v2:0',
    body=json.dumps({"inputText": text})
)
embedding = json.loads(response['body'].read())['embedding']

# Insert into S3 Vectors with metadata
s3vectors.put_vectors(
    vectorBucketName="star-wars-vector-bucket",
    indexName="come-to-the-dark-side-index",
    vectors=[{
        "key": "doc1",
        "data": {"float32": embedding},
        "metadata": {"title": "Star Wars", "genre": "scifi"}
    }]
)

```

Generate embeddings using Bedrock and insert them into your index:


```python
# Generate embeddings with Bedrock
bedrock = boto3.client("bedrock-runtime", region_name="us-west-2")

text = "Star Wars: A farm boy joins rebels to fight an evil empire"
response = bedrock.invoke_model(
    modelId='amazon.titan-embed-text-v2:0',
    body=json.dumps({"inputText": text})
)
embedding = json.loads(response['body'].read())['embedding']

# Insert into S3 Vectors with metadata
s3vectors.put_vectors(
    vectorBucketName="star-wars-vector-bucket",
    indexName="come-to-the-dark-side-index",
    vectors=[{
        "key": "doc1",
        "data": {"float32": embedding},
        "metadata": {"title": "Star Wars", "genre": "scifi"}
    }]
)

```

Query your vectors with similarity search:

```python
# Search for similar content
query_text = "Movies about space adventures"
query_embedding = generate_embedding(query_text)  # Same Bedrock process

results = s3vectors.query_vectors(
    vectorBucketName="star-wars-vector-bucket",
    indexName="come-to-the-dark-side-index",
    queryVector={"float32": query_embedding},
    topK=5,
    filter={"genre": "scifi"},  # Optional metadata filter
    returnDistance=True
)

results = query["vectors"]
print(results)
```