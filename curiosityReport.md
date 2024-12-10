# Curiosity Report

**AWS CloudFormation and API Gateway**

During the difficult process of manually setting up API Gateway for another class, I decided to look into how I would go about automating that process with AWS CloudFormation.

Relevant resources appear under the AWS::ApiGateway category.
For example, AWS::ApiGateway::RestApi and AWS::ApiGateway::Resource

The AWS CloudFormation documentation contains information on these resources that you can create.

I had to double check the CloudFormation templates that we used in class to refresh my understanding of how CloudFormation works. I also looked up what the Outputs section does, as I hadn't really understood in class what the purpose was (it seemed to just do the same thing as Resources). It's apparently used primarily for communication between stacks, and to capture important data for ease of use (for example, outputting a bucket name to make it easier to find).

Next, I went back to what my project expected. We had been working with API Gateway and Lambdas, both of which can be automated with AWS CloudFormation, but I wanted to focus on API Gateway for now. The way I had set mine up, each lambda had a separate path at the same level of depth, that is, I didn't use nested paths, but if I were redoing it, I would have done it differently (I didn't want to risk messing up my toilsome hard work over that, which wouldn't have been a problem if it had been automated).
Each endpoint needed
  - A resource/actual endpoint
  - Lambda integration
  - Error code handling
  - Documentation
  - CORS handling

I decided to figure this out piecemeal through experimentation. I started by making a Rest API. I went into the builder for making it manually, to see what information I had been providing manually.

I ended up with this JSON:
```
{
  "AWSTemplateFormatVersion": "2010-09-09",
  "Description": "Automated Tweeter API",
  "Parameters": {
    "APIName": {
      "Type": "String",
      "Description": "What to name the automated tweeter API",
      "Default": ""
    }
  },
  "Resources": {
    "API": {
      "Type": "AWS::ApiGateway::RestApi",
      "Properties": {
        "Name": {
          "Ref": "APIName"
        },
        "Description": "Automated Tweeter API",
        "Parameters": {
          "endpointConfigurationTypes": "REGIONAL"
        }
      }
    }
  }
}
```

After testing, this successfully created an empty Rest API, without specific deletion policy or outputs.
While working on the next part (the resources), I noticed two things. Firstly, I had my first CloudFormation error to fix, and secondly, resources don't have the same ability to enable CORS as they do when you manually create them. Presumably, this means that you have to enable CORS later on in the process.

The error I received was this:
This AWS::ApiGateway::Resource resource is in a CREATE_FAILED state.

Resource handler returned message: "Invalid API identifier specified 750056739266:APIID"

I was able to use CloudTrail to figure out the problem (I was passing APIID, not a ref to it), and tried to move on. I had more problems, and had to look up the error on the Google. I found this snippet of code
```
"ResourceId": {
  "Fn::GetAtt": [
    "MyApi",
    "RootResourceId"
  ]
}
```
And adapted it, bug tested some more, and finally got a working API structure:
```
{
  "AWSTemplateFormatVersion": "2010-09-09",
  "Description": "Automated Tweeter API",
  "Parameters": {
    "APIName": {
      "Type": "String",
      "Description": "What to name the automated tweeter API",
      "Default": ""
    }
  },
  "Resources": {
    "API": {
      "Type": "AWS::ApiGateway::RestApi",
      "Properties": {
        "Name": {
          "Ref": "APIName"
        },
        "Description": "Automated Tweeter API",
        "Parameters": {
          "endpointConfigurationTypes": "REGIONAL"
        }
      }
    },
    "AuthPath": {
      "Type": "AWS::ApiGateway::Resource",
      "Properties": {
        "ParentId": {
          "Fn::GetAtt": [
            "API",
            "RootResourceId"
          ]
        },
        "PathPart": "auth",
        "RestApiId": {
          "Ref": "API"
        }
      },
      "DependsOn": ["API"]
    },
    "LoginPath": {
      "Type": "AWS::ApiGateway::Resource",
      "Properties": {
        "ParentId": {
          "Ref": "AuthPath"
        },
        "PathPart": "login",
        "RestApiId": {
          "Ref": "API"
        }
      },
      "DependsOn": ["API", "AuthPath"]
    },
    "LogoutPath": {
      "Type": "AWS::ApiGateway::Resource",
      "Properties": {
        "ParentId": {
          "Ref": "AuthPath"
        },
        "PathPart": "logout",
        "RestApiId": {
          "Ref": "API"
        }
      },
      "DependsOn": ["API", "AuthPath"]
    }
  },
  "Outputs": {
    "APIID": {
      "Description": "ID for the API",
      "Value": {
        "Ref": "API"
      }
    }
  }
}
```
This creates a Rest API with four resources:
  - /
  - /auth
  - /auth/login
  - /auth/logout

