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