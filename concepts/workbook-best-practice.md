---
title: "Best practices for Excel APIs in Microsoft Graph"
description: "List best practices and examples for Excel APIs in Microsoft Graph"
author: "grangeryy"
localization_priority: Normal
ms.prod: "excel"
---

# Best practices for working with the Excel API in Microsoft Graph

This article provides recommendations for working with the Excel APIs in Microsoft Graph.

## Manage sessions in the most efficient way

If you have more than one calls to make within a certain period of time, we strongly suggest you create a session and pass the session ID with each request. To represent the session in the API, use the `workbook-session-id: {session-id}` header. By doing so you can use the Excel APIs in the most efficient way.

Here is an example on a typical scenario you should choose this mode: “I want to add a new number to the table and then find one record in my workbook.”

Step 1: create a session.

```http
POST https://graph.microsoft.com/v1.0/me/drive/items/{id}/workbook/createSession
Content-type: application/json
Content-length: 52

{
  "persistChanges": true
}
```

You receive a response once the request is successful.

```http
HTTP/1.1 201 Created
Content-type: application/json
Content-length: 52

{
  "id": "id-value",
  "persistChanges": true
}
```

Step 2: add a new row to the table.

```http
POST https://graph.microsoft.com/v1.0/me/drive/items/{id}/workbook/tables/Table1/rows/add
Content-type: application/json
workbook-session-id: {session-id}

{
  "values": [[“east”, “pear”, 4]]
}
```

The successful response looks like:

```http
HTTP/1.1 200 OK
Content-type: application/json
Content-length: 42

{
  "index": 6,
  "values": [[“east”, “pear”, 4]]
}
```

Step 3: look up a value in the updated table.

```http
POST https://graph.microsoft.com/v1.0/me/drive/items/{id}/workbook/functions/vlookup
Content-type: application/json
workbook-session-id: {session-id}

{
    "lookupValue":"pear",
    "tableArray":{"Address":"Sheet1!B2:C7"},
    "colIndexNum":2,
    "rangeLookup":false
}
```

The successful response looks like:

```http
HTTP code: 200 OK
content-type: application/json

{
    "value": 5
}
```

Step 4: close the session once all of the requests are completed.

```http
POST https://graph.microsoft.com/v1.0/me/drive/items/{id}/workbook/closeSession
Content-type: application/json
workbook-session-id: {session-id}
Content-length: 0

{
}
```

The successful response looks similar as following.

```http
HTTP/1.1 204 No Content
```

For more details, see [Create session](/graph/api/workbook-createsession?view=graph-rest-1.0) and [Close session](/graph/api/workbook-closesession?view=graph-rest-1.0).

## Working with APIs that may take a long time to complete

You may notice some API responses require indeterminate time to complete, for example, open a workbook with large size. However, it is easy to hit timeout while waiting for the response to such kind of request. To resolve this issue, we provide the long running operation pattern and by using this pattern, you do not need to  worry about the timeout for the request.

Currently, Excel Graph has enabled long running operation pattern for session creation API. Here are the steps:

1. Adds a header of `Prefer: respond-async` in the request to indicate it as a long running operation when creating a session.
2. In long running operation pattern, it will return a `202 Accepted` response along with a Location header to retrieve operation status. Otherwise, if session creation completes in several seconds, it will return as regular create session instead of going to long running operation pattern.
3. With the `202 Accepted` response, you can retrieve the operation status through specified location. Operation status includes `notStarted`, `running`, `succeeded`, and `failed`.
4. After operation completes, you can get the session creation result through the specified URL in succeeded response.

Following is an example of create session with long running operation pattern.

### Initial request to create session

```http
POST https://graph.microsoft.com/v1.0/me/drive/items/{drive-item-id}/workbook/worksheets({id})/createSession
Prefer: respond-async
Content-type: application/json
{
    "persistChanges": true
}
```

Long running operation pattern will return a `202 Accepted` response similar as following.

```http
HTTP/1.1 202 Accepted
Location: https://graph.microsoft.com/v1.0/me/drive/items/{drive-item-id}/workbook/operations/{operation-id}
Content-type: application/json

{
}
```

In some cases, if the creation succeeds directly in seconds, it won't enter long running operation pattern, instead it returns as a regular create session and the successful request will return a `201 Created` response like following.

```http
HTTP/1.1 201 Created
Content-type: application/json
Content-length: 52

{
  "id": "id-value",
  "persistChanges": true
}
```

The failed request of regular create session will look as following.
>**Note:** The response object shown here might be shortened for readability.

```http
HTTP/1.1 500 Internal Server Error
Content-type: application/json
{
  "error":{
    "code": "internalServerError",
    "message": "An internal server error occurred while processing the request.",
    "innerError": {
      "code": ""internalServerErrorUncategorized",
      "message": "An unspecified error has occurred.",
      "innerError": {
        "code": "GenericFileOpenError",
        "message": "The workbook cannot be opened."
      }
    }
  }
}


### Poll status of the long running create session

In long running operation pattern, you can get creation status with specified location with request similar as following. The suggested interval to poll status is around 30 seconds and the maximum internal should be no more than 4 minutes.

```http
GET https://graph.microsoft.com/v1.0/me/drive/items/{drive-item-id}/workbook/operations/{operation-id}
{
}
```

Here are the examples of response with status `running`, `succeeded`, and `failed`.

Case of `running`: the operation is still in execution.

```http
HTTP/1.1 200 OK
Content-type: application/json
{
    "id": {operation-id},
    "status": "running"
}
```

Case of `succeeded`: the operation is successful.

```http
HTTP/1.1 200 OK
Content-type: application/json
{
    "id": {operation-id},
    "status": "succeeded",
    "resourceLocation": "https://graph.microsoft.com/v1.0/me/drive/items/{drive-item-id}/workbook/sessionInfoResource(key='{key}')
}
```

Case of `failed`: the operation fails.

```http
HTTP/1.1 200 OK
Content-type: application/json
{
  "id": {operation-id},
  "status": "failed",
  "error":{
    "code": "internalServerError",
    "message": "An internal server error occurred while processing the request.",
    "innerError": {
      "code": ""internalServerErrorUncategorized",
      "message": "An unspecified error has occurred.",
      "innerError": {
        "code": "GenericFileOpenError",
        "message": "The workbook cannot be opened."
      }
    }
  }
}
```

For more details on error codes, see [Error codes](/concepts/workbook-error-codes.md)

### Acquire session information

With status of `succeeded`, you can get the created session information through `resourceLocation` with request similar as following.

```http
GET https://graph.microsoft.com/v1.0/me/drive/items/{drive-item-id}/workbook/sessionInfoResource(key='{key}')
{
}
```

Corresponding response looks like below.

```http
HTTP/1.1 200 OK
Content-type: application/json
{
    "id": "id-value",
    "persistChanges": true
}
```

>**Note:** Acquire session information depends on the initial request, and there is no need to acquire the result if the initial request doesn't return a response body.