At this point, time was starting to become an issue, and I figured I would need to cut down my proof of concept and save the rest for another time. I had already looked up documentation, but that didn't seem particularly interesting to me, so the last thing I would try to do was integrate a lambda with an endpoint. To do this I would need a method resource.

I had to do a bit of searching more more specific information/examples in dev blogs and other sections of the Amazon API than cloudfront. The way we did it in class was with Custom Lambda Integration, rather than Lambda Proxy Integration, so I had to find the right keyword from a different section.

At this point, I had the core of the proof of concept down. I knew generally what I had to do to get documentation and error codes down (the latter of which were a subsection of a method I didn't have time for, as was CORS, according to something I found online.) I was able to add a method, probably the most complicated resource, to my API, and get it created. The "finished" code is below; I think the core of what I set out to learn is there.
My key takeaways, aside from the knowledge to create Rest APIs via CloudFormation, are as follows:
 - I learned more comprehensively how CloudFormation works, as I was writing my own code, not just copying and pasting
 - I learned how to debug CloudFormation code, especially using CloudTrail, and how to read the documentation for CloudFormation resources
 - I learned that, to the contrary of what I had initially believed, doing this project manually (all 14 endpoints) probably took less time than if I had tried to do it automatically. Once I got a working proof of concept, it could be quite simple to add an arbitrary number of endpoints, and creation and deletion, or editing, would be much easier, but it is actually quite complicated to get CloudFormation to work, and for the school project, without needing my code to be highly resilient to unforeseen circumstances, and without it needing to be modular, it makes sense why we didn't learn CloudFormation for the assignment.

 The Finished Proof of Concept Code for 'createRESTAPI.json' :
 ```
{
  "AWSTemplateFormatVersion": "2010-09-09",
  "Description": "Automated Tweeter API",
  "Parameters": {
    "APIName": {
      "Type": "String",
      "Description": "What to name the automated tweeter API",
      "Default": ""
    }
  },
  "Resources": {
    "API": {
      "Type": "AWS::ApiGateway::RestApi",
      "Properties": {
        "Name": {
          "Ref": "APIName"
        },
        "Description": "Automated Tweeter API",
        "Parameters": {
          "endpointConfigurationTypes": "REGIONAL"
        }
      }
    },
    "AuthPath": {
      "Type": "AWS::ApiGateway::Resource",
      "Properties": {
        "ParentId": {
          "Fn::GetAtt": [
            "API",
            "RootResourceId"
          ]
        },
        "PathPart": "auth",
        "RestApiId": {
          "Ref": "API"
        }
      },
      "DependsOn": ["API"]
    },
    "LoginPath": {
      "Type": "AWS::ApiGateway::Resource",
      "Properties": {
        "ParentId": {
          "Ref": "AuthPath"
        },
        "PathPart": "login",
        "RestApiId": {
          "Ref": "API"
        }
      },
      "DependsOn": ["API", "AuthPath"]
    },
    "LogoutPath": {
      "Type": "AWS::ApiGateway::Resource",
      "Properties": {
        "ParentId": {
          "Ref": "AuthPath"
        },
        "PathPart": "logout",
        "RestApiId": {
          "Ref": "API"
        }
      },
      "DependsOn": ["API", "AuthPath"]
    },
    "LogoutMethod": {
      "Type": "AWS::ApiGateway::Method",
      "Properties": {
        "RestApiId": {
          "Ref": "API"
        },
        "ResourceId": {
          "Ref": "LogoutPath"
        },
        "HttpMethod": "POST",
        "AuthorizationType": "NONE",
        "Integration": {
          "Type": "AWS",
          "IntegrationHttpMethod": "POST",
          "Uri": "arn:aws:apigateway:us-east-1:lambda:path/2015-03-31/functions/arn:aws:lambda:us-east-1:750056739266:function:logout/invocations"
        }
      }
    }
  },
  "Outputs": {
    "APIID": {
      "Description": "ID for the API",
      "Value": {
        "Ref": "API"
      }
    }
  }
}
 ```