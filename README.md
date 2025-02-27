# Nest AMQP 1.0 Module

[![Build Status](https://github.com/team-supercharge/nest-amqp/actions/workflows/master-branch.yml/badge.svg)](https://github.com/team-supercharge/nest-amqp/actions/workflows/master-branch.yml)
[![Coverage Status](https://codecov.io/github/team-supercharge/nest-amqp/coverage.svg?branch=master)](https://codecov.io/github/team-supercharge/nest-amqp)
<a href="https://www.npmjs.com/@team-supercharge/nest-amqp" target="_blank"><img src="https://img.shields.io/npm/v/@team-supercharge/nest-amqp.svg" alt="NPM Version" /></a>
<a href="https://www.npmjs.com/@team-supercharge/nest-amqp" target="_blank"><img src="https://img.shields.io/npm/l/@team-supercharge/nest-amqp.svg" alt="Package License" /></a>

## Description

AMQP 1.0 module for [Nest](https://github.com/nestjs/nest). It is based on the
[rhea-promise](https://www.npmjs.com/package/rhea-promise) package. This package is for
working with a message broker with extra data validation functionality. It uses the
[class-transformer](https://www.npmjs.com/package/class-transformer) and
[class-validator](https://www.npmjs.com/package/class-validator) packages to transform
and validate the payload of the messages. With this we can easily verify the message's
payload structure and its data types with Data Transfer Objects. Besides, it gives
control over the message how to handle it: accept, reject or release.

## Installation

```bash
$ npm install --save @team-supercharge/nest-amqp
```

To use the library, the peer dependencies must also be installed:
```bash
$ npm install --save class-transformer class-validator
```

## Usage

In the following subsections you can see how to send and receive messages, how to handle message transfer and how to use DTO classes for message
payload transformation and validation.

### Module import and connection options

To create a connection, you have to set the connection details. The library provides an easy way to set the connection details via a string
connection URI. The library will parse this connection URI and set the appropriate connection options. Besides, you can add your custom
connection options or other library settings with the module options.

#### Connection URI

This library provides an easier way to set the connection options with a connection URI: you can describe the connection settings with a
URL. The library will parse the URL and set the corresponding options. Here you can see the description of the URL:
```bash
protocol://[username:password@]host:port
```

The `username` and `password` components are optional, with these you can set the authentication credentials to the message queue server.

> Note: if you would like to use special characters in username or password, then you have to `encodeURIComponent()` these values, because
> the library will decode the values right before connection creation. If you don't encode these values, then you will get a URL parse
> error.
>
> If you use environment variables to connection username or password, then use the env values with `encodeURIComponent()` as well.
>
> Example:
> ```
> const username = encodeURIComponent('Jörg');
> const password = encodeURIComponent('Gt|N#R=6$5(TE@rH"Pvc$7a');
> const connectionUri = `amqps://${username}:${password}@localhost:5672`;
> ```

You can set custom protocol which will set the connection transport automatically, so you don't have to add the `transport` to the connection
options object. The protocol can be:
* **amqp**: in this case the `transport` will be `tcp`
* **amqps**: in this case the `transport` will be `ssl`
* **amqp+ssl**: in this case the `transport` will be `ssl`
* **amqp+tls**: in this case the `transport` will be `tls`

Examples:
* `amqp://localhost:5672`
* `amqps://user:password@my-server.com:5672`
* `amqp+tls://admin:secret@127.0.0.1:5672`

#### Create connection

To create a connection, you have to import the `QueueModule.forRoot()` module into your application's root module. The `forRoot()`
static method has multiple parameters:
* for single connection:
  * the first and required is either the connection URI string, or the module configuration object
  * the *optional* second parameter is the module and connection configuration object sans the connection URI, if you provided the URI as a first parameter
* for multiple connections:
  * the first and required parameter is an array of *connection configuration* objects
  * the *optional* second parameter is the module configuration object

To see the available module options, scroll down.
Here are some simple examples:

```typescript
// for single connection

import { Module } from '@nestjs/common';
import { QueueModule } from '@team-supercharge/nest-amqp';

@Module({
  imports: [
    QueueModule.forRoot('amqp://user:password@localhost:5672'),
    // or alternatively
    // QueueModule.forRoot({ connectionUri: 'amqp://user:password@localhost:5672' }),
  ],
})
export class AppModule {}
```

```typescript
// for multiple connections

import { Module } from '@nestjs/common';
import { QueueModule } from '@team-supercharge/nest-amqp';

@Module({
  imports: [
    QueueModule.forRoot([
      { connectionUri: 'amqp://user1:password1@localhost:5671' },
      { connectionUri: 'amqp://user2:password2@localhost:5672', name: Connections.WORKER },
    ]),
  ],
})
export class AppModule {}
```

#### Create connection with asynchronous module configuration

Generally we are using environment variables to configure our application. Nest provides the
[ConfigService](https://docs.nestjs.com/techniques/configuration) to use these env variables nicely. If you would like to configure the
AMQP module with env variables, or you are using another asynchronous way to get the configuration, then you have to use the
`forRootAsync()` static method instead of `forRoot()` on the `QueueModule` class. To see the available module options, scroll down.
Here is an example:

> Note: the @team-supercharge/nest-amqp package does not support multiple connection on asynchronous module configuration!

```typescript
import { Module } from '@nestjs/common';
import { ConfigModule, ConfigService } from '@nestjs/config';
import { QueueModule, QueueModuleOptions } from '@team-supercharge/nest-amqp';

@Module({
  imports: [
    QueueModule.forRootAsync({
      imports: [ConfigModule],
      useFactory: (configService: ConfigService): QueueModuleOptions => ({
        connectionUri: configService.get<string>('AMQP_CONNECTION_URI'),
        connectionOptions: {
          transport: configService.get<string>('AMQP_CONNECTION_TRANSPORT'),
        }
      }),
      inject: [ConfigService],
    }),
  ],
})
export class AppModule {}
```

Instead of `useFactory` you can use `useClass` or `useExisting` to set module options. You can see the examples in the
[test file](https://github.com/team-supercharge/nest-amqp/tree/master/src/queue.module.spec.ts).

#### Module options

The module options object needs to be added to the `forRoot()` or `forRootAsync()` static method. The possible options can be these:
* **isGlobal**?: A boolean value. If this property is `true`, then you can skip the import with`.forFeature()` method in the feature
  modules, because the public services will be available in all modules which imports the root module. Default value is
  `false`. (You can read more about it on [Nest modules page](https://docs.nestjs.com/modules#global-modules))
* **logger**?: A custom object or class instance which implements the `LoggerService` interface from the `@nestjs/common` package.
  With this option, you can use your own logging method to log information in `@team-supercharge/nest-amqp` library. If it is
  not set then the library will use Nest's `Logger` service to logging. You can see a full example below in the readme.
  Default value is `undefined`.
* **throwExceptionOnConnectionError**?: A boolean value. If it's `true` then QueueModule will throw forward the exception which occurs
  during the connection creation. Default value is `false`.
* **connectionUri**?: It is an optional string property. This is required only when you are use the `forRootAsync()` method or the
  `forRoot()` method for single connection with only an object argument.
* **connectionOptions**?: It is an optional object. With this you can set
  [Rhea's connection options](https://www.npmjs.com/package/rhea#connectoptions) which will be passed to the `Connection` object
  during connection creation. The default value is `{}`.

First basic example:
```typescript
@Module({
  imports: [
    QueueModule.forRoot(
      'amqp://user:password@localhost:5672',
      {
        isGlobal: true,
        throwExceptionOnConnectionError: true,
        connectionOptions: {
          transport: 'tcp'
        }
      }
    ),
  ],
})
export class AppModule {}
```

Second example with asynchronous configuration:
```typescript
@Module({
  imports: [
    QueueModule.forRootAsync({
      // in case of `QueueModule.forRootAsync`, isGlobal property goes here and
      // not into the object returned by 'useFactory' or 'useClass' or 'useExisting'
      isGlobal: true,
      imports: [ConfigModule],
      useFactory: (configService: ConfigService): QueueModuleOptions => ({
        connectionUri: configService.get<string>('AMQP_CONNECTION_URI'),
        throwExceptionOnConnectionError: true,
        connectionOptions: {
          transport: configService.get<string>('AMQP_CONNECTION_TRANSPORT'),
        }
      }),
      inject: [ConfigService],
    }),
  ],
})
export class AppModule {}
```

### Custom logger

You can use your own logger solution in the library. To do this, you have to create a class which implements the `LoggerService` interface
from the `@nestjs/common` package. After that you can add the class instance to the module options, and the logging will work with your
custom solution. The `@team-supercharge/nest-amqp` library will use this one instance/object to log everything inside, so it is good if you
use the `context` argument in the log methods to determine who is logging currently. Here is an example:

```typescript
import { LoggerService, Module } from "@nestjs/common";

class MyLogger implements LoggerService {
  public log(message: any, context: string): void {
    console.log(`[${context}] ${message}`);
  }
  public error(message: any, trace: string, context: string): void {
    console.error(`[${context}] ${message}`);
  }
  public warn(message: any, context: string): void {
    console.warn(`[${context}] ${message}`);
  }
  public debug(message: any, context: string): void {
    console.debug(`[${context}] ${message}`);
  }
  public verbose(message: any, context: string): void {
    console.log(`[${context}] ${message}`);
  }
}

@Module({
  imports: [
    QueueModule.forRoot(
      'amqp://user:password@localhost:5672',
      {
        logger: new MyLogger()
      }
    ),
  ],
})
export class AppModule {}
```

### Send a message

#### Single connection
You can send messages with the `QueueService`. First you have to inject the service instance in the constructor, then you can use it to
send a message. Here is an example:

```typescript
import { Injectable } from '@nestjs/common';
import { QueueService } from '@team-supercharge/nest-amqp';

@Injectable()
export class TestService {
  constructor(private readonly queueService: QueueService) {}

  public async sendMessage(): Promise<void> {
    const payload = { text: 'hello world', date: new Date() };
    await this.queueService.send<TestDto>('example', payload);
  }
}
```

In the example we send a message to the `example` queue with an object payload. The `QueueService` will stringify the object and send the
message with the stringified object as payload. You can add a DTO class as generic type to the `send<T>()` method and TypeScript will
check that the second argument is matches with the DTO class.

> Note: the `send<T>()` method won't validate and transform the payload, it only stringifies the payload to a string value and send objects
> as string payload.

The `send<T>()` method's third parameter is the options which is an optional object. You can set here the
[rhea connection options](https://github.com/amqp/rhea/blob/2cbe55795b86a43375111f2f446089f55b948ecf/typings/connection.d.ts#L525)
as well. Other custom available options:
* **schedule.divideMinute**?: Send the message multiple times a minute given by this number.
* **schedule.cron**?: Send the message multiple times given by this standard cron string.
* **schedule.afterSeconds**?: Message sending delay in seconds.

#### Multiple connections

Sending messages with multiple connections works mostly the same, the only difference is that we need to define to which connection do we want to send the message. Here is an example:

```typescript
import { Injectable } from '@nestjs/common';
import { QueueService } from '@team-supercharge/nest-amqp';

import { Connections } from './constants';

@Injectable()
export class TestService {
  constructor(private readonly queueService: QueueService) {}

  public async sendMessage(): Promise<void> {
    const payload = { text: 'hello world', date: new Date() };

    await this.queueService.send<TestDto>('example', payload, Connections.TEST);
  }

  public async sendDelayedMessage(): Promise<void> {
    const payload = { text: 'hello world', date: new Date() };
    const sendOptions = { schedule: { afterSeconds: 10 } };

    await this.queueService.send<TestDto>('example', payload, sendOptions, Connections.TEST);
  }
}
```

> Note: It is assumed that there is an enum named `Connections`, and
the connections defined in the `forRoot` method contains a connection named `Connections.TEST`.

> Note: If you leave out the name of the connection, it will be sent a to the default connection (if exists).

### Listen to a queue

#### Single connection

If you want to receive the messages which arrive at a queue then you have to listen on the specific queue. You can do this with the
`@Listen()` decorator which you can add to a method of a class which has the `@Injectable()` decorator. Here is an example:

```typescript
@Injectable()
export class ExampleListener {
  @Listen('example', { type: PayloadDto })
  public async listenForQueueMessages(data: PayloadDto, control: MessageControl): Promise<void> {
    console.log('new message arrived on the "example" queue:', data);
  }
}
```

The `@Listen()` decorator's first parameter is the name of the queue, which we are listening to. When a new message arrives, this method
will be invoked with 2 arguments in order: `message payload` and `message control`. The `message payload` is a Data Transfer Object (DTO)
class's instance. The task of this DTO class is to describe the payload's data structure for validation and transformation. We do this with
decorators. The second argument is the `message control` object which gives control how to handle the message transfer.

The `@Listen()` decorator has a second argument which is the options object. The options object can have these **optional** properties:
* **type**?: A class reference which is decorated with [class-transformer](https://www.npmjs.com/package/class-transformer) and
  [class-validator](https://www.npmjs.com/package/class-validator) decorators. The data validation and transformation will be done
  with this class.
  > Note: if `type` is not provided, the incoming message will not be processed, and the handler will not receive the message, but `null` instead
* **skipValidation**?: if it is `true` then the message payload won't be validated and transformed by the `type` property's value. The
  default value is `false`.
* **acceptValidationNullObjectException**?: A boolean value. If it's `true` then QueueModule will accept the message when a
`ValidationNullObjectException` error is thrown during message transformation and validation. (`ValidationNullObjectException` will be thrown when message body is null). Otherwise the
message will be rejected on `ValidationNullObjectException` error. Default value is `false`.
* **parallelMessageProcessing**?: The most number of messages that should be processed at the same time. The default value is `1`.
* **transformerOptions**?: [class-transformer options](https://github.com/typestack/class-transformer/blob/develop/src/interfaces/class-transformer-options.interface.ts)
* **validatorOptions**?: [class-validator options](https://github.com/typestack/class-validator#passing-options)

#### Multiple connections

Listening for messages with multiple connections works mostly the same, the only difference is that we need to define on which connection are we listening for messages. Here is an example:

```typescript
@Injectable()
export class ExampleListener {
  @Listen('example', Connections.INTERNAL)
  public async listenForInternalTicks(_tick: unknown, control: MessageControl): Promise<void> {
    console.log(`new tick arrived on the "example" queue on connection ${Connections.INTERNAL}:`, data);
  }

  @Listen('example', { type: PayloadDto }, Connections.WORKER)
  public async listenForWorkerMessages(data: PayloadDto, control: MessageControl): Promise<void> {
    console.log(`new message arrived on the "example" queue on connection ${Connections.WORKER}:`, data);
  }
}
```

> Note: It is assumed that there is an enum named `Connections`, and
the connections defined in the `forRoot` method contains a connection named `Connections.INTERNAL` and `Connections.WORKER`.

> Note: If you leave out the name of the connection, the listener will be attached to the default connection (if exists).


### Message control

When a new message arrives at a queue, the assigned method with `@Listen()` decorator receives the transformed and validated message body
and the message control. The latter object is to control the message transfer. It is possible to accept, reject or release the transfer.
Here are the examples:

```typescript
// accept the message
control.accept();

// reject the message
control.reject('processing failed');

// release the message
control.release();
```

Use `accept` when message has been handled normally. It will remove the message from the queue.
Use `reject` when message was unprocessable. It contained either malformed or semantically incorrect data. In other words
it can't be successfully processed in the future without modifications. It will remove the message from the queue.
Use `release` when a temporary problem happened during message handling, e.g. could not save record to DB, 3rd party service
errored, etc. The message is not malformed and theoretically can be processed at a later time without modifications. The
message will not be removed from the queue but will be processed by another consumer.

> If the message was not handled manually and the method with `@Listen()` decorator executed successfully then the message
will be accepted. If the message was not handled manually and there was an `Error` or `Exception` then the message will be rejected
automatically.

> Note relating to ActiveMQ: Active MQ current version (5.16.2 as of this writing) does not differentiate between `release` and `reject`, both will be understood as `release`.

### Payload validation and transformation

The module provides an opportunity for adding transformation and validation process to the message payload.
With transformation, we can set what properties are important for us and what properties should be skipped during the transformation.
With validation, we can check that the payload has the expected properties with valid values.

Here is an example DTO class:

```typescript
import { Exclude, Expose, Transform } from 'class-transformer';
import { IsIn, IsDateString, IsString } from 'class-validator';

@Exclude()
export class LogEntryDto {
  @Expose()
  @IsString()
  public readonly message: string;

  @Expose()
  @IsDateString()
  @Transform(value => new Date(value), { toClassOnly: true })
  public readonly date: Date;

  @Expose()
  @IsIn([1, 2, 3, 4, 5])
  public readonly level: number;

  constructor(props: Partial<LogEntryDto>) {
    Object.assign(this, props);
  }
}
```

The `@Expose()` decorator says that after the transformation, the properties with this decorator should stay in the `LogEntryDto` class
instance. The DTO class has an `@Exclude()` decorator which means that the properties that are not described in the class definition or
don't have `@Expose()` decorator will be removed from the resulting instance during the transformation . With these 2 decorators we can specify which
properties we want from the payload when the message arrives at the consumer side.

The `@Transform()` decorator is to manually cast or transform a value during the transformation process. In the example we can see that when the message arrives, the body will contain a JSON object which is converted to string.
The stringified JSON object has only primitive values so the `date` property will be a string
date. We want it as a `Date` object instance, so we manually transform it into a Date object with the `@Transform()` decorator. We have to notice that only when the payload will be transformed to the DTO object (i.e. when the message arrives at the consumer side) and not when
the object will be transformed to a string (i.e. before the sending on the producer side), because the DTO object will be represented as
a string in the message payload. The other decorators are to validate the property values.

An object payload's lifetime during the whole sending-receiving process:
```
                                        send                               receive
Message's way:     sender (producer)  --------->  queue (message broker)  --------->  receiver (consumer)
                                      transform                           transform
Payload's format:   DTO instance or   --------->          string          --------->     DTO instance
                   JavaScript object
```

You can see that the sender sends a DTO object instance or a plain object which will be converted to string and the message broker receives this string as message body.
When the consumer gets the message, the body is a string and the transformation process will transform it into a DTO instance.
Only one thing left: set this DTO class as message payload:

```typescript
@Listen('logsQueue', { type: LogEntryDto })
public async listenForLogsQueueMessages(data: LogEntryDto): Promise<void> {
  console.log('log entry:', data);
}
```

You can send any other primitive values as message payload but in this case you have to take care to disable the validation and
transformation process. Here is the example:

```typescript
@Listen('stringQueue', { type: String, skipValidation: true })
public async listenForStringQueueMessages(data: string): Promise<void> {
  console.log('message payload:', data);
}
```

> Note: Defining `type` is required if we want to receive primitive values as the message.
### Module logs

The module uses [debug](https://www.npmjs.com/package/debug) package for logging. If you start the Nest application with the `DEBUG`
environment variable, then you can set the level of logging:

```bash
# log everything
$ DEBUG=nest-amqp:* nest start

# log only the errors
$ DEBUG=nest-amqp:error:* nest start
```

### Shutdown

If you stop your application, then Nest will wait for the end of the currently processing messages and will exit after the processes
finished.

## Example

First you have to import the Queue module into the app module. The `QueueModule.forRoot()` method's first parameter
is the connection URI for the message broker server:

> Note: the `QueueModule.forRoot()` can be added only to the application's root module and only once

```typescript
import { QueueModule } from '@team-supercharge/nest-amqp';
// ...

@Module({
  imports: [
    QueueModule.forRoot('amqp://user:password@localhost:5672'),
    // ...
  ],
})
export class AppModule {}
```

Then create a `user.module.ts` feature module what will give all the functionality which belongs to the users.

> Note: the `QueueModule.forFeature()` module must be imported to each feature module of the application.

```typescript
import { Module } from '@nestjs/common';
import { QueueModule } from '@team-supercharge/nest-amqp';

@Module({
  imports: [QueueModule.forFeature()],
  controllers: [],
  providers: [],
})
export class UserModule {}
```

Import this `UserModule` in the `AppModule`.

After that create the `user.dto.ts` file and add a data transfer object (DTO) class to it which will be sent as body in the queue message.
The [class-transformer](https://www.npmjs.com/package/class-transformer) package is used to transform (rename, remove, alter type, etc...) the object properties before sending it to / after receiving it from the queue and
the [class-validator](https://www.npmjs.com/package/class-validator) package is used to validate the received object on the consumer side:

```typescript
import { Expose } from 'class-transformer';
import { IsInt, IsString } from 'class-validator';

@Expose()
export class AddUserDto {
  @IsString()
  public readonly name: string;

  @IsInt()
  public readonly age: number;

  constructor(userData: AddUserDto) {
    Object.assign(this, userData);
  }
}
```

After that create the `user.listener.ts` file and add  a new listener class to it which has a method
with the `@Listen()` decorator to listen the specified queue's messages:

> Note: you can add the `@Listen()` decorator for any class which have the `@Injectable()` decorator.

```typescript
import { Injectable } from '@nestjs/common';
import { Listen, MessageControl } from '@team-supercharge/nest-amqp';

import { AddUserDto } from './user.dto';

@Injectable()
export class UserListener {
  @Listen('addUser', { type: AddUserDto })
  public async listenForQueueNameMessages(data: AddUserDto, control: MessageControl): Promise<void> {
    console.log('new message arrived on the "addUser" queue:', data);
    control.accept();
  }
}
```

Then we create a `user.controller.ts` file and add a HTTP endpoint which will send the message to the queue with the payload what it gets as HTTP body:

```typescript
import { Body, Controller, Post } from '@nestjs/common';
import { QueueService } from '@team-supercharge/nest-amqp';

import { AddUserDto } from './user.dto';

@Controller('user')
export class UserController {
  constructor(private readonly queueService: QueueService) {}

  @Post()
  public async sendAddUserMessage(@Body() body: AddUserDto): Promise<string> {
    await this.queueService.send<AddUserDto>('addUser', body);

    return 'Add user message sent';
  }
}
```

We can see that the `send()` method is responsible to add a message to the given queue.

The last thing is to add this controller to the corresponding module:

```typescript
import { UserController } from './user.controller';
import { UserListener } from './user.listener';
// ...

@Module({
  controllers: [UserController],
  providers: [UserListener],
})
export class UserModule {}
```

Finally, start the app with `npm run start` and it will listen on http://localhost:4444 URL. You can test the functionality with sample HTTP requests which are in the `examples/01-Basic_Connection/http-requests/add-user.http` file.

## License

@team-supercharge/nest-amqp is [MIT licensed](LICENSE).
