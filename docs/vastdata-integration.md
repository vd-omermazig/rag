<!--
  SPDX-FileCopyrightText: Copyright (c) 2025 NVIDIA CORPORATION & AFFILIATES. All rights reserved.
  SPDX-License-Identifier: Apache-2.0
-->

# VastData Vector Store Integration

This guide explains how to deploy the NVIDIA RAG Blueprint with VastData's vector store instead of the default Milvus vector database. VastData provides a high-performance HTTP-based vector store that can be used as a drop-in replacement for Milvus.

## Table of Contents

- [Prerequisites](#prerequisites)
- [VastData Setup](#vastdata-setup)
- [Configuration](#configuration)
- [Deployment](#deployment)
- [Data Ingestion](#data-ingestion)
- [Troubleshooting](#troubleshooting)

## Prerequisites

1. **NVIDIA API Key**: Follow the [quickstart guide](quickstart.md#obtain-an-api-key) to obtain your NGC/NVIDIA API key for cloud-hosted NIMs.

2. **VastData Vector Store Service**: You need a running VastData vector store HTTP service. This should be accessible from your Docker containers.

3. **Docker & Docker Compose**: Ensure you have Docker and Docker Compose installed. See [quickstart prerequisites](quickstart.md#prerequisites).

## VastData Setup

### 1. Start VastData HTTP Service

Ensure your VastData vector store HTTP service is running and accessible. The service should be available at a URL like `http://localhost:8088` or your specific VastData endpoint.

**Note**: If running in Docker, use `host.docker.internal:8088` instead of `localhost:8088` for container-to-host communication.

### 2. Verify VastData Service

Test that your VastData service is responding:

```bash
curl -X GET http://localhost:8088/health/readiness
# or
curl -X GET http://host.docker.internal:8088/health/readiness
```

## Configuration

### 1. Environment Configuration

The RAG Blueprint includes VastData configuration in `deploy/compose/.env`. Update the following variables:

```bash
# Vector DB Configuration - Switch to VastData
export APP_VECTORSTORE_NAME=vast

# VastData VectorStore Configuration
export VAST_VECTORSTORE_BASE_URL=http://host.docker.internal:8088
export VAST_VECTORSTORE_AUTH_TOKEN=your_auth_token_here

# Cloud NIMs Configuration (recommended for VastData integration)
export APP_EMBEDDINGS_MODELNAME="nvidia/nv-embedqa-e5-v5"
```

### 2. API Keys

Set your NVIDIA API keys for cloud-hosted NIMs:

```bash
export NGC_API_KEY=your_ngc_api_key_here
export NVIDIA_API_KEY=your_nvidia_api_key_here
export NVIDIA_BUILD_API_KEY=your_nvidia_api_key_here
```

**Important**: VastData integration works best with cloud-hosted NVIDIA NIMs. Make sure you have a valid NVIDIA API key from [NGC](https://org.ngc.nvidia.com/setup/api-keys).

## Deployment

### 1. Start the RAG Server

Deploy the RAG server with VastData configuration:

```bash
# Navigate to the repository root
cd ..

# Start the RAG server with VastData configuration
docker compose -f deploy/compose/docker-compose-rag-server.yaml up -d
```

### 2. Verify Deployment

Check that all services are running:

```bash
docker ps --format "table {{.Names}}\t{{.Status}}"
```

You should see:
- `rag-server` - Up and healthy
- `rag-frontend` - Up and healthy
- A lot of other vast containers for the separate deployment like `nvidia-api`, `langchain-ingest-docs` and such  

### 3. Test VastData Connection

Check the RAG server logs to ensure VastData connection is successful:

```bash
docker logs rag-server | grep -i "vast"
```

You should see messages like:
```
INFO:src.utils:Creating VastData VectorStore with endpoint: http://host.docker.internal:8088
```

## Data Ingestion

VastData integration requires external ingestion since the data needs to be processed and stored in your VastData vector store before querying.

### Document Ingestion

Since VastData is an external vector store, you'll need to ingest documents using your VastData ingestion pipeline. This typically involves:

1. **Document Processing**: Extract text from your documents
2. **Embedding Generation**: Generate embeddings using the same model as the RAG server (`nvidia/nv-embedqa-e5-v5`)
3. **Vector Storage**: Store the embeddings in your VastData collection

**External Ingestion Required:**

Since VastData integration is retrieval-only, you must ingest documents using your own VastData ingestion pipeline **before** using the RAG server. The RAG server only connects to VastData for retrieval operations.

**Example external ingestion workflow (outside of the RAG system):**
1. Use VastData's native ingestion tools or APIs
2. Process documents and generate embeddings with `nvidia/nv-embedqa-e5-v5` model
3. Store embeddings in your VastData collection with proper metadata structure:
   ```json
   {
     "source": "document_name.pdf",
     "page_number": 1,
     "title": "Document Title"
   }
   ```

**Note**: The RAG server does **not** perform ingestion for VastData - it only retrieves from pre-populated collections.

### 3. Verify Ingestion

Test document retrieval through the RAG server:

```bash
curl -X POST "http://localhost:8081/v1/search" \
  -H "Content-Type: application/json" \
  -d '{
    "query": "test query",
    "collection_name": "my_collection_name",
    "reranker_top_k": 5,
    "vdb_top_k": 10
  }'
```

## Usage

### 1. Query via Frontend

1. Open the RAG frontend at http://localhost:8090
2. Select your VastData collection
3. Ask questions about your ingested documents

### 2. Query via API

```bash
curl -X POST "http://localhost:8081/v1/generate" \
  -H "Content-Type: application/json" \
  -d '{
    "messages": [{"role": "user", "content": "What is the main topic of the documents?"}],
    "use_knowledge_base": true,
    "collection_name": "my_collection_name",
    "temperature": 0.7,
    "max_tokens": 512
  }'
```

## Troubleshooting

### Common Issues

1. **Connection Refused Error**
   ```
   Failed to establish a new connection: [Errno 111] Connection refused
   ```
   **Solution**: Ensure VastData service is running and use `host.docker.internal:8088` instead of `localhost:8088` in Docker.

2. **Authentication Errors**
   ```
   401 Unauthorized - Authentication failed
   ```
   **Solution**: Verify your `NVIDIA_API_KEY` is valid and has the correct permissions.

3. **Collection Not Found**
   ```
   Collection 'my_collection_name' does not exist
   ```
   **Solution**: Create the collection first using the frontend or API before querying.

4. **Empty Results**
   - Verify documents are properly ingested in VastData
   - Check that embeddings were generated with the correct model (`nvidia/nv-embedqa-e5-v5`)
   - Ensure metadata includes required fields like `doc_path` and `page_number`

### Debug Mode

Enable debug logging to troubleshoot issues:

```bash
# Add to your environment
export LOGLEVEL=DEBUG

# Restart the RAG server
docker restart rag-server

# Check detailed logs
docker logs rag-server --tail 100
```

### Health Checks

Check service health status:

```bash
curl -X GET "http://localhost:8081/v1/health?check_dependencies=true"
```

For more troubleshooting tips, see the [general troubleshooting guide](troubleshooting.md).

## Architecture Notes

- **Vector Store**: VastData replaces Milvus as the vector database
- **Embeddings**: Uses cloud-hosted `nvidia/nv-embedqa-e5-v5` model
- **Reranking**: Uses cloud-hosted `nvidia/nv-rerankqa-mistral-4b-v3` model  
- **LLM**: Uses cloud-hosted NVIDIA language models
- **Citations**: Adapted to work with VastData's metadata structure (`doc_path`, `page_number`)

## Related Documentation

- [Quickstart Guide](quickstart.md) - General deployment instructions
- [API Reference](api_reference/) - Complete API documentation  
- [Troubleshooting](troubleshooting.md) - Common issues and solutions
- [Change Models](change-model.md) - How to switch between different models