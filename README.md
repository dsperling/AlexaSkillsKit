# AlexaSkillsKit
[![Build Status](https://travis-ci.org/choefele/AlexaSkillsKit.svg?branch=master)](https://travis-ci.org/choefele/AlexaSkillsKit)

AlexaSkillsKit is a Swift library that allows you to develop custom skills for [Amazon Alexa](https://developer.amazon.com/alexa), the voice service that powers Echo. It takes care of parsing JSON requests from Amazon, generating the proper responses and providing convenience methods to handle all other features that Alexa offers.

AlexaSkillsKit has been inspired by [alexa-app](https://github.com/matt-kruse/alexa-app), [SwiftOnLambda](https://github.com/algal/SwiftOnLambda) and [alexa-skills-kit-java](https://github.com/amzn/alexa-skills-kit-java).

It's early days – expect API changes until we reach 1.0!

## Implementing a Custom Alexa Skill

Start with implementing the `RequestHandler` protocol. AlexaSkillsKit parses requests from Alexa and passes the data on to methods required by this protocol. For example, a [launch request](https://developer.amazon.com/public/solutions/alexa/alexa-skills-kit/docs/custom-standard-request-types-reference#launchrequest) would result in AlexaSkillsKit calling the `handleLaunch()` method.

```
import Foundation
import AlexaSkillsKit

public class AlexaSkillHandler : RequestHandler {
    public init() {}
    
    public func handleLaunch(request: LaunchRequest, session: Session, next: @escaping (StandardResult) -> ()) {
        let standardResponse = generateResponse(message: "Alexa Skill received launch request")
        next(.success(standardResponse: standardResponse, sessionAttributes: session.attributes))
    }
    
    public func handleIntent(request: IntentRequest, session: Session, next: @escaping (StandardResult) -> ()) {
        let standardResponse = generateResponse(message: "Alexa Skill received intent \(request.intent.name)")
        next(.success(standardResponse: standardResponse, sessionAttributes: session.attributes))
    }
    
    public func handleSessionEnded(request: SessionEndedRequest, session: Session, next: @escaping (VoidResult) -> ()) {
        next(.success())
    }
    
    func generateResponse(message: String) -> StandardResponse {
        let outputSpeech = OutputSpeech.plain(text: message)
        return StandardResponse(outputSpeech: outputSpeech)
    }
}
```

In the request handler, your custom skill can implement any logic your skill requires. To enable asynchronous code (for example calling another HTTP service), the result is passed on via the `next` callback. `next` takes a enum that's either `.success` and contains an Alexa response or `.failure` in case a problem occurred.

## Deployment

You can run your custom skill on AWS Lambda using an [Alexa Skills Kit trigger](https://developer.amazon.com/public/solutions/alexa/alexa-skills-kit/docs/developing-an-alexa-skill-as-a-lambda-function) as well as on any other Swift server environment via [Alexa's HTTPS API](https://developer.amazon.com/public/solutions/alexa/alexa-skills-kit/docs/developing-an-alexa-skill-as-a-web-service). 

Using Lambda, Amazon will take care of scaling and running your Swift code. Lambda, however, doesn't support Swift executables natively thus the sample comes with a script that packages your Swift executable and its dependencies so it can be executed as a Node.js Lambda function.

A stand-alone server allows you to use alternate cloud providers and run multiple skills on the same server using any Swift web framework such as [Kitura](https://github.com/IBM-Swift/Kitura), [Vapor](https://github.com/vapor/vapor) or [Perfect](https://github.com/PerfectlySoft/Perfect). 

Even if you use Lambda for execution, configuring a server allows you to easily run and debug your custom skill in Xcode on a local computer. For this reason, the sample is configured to build both a Lambda executable as well as an HTTP server. You can use the same `RequestHandler` code in both cases.

### Lambda

```
import Foundation
import AlexaSkillsKit
import AlexaSkill

do {
    let data = FileHandle.standardInput.readDataToEndOfFile()
    let requestDispatcher = RequestDispatcher(requestHandler: AlexaSkillHandler())
    let responseData = try requestDispatcher.dispatch(data: data)
    FileHandle.standardOutput.write(responseData)
} catch let error as MessageError {
    let data = error.message.data(using: .utf8) ?? Data()
    FileHandle.standardOutput.write(data)
}
```

A sample for a custom skill using Lambda is provided in [Samples](https://github.com/choefele/AlexaSkillsKit/tree/master/Samples). You'll need [Xcode 8](https://developer.apple.com/xcode/) and [docker](https://www.docker.com/products/overview) to build it.

- Make sure the sample builds by running `swift build`. This will install AlexaSkillsKit in the Packages folder and build the executable for macOS
- Lambda executables take their input via `stdin` and provide output via `stdout` – you can try this out by piping a test file into the executable `swift build && cat ../../Tests/AlexaSkillsKitTests/launch_request.json | ./.build/debug/Lambda`
- Execute `./build-lambda-package.sh` to build the executable for Linux. The Swift compiler will run inside a docker container that provides a build environment for Linux
- This will result in a zip file at .build/lambda/lambda.zip that contains the executable, its libraries and a Node.js shim that provides an execution context.
- Create a new Lambda function in the [AWS Console](https://console.aws.amazon.com/lambda/home) in the US East (N. Virginia) region
 - Use an Alexa Skills Kit trigger
 - Runtime: NodeJS 4.3
 - Code entry type: ZIP file (upload the lambda.zip file from the previous step)
 - Handler: index.handler
 - Role: Create from template or use existing role
- After creating the Lambda function, you can now test your code using an Alexa test event in the AWS console
- To create an Alexa skill
 - Go to the [Alexa console](https://developer.amazon.com/edw/home.html#/skills/list) and create a new skill
 - Skill type: Custom Interaction Model
 - Intent: `{ "intents": [{"intent": "TestIntent"}]}`
 - Sample utterances: "TestIntent test swift"
 - Service endpoint type: AWS Lambda ARN (use the ARN from the AWS Console)
 
Now you can test the skill in the Alexa Console using the utterance "test swift". More details on configuring Alexa skills can be found on [Amazon's developer portal](https://developer.amazon.com/public/solutions/alexa/alexa-skills-kit/overviews/steps-to-build-a-custom-skill).

### Stand-Alone Server
_Coming soon_

```
import Foundation
import AlexaSkillsKit
import AlexaSkill
import Kitura

router.all("/") { request, response, next in
    var data = Data()
    let _ = try? request.read(into: &data)

    let requestDispatcher = RequestDispatcher(requestHandler: AlexaSkillHandler())
    requestDispatcher.dispatch(data: data) { result in
        switch result {
        case .success(let data):
            response.send(data: data).status(.OK)
        case .failure(let error):
            response.send(error.message).status(.badRequest)
        }
        
        next()
    }
}

Kitura.addHTTPServer(onPort: 8090, with: router)
Kitura.run()
```

## Supported Features
### Request Envelope
| Feature | Supported |
| --- | --- |
| version | yes |
| session | yes |
| context | |
| request | partially, see below |

### Requests
| Feature | Supported |
| --- | --- |
| LaunchRequest | yes |
| IntentRequest | yes |
| SessionEndedRequest | yes |
| AudioPlayer Requests | |
| PlaybackController Requests | |

### Response Envelope
| Feature | Supported |
| --- | --- |
| version | yes |
| sessionAttributes | yes |
| response | partially, see below |

### Response
| Feature | Supported |
| --- | --- |
| outputSpeech | partially (plain yes, SSML no) |
| card | yes |
| reprompt | yes |
| directives | |
| shouldEndSession | yes |

### Other
| Feature | Supported |
| --- | --- |
| Request handler | |
| Account Linking | |
| Multiple Languages | partially (locale attribute supported) |
| Response validation | |
| Request verification (stand-alone server) | |
