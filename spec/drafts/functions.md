# Serverless Functions Signature Specification - WIP

## Abstract

A vendor-neutral specification for defining the function signature for serveress functions.

## Table of Contents

- [Overview](#overview)
- [Out of Scope](#out-of-scope)
- [Notations and Terminology](#notations-and-terminology)
- [Function Definition](#function-definition)
- [Context Object Model](#context-attributes)
- [Example](#example)

## Overview

Today every competing JavaScript functions API is different enough that we end up with lower developer productivity and vendor lock-in. Lower developer productivity because developers have to learn multiple ways to write the JavaScript function code and may have to write more complex code to avoid vendor lock-in. Organizations wanting to leverage multiple functions providers or move from one provider to another incur significant additional cost.

The Goal is to define a JavaScript functions API that avoids vendor lock-in and facilitates developer productivity while being general enough to allow different implementations and optimizations behind the scenes.

### Out of Scope

Below are the items that this specification does not define:

- Use of ESM or CJS.
- Underlying framework that “serves” the function.
- Cloud Platform for running/serving the function.
- Any Optional context object model attributes
- Format of the function return value.

## Notations and Terminology

### Notational Conventions

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD",
"SHOULD NOT", "RECOMMENDED", "MAY", and "OPTIONAL" in this document are to be
interpreted as described in [RFC 2119](https://tools.ietf.org/html/rfc2119).

### Terminology

This specification defines the following terms:

#### Serverless Function

A serverless function is a programmatic function written by a software developer for a single purpose. It's then hosted and maintained on infrastructure by cloud computing companies.

#### Function Developer

A function developer is someone who is creating a serverless function.

#### Context Object Model

A context object model is a parameter of the function signature that exposes certain data and Attributes to the function developer.

#### CloudEvents

CloudEvents is a specification for describing event data in common formats to provide interoperability across services, platforms and systems.

## Function Definition

A serverless function SHOULD be exported as either a function or an object with a “handle” function.  The function SHOULD use `return` to send results. Calling `return` ends the function execution.

This specification does not define what the function return value is, but it is RECOMMENDED to return an Object that is JSON serializable.

```
function handlerFunction(context: Object){
  ...
  return { ... };
}

module.exports = handlerFunction ;

//or

module.exports = {
  handle: handlerFunction
}
```

## Context Object Model

Functions are invoked with a context object as the first parameter. This object provides access to the incoming request information.  Function developers may access several properties including, but not limited to, the HTTP request method, any query strings sent with the request, the headers, the request body, a logger and a CloudEvent.

const context = {
  request: {
    query: Object,
    body: Object,
    headers: Object,
    method: String
  },
  log: Logger,
  event: CloudEvent
};

### Required Context Object Attributes

The following Attributes are REQUIRED to be present in all Context Objects:

#### request

- Type: `Object`
- Description: An object holds information about the incoming request

#### request.query

- Type: `Object`
- Description: the query string deserialized as an object, if any
- Constraints:
  - For HTTP GET requests, a `request.query` SHOULD be available, if query string Attributes exist
- Examples:
  - `{ param1: value, param2: value }`

#### request.body

- Type: `Object`
- Description: The request body if any
- Constraints:
  - The request body SHOULD NOT expose the underlying framework(express, fastify)
  - For HTTP POST requests, a `request.body` SHOULD be available
- Examples:
  - TODO

#### request.headers

- Type: `Object`
- Description: The HTTP request headers
- Constraints:
- Examples:
  - `{ HEADER_1: value }`

#### request.method

- Type: `String`
- Description: The HTTP request method
- Constraints:
- Examples:
  - GET
  - POST

#### log

- Type: `Object`
- Description: A Logging object with methods for 'info', 'warn', 'error', etc.
- Constraints:
  - Up to the implementation on where the result is logged out to
- Examples:
  - console.log/console.error/etc...
  - an instance of a Pino logger

#### event

- Type: `CloudEvent`
- Description: The event that triggered the function.
- Constraints:
  - If a CloudEvent is detected, then this attribute is REQUIRED
- Examples:
  - A CloudEvent Serialized as JSON:
  ```
  {
      "specversion" : "1.0",
      "type" : "com.github.pull_request.opened",
      "source" : "https://github.com/cloudevents/spec/pull",
      "subject" : "123",
      "id" : "A234-1234-1234"
      ...
  }
  ```

### OPTIONAL Context Object Attributes

This specification does not define what OPTIONAL Context Object Attributes to provide.
