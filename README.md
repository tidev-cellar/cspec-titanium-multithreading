# CSPEC 1 - Titanium Multithreading

This CSPEC is for adding an official multithreading API to Titanium.

## Audience

The primary audience for this API is a Titanium application developer.

## Goals

The goals are to provide an official cross-platform way to perform asynchoronous tasks in a Titanium application.

## Description

The problem is that Titanium has no official way to control what types of tasks are performed and how they are scheduled.  There are currently two different "modes" -- using a Kroll thread (i.e. JS runs on it's own separate thread than the "main" UI thread) or main thread (i.e. JS runs on the main thread).  Starting in 6.0, we intend to move the default to "main thread" for application execution.  However, this potentially presents problems where certain operations should not block the UI thread and current hacks (setTimeout) don't provide sufficient control or predicitability and can be unstable or different per platform.

## Proposal

The proposal would be to create a new API which specifically handles scheduling tasks to be run on either a new thread or the main UI thread.  The API would give the developer the specific control to handle when and where tasks should run and provide clear intentions on how to run them.  The API would also hide the complexities and platform specific differences between executing JS on different threads in different JS Contexts.

### API Definition

#### Namespace

The proposed namespace is `Titanium.Async`.  All APIs will be defined as properties that are children of this namespace.

#### Creating Queues

A Queue will define a set of tasks that run serially on a system thread or other similar internal mechanism defined by the platform implementation.  The API will provide a default queue called the "main queue" which is automatically created.  The main queue is special and cannot be deleted.

You can create a new Queue using the `createQueue` method.

```javascript
var queue = Ti.Async.createQueue();
```

#### Destroying Queues

You can destroy a user-generated Queue with `destroy` using the queue instance.

```javascript
queue.destroy();
```

Once a queue is destroyed, any further usage will result in unpredictable behavior.

Any pending jobs in the queue will automatically be cancelled.

##### Referencing the Main Queue

Since the main queue is a system provided queue, you can reference it by using the `getMainQueue` method.

```javascript
var mainQueue = Ti.Async.getMainQueue();
```

##### Dispatching a Job

You dispatch a job using the `dispatch` method on the queue instance.

```javascript
var job = queue.dispatch(function() {
   // my job is contained in this function
});
```

When you dispatch a job, the queue will return a Promise instance for the job.  The Promise instance has several methods for interacting with the job.

##### Passing data to the Job

Jobs cannot directly reference JS variables or functions not contained inside the job function.  When a queue is created, a new JS context is created which does not share any application state (variables, functions, etc) with your application or with other queues or other jobs.

In order to pass data between jobs, you must pass serialized data (serializable as JSON although implementations likely are free to "copy by value" the data) to and from the job function.

To pass data into the job, you can use with `with` method on the job instance.

```javascript
job.with({foo:1});
```

And then in your job function, the first argument would receive your data.

```javascript
var job = queue.dispatch(function (imports) {
    // imports.foo === 1
})
.with({foo:1});
```

##### Passing data from the Job

Since jobs cannot pass data directly back to the application or set variables in the application scope, it must pass any optional data back through a provided Object which is passed as the second argument to the job function.

```javascript
var job = queue.dispatch(function (imports, exports) {
   exports.foo = 'bar';
});
```

To receive the data, the job instance provides a `then` method which you can pass a callback function to receive any data.

```javascript
var job = queue.dispatch(function (imports, exports) {
   exports.foo = 'bar';
})
.then(function (exports) {
   // exports.foo === 'bar'
});
```

The `then` function is always called (if set) regardless if you set any data in the `exports` or not.

#### Tracking Job Status

You can receive callbacks when the job state changes using the `status` method to pass a callback function.

```javascript
var job = queue.dispatch(function (imports, exports) {
})
.status(function (status) {
    switch (status) {
       case Ti.Async.STATUS_CREATED:
           Ti.API.debug('job created');
       default: break;
    }
});
```

The following are the valid job states:

- *STATUS_CREATED* - the job has been created ("queued") but not yet executed
- *RUNNING* - the job is currently running
- *COMPLETED* - the job has completed running without error or cancellation
- *ERRORED* - the job encountered an unhandled exception
- *CANCELLED* - the job was cancelled
 
#### Handling Job Errors

You can receive a callback when an unhandled exception happens processing a job using the `error` method to pass a callback function.

```javascript
var job = queue.dispatch(function (imports, exports) {
})
.error(function (err) {
   // err is the Error object for the exception
});
```

## Timeline

The goal of this proposal is to get flushed out ASAP so that it can be incorporated into Titanium 6.0 in the Q1 2016 timeframe.

## Status

12-02-2016 - Initial Draft

## Legal Stuff

This proposal is non-binding and may not be implemented, may be implemented partially or not at all. Any intellectual property developed as part of this proposal is owned and Copyright (c) 2015 by Appcelerator, Inc. All Rights Reserved.
