# LambdaSharp.Chat - A Chat app built with Blazor WebAssembly, AWS Cognito, WebSockets, DynamoDB, and Lambda Functions

[This sample requires the LambdaSharp CLI to deploy.](https://lambdasharp.net/)

## Overview

This LambdaSharp module creates a chat application built with [ASP.NET Core Blazor WebAssembly](https://docs.microsoft.com/en-us/aspnet/core/blazor/get-started) for the front-end, [Amazon Cognito](https://aws.amazon.com/cognito/) for user authentication, [API Gateway WebSocket](https://aws.amazon.com/blogs/compute/announcing-websocket-apis-in-amazon-api-gateway/) for communication, [Amazon DynamoDB](https://aws.amazon.com/dynamodb/) for storage using a single-table design, and [AWS Lambda](https://aws.amazon.com/lambda/) for the business logic written in C#. Finally, the application is delivered as as self-contained [AWS CloudFormation](https://aws.amazon.com/cloudformation/) template.

> **NOTE:** This LambdaSharp module requires .NET Core 3.1.300 and LambdaSharp.Tool 0.8.0.5, or later.

![LambdaSharp.Chat](Assets/LambdaSharpWebChat.png)

## Deploy LambdaSharp.Chat

_LambdaSharp.Chat_ requires an [AWS account](https://aws.amazon.com/) and a LambdaSharp deployment tier. Follow the [_Getting Started_](https://lambdasharp.net/articles/Setup.html) instructions for the initial setup.

> **NOTE:** Creating the CloudFront distribution takes up to 5 minutes. Granting permission for CloudFront to access the private S3 bucket can take up to an hour! Please be patient.

**To deploy the application directly:**

Use the LambdaSharp CLI to import and deploy the module to the deployment tier as follows:
```
lash deploy LambdaSharp.Chat:1.0-DEV@lambdasharp
```

**To build and deploy application from a git checkout:**

To check out the git repository, build it locally, and then deploy the application, follow these steps:
```
git clone https://github.com/LambdaSharp/LambdaSharp-Chat.git
cd LambdaSharp-Chat
lash deploy
```

### Deployment Details

The following resources are created by CloudFormation during the deployment:

1. A DynamoDB table with a secondary index for persistent storage.
1. A Cognito User Pool to manage users.
1. An API Gateway WebSocket to connect the front-end and back-end.
1. A Lambda function to authenticate WebSocket connections.
1. A Lambda function to handle WebSocket requests.
1. A Lambda function to broadcast messages to all open connections.
1. A private S3 bucket for hosting the application front-end assets.
1. A custom resource top copy the _wwwroot_ files to the S3 bucket using [brotli compression](https://en.wikipedia.org/wiki/Brotli).
1. A CloudFront distribution to enable caching and `https://` access to the front-end assets.
1. A Lambda function to invalidate cached assets in the CloudFront distribution when they get updated.
1. A custom resource to generate the `cognito.json` file with the Cognito configuration.

> **NOTE:** Creating the CloudFront distribution takes up to 5 minutes. Granting permission to CloudFront to access the private S3 bucket can take up to an hour!

## ASP .NET Core Blazor WebAssembly

The _BlazorWebSocket_ folder contains the front-end Blazor code, which integrates with the WebSocket and Cognito. The front-end files are served by an S3 bucket that is edge-accelerated and secured by a [Amazon CloudFront](https://aws.amazon.com/cloudfront/) distribution. A Lambda function monitors the S3 bucket and automatically invalidates the CloudFront cache when files are modified by a fresh deployment.

Information about the Cognito User Pool is transferred to the front-end application via a JSON file generated by a custom CloudFormation resource during deployment. The JSON file is copied to the same S3 bucket as the Blazor files. Once the Blazor application starts, it fetches the Cognito configuration from the `/cognito.json` location.

## API Gateway WebSocket

To open a WebSocket connection, the front-end must supply a valid JWT authentication token obtained from Cognito. The token is validated by the _Authorization::JwtAuthorizer_ Lambda function during the connection attempt.

During the build phase, LambdaSharp extracts the message schema from the C# implementation and uses it to configure the API Gateway WebSocket instance. If an incoming message does not conform to the expected schema of the WebSocket route, API Gateway will automatically reject it before it reaches the Lambda function.

```yaml
- Function: ChatFunction
  Description: Handle web-socket messages
  Memory: 256
  Timeout: 30
  Sources:

    - WebSocket: $connect
      Invoke: OpenConnectionAsync
      AuthorizationType: CUSTOM
      AuthorizerId: !Ref Authorization::JwtAuthorizer

    - WebSocket: $disconnect
      Invoke: CloseConnectionAsync

    - WebSocket: send
      Invoke: SendMessageAsync
```

Defining the JSON schema for the web-socket route doesn't require any special effort.

```csharp
public abstract class AMessageRequest {

    //--- Properties ---
    public string Action { get; set; }
}

public class SendMessageRequest : AMessageRequest {

    //--- Constructors ---
    public SendMessageRequest() => Action = "send";

    //--- Properties ---
    public string ChannelId { get; set; }
    public string Text { get; set; }
}
```

Sample Payload

```json
{
  "Action": "send",
  "ChannelId": "General",
  "Text": "Hello world!"
}
```

## User Interface Flow

1. Show splash screen in `index.html`
1. Continue showing the same splash screen when `Index.razor` loads
1. Check if we have a JWT token stored.
  1. If we do, attempt to log in with it. (optional: check if it has expired)
  1. If login is successful, then prepare to show the full interface
1. If we don't have JWT token or we failed to login with the one we had (probably b/c it's expired), show a `Login` button
1. Button redirects to Cognito login form
1. Cognito redirects back to Blazor app with `id_token=JWT` in URI fragment
1. Store JWT in local storage
1. Log into WebSocket

## DynamoDB Table Design

The application uses a single DynamoDB table to store all records and their relationships.

### User Record

Every user has exactly one user record associated with them. Each user is uniquely identified by the value in the `UserId` column. The user name can be customized by the user and may not be unique across all users.

The primary index is used to resolve user records by `UserId`.

The secondary index is used to list all existing users.

|Column       |Value
|-------------|------------------
|PK           |"USER#{UserId}"
|SK           |"INFO"
|GS1PK        |"USERS"
|GS1SK        |"USER#{UserId}"
|UserId       |String
|UserName     |String


### Connection Record

A connection record is created by a new connection is opened wit a user. A user can have multiple, simultaneous connections active. Each connection is uniquely identified by the value in the `ConnectionId` column.

The primary index is used to resolve connections by `ConnectionId`.

The secondary index is used to find all open connections per `UserId`.

|Column       |Value
|-------------|------------------
|PK           |"WS#{ConnectionId}"
|SK           |"INFO"
|GS1PK        |"USER#{UserId}"
|GS1SK        |"WS#{ConnectionId}"
|ConnectionId |String
|UserId       |String

### Channel Record

The channel record is created for each channel. The `Finalizer` ensures that the `General` channel always exists by default. Each channel is uniquely identified by the value in the `ChannelId` column.

The primary index is used to resolve channels by `ChannelId`.

The secondary index is used to find all existing channels.

|Column       |Value
|-------------|------------------
|PK           |"ROOM#{ChannelId}"
|SK           |"INFO"
|GS1PK        |"CHANNELS"
|GS1SK        |"ROOM#{ChannelId}"
|ChannelId    |String
|ChannelName  |String

### Subscription Record

The subscription record is created when a user subscribes to a channel. The value in the `LastSeenTimestamp` column is the UNIX epoch timestamp in milliseconds for the last message seen by the user in the given channel.

The primary index is used for finding all users subscribed by `ChannelId`.

The secondary index is used fod finding all channels subscribed by `UserId`.

|Column           |Value
|-----------------|------------------
|PK               |"ROOM#{ChannelId}"
|SK               |"USER#{UserId}"
|GS1PK            |"USER#{UserId}"
|GS1SK            |"ROOM#{ChannelId}"
|ChannelId        |String
|UserId           |String
|LastSeenTimestamp|Number

### Message Record

The message record is created for each message sent by a user on a channel. The value in the `Timestamp` column is the UNIX epoch timestamp in milliseconds when the message was sent by the user. The value in the `Jitter` column is used to minimize risk of row conflicts, in case a user sends two messages at the same time.

|Column       |Value
|-------------|------------------
|PK           |"ROOM#{ChannelId}"
|SK           |"WHEN#{Timestamp:0000000000000000}#{Jitter}"
|UserId       |String
|ChannelId    |String
|Timestamp    |Number
|Message      |String
|Jitter       |String

## Chat Protocol

### Requests

|Name                 |Description
|---------------------|-----------------------
|CreateChannel        |Create a new channel and join it
|RenameUser           |Change the current user name
|SendMessage          |Send a message on a chat channel

### Notifications

|Name                 |Description
|---------------------|-----------------------
|JoinedChannel        |Received by subscribers of a channel when a user joins
|UserNameChanged      |Received by all clients when a user changes their name
|Welcome              |Received by each client when they first connect

## Future Improvements
- [x] Allow users to rename themselves.
- [x] Remember a user's name from a previous session using local storage.
- [x] Restrict access to S3 bucket to only allow CloudFront.
- [x] Show previous messages when a user connects.
- [x] Route WebSocket requests via CloudFront.
- [x] Allow users to create or join chat rooms.
- [x] Add UI for logging in.
- [x] Add Cognito user pool for user management.
- [ ] Improve login experience.
- [ ] Add logout experience.
- [ ] Create user record at sign-in time with custom username (Cognito sign-up flow).
- [ ] Enhance user interface with Blazor UI components.
  - https://blazorise.com/
  - https://blazor.radzen.com/
- [ ] Enhance Cognito login user interface.
- [ ] Enhance Splash graphic by changing colors based on "Loading, Connecting, Connected" state sequence.
- [ ] Add user interface for reporting errors/warnings/etc to the user.
- [ ] Automatically refresh tokens and reconnect websocket in the background.
- [ ] Allows users to create and join rooms.
- [ ] Improve chat protocol to not send message during `$connect` route since the socket is not open yet.
- [ ] Secure WebSocket so they must come through CloudFront.
- [ ] Improve fan-out mechanism for sending messages to open connections.
- [ ] Create a "Website" group in module to encapsulate the S3 bucket, the CloudFront distribution, the cache invalidation Lambda function.
- [ ] Add diagram for data record relationships.
- [ ] Describe message protocol between front-end and back-end.
- [ ] Define WebSocket sub-protocol

## Contributors

Many thanks to our contributors (in alphabetical order):

* [@bjorg](https://github.com/bjorg)
* [@JuansonGrajales](https://github.com/JuansonGrajales)
* [@onema](https://github.com/onema)
* [@pattyr](https://github.com/pattyr)
* [@yurigorokhov](https://github.com/yurigorokhov)

## Acknowledgements

This LambdaSharp module is a port of the [netcore-simple-websockets-chat-app](https://github.com/normj/netcore-simple-websockets-chat-app) sample for AWS Lambda. For more information [Announcing WebSocket APIs in Amazon API Gateway](https://aws.amazon.com/blogs/compute/announcing-websocket-apis-in-amazon-api-gateway/) blog post.

Inspiration for the chat logic was taken from [Node.js & WebSocket — Simple chat tutorial](https://medium.com/@martin.sikora/node-js-websocket-simple-chat-tutorial-2def3a841b61).

## License

_Apache 2.0_ for the module and code.
