# Resource Management Service Detailed Design

## Overview

This document outlines the detailed design for the Resource Management Service, including its domain-driven design, APIs, and interactions with other services.

## Domain-Driven Design

*   **Aggregate Root:** ResourcePool
    *   Fields:
        *   `id`: UUID (Unique identifier)
        *   `name`: string (Name of the resource pool)
        *   `resource_type`: string (Type of resource, e.g., "GPU", "CPU", "Memory")
        *   `total_capacity`: integer (Total capacity of the resource pool)
        *   `available_capacity`: integer (Available capacity of the resource pool)
        *   `location`: string (Location of the resources, e.g., "us-east-1", "eu-west-1")
        *   `created_at`: timestamp (Creation timestamp)
        *   `updated_at`: timestamp (Last updated timestamp)
*   **Entities:**
    *   `ResourceAllocation`
        *   Fields:
            *   `id`: UUID (Unique identifier)
            *   `resource_pool_id`: UUID (ID of the ResourcePool)
            *   `service_id`: UUID (ID of the ModelService)
            *   `allocated_capacity`: integer (Capacity allocated to the service)
            *   `created_at`: timestamp (Creation timestamp)
            *   `updated_at`: timestamp (Last updated timestamp)
*   **Value Objects:**
    *   None

## Interfaces

```go
type ResourcePool struct {
	ID              string    `json:"id"`
	Name            string    `json:"name"`
	ResourceType    string    `json:"resource_type"`
	TotalCapacity   int       `json:"total_capacity"`
	AvailableCapacity int       `json:"available_capacity"`
	Location        string    `json:"location"`
	CreatedAt       time.Time `json:"created_at"`
	UpdatedAt        time.Time `json:"updated_at"`
}

type ResourceAllocation struct {
	ID              string    `json:"id"`
	ResourcePoolID  string    `json:"resource_pool_id"`
	ServiceID       string    `json:"service_id"`
	AllocatedCapacity int       `json:"allocated_capacity"`
	CreatedAt       time.Time `json:"created_at"`
	UpdatedAt        time.Time `json:"updated_at"`
}

type ResourceManagement interface {
	CreateResourcePool(pool *ResourcePool) (string, error)
	GetResourcePool(poolID string) (*ResourcePool, error)
	UpdateResourcePool(pool *ResourcePool) error
	DeleteResourcePool(poolID string) error
	ListResourcePools() ([]*ResourcePool, error)
	AllocateResources(poolID string, serviceID string, capacity int) (string, error)
	ReleaseResources(allocationID string) error
	GetResourceAllocation(allocationID string) (*ResourceAllocation, error)
	ListResourceAllocations(poolID string) ([]*ResourceAllocation, error)
}

type ResourceManagementRepository interface {
	CreateResourcePool(pool *ResourcePool) (string, error)
	GetResourcePool(poolID string) (*ResourcePool, error)
	UpdateResourcePool(pool *ResourcePool) error
	DeleteResourcePool(poolID string) error
	ListResourcePools() ([]*ResourcePool, error)
	CreateResourceAllocation(allocation *ResourceAllocation) (string, error)
	GetResourceAllocation(allocationID string) (*ResourceAllocation, error)
	DeleteResourceAllocation(allocationID string) error
	ListResourceAllocations(poolID string) ([]*ResourceAllocation, error)
}

type ResourceManagementService struct {
	repo ResourceManagementRepository
}

func NewResourceManagementService(repo ResourceManagementRepository) *ResourceManagementService {
	return &ResourceManagementService{repo: repo}
}

func (s *ResourceManagementService) CreateResourcePool(pool *ResourcePool) (string, error) {
	return s.repo.CreateResourcePool(pool)
}

func (s *ResourceManagementService) GetResourcePool(poolID string) (*ResourcePool, error) {
	return s.repo.GetResourcePool(poolID)
}

func (s *ResourceManagementService) UpdateResourcePool(pool *ResourcePool) error {
	return s.repo.UpdateResourcePool(pool)
}

func (s *ResourceManagementService) DeleteResourcePool(poolID string) error {
	return s.repo.DeleteResourcePool(poolID)
}

func (s *ResourceManagementService) ListResourcePools() ([]*ResourcePool, error) {
	return s.repo.ListResourcePools()
}

func (s *ResourceManagementService) AllocateResources(poolID string, serviceID string, capacity int) (string, error) {
    // 1. Check if there is enough capacity in the pool
    // 2. Create a new ResourceAllocation
    // 3. Update the ResourcePool's available capacity
	return "", nil
}

func (s *ResourceManagementService) ReleaseResources(allocationID string) error {
    // 1. Get the ResourceAllocation
    // 2. Update the ResourcePool's available capacity
    // 3. Delete the ResourceAllocation
	return nil
}

func (s *ResourceManagementService) GetResourceAllocation(allocationID string) (*ResourceAllocation, error) {
	return s.repo.GetResourceAllocation(allocationID)
}

func (s *ResourceManagementService) ListResourceAllocations(poolID string) ([]*ResourceAllocation, error) {
	return s.repo.ListResourceAllocations(poolID)
}
```

## APIs Provided (REST)

*   `POST /resource_pools`: Create a new resource pool.
    *   Request Body: Resource pool metadata (name, resource type, total capacity, location).
    *   Response: Resource pool ID.
    *   Permissions: `resource_pool:create`
*   `GET /resource_pools/{pool_id}`: Retrieve resource pool information.
    *   Response: Resource pool metadata.
    *   Permissions: `resource_pool:get`
*   `PUT /resource_pools/{pool_id}`: Update resource pool.
    *   Request Body: Resource pool metadata (name, resource type, total capacity, location).
    *   Response: Success/Failure.
    *   Permissions: `resource_pool:update`
*   `DELETE /resource_pools/{pool_id}`: Delete a resource pool.
    *   Response: Success/Failure.
    *   Permissions: `resource_pool:delete`
*   `GET /resource_pools`: List all resource pools.
    *   Response: List of resource pool metadata.
    *   Permissions: `resource_pool:list`
*   `POST /resource_pools/{pool_id}/allocations`: Allocate resources to a service.
    *   Request Body: Service ID, capacity.
    *   Response: Resource allocation ID.
    *   Permissions: `resource_allocation:create`
*   `DELETE /resource_pools/{pool_id}/allocations/{allocation_id}`: Release resources from a service.
    *   Response: Success/Failure.
    *   Permissions: `resource_allocation:delete`
*   `GET /resource_pools/{pool_id}/allocations/{allocation_id}`: Retrieve resource allocation information.
    *   Response: Resource allocation metadata.
    *   Permissions: `resource_allocation:get`
*   `GET /resource_pools/{pool_id}/allocations`: List all resource allocations for a given pool.
    *   Response: List of resource allocation metadata.
    *   Permissions: `resource_allocation:list`

## APIs Required

*   None

## Role-Based Access Control (RBAC)

The Resource Management Service will use Casbin for RBAC. The following is the Casbin DSL model:

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
p, resource-pool-owner, resource_pools/{pool_id}, resource_pool:get
g, alice, resource-pool-owner
```

## Storage

The Resource Management Service will use PostgreSQL as the storage for state data (resource pool and allocation metadata).

## Storage Abstraction (Dapr)

The Resource Management Service will use Dapr to interact with PostgreSQL.

Dapr components:

*   State Store: Used to store resource pool and allocation metadata (PostgreSQL).