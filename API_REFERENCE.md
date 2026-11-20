# Open POS API Reference

A practical, hands-on guide for using and testing the Open POS REST API with Postman. This document covers authentication, all core endpoints, the new **Task module** (CRUD + filtering + pagination), and a quick developer guide on how to add new modules to this project.

---

## 1. Quick Setup & Postman Variables

Set up two collection-level variables in Postman so your URLs stay clean:

| Variable | Value | What it's for |
| :--- | :--- | :--- |
| `base_url` | `http://127.0.0.1:5000` | Root host and Flask-Admin routes (`/admin/...`) |
| `auth_base_url` | `http://127.0.0.1:5000/api/v1` | Versioned REST API endpoints (`/api/v1/...`) |

---

## 2. Authentication: How to Get & Attach Your Token

Most REST endpoints require you to be logged in. Here is the exact workflow:

1. Send a `POST` request to **`{{auth_base_url}}/login/`** with your email and password.
2. The response returns an `authentication_token`:
   ```json
   {
     "id": "552c97d8-1a93-420f-b56f-8d5a5045d6dc",
     "authentication_token": "WyI1NTJjOTdkOC...=="
   }
   ```
3. Copy that token.
4. For every protected call, open the **Headers** tab in Postman and add:
   - **Key**: `Authorization`
   - **Value**: `<your_authentication_token>`

> **Pro-Tip (Auto-save token in Postman):**  
> In your `login-user` request, click the **Scripts > Post-response (Tests)** tab and paste this snippet:
> ```javascript
> const res = pm.response.json();
> if (res.authentication_token) {
>     pm.collectionVariables.set("auth_token", res.authentication_token);
> }
> ```
> Now you can simply put `{{auth_token}}` in the `Authorization` header of all protected requests.

---

## 3. Endpoints Overview

| Request Name | Method | URL Path | Auth Required? | Payload |
| :--- | :--- | :--- | :--- | :--- |
| **register-user** | `POST` | `{{auth_base_url}}/register/` | No | JSON |
| **login-user** | `POST` | `{{auth_base_url}}/login/` | No | JSON |
| **get-all-user** | `GET` | `{{auth_base_url}}/user` | Yes (`admin`/`owner`) | None |
| **get-role** | `GET` | `{{auth_base_url}}/role` | Yes | None |
| **create-user (admin)**| `POST` | `{{base_url}}/admin/user/new/` | Yes (Admin session) | `form-data` |
| **create-role (admin)**| `POST` | `{{base_url}}/admin/role/new/` | Yes (Admin session) | `form-data` |
| **get-products (admin)**| `GET` | `{{base_url}}/admin/product` | Yes (Admin) | None (HTML) |
| **get-retail-shop (admin)**| `GET` | `{{base_url}}/admin/retailshop/` | Yes (Admin) | None (HTML) |
| **list-tasks** | `GET` | `{{auth_base_url}}/tasks/` | Yes (Any logged-in user) | None |
| **create-task** | `POST` | `{{auth_base_url}}/tasks/` | Yes (Any logged-in user) | JSON |
| **get-task-by-id** | `GET` | `{{auth_base_url}}/task/<id>/` | Yes (Any logged-in user) | None |
| **update-task** | `PUT` | `{{auth_base_url}}/task/<id>/` | Yes (Any logged-in user) | JSON |
| **delete-task** | `DELETE`| `{{auth_base_url}}/task/<id>/` | Yes (Any logged-in user) | None |

---

## 4. Auth & User Endpoints

### 1. Register User (`register-user`)
Creates a new user account through Flask-Security.
- **Method**: `POST`
- **URL**: `{{auth_base_url}}/register/`
- **Headers**: `Content-Type: application/json`
- **Body (`raw JSON`)**:
  ```json
  {
    "email": "ali123456@gmail.com",
    "password": "12345678",
    "name": "Arashad Ali",
    "mobile_number": "9878878787"
  }
  ```
- **Success (`200 OK`)**:
  ```json
  {
    "response": {
      "user": {
        "id": "c1f7a014-bf4c-473d-82d2-5a9e14e1a0b3",
        "email": "ali123456@gmail.com"
      }
    }
  }
  ```
- **Errors**: `400 Bad Request` (missing field / invalid email), `409 Conflict` (duplicate email or mobile).

---

### 2. Login User (`login-user`)
Authenticates credentials and returns the authorization token.
- **Method**: `POST`
- **URL**: `{{auth_base_url}}/login/`
- **Headers**: `Content-Type: application/json`
- **Body (`raw JSON`)**:
  ```json
  {
    "email": "aliarshad0786@gmail.com",
    "password": "12345678"
  }
  ```
