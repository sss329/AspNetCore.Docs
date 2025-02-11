---
title: JsonPatch in ASP.NET Core web API
author: rick-anderson
description: Learn how to handle JSON Patch requests in an ASP.NET Core web API.
ms.author: riande
ms.custom: mvc
ms.date: 03/24/2019
uid: web-api/jsonpatch
---

# JsonPatch in ASP.NET Core web API

By [Tom Dykstra](https://github.com/tdykstra) and [Kirk Larkin](https://github.com/serpent5)

::: moniker range=">= aspnetcore-3.0"

<!--   Uncomment this when you update 
This article explains how to handle JSON Patch requests in an ASP.NET Core web API using `Microsoft.AspNetCore.JsonPatch`. For instructions on using [Microsoft.AspNetCore.Mvc.NewtonsoftJson](https://www.nuget.org/packages/Microsoft.AspNetCore.Mvc.NewtonsoftJson/), use the version selector on this page to select an ASP.NET Core 2.x version.
-->

## Package installation

Support for JsonPatch is enabled using the `Microsoft.AspNetCore.Mvc.NewtonsoftJson` package. To enable this feature, apps must:

* Install the [Microsoft.AspNetCore.Mvc.NewtonsoftJson](https://www.nuget.org/packages/Microsoft.AspNetCore.Mvc.NewtonsoftJson/) NuGet package.
* Update the project's `Startup.ConfigureServices` method to include a call to `AddNewtonsoftJson`:

  ```csharp
  services
      .AddControllersWithViews()
      .AddNewtonsoftJson();
  ```

`AddNewtonsoftJson` is compatible with the MVC service registration methods:

  * `AddRazorPages`
  * `AddControllersWithViews`
  * `AddControllers`

## PATCH HTTP request method

The PUT and [PATCH](https://tools.ietf.org/html/rfc5789) methods are used to update an existing resource. The difference between them is that PUT replaces the entire resource, while PATCH specifies only the changes.

## JSON Patch

[JSON Patch](https://tools.ietf.org/html/rfc6902) is a format for specifying updates to be applied to a resource. A JSON Patch document has an array of *operations*. Each operation identifies a particular type of change, such as add an array element or replace a property value.

For example, the following JSON documents represent a resource, a JSON patch document for the resource, and the result of applying the patch operations.

### Resource example

[!code-json[](jsonpatch/samples/2.2/JSON/customer.json)]

### JSON patch example

[!code-json[](jsonpatch/samples/2.2/JSON/add.json)]

In the preceding JSON:

* The `op` property indicates the type of operation.
* The `path` property indicates the element to update.
* The `value` property provides the new value.

### Resource after patch

Here's the resource after applying the preceding JSON Patch document:

```json
{
  "customerName": "Barry",
  "orders": [
    {
      "orderName": "Order0",
      "orderType": null
    },
    {
      "orderName": "Order1",
      "orderType": null
    },
    {
      "orderName": "Order2",
      "orderType": null
    }
  ]
}
```

The changes made by applying a JSON Patch document to a resource are atomic: if any operation in the list fails, no operation in the list is applied.

## Path syntax

The [path](https://tools.ietf.org/html/rfc6901) property of an operation object has slashes between levels. For example, `"/address/zipCode"`.

Zero-based indexes are used to specify array elements. The first element of the `addresses` array would be at `/addresses/0`. To `add` to the end of an array, use a hyphen (-) rather than an index number: `/addresses/-`.

### Operations

The following table shows supported operations as defined in the [JSON Patch specification](https://tools.ietf.org/html/rfc6902):

|Operation  | Notes |
|-----------|--------------------------------|
| `add`     | Add a property or array element. For existing property: set value.|
| `remove`  | Remove a property or array element. |
| `replace` | Same as `remove` followed by `add` at same location. |
| `move`    | Same as `remove` from source followed by `add` to destination using value from source. |
| `copy`    | Same as `add` to destination using value from source. |
| `test`    | Return success status code if value at `path` = provided `value`.|

## JsonPatch in ASP.NET Core

The ASP.NET Core implementation of JSON Patch is provided in the [Microsoft.AspNetCore.JsonPatch](https://www.nuget.org/packages/microsoft.aspnetcore.jsonpatch/) NuGet package. The package is included in the [Microsoft.AspnetCore.App](xref:fundamentals/metapackage-app) metapackage.

## Action method code

In an API controller, an action method for JSON Patch:

* Is annotated with the `HttpPatch` attribute.
* Accepts a `JsonPatchDocument<T>`, typically with `[FromBody]`.
* Calls `ApplyTo` on the patch document to apply the changes.

Here's an example:

[!code-csharp[](jsonpatch/samples/2.2/Controllers/HomeController.cs?name=snippet_PatchAction&highlight=1,3,9)]

This code from the sample app works with the following `Customer` model.

[!code-csharp[](jsonpatch/samples/2.2/Models/Customer.cs?name=snippet_Customer)]

[!code-csharp[](jsonpatch/samples/2.2/Models/Order.cs?name=snippet_Order)]

The sample action method:

* Constructs a `Customer`.
* Applies the patch.
* Returns the result in the body of the response.

 In a real app, the code would retrieve the data from a store such as a database and update the database after applying the patch.

### Model state

The preceding action method example calls an overload of `ApplyTo` that takes model state as one of its parameters. With this option, you can get error messages in responses. The following example shows the body of a 400 Bad Request response for a `test` operation:

```json
{
    "Customer": [
        "The current value 'John' at path 'customerName' is not equal to the test value 'Nancy'."
    ]
}
```

### Dynamic objects

The following action method example shows how to apply a patch to a dynamic object.

[!code-csharp[](jsonpatch/samples/2.2/Controllers/HomeController.cs?name=snippet_Dynamic)]

## The add operation

* If `path` points to an array element: inserts new element before the one specified by `path`.
* If `path` points to a property: sets the property value.
* If `path` points to a nonexistent location:
  * If the resource to patch is a dynamic object: adds a property.
  * If the resource to patch is a static object: the request fails.

The following sample patch document sets the value of `CustomerName` and adds an `Order` object to the end of the `Orders` array.

[!code-json[](jsonpatch/samples/2.2/JSON/add.json)]

## The remove operation

* If `path` points to an array element: removes the element.
* If `path` points to a property:
  * If resource to patch is a dynamic object: removes the property.
  * If resource to patch is a static object:
    * If the property is nullable: sets it to null.
    * If the property is non-nullable, sets it to `default<T>`.

The following sample patch document sets `CustomerName` to null and deletes `Orders[0]`.

[!code-json[](jsonpatch/samples/2.2/JSON/remove.json)]

## The replace operation

This operation is functionally the same as a `remove` followed by an `add`.

The following sample patch document sets the value of `CustomerName` and replaces `Orders[0]`with a new `Order` object.

[!code-json[](jsonpatch/samples/2.2/JSON/replace.json)]

## The move operation

* If `path` points to an array element: copies `from` element to location of `path` element, then runs a `remove` operation on the `from` element.
* If `path` points to a property: copies value of `from` property to `path` property, then runs a `remove` operation on the `from` property.
* If `path` points to a nonexistent property:
  * If the resource to patch is a static object: the request fails.
  * If the resource to patch is a dynamic object: copies `from` property to location indicated by `path`, then runs a `remove` operation on the `from` property.

The following sample patch document:

* Copies the value of `Orders[0].OrderName` to `CustomerName`.
* Sets `Orders[0].OrderName` to null.
* Moves `Orders[1]` to before `Orders[0]`.

[!code-json[](jsonpatch/samples/2.2/JSON/move.json)]

## The copy operation

This operation is functionally the same as a `move` operation without the final `remove` step.

The following sample patch document:

* Copies the value of `Orders[0].OrderName` to `CustomerName`.
* Inserts a copy of `Orders[1]` before `Orders[0]`.

[!code-json[](jsonpatch/samples/2.2/JSON/copy.json)]

## The test operation

If the value at the location indicated by `path` is different from the value provided in `value`, the request fails. In that case, the whole PATCH request fails even if all other operations in the patch document would otherwise succeed.

The `test` operation is commonly used to prevent an update when there's a concurrency conflict.

The following sample patch document has no effect if the initial value of `CustomerName` is "John", because the test fails:

[!code-json[](jsonpatch/samples/2.2/JSON/test-fail.json)]

## Get the code

[View or download sample code](https://github.com/aspnet/AspNetCore.Docs/tree/master/aspnetcore/web-api/jsonpatch/samples/2.2). ([How to download](xref:index#how-to-download-a-sample)).

To test the sample, run the app and send HTTP requests with the following settings:

* URL: `http://localhost:{port}/jsonpatch/jsonpatchwithmodelstate`
* HTTP method: `PATCH`
* Header: `Content-Type: application/json-patch+json`
* Body: Copy and paste one of the JSON patch document samples from the *JSON* project folder.

## Additional resources

* [IETF RFC 5789 PATCH method specification](https://tools.ietf.org/html/rfc5789)
* [IETF RFC 6902 JSON Patch specification](https://tools.ietf.org/html/rfc6902)
* [IETF RFC 6901 JSON Patch path format spec](https://tools.ietf.org/html/rfc6901)
* [JSON Patch documentation](https://jsonpatch.com/). Includes links to resources for creating JSON Patch documents.
* [ASP.NET Core JSON Patch source code](https://github.com/aspnet/AspNetCore/tree/master/src/Features/JsonPatch/src)

::: moniker-end

::: moniker range="< aspnetcore-3.0"

This article explains how to handle JSON Patch requests in an ASP.NET Core web API.

## PATCH HTTP request method

The PUT and [PATCH](https://tools.ietf.org/html/rfc5789) methods are used to update an existing resource. The difference between them is that PUT replaces the entire resource, while PATCH specifies only the changes.

## JSON Patch

[JSON Patch](https://tools.ietf.org/html/rfc6902) is a format for specifying updates to be applied to a resource. A JSON Patch document has an array of *operations*. Each operation identifies a particular type of change, such as add an array element or replace a property value.

For example, the following JSON documents represent a resource, a JSON patch document for the resource, and the result of applying the patch operations.

### Resource example

[!code-json[](jsonpatch/samples/2.2/JSON/customer.json)]

### JSON patch example

[!code-json[](jsonpatch/samples/2.2/JSON/add.json)]

In the preceding JSON:

* The `op` property indicates the type of operation.
* The `path` property indicates the element to update.
* The `value` property provides the new value.

### Resource after patch

Here's the resource after applying the preceding JSON Patch document:

```json
{
  "customerName": "Barry",
  "orders": [
    {
      "orderName": "Order0",
      "orderType": null
    },
    {
      "orderName": "Order1",
      "orderType": null
    },
    {
      "orderName": "Order2",
      "orderType": null
    }
  ]
}
```

The changes made by applying a JSON Patch document to a resource are atomic: if any operation in the list fails, no operation in the list is applied.

## Path syntax

The [path](https://tools.ietf.org/html/rfc6901) property of an operation object has slashes between levels. For example, `"/address/zipCode"`.

Zero-based indexes are used to specify array elements. The first element of the `addresses` array would be at `/addresses/0`. To `add` to the end of an array, use a hyphen (-) rather than an index number: `/addresses/-`.

### Operations

The following table shows supported operations as defined in the [JSON Patch specification](https://tools.ietf.org/html/rfc6902):

|Operation  | Notes |
|-----------|--------------------------------|
| `add`     | Add a property or array element. For existing property: set value.|
| `remove`  | Remove a property or array element. |
| `replace` | Same as `remove` followed by `add` at same location. |
| `move`    | Same as `remove` from source followed by `add` to destination using value from source. |
| `copy`    | Same as `add` to destination using value from source. |
| `test`    | Return success status code if value at `path` = provided `value`.|

## JsonPatch in ASP.NET Core

The ASP.NET Core implementation of JSON Patch is provided in the [Microsoft.AspNetCore.JsonPatch](https://www.nuget.org/packages/microsoft.aspnetcore.jsonpatch/) NuGet package. The package is included in the [Microsoft.AspnetCore.App](xref:fundamentals/metapackage-app) metapackage.

## Action method code

In an API controller, an action method for JSON Patch:

* Is annotated with the `HttpPatch` attribute.
* Accepts a `JsonPatchDocument<T>`, typically with `[FromBody]`.
* Calls `ApplyTo` on the patch document to apply the changes.

Here's an example:

[!code-csharp[](jsonpatch/samples/2.2/Controllers/HomeController.cs?name=snippet_PatchAction&highlight=1,3,9)]

This code from the sample app works with the following `Customer` model.

[!code-csharp[](jsonpatch/samples/2.2/Models/Customer.cs?name=snippet_Customer)]

[!code-csharp[](jsonpatch/samples/2.2/Models/Order.cs?name=snippet_Order)]

The sample action method:

* Constructs a `Customer`.
* Applies the patch.
* Returns the result in the body of the response.

 In a real app, the code would retrieve the data from a store such as a database and update the database after applying the patch.

### Model state

The preceding action method example calls an overload of `ApplyTo` that takes model state as one of its parameters. With this option, you can get error messages in responses. The following example shows the body of a 400 Bad Request response for a `test` operation:

```json
{
    "Customer": [
        "The current value 'John' at path 'customerName' is not equal to the test value 'Nancy'."
    ]
}
```

### Dynamic objects

The following action method example shows how to apply a patch to a dynamic object.

[!code-csharp[](jsonpatch/samples/2.2/Controllers/HomeController.cs?name=snippet_Dynamic)]

## The add operation

* If `path` points to an array element: inserts new element before the one specified by `path`.
* If `path` points to a property: sets the property value.
* If `path` points to a nonexistent location:
  * If the resource to patch is a dynamic object: adds a property.
  * If the resource to patch is a static object: the request fails.

The following sample patch document sets the value of `CustomerName` and adds an `Order` object to the end of the `Orders` array.

[!code-json[](jsonpatch/samples/2.2/JSON/add.json)]

## The remove operation

* If `path` points to an array element: removes the element.
* If `path` points to a property:
  * If resource to patch is a dynamic object: removes the property.
  * If resource to patch is a static object:
    * If the property is nullable: sets it to null.
    * If the property is non-nullable, sets it to `default<T>`.

The following sample patch document sets `CustomerName` to null and deletes `Orders[0]`.

[!code-json[](jsonpatch/samples/2.2/JSON/remove.json)]

## The replace operation

This operation is functionally the same as a `remove` followed by an `add`.

The following sample patch document sets the value of `CustomerName` and replaces `Orders[0]`with a new `Order` object.

[!code-json[](jsonpatch/samples/2.2/JSON/replace.json)]

## The move operation

* If `path` points to an array element: copies `from` element to location of `path` element, then runs a `remove` operation on the `from` element.
* If `path` points to a property: copies value of `from` property to `path` property, then runs a `remove` operation on the `from` property.
* If `path` points to a nonexistent property:
  * If the resource to patch is a static object: the request fails.
  * If the resource to patch is a dynamic object: copies `from` property to location indicated by `path`, then runs a `remove` operation on the `from` property.

The following sample patch document:

* Copies the value of `Orders[0].OrderName` to `CustomerName`.
* Sets `Orders[0].OrderName` to null.
* Moves `Orders[1]` to before `Orders[0]`.

[!code-json[](jsonpatch/samples/2.2/JSON/move.json)]

## The copy operation

This operation is functionally the same as a `move` operation without the final `remove` step.

The following sample patch document:

* Copies the value of `Orders[0].OrderName` to `CustomerName`.
* Inserts a copy of `Orders[1]` before `Orders[0]`.

[!code-json[](jsonpatch/samples/2.2/JSON/copy.json)]

## The test operation

If the value at the location indicated by `path` is different from the value provided in `value`, the request fails. In that case, the whole PATCH request fails even if all other operations in the patch document would otherwise succeed.

The `test` operation is commonly used to prevent an update when there's a concurrency conflict.

The following sample patch document has no effect if the initial value of `CustomerName` is "John", because the test fails:

[!code-json[](jsonpatch/samples/2.2/JSON/test-fail.json)]

## Get the code

[View or download sample code](https://github.com/aspnet/AspNetCore.Docs/tree/master/aspnetcore/web-api/jsonpatch/samples/2.2). ([How to download](xref:index#how-to-download-a-sample)).

To test the sample, run the app and send HTTP requests with the following settings:

* URL: `http://localhost:{port}/jsonpatch/jsonpatchwithmodelstate`
* HTTP method: `PATCH`
* Header: `Content-Type: application/json-patch+json`
* Body: Copy and paste one of the JSON patch document samples from the *JSON* project folder.

## Additional resources

* [IETF RFC 5789 PATCH method specification](https://tools.ietf.org/html/rfc5789)
* [IETF RFC 6902 JSON Patch specification](https://tools.ietf.org/html/rfc6902)
* [IETF RFC 6901 JSON Patch path format spec](https://tools.ietf.org/html/rfc6901)
* [JSON Patch documentation](https://jsonpatch.com/). Includes links to resources for creating JSON Patch documents.
* [ASP.NET Core JSON Patch source code](https://github.com/aspnet/AspNetCore/tree/master/src/Features/JsonPatch/src)

::: moniker-end
