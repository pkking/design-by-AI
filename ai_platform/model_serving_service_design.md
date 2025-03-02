# Model Serving Service Detailed Design

## Overview

This document outlines the detailed design for the Model Serving Service, including its domain-driven design, APIs, and interactions with other services.

## Domain-Driven Design

*   **Aggregate Root:** ModelService (Management Object)
    *   Fields:
        *   `id`: UUID (Unique identifier)
        *   `model_id`: UUID (ID of the model being served)
        *   `endpoint`: string (API endpoint for the service)
        *   `resource_requirements`: JSON (Resource requirements for the service, e.g., GPU, memory, CPU)
        *   `created_at`: timestamp (Creation timestamp)
        *   `updated_at`: timestamp (Last updated timestamp)
*   **Entities:**
    *   `ModelServiceInstance` (Instance Object)
        *   Fields:
            *   `id`: UUID (Unique identifier)
            *   `model_service_id`: UUID (ID of the ModelService)
            *   `status`: string (Status of the service instance: "creating", "running", "stopped", "error")
            *   `start_time`: timestamp (Start time of the service instance)
            *   `run_time`: duration (Run time of the service instance)
            *   `processed_requests`: integer (Number of requests processed by the service instance)
            *   `node`: string (Node the service instance is running on)
            *   `created_at`: timestamp (Creation timestamp)
            *   `updated_at`: timestamp (Last updated timestamp)
*   **Value Objects:**
    *   `ResourceRequirements`: Describes the resource requirements for a model service.

## Interfaces

```go
type ModelService struct {
 ID               string    `json:"id"`
 ModelID          string    `json:"model_id"`
 Endpoint         string    `json:"endpoint"`
 ResourceRequirements interface{} `json:"resource_requirements"`
 CreatedAt        time.Time `json:"created_at"`
 UpdatedAt        time.Time `json:"updated_at"`
}

type ModelServiceInstance struct {
 ID              string    `json:"id"`
 ModelServiceID  string    `json:"model_service_id"`
 Status          string    `json:"status"`
 StartTime       time.Time `json:"start_time"`
 RunTime         string    `json:"run_time"`
 ProcessedRequests int       `json:"processed_requests"`
 Node            string    `json:"node"`
 CreatedAt       time.Time `json:"created_at"`
 UpdatedAt       time.Time `json:"updated_at"`
}

type ModelServing interface {
 CreateModelService(service *ModelService) (string, error)
 GetModelService(serviceID string) (*ModelService, error)
 UpdateModelService(service *ModelService) error
 DeleteModelService(serviceID string) error
 ListModelServices() ([]*ModelService, error)
 CreateModelServiceInstance(service *ModelService, instance *ModelServiceInstance) (string, error)
}

type ModelServingRepository interface {
 CreateModelService(service *ModelService) (string, error)
 GetModelService(serviceID string) (*ModelService, error)
 UpdateModelService(service *ModelService) error
 DeleteModelService(serviceID string) error
 ListModelServices() ([]*ModelService, error)
}

type ModelServingService struct {
 repo ModelServingRepository
}

func NewModelServingService(repo ModelServingRepository) *ModelServingService {
 return &ModelServingService{repo: repo}
}

func (s *ModelServingService) CreateModelService(service *ModelService) (string, error) {
 return s.repo.CreateModelService(service)
}

func (s *ModelServingService) GetModelService(serviceID string) (*ModelService, error) {
 return s.repo.GetModelService(serviceID)
}

func (s *ModelServingService) UpdateModelService(service *ModelService) error {
 return s.repo.UpdateModelService(service)
}

func (s *ModelServingService) DeleteModelService(serviceID string) error {
 return s.repo.DeleteModelService(serviceID)
}

func (s *ModelServingService) ListModelServices() ([]*ModelService, error) {
 return s.repo.ListModelServices()
}

func (s *ModelServingService) CreateModelServiceInstance(service *ModelService, instance *ModelServiceInstance) (string, error) {
 // 1. Create the ModelServiceInstance in the database
 //instanceID, err := s.repo.CreateModelServiceInstance(instance)
 //if err != nil {
 // return "", err
 //}

 // 2. Call KubeRay API to create vLLM service
 // ...

 return "", nil
}

## APIs Provided (REST)

*   `POST /model_services`: Create a new model service.
    *   Request Body: Model ID, version, resource requirements.
    *   Response: Model service instance ID.
    *   Permissions: `model_service:create`
*   `GET /model_services/{instance_id}`: Retrieve model service instance information.
    *   Response: Model service instance metadata.
    *   Permissions: `model_service:get`
*   `PUT /model_services/{instance_id}`: Update model service instance.
    *   Request Body: Status, resource requirements.
    *   Response: Success/Failure.
    *   Permissions: `model_service:update`
*   `DELETE /model_services/{instance_id}`: Delete a model service instance.
    *   Response: Success/Failure.
    *   Permissions: `model_service:delete`
*   `GET /model_services`: List all model service instances.
    *   Response: List of model service instance metadata.
    *   Permissions: `model_service:list`

## APIs Required

*   **Model Management Service:**
    *   `GET /models/{model_id}/versions/{version}`: Retrieve model file URL.
*   **Resource Management Service:**
    *   Allocate resources.
    *   Release resources.

## Role-Based Access Control (RBAC)

The Model Serving Service will use Casbin for RBAC. The following is the Casbin DSL model:

```casbin
[request_definition]
r = sub, obj, act

[policy_definition]
p = sub, obj, act

[role_definition]
g = _, _

[policy_effect]
e = some(where (p.eft == allow))

[matchers]
m = g(r.sub, p.sub) && r.obj == p.obj && r.act == p.act
```

Example policies:

```casbin
p, admin, *, *  # Admin has full access
p, model-service-owner, model_services/{instance_id}, model_service:get
g, alice, model-service-owner
```

## Storage

The Model Serving Service will use PostgreSQL as the storage for state data (model service instance metadata).

## Model Serving

The Model Serving Service will use a containerized model server (e.g., TorchServe, TensorFlow Serving, Triton Inference Server) to host the models.

When a new model service instance is created, the Model Serving Service will:

1.  Retrieve the model file URL from the Model Management Service.
2.  Allocate resources from the Resource Management Service.
3.  Deploy the model to the containerized model server.
4.  Update the status of the model service instance to "running".

## Storage Abstraction (Dapr)

The Model Serving Service will use Dapr to interact with PostgreSQL.

Dapr components:

*   State Store: Used to store model service instance metadata (PostgreSQL).