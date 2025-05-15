---
title: "🦾 Picture Perfect Match: Building an Image Similarity Search Engine with Vector Databases🤖"
date: 2025-05-15T13:06:41+02:00
draft: true
tags:
  - vector-database
  - AI
  - data-engineering
  - python
  - deep-learning
  - computer-vision
cover:
  image: ""
  alt: "vector-database"
  caption: "vector-database"
---

# Introduction

Have you ever wondered how Pinterest finds visually **similar images** or how Google Photos recognizes faces across thousands of pictures? The technology that powers these features isn't magic—it's vector similarity search. Today, modern vector databases make it possible for developers to build these powerful visual search capabilities without needing a PhD in computer vision.


In this post, I'll guide you through the process of building your own **image similarity search engine**. We'll cover everything from understanding vector embeddings to implementing a working solution that can find visually similar images in milliseconds.

# Understanding Vector Embeddings for Images

Before we dive into the implementation details, let's understand what vector embeddings are and why they're crucial for image similarity search.

## What is vector?

A **vector** is a simple mathematical idea that describes something with both a size (how much) and a direction (which way). Think of a vector like an arrow: the length of the arrow shows how big it is, and the way it points shows the direction.

#### Everyday Example
Imagine you’re giving someone directions:

“Walk 3 steps forward and 4 steps to the right.”

This instruction can be shown as an arrow from where you start to where you end up. That arrow is a vector: it tells you both how far to go (the magnitude) and which way to go (the direction)

3 steps forward would be the magnitude of a vector and direction would be 4 steps to the right

// Add here picture

## What is dimension of vector?

Vector dimension is a way to describe how much information is needed to specify a vector, or how many "directions" you can move in that space.

If you look at this image this vector wil have **3 dimensions** (1, 4, 3). It can contain as many dimesions as you want. Everything more than 3 dimesions is hard to imagine and plot. 

![3-dimensions-vector](/posts/image-similarity-vector-db/3-dimesions-vector.jpg)


## What is vector space?

Vector space is container of all vectors, imagine this as a bucket for all vectors. 

// create image here for this

## What Are Vector Embeddings?

Vector embeddings are numerical representations of data in multi-dimensional (vector) space. For images, these embeddings capture visual features like shapes, textures, colors, and objects within a fixed-length vector (typically hundreds or thousands of dimensions).


**In simple words, machines aren't naturally good at understanding data like images, audio, and text. To help them process this information, we convert it into a mathematical form a domain where machines know how to operate effectively and can detect patterns in data.**


The beauty of vector embeddings is that similar images will have vectors that are closer to each other in this high-dimensional space. This geometric property allows us to find similar images by measuring the distance between vectors.

// create in excalidraw vector embeding images

# Vector Databases: The Engine Behind Similarity Search

Traditional databases are great for exact matches and range queries, but they fall short when it comes to finding "similar" items in high-dimensional space. This is where vector databases shine.

## What Makes Vector Databases Special?

Vector databases are purpose-built for storing and querying vector embeddings. They implement specialized algorithms like Approximate Nearest Neighbor (ANN) search that can efficiently find the closest vectors to a query vector, even in datasets with millions of entries.

Key features that set vector databases apart:

- **Similarity metrics:** Support for cosine similarity, Euclidean distance, and other vector comparison methods
- **Indexing structures:** Advanced indexing like HNSW or IVF that enable sub-linear search times
- **Filtering capabilities:** Combining vector similarity with metadata filtering
- **Scalability:** Distributed architecture for handling large collections

### Popular Vector Database Options

Several powerful vector databases have emerged in recent years:

| Database  |  Key Features   ⭐                        | 🚀 Best For                           |
|-----------|------------------------------------------|---------------------------------------|
| **Pinecone**  |  Fully managed, serverless  🛠️          |  Production deployments  🏭           |
| **Milvus**    |  Open-source, highly scalable   🌱       | Self-hosted large-scale applications 🏢 |
| **Weaviate**  |  Schema-based, multimodal    🧩          |  Complex data relationships 🔗         |
| **Qdrant**    | Simple API, filtering, self-hosted ⚡   | Quick prototyping, startups 🚀        |

# Building Your Image Similarity Search Pipeline

Let's dive into the practical steps for building an image similarity search system:

## Step 1: Image Collection and Preprocessing

First, you'll need a collection of images. For this tutorial, let's assume we have a directory of some images. For this tutorial I will chose different type of fruits and vegetables.

```python

import os
from pathlib import Path

# Define image directory
image_dir = Path("./data/images/")

# Get all image paths
image_paths = [str(f) for f in image_dir.glob("*.jpg")]
print(f"Found {len(image_paths)} images")


```

## Step 2: Generating Vector Embeddings

```python

import numpy as np
from tqdm import tqdm
import torch
from torchvision import models, transforms
from PIL import Image

model = models.resnet50(pretrained=True)

# Remove the classification layer to get embeddings
model = torch.nn.Sequential(*(list(model.children())[:-1]))
model.eval()

# Prepare image transformation
# We are scaling all images and normalizng them
transform = transforms.Compose([
    transforms.Resize(256),
    transforms.CenterCrop(224),
    transforms.ToTensor(),
    transforms.Normalize(mean=[0.485, 0.456, 0.406], 
                         std=[0.229, 0.224, 0.225])
])

def get_image_embedding(image_path):
    # Load and transform image
    img = Image.open(image_path).convert('RGB')
    img_t = transform(img)
    batch_t = torch.unsqueeze(img_t, 0)
    
    # Get embedding
    # Using already trained model resnet50
    with torch.no_grad():
        embedding = model(batch_t)
    
    # Flatten to 1D vector and return as numpy array
    return embedding.squeeze().cpu().numpy()




# Generate embeddings for all images
embeddings = {}
for img_path in tqdm(image_paths, desc="Generating embeddings"):
    try:
        # Get the image ID from the filename
        img_id = os.path.basename(img_path).split('.')[0]
        # Generate embedding
        embedding = get_image_embedding(img_path)
        embeddings[img_id] = embedding
    except Exception as e:
        print(f"Error processing {img_path}: {e}")

print(f"Generated embeddings for {len(embeddings)} images")

```