- **Success (`200 OK`)**:
  ```json
  {
    "id": "552c97d8-1a93-420f-b56f-8d5a5045d6dc",
    "authentication_token": "WyI1NTJjOTdkOC0xYTkzLTQyMGYtYjU2Zi04ZDVhNTA0NWQ2ZGMi...=="
  }
  ```
- **Errors**: `403 Forbidden` (`{"meta": {"code": 403}}`) for bad credentials.

---

### 3. Get All Users (`get-all-user`)
Lists users accessible to the current logged-in user's retail brand.
- **Method**: `GET`
- **URL**: `{{auth_base_url}}/user`
- **Headers**: `Authorization: {{auth_token}}`
- **Success (`200 OK`)**:
  ```json
  {
    "data": [
      {
        "id": "552c97d8-1a93-420f-b56f-8d5a5045d6dc",
        "email": "aliarshad0786@gmail.com",
        "name": "Admin User",
        "mobile_number": "9999999999",
        "active": true
      }
    ],
    "success": true,
    "total": 1
  }
  ```
- **Errors**: `401 Unauthorized` (missing/invalid token), `403 Forbidden` (non-admin role).

---

### 4. Get Roles (`get-role`)
Lists system roles and their assigned permissions.
- **Method**: `GET`
- **URL**: `{{auth_base_url}}/role`
- **Headers**: `Authorization: {{auth_token}}`
- **Success (`200 OK`)**:
  ```json
  {
    "data": [
      {
        "id": "e932bce7-d2c9-4b67-938c-3088a8767980",
        "name": "admin",
        "permissions": []
      }
    ],
    "success": true,
    "total": 1
  }
  ```

---

### 5. Admin Panel Endpoints (Flask-Admin Forms & Views)

These endpoints run on the Flask-Admin web engine.

- **Create User (`POST {{base_url}}/admin/user/new/`)**:
  - Requires **`form-data`** body (NOT JSON), with keys: `name`, `email`, `mobile_number`, `password`, `active` (`y`).
  - Response: `302 Found` redirect on success.
- **Create Role (`POST {{base_url}}/admin/role/new/`)**:
  - Requires **`form-data`** with key `name` (e.g. `staff`).
  - Response: `302 Found` redirect on success.
- **Get Products View (`GET {{base_url}}/admin/product`)**:
  - Returns the Flask-Admin HTML dashboard table for products.
- **Get Retail Shop View (`GET {{base_url}}/admin/retailshop/`)**:
  - Returns the Flask-Admin HTML dashboard table for shops.

---

## 5. Task Module Endpoints (CRUD)

The Task module uses `CommonResource`, meaning **any authenticated user** (admin, owner, staff, or regular user) can call these endpoints as long as a valid token is provided in the `Authorization` header.

Allowed task statuses: `pending`, `in_progress`, `done`.

---

### 1. Create a Task
Creates a new task assigned to a user.
- **Method**: `POST`
- **URL**: `{{auth_base_url}}/tasks/`
- **Headers**:
  - `Authorization`: `{{auth_token}}`
  - `Content-Type`: `application/json`
#### Request Body (`raw JSON`)
```json
{
  "title": "Prepare End of Month Sales Report",
  "description": "Compile report for retail brand shops",
  "status": "pending",
  "due_date": "2026-11-30T18:00:00",
  "user_id": "552c97d8-1a93-420f-b56f-8d5a5045d6dc"
}
```
#### Success Response (`201 Created`)
```json
{
  "data": [
    {
      "id": "7b822d87-34f1-4db4-bb14-9b841a1a5209",
      "title": "Prepare End of Month Sales Report",
      "description": "Compile report for retail brand shops",
      "status": "pending",
      "due_date": "2026-11-30T18:00:00",
      "user_id": "552c97d8-1a93-420f-b56f-8d5a5045d6dc",
      "created_on": "2026-11-20T12:15:30.123456"
    }
  ],
  "message": "Resource added successfully",
  "success": true
}
```
#### Error Codes
| Status | When it happens | Sample response |
| :--- | :--- | :--- |
| `400 Bad Request` | Missing `title` or `user_id` | `{"error": true, "message": "{'title': ['Missing data for required field.']}"}` |
| `400 Bad Request` | Invalid `status` value | `{"error": true, "message": "{'status': ['status must be one of: pending, in_progress, done']}"}` |
| `400 Bad Request` | Invalid datetime string | `{"error": true, "message": "{'due_date': ['Not a valid datetime.']}"}` |
| `401 Unauthorized`| Missing / invalid `Authorization` header | Standard Flask-Security redirect or unauthorized response |
---
### 2. List Tasks (With Filtering & Pagination)
Retrieves a paginated list of tasks. Supports filtering by status, title, and user.
- **Method**: `GET`
- **URL**: `{{auth_base_url}}/tasks/`
- **Headers**:
  - `Authorization`: `{{auth_token}}`
