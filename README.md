# TwenyCode Approval System

A flexible, configurable approval workflow package for Laravel applications.

## Table of Contents

- [Introduction](#introduction)
- [Installation](#installation)
- [Configuration](#configuration)
- [Core Concepts](#core-concepts)
  - [Approval Flows](#approval-flows)
  - [Approval Levels](#approval-levels)
  - [Approvers](#approvers)
  - [Approval Requests](#approval-requests)
  - [Approval Actions](#approval-actions)
- [Usage](#usage)
  - [Setting Up Approval Flows](#setting-up-approval-flows)
  - [Making Models Approvable](#making-models-approvable)
  - [Initiating Approval Processes](#initiating-approval-processes)
  - [Processing Approval Actions](#processing-approval-actions)
  - [Checking Approval Status](#checking-approval-status)
- [Blade Components](#blade-components)
- [Notifications](#notifications)
- [Events](#events)
- [Customization](#customization)
- [Advanced Usage](#advanced-usage)
  - [Custom Approval Logic](#custom-approval-logic)
  - [Delegation](#delegation)
  - [Multi-level Approvals](#multi-level-approvals)
- [API Reference](#api-reference)
- [Troubleshooting](#troubleshooting)
- [Contributing](#contributing)
- [License](#license)

## Introduction

The TwenyCode Approval System is a Laravel package that provides a robust and flexible framework for implementing approval workflows in your applications. It supports multi-level approvals, different approval flows for different types of requests, and a comprehensive system of notifications to keep all stakeholders informed.

This documentation will guide you through the installation, configuration, and usage of the Approval System.

## Installation

You can install the package via composer:

```bash
composer require twenycode/approval-system
```

After installation, publish the configuration, migrations, and views:

```bash
php artisan vendor:publish --provider="TwenyCode\ApprovalSystem\ApprovalSystemServiceProvider" --tag="approval-system-config"
php artisan vendor:publish --provider="TwenyCode\ApprovalSystem\ApprovalSystemServiceProvider" --tag="approval-system-migrations"
php artisan vendor:publish --provider="TwenyCode\ApprovalSystem\ApprovalSystemServiceProvider" --tag="approval-system-views"
```

Run the migrations:

```bash
php artisan migrate
```

## Configuration

The package configuration file is located at `config/approval-system.php` after publishing. Here you can configure:

- Default approval flow
- User, employee, and department models
- Notification classes
- Available approval statuses

### Example Configuration

```php
return [
    // Default approval flow code
    'default_flow' => 'general',
    
    // Define the user model that will be used for approvers
    'user_model' => 'App\\Models\\User',
    
    // Define the employee model that will be used for approvers
    'employee_model' => 'App\\Models\\Hra\\Employee',
    
    // Department model for approvers
    'department_model' => 'App\\Models\\Department',
    
    // Define notification classes
    'notifications' => [
        'approval_request' => \TwenyCode\ApprovalSystem\Notifications\ApprovalRequestNotification::class,
    ],
    
    // Available approval statuses
    'statuses' => [
        'draft',
        'pending',
        'returned',
        'delegated',
        'approved', 
        'rejected',
        'canceled'
    ],
];
```

## Core Concepts

### Approval Flows

An approval flow defines a sequence of approval steps that a particular type of request must go through. For example, you might have different approval flows for leave requests, expense reports, and contract approvals.

#### Database Schema

| Field | Type | Description |
|-------|------|-------------|
| id | bigint | Primary key |
| name | string | Name of the approval flow |
| code | string | Unique identifier code |
| descriptions | text | Optional description |
| isActive | boolean | Whether this flow is active |
| created_at | timestamp | Creation timestamp |
| updated_at | timestamp | Update timestamp |
| deleted_at | timestamp | Soft delete timestamp |

### Approval Levels

An approval level represents a single step in an approval flow. Each level has a sequence number that determines its order in the flow.

#### Database Schema

| Field | Type | Description |
|-------|------|-------------|
| id | bigint | Primary key |
| name | string | Name of the approval level |
| approval_flow_id | bigint | Foreign key to approval_flows |
| sequence | integer | Order in the approval flow |
| isActive | boolean | Whether this level is active |
| created_at | timestamp | Creation timestamp |
| updated_at | timestamp | Update timestamp |

### Approvers

Approvers are individuals who are authorized to approve requests at specific approval levels. You can assign different approvers for different departments.

#### Database Schema

| Field | Type | Description |
|-------|------|-------------|
| id | bigint | Primary key |
| approval_level_id | bigint | Foreign key to approval_levels |
| department_id | bigint | Optional foreign key to departments |
| employee_id | bigint | Foreign key to employees |
| isActive | boolean | Whether this approver is active |
| created_at | timestamp | Creation timestamp |
| updated_at | timestamp | Update timestamp |

### Approval Requests

An approval request represents a specific instance of an approval process. It tracks the current status and level of the request.

#### Database Schema

| Field | Type | Description |
|-------|------|-------------|
| id | bigint | Primary key |
| requestable_type | string | Polymorphic relation type |
| requestable_id | bigint | Polymorphic relation ID |
| approval_flow_id | bigint | Foreign key to approval_flows |
| employee_id | bigint | Foreign key to employees |
| current_level_id | bigint | Current approval level (nullable) |
| status | enum | Current status of the request |
| created_at | timestamp | Creation timestamp |
| updated_at | timestamp | Update timestamp |
| deleted_at | timestamp | Soft delete timestamp |

### Approval Actions

Approval actions track each action taken on an approval request, including who took the action and any comments they provided.

#### Database Schema

| Field | Type | Description |
|-------|------|-------------|
| id | bigint | Primary key |
| approval_request_id | bigint | Foreign key to approval_requests |
| approval_level_id | bigint | Foreign key to approval_levels (nullable) |
| employee_id | bigint | Foreign key to employees |
| action | enum | Action taken (approved, rejected, etc.) |
| comments | text | Optional comments |
| created_at | timestamp | Creation timestamp |
| updated_at | timestamp | Update timestamp |

## Usage

### Setting Up Approval Flows

Before you can start using the approval system, you need to set up your approval flows and levels. You can do this through the provided admin UI or programmatically.

#### Admin UI

The package comes with a set of admin views that you can use to manage approval flows, levels, and approvers:

- `/approval-system/approval-flows` - Manage approval flows
- `/approval-system/approval-levels` - Manage approval levels
- `/approval-system/approvers` - Manage approvers

#### Programmatic Setup

You can also set up approval flows programmatically:

```php
use TwenyCode\ApprovalSystem\Models\ApprovalFlow;
use TwenyCode\ApprovalSystem\Models\ApprovalLevel;
use TwenyCode\ApprovalSystem\Models\Approver;

// Create an approval flow
$flow = ApprovalFlow::create([
    'name' => 'Leave Request Approval',
    'code' => 'leave-request',
    'descriptions' => 'Approval flow for leave requests',
    'isActive' => true,
]);

// Create approval levels
$level1 = ApprovalLevel::create([
    'name' => 'Supervisor',
    'approval_flow_id' => $flow->id,
    'sequence' => 1,
    'isActive' => true,
]);

$level2 = ApprovalLevel::create([
    'name' => 'Department Head',
    'approval_flow_id' => $flow->id,
    'sequence' => 2,
    'isActive' => true,
]);

// Assign approvers
Approver::create([
    'approval_level_id' => $level1->id,
    'department_id' => $departmentId,
    'employee_id' => $supervisorId,
    'isActive' => true,
]);

Approver::create([
    'approval_level_id' => $level2->id,
    'department_id' => $departmentId,
    'employee_id' => $deptHeadId,
    'isActive' => true,
]);
```

### Making Models Approvable

To make a model subject to approval, use the `Approvable` trait:

```php
<?php

namespace App\Models;

use Illuminate\Database\Eloquent\Model;
use TwenyCode\ApprovalSystem\Traits\Approvable;

class LeaveRequest extends Model
{
    use Approvable;
    
    // Define the approval flow code for this model
    protected $approvalFlowCode = 'leave-request';
    
    // ... your model code
}
```

The `Approvable` trait adds the following functionality to your model:

- A polymorphic relationship to approval requests
- Methods for initiating and checking approval status
- Events for approval lifecycle

### Initiating Approval Processes

When a new request is created, you need to initiate the approval process:

```php
use TwenyCode\ApprovalSystem\Facades\ApprovalSystem;

// Create a new leave request
$leaveRequest = LeaveRequest::create([
    'employee_id' => $employeeId,
    'start_date' => $startDate,
    'end_date' => $endDate,
    'reason' => $reason,
]);

// Initiate the approval process
$approvalRequest = ApprovalSystem::initializeApproval(
    $leaveRequest,
    $employeeId,
    'leave-request' // This can be omitted if $approvalFlowCode is defined on the model
);
```

Alternatively, if you're using the `Approvable` trait, you can use the `initiateApproval` method:

```php
$leaveRequest->initiateApproval($employeeId);
```

### Processing Approval Actions

When an approver takes action on a request, you process it using the ApprovalService:

```php
use TwenyCode\ApprovalSystem\Facades\ApprovalSystem;

// Process an approval action
ApprovalSystem::processAction([
    'action_taken' => 'approved', // Can be 'approved', 'rejected', 'returned', 'canceled'
    'approver_id' => $approverId,
    'comments' => 'Approved, enjoy your vacation!'
], $leaveRequest);
```

Or, using the `Approvable` trait:

```php
$leaveRequest->processApprovalAction(
    'approved',
    $approverId,
    'Approved, enjoy your vacation!'
);
```

### Checking Approval Status

You can check the status of an approval request:

```php
// Using the facade
$status = ApprovalSystem::getApprovalStatus($leaveRequest);

// Using the trait
$status = $leaveRequest->approvalStatus();

// Check if approved
if ($leaveRequest->isApproved()) {
    // Do something for approved requests
}

// Check if pending
if ($leaveRequest->isPending()) {
    // Do something for pending requests
}

// Check if rejected
if ($leaveRequest->isRejected()) {
    // Do something for rejected requests
}
```

## Blade Components

The package provides several Blade components to help you integrate with your UI:

### Approval Flow Component

```blade
<x-approval-system::approval-flow-component 
    name="approval_flow_id" 
    :value="$currentValue" 
    id="approval-flow-select"
/>
```

### Approval Level Component

```blade
<x-approval-system::approval-level-component 
    name="approval_level_id" 
    :value="$currentValue" 
    id="approval-level-select"
/>
```

## Notifications

The package includes a notification system that keeps all stakeholders informed about the approval process. When an approval request status changes, notifications are sent to:

- The requestor
- The current approvers
- Previous approvers (for certain actions)

The default notification is `ApprovalRequestNotification`, which sends mail and database notifications. You can customize this by extending the class or creating your own notification class and updating the configuration.

### Customizing Notifications

Create your custom notification class:

```php
<?php

namespace App\Notifications;

use TwenyCode\ApprovalSystem\Notifications\ApprovalRequestNotification as BaseNotification;

class CustomApprovalNotification extends BaseNotification
{
    // Override methods as needed
    
    public function toMail($notifiable)
    {
        // Custom mail notification
    }
    
    public function toArray($notifiable)
    {
        // Custom database notification
    }
}
```

Update your configuration:

```php
// config/approval-system.php
'notifications' => [
    'approval_request' => \App\Notifications\CustomApprovalNotification::class,
],
```

## Events

The package dispatches several events that you can listen for:

- `ApprovalRequested` - When a new approval request is created
- `ApprovalActionPerformed` - When an action is taken on an approval request
- `ApprovalCompleted` - When an approval process is completed (approved or rejected)

### Listening for Events

Register your event listeners in your `EventServiceProvider`:

```php
protected $listen = [
    \TwenyCode\ApprovalSystem\Events\ApprovalRequested::class => [
        \App\Listeners\HandleNewApprovalRequest::class,
    ],
    \TwenyCode\ApprovalSystem\Events\ApprovalActionPerformed::class => [
        \App\Listeners\HandleApprovalAction::class,
    ],
    \TwenyCode\ApprovalSystem\Events\ApprovalCompleted::class => [
        \App\Listeners\HandleApprovalCompletion::class,
    ],
];
```

## Customization

### Custom Approval Logic

You can extend the `ApprovalService` to implement custom approval logic:

```php
<?php

namespace App\Services;

use TwenyCode\ApprovalSystem\Services\ApprovalService as BaseService;

class CustomApprovalService extends BaseService
{
    // Override methods to implement custom logic
    
    public function processApproved(int $requestId, int $currentLevelId)
    {
        // Custom approved logic
        
        // Maybe skip a level based on some condition
        if ($someCondition) {
            $nextLevel = $this->approvalLevelRepository->getNextLevel($currentLevelId);
            if ($nextLevel) {
                $nextLevel = $this->approvalLevelRepository->getNextLevel($nextLevel->id);
            }
            
            if ($nextLevel) {
                return $this->repository->update($requestId, [
                    'current_level_id' => $nextLevel->id,
                    'status' => 'pending',
                ]);
            } else {
                return $this->repository->update($requestId, [
                    'status' => 'approved',
                    'current_level_id' => null,
                ]);
            }
        }
        
        // Otherwise, use the default logic
        return parent::processApproved($requestId, $currentLevelId);
    }
}
```

Register your custom service in your app's service provider:

```php
$this->app->bind(
    \TwenyCode\ApprovalSystem\Services\ApprovalServiceInterface::class,
    \App\Services\CustomApprovalService::class
);
```

### View Customization

After publishing the views, you can find them in `resources/views/vendor/approval-system`. Modify them as needed to match your application's design.

## Advanced Usage

### Custom Approval Logic

You can implement custom approval logic based on various factors such as request type, amount, department, etc.:

```php
// In your custom approval service
public function processAction(array $data, $requestType)
{
    // Special handling for high-value requests
    if ($requestType instanceof ExpenseReport && $requestType->amount > 10000) {
        // Add an additional approval level for high-value expenses
        // ...
    }
    
    return parent::processAction($data, $requestType);
}
```

### Delegation

You can implement delegation to allow approvers to delegate their approval authority to others:

```php
// In your custom approval service
public function delegateApproval(int $requestId, int $fromApproverId, int $toApproverId, string $reason)
{
    // Record the delegation
    $this->recordApprovalAction(
        $this->repository->findById($requestId),
        'delegated',
        $reason,
        $fromApproverId
    );
    
    // Create a new approver or update existing one
    // ...
    
    return $this->repository->update($requestId, [
        'status' => 'delegated',
    ]);
}
```

### Multi-level Approvals

The package supports multi-level approvals out of the box. You can configure as many approval levels as needed for each flow.

## API Reference

### ApprovalSystem Facade

- `initializeApproval($requestable, int $employeeId, string $flowCode = null)`
- `processAction(array $data, $requestable)`
- `isApproverForRequest(int $requestId, int $employeeId)`
- `recordApprovalAction($approvalRequest, string $action, string $comments, int $employeeId = null)`
- `processApproved(int $requestId, int $currentLevelId)`
- `notifyApprovers($approvalRequest)`
- `notifyCancellation($approvalRequest, $canceledBy)`
- `notifyRequestor($approvalRequest, string $action, $approvalAction = null)`

### Approvable Trait

- `approval_request()` - Relationship to ApprovalRequest
- `initiateApproval(int $employeeId, string $flowCode = null)`
- `processApprovalAction(string $action, int $approverId, string $comments = null)`
- `approvalStatus()`
- `isApproved()`
- `isPending()`
- `isRejected()`
- `isReturned()`
- `isCanceled()`
- `getCurrentApprovalLevel()`
- `getApprovalHistory()`

## Troubleshooting

### Common Issues

1. **Missing approvers**: Ensure that approvers are set up for each level in your approval flow.
2. **Incorrect approval flow**: Check that the correct approval flow code is being used for each model.
3. **Notification issues**: Make sure your email configuration is correct and notifications are properly configured.

### Logging

The package logs important events and errors. Check your Laravel logs for more information when troubleshooting:

```bash
tail -f storage/logs/laravel.log
```

## Contributing

Contributions are welcome! Please feel free to submit a Pull Request.

## License

The TwenyCode Approval System is open-sourced software licensed under the [MIT license](LICENSE.md).
