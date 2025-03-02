# Model Management Service Detailed Design

## Overview

This document outlines the detailed design for the Model Management Service, including its domain-driven design, APIs, and interactions with other services.

## Domain-Driven Design

*   **Aggregate Root:** Model
    *   Fields:
        *   `id`: UUID (Unique identifier)
        *   `name`: string (Model name)
        *   `description`: string (Model description)
        *   `owner_id`: UUID (ID of the user who owns the model)
        *   `created_at`: timestamp (Creation timestamp)
        *   `updated_at`: timestamp (Last updated timestamp)
        *   `versions`: List of ModelVersion entities
        *   `metadata`: ModelMetadata entity
*   **Entities:**
    *   `ModelVersion`:
        *   Fields:
            *   `version`: integer (Version number)
            *   `model_file_url`: string (URL of the model file in cloud storage)
            *   `created_at`: timestamp (Creation timestamp)
    *   `ModelMetadata`:
        *   Fields:
            *   `input_schema`: JSON (Input schema of the model)
            *   `output_schema`: JSON (Output schema of the model)
            *   `token_context_limit`: integer (Token context limit for the model)
*   **Value Objects:**
    *   `InputSchema`: JSON schema for model input.
    *   `OutputSchema`: JSON schema for model output.

## Interfaces

```typescript
interface IModel {
    id: string;
    name: string;
    description: string;
    owner_id: string;
    created_at: Date;
    updated_at: Date;
    versions: IModelVersion[];
    metadata: IModelMetadata;
}

interface IModelVersion {
    version: number;
    model_file_url: string;
    created_at: Date;
}

interface IModelMetadata {
    input_schema: any;
    output_schema: any;
    token_context_limit: number;
}

interface IModelRepository {
    create(model: IModel): Promise<string>;
    get(model_id: string): Promise<IModel | null>;
    update(model: IModel): Promise<void>;
    delete(model_id: string): Promise<void>;
    list(): Promise<IModel[]>;
    getVersion(model_id: string, version: number): Promise<IModelVersion | null>;
}
```

## APIs Provided (REST)

*   `POST /models`: Register a new model.
    *   Request Body: Model metadata (name, description, input schema, output schema, token context limit), model file.
    *   Response: Model ID.
    *   Permissions: `admin`, `model.create`
*   `GET /models/{model_id}`: Retrieve model information.
    *   Response: Model metadata.
    *   Permissions: `admin`, `model.read`, `model.read.self` (if `owner_id` matches user ID)
*   `PUT /models/{model_id}`: Update model metadata.
    *   Request Body: Model metadata (name, description, input schema, output schema, token context limit).
    *   Response: Success/Failure.
    *   Permissions: `admin`, `model.update`, `model.update.self` (if `owner_id` matches user ID)
*   `DELETE /models/{model_id}`: Delete a model.
    *   Response: Success/Failure.
    *   Permissions: `admin`, `model.delete`, `model.delete.self` (if `owner_id` matches user ID)
*   `GET /models`: List all models.
    *   Response: List of model metadata.
    *   Permissions: `admin`, `model.list`
*   `GET /models/{model_id}/versions/{version}`: Retrieve a specific version of a model.
    *   Response: Model file URL.
    *   Permissions: `admin`, `model.read`, `model.read.self` (if `owner_id` matches user ID)

## APIs Required

*   None

## Role-Based Access Control (RBAC)

*   **Roles:**
    *   `admin`: Full access to all resources.
    *   `model.create`: Permission to create new models.
    *   `model.read`: Permission to read any model.
    *   `model.read.self`: Permission to read models owned by the user.
    *   `model.update`: Permission to update any model.
    *   `model.update.self`: Permission to update models owned by the user.
    *   `model.delete`: Permission to delete any model.
    *   `model.delete.self`: Permission to delete models owned by the user.

## Storage Abstraction (Dapr)

The Model Management Service will use Dapr to abstract the underlying database. This allows us to easily switch between different database implementations without modifying the service code.

Dapr components:

*   State Store: Used to store model metadata.