#### Query Parameters (All Optional)
- `__page`: Page number (default: `1`)
- `__limit`: Results per page (default: `50`, max: `100`)
- `__order_by`: Sort field (`created_on`, `-created_on`, `due_date`, `title`, `status`)
- `__status__equal`: Exact status match (`pending`, `in_progress`, `done`)
- `__title__contains`: Partial title search (e.g. `Report`)
- `__user_id__equal`: Filter by assigned user UUID
#### Example Request URL
```
{{auth_base_url}}/tasks/?__page=1&__limit=10&__status__equal=pending&__order_by=-created_on
```
#### Success Response (`200 OK`)
```json
{
  "data": [
    {
      "id": "7b822d87-34f1-4db4-bb14-9b841a1a5209",
      "title": "Prepare End of Month Sales Report",
      "description": "Compile report for retail brand shops",
      "status": "pending",
      "due_date": "2026-11-30T18:00:00",
      "user_id": "552c97d8-1a93-420f-b56f-8d5a5045d6dc",
      "created_on": "2026-11-20T12:15:30.123456"
    }
  ],
  "success": true,
  "total": 1
}
```

#### Error Codes
| Status | Cause |
| :--- | :--- |
| `401 Unauthorized` | Missing / invalid token |
| `404 Not Found` | No tasks match the query filters |

---

### 3. Get Single Task by ID
Retrieves details of one specific task.

- **Method**: `GET`
- **URL**: `{{auth_base_url}}/task/<task_id>/`
- **Headers**:
  - `Authorization`: `{{auth_token}}`

#### Success Response (`200 OK`)
```json
{
  "id": "7b822d87-34f1-4db4-bb14-9b841a1a5209",
  "title": "Prepare End of Month Sales Report",
  "description": "Compile report for retail brand shops",
  "status": "pending",
  "due_date": "2026-11-30T18:00:00",
  "user_id": "552c97d8-1a93-420f-b56f-8d5a5045d6dc",
  "created_on": "2026-11-20T12:15:30.123456"
}
```

#### Error Codes
| Status | Cause |
| :--- | :--- |
| `401 Unauthorized` | Missing or invalid auth token |
| `404 Not Found` | No task exists with the given UUID (`{"error": true, "message": "Resource not found"}`) |

---

### 4. Update Task (Partial Update)
Updates one or more fields of an existing task (e.g., advancing status or changing due date).

- **Method**: `PUT`
- **URL**: `{{auth_base_url}}/task/<task_id>/`
- **Headers**:
  - `Authorization`: `{{auth_token}}`
  - `Content-Type`: `application/json`

#### Request Body (`raw JSON`)
Send only the fields you want to change:
```json
{
  "status": "in_progress",
  "description": "Report compilation started, numbers verified"
}
```

#### Success Response (`200 OK`)
```json
{
  "data": {
    "id": "7b822d87-34f1-4db4-bb14-9b841a1a5209",
    "title": "Prepare End of Month Sales Report",
    "description": "Report compilation started, numbers verified",
    "status": "in_progress",
    "due_date": "2026-11-30T18:00:00",
    "user_id": "552c97d8-1a93-420f-b56f-8d5a5045d6dc",
    "created_on": "2026-11-20T12:15:30.123456"
  },
  "message": "obj updated successfully",
  "success": true
}
```

#### Error Codes
| Status | Cause |
| :--- | :--- |
| `400 Bad Request` | Invalid status or bad date format |
| `401 Unauthorized` | Missing / invalid token |
| `404 Not Found` | Task ID does not exist |

---

### 5. Delete Task
Permanently deletes a task.

- **Method**: `DELETE`
- **URL**: `{{auth_base_url}}/task/<task_id>/`
- **Headers**:
  - `Authorization`: `{{auth_token}}`

#### Success Response (`204 No Content`)
- Empty response body with HTTP status `204`.

#### Error Codes
| Status | Cause |
| :--- | :--- |
| `401 Unauthorized` | Missing / invalid token |
| `404 Not Found` | Task ID does not exist |

---