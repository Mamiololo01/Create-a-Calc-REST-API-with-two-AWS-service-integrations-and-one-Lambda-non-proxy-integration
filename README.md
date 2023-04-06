# Create-a-Calc-REST-API-with-two-AWS-service-integrations-and-one-Lambda-non-proxy-integration
Create a Calc REST API with two AWS service integrations and one Lambda non-proxy integration

The Getting Started non-proxy integration tutorial uses Lambda Function integration exclusively. Lambda Function integration is a special case of the AWS Service integration type that performs much of the integration setup for you, such as automatically adding the required resource-based permissions for invoking the Lambda function. Here, two of the three integrations use AWS Service integration. In this integration type, you have more control, but you'll need to manually perform tasks like creating and specifying an IAM role containing appropriate permissions.

In this tutorial, you'll create a Calc Lambda function that implements basic arithmetic operations, accepting and returning JSON-formatted input and output. Then you'll create a REST API and integrate it with the Lambda function in the following ways:

By exposing a GET method on the /calc resource to invoke the Lambda function, supplying the input as query string parameters. (AWS Service integration)

By exposing a POST method on the /calc resource to invoke the Lambda function, supplying the input in the method request payload. (AWS Service integration)

By exposing a GET on nested /calc/{operand1}/{operand2}/{operator} resources to invoke the Lambda function, supplying the input as path parameters. (Lambda Function integration)

In addition to trying out this tutorial, you may wish to study the OpenAPI definition file for the Calc API, which you can import into API Gateway by following the instructions in Configuring a REST API using OpenAPI.

Topics

Create an assumable IAM role
Create a Calc Lambda function
Test the Calc Lambda function
Create a Calc API
Integration 1: Create a GET method with query parameters to call the Lambda function
Integration 2: Create a POST method with a JSON payload to call the Lambda function
Integration 3: Create a GET method with path parameters to call the Lambda function
OpenAPI definitions of sample API integrated with a Lambda function
Create an assumable IAM role
In order for your API to invoke your Calc Lambda function, you'll need to have an API Gateway assumable IAM role, which is an IAM role with the following trusted relationship:


{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "",
      "Effect": "Allow",
      "Principal": {
        "Service": "apigateway.amazonaws.com"
      },
      "Action": "sts:AssumeRole"
    }
  ]
}
The role you create will need to have Lambda InvokeFunction permission. Otherwise, the API caller will receive a 500 Internal Server Error response. To give the role this permission, you'll attach the following IAM policy to it:


{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": "lambda:InvokeFunction",
            "Resource": "*"
        }
    ]
}
Here's how to accomplish all this:

Create an API Gateway assumable IAM role
Log in to to the IAM console.

Choose Roles.

Choose Create Role.

Under Select type of trusted entity, choose AWS Service.

Under Choose the service that will use this role, choose Lambda.

Choose Next: Permissions.

Choose Create Policy.

A new Create Policy console window will open up. In that window, do the following:

In the JSON tab, replace the existing policy with the following:



{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": "lambda:InvokeFunction",
            "Resource": "*"
        }
    ]
}                        
        
Choose Review policy.

Under Review Policy, do the following:

For Name, type a name such as lambda_execute.

Choose Create Policy.

In the original Create Role console window, do the following:

Under Attach permissions policies, choose your lambda_execute policy from the dropdown list.

If you don't see your policy in the list, choose the refresh button at the top of the list. (Don't refresh the browser page!)

Choose Next:Tags.

Choose Next:Review.

For the Role name, type a name such as lambda_invoke_function_assume_apigw_role.

Choose Create role.

Choose your lambda_invoke_function_assume_apigw_role from the list of roles.

Choose the Trust relationships tab.

Choose Edit trust relationship.

Replace the existing policy with the following:



{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "",
      "Effect": "Allow",
      "Principal": {
        "Service": [
          "lambda.amazonaws.com",
          "apigateway.amazonaws.com"
        ]
      },
      "Action": "sts:AssumeRole"
    }
  ]
}                        
        
Choose Update Trust Policy.

Make a note of the role ARN for the role you just created. You'll need it later.

Create a Calc Lambda function
Next you'll create a Lambda function using the Lambda console.

In the Lambda console, choose Create function.

Choose Author from Scratch.

For Name, enter Calc.

Choose Create function.

Copy the following Lambda function and paste it into the code editor in the Lambda console.


export const handler = async function (event, context) {
  console.log("Received event:", JSON.stringify(event));

  if (
    event.a === undefined ||
    event.b === undefined ||
    event.op === undefined
  ) {
    return "400 Invalid Input";
  }

  const res = {};
  res.a = Number(event.a);
  res.b = Number(event.b);
  res.op = event.op;
  if (isNaN(event.a) || isNaN(event.b)) {
    return "400 Invalid Operand";
  }
  switch (event.op) {
    case "+":
    case "add":
      res.c = res.a + res.b;
      break;
    case "-":
    case "sub":
      res.c = res.a - res.b;
      break;
    case "*":
    case "mul":
      res.c = res.a * res.b;
      break;
    case "/":
    case "div":
      if (res.b == 0) {
        return "400 Divide by Zero";
      } else {
        res.c = res.a / res.b;
      }
      break;
    default:
      return "400 Invalid Operator";
  }

  return res;
};
Under Execution role, choose Choose an existing role.

Enter the role ARN for the lambda_invoke_function_assume_apigw_role role you created earlier.

Choose Deploy.

This function requires two operands (a and b) and an operator (op) from the event input parameter. The input is a JSON object of the following format:



{
  "a": "Number" | "String",
  "b": "Number" | "String",
  "op": "String"
}
        
This function returns the calculated result (c) and the input. For an invalid input, the function returns either the null value or the "Invalid op" string as the result. The output is of the following JSON format:



{
  "a": "Number",
  "b": "Number",
  "op": "String",
  "c": "Number" | "String"
}
        
You should test the function in the Lambda console before integrating it with the API in the next step.

Test the Calc Lambda function
Here's how to test your Calc function in the Lambda console:

In the Saved test events dropdown menu, choose Configure test events.

For the test event name, enter calc2plus5.

Replace the test event definition with the following:



{
  "a": "2",
  "b": "5",
  "op": "+"
}
        
Choose Save.

Choose Test.

Expand Execution result: succeeded. You should see the following:



{
  "a": 2,
  "b": 5,
  "op": "+",
  "c": 7
}
        
Create a Calc API
The following procedure shows how to create an API for the Calc Lambda function you just created. In subsequent sections, you'll add resources and methods to it.

Create the Calc API
Sign in to the API Gateway console at https://console.aws.amazon.com/apigateway.

If this is your first time using API Gateway, you see a page that introduces you to the features of the service. Under REST API, choose Build. When the Create Example API popup appears, choose OK.

If this is not your first time using API Gateway, choose Create API. Under REST API, choose Build.

Under Create new API, choose New API.

For API Name, enter LambdaCalc.

Leave the Description blank, and leave the Endpoint Type set to Regional.

Choose Create API.

Integration 1: Create a GET method with query parameters to call the Lambda function
By creating a GET method that passes query string parameters to the Lambda function, you enable the API to be invoked from a browser. This approach can be useful, especially for APIs that allow open access.

To set up the GET method with query string parameters
In the API Gateway console, under your LambdaCalc API's Resources, choose /.

In the Actions drop-down menu, choose Create Resource.

For Resource Name, enter calc.

Choose Create Resource.

Choose the /calc resource you just created.

In the Actions drop-down menu, choose Create Method.

In the method drop-down menu that appears, choose GET.

Choose the checkmark icon to save your choice.

In the Set up pane:

For Integration type, choose AWS Service.

For AWS Region, choose the region (e.g., us-west-2) where you created the Lambda function.

For AWS Service, choose Lambda.

Leave AWS Subdomain blank, because our Lambda function is not hosted on any AWS subdomain.

For HTTP method, choose POST and choose the checkmark icon to save your choice. Lambda requires that the POST request be used to invoke any Lambda function. This example shows that the HTTP method in a frontend method request can be different from the integration request in the backend.

Choose Use path override for Action Type. This option allows us to specify the ARN of the Invoke action to execute our Calc function.

Enter /2015-03-31/functions/arn:aws:lambda:region:account-id:function:Calc/invocations in Path override, where region is the region where you created your Lambda function and account-id is the account number for the AWS account.

For Execution role, enter the role ARN for the lambda_invoke_function_assume_apigw_role IAM role you created earlier.

Leave Content Handling set to Passthrough, because this method will not deal with any binary data.

Leave Use default timeout checked.

Choose Save.

Choose Method Request.

Now you will set up query parameters for the GET method on /calc so it can receive input on behalf of the backend Lambda function.

Choose the pencil icon next to Request Validator and choose Validate query string parameters and headers from the drop-down menu. This setting will cause an error message to return to state the required parameters are missing if the client does not specify them. You will not get charged for the call to the backend.

Choose the checkmark icon to save your changes.

Expand the URL Query String Parameters section.

Choose Add query string.

For Name, type operand1.

Choose the checkmark icon to save the parameter.

Repeat the previous steps to create parameters named operand2 and operator.

Check the Required option for each parameter to ensure that they are validated.

Choose Method Execution and then choose Integration Request to set up the mapping template to translate the client-supplied query strings to the integration request payload as required by the Calc function.

Expand the Mapping Templates section.

Choose When no template matches the request Content-Type header for Request body passthrough.

Under Content-Type, choose Add mapping template.

Type application/json and choose the checkmark icon to open the template editor.

Choose Yes, secure this integration to proceed.

Copy the following mapping script into the mapping template editor:


{
    "a":  "$input.params('operand1')",
    "b":  "$input.params('operand2')", 
    "op": "$input.params('operator')"   
}
This template maps the three query string parameters declared in Method Request into designated property values of the JSON object as the input to the backend Lambda function. The transformed JSON object will be included as the integration request payload.

Choose Save.

Choose Method Execution.

You can now test your GET method to verify that it has been properly set up to invoke the Lambda function:

For Query Strings, type operand1=2&operand2=3&operator=+.

Choose Test.

The results should look similar to this:


                                Create an API in API Gateway as a Lambda proxy
                            
Integration 2: Create a POST method with a JSON payload to call the Lambda function
By creating a POST method with a JSON payload to call the Lambda function, you make it so that the client must provide the necessary input to the backend function in the request body. To ensure that the client uploads the correct input data, you'll enable request validation on the payload.

To set up the POST method with a JSON payload to invoke a Lambda function
Go to the API Gateway console.

Choose APIs.

Choose the LambdaCalc API you created previously.

Choose the /calc resource from Resources pane.

In the Actions menu, choose Create Method.

Choose POST from the method drop-down list.

Choose the checkmark icon to save your choice.

In the Set up pane:

For Integration type, choose AWS Service.

For AWS Region, choose the region (e.g., us-west-2) where you created the Lambda function.

For AWS Service, choose Lambda.

Leave AWS Subdomain blank, because our Lambda function is not hosted on any AWS subdomain.

For HTTP method, choose POST. This example shows that the HTTP method in a frontend method request can be different from the integration request in the backend.

For Action, choose Use path override for Action Type. This option allows you to specify the ARN of the Invoke action to execute your Calc function.

For Path override, type /2015-03-31/functions/arn:aws:lambda:region:account-id:function:Calc/invocations , where region is the region where you created your Lambda function and account-id is the account number for the AWS account.

For Execution role, enter the role ARN for the lambda_invoke_function_assume_apigw_role IAM role you created earlier.

Leave Content Handling set to Passthrough, because this method will not deal with any binary data.

Leave Use Default Timeout checked.

Choose Save.

Choose Models under your LambdaCalc API in the API Gateway console's primary navigation pane to create data models for the method's input and output:

Choose Create in the Models pane. Type Input in Model name, type application/json in Content type, and copy the following schema definition into the Model schema box:


{
    "type":"object",
    "properties":{
        "a":{"type":"number"},
        "b":{"type":"number"},
        "op":{"type":"string"}
    },
    "title":"Input"
}
This model describes the input data structure and will be used to validate the incoming request body.

Choose Create model.

Choose Create in the Models pane. Type Output in Model name, type application/json in Content type, and copy the following schema definition into the Model schema box:


{
    "type":"object",
    "properties":{
        "c":{"type":"number"}
    },
    "title":"Output"
}
This model describes the data structure of the calculated output from the backend. It can be used to map the integration response data to a different model. This tutorial relies on the passthrough behavior and does not use this model.

Locate the API ID for your API at the top of the console screen and make a note of it. It will appear in parentheses after the API name.

In the Models pane, choose Create.

Type Result in Model name.

Type application/json in Content type.

Copy the following schema definition, where restapi-id is the REST API ID you noted earlier, into the Model schema box:


{
    "type":"object",
    "properties":{
        "input":{
            "$ref":"https://apigateway.amazonaws.com/restapis/restapi-id/models/Input"
        },
        "output":{
            "$ref":"https://apigateway.amazonaws.com/restapis/restapi-id/models/Output"
        }
    },
    "title":"Output"
}
This model describes the data structure of the returned response data. It references both the Input and Output schemas defined in the specified API (restapi-id). Again, this model is not used in this tutorial because it leverages the passthrough behavior.

Choose Create model.

In the main navigation pane, under your LambdaCalc API, choose Resources.

In the Resources pane, choose the POST method for your API.

Choose Method Request.

In the Method Request configuration settings, do the following to enable request validation on the incoming request body:

Choose the pencil icon next to Request Validator to choose Validate body. Choose the checkmark icon to save your choice.

Expand the Request Body section, choose Add model

Type application/json in the Content-Type input field and choose Input from the drop-down list in the Model name column. Choose the checkmark icon to save your choice.

To test your POST method, do the following:

Choose Method Execution.

Choose Test.

Copy the following JSON payload into the Request Body:



{
    "a": 1,
    "b": 2,
    "op": "+"
}
                
Choose Test.

You should see the following output in Response Body:



{
  "a": 1,
  "b": 2,
  "op": "+",
  "c": 3
}                    
                
Integration 3: Create a GET method with path parameters to call the Lambda function
Now you'll create a GET method on a resource specified by a sequence of path parameters to call the backend Lambda function. The path parameter values specify the input data to the Lambda function. You'll use a mapping template to map the incoming path parameter values to the required integration request payload.

This time you'll use the built-in Lambda integration support in the API Gateway console to set up the method integration.

The resulting API resource structure will look like this:


                Create an API in API Gateway as a Lambda proxy
            
To set up a GET method with URL path parameters
Go to the API Gateway console.

Under APIs, choose the LambdaCalc API you created previously.

In the API's Resources navigation pane, choose /calc.

From the Actions drop-down menu, choose Create Resource.

For Resource Name, type {operand1}.

For Resource Path, type {operand1}.

Choose Create Resource.

Choose the /calc/{operand1} resource you just created.

From the Actions drop-down menu, choose Create Resource.

For Resource Name, type {operand2}.

For Resource Path, type {operand2}.

Choose Create Resource.

Choose the /calc/{operand1}/{operand2} resource you just created.

From the Actions drop-down menu, choose Create Resource.

For Resource Path, type {operator}.

For Resource Name, type {operator}.

Choose Create Resource.

Choose the /calc/{operand1}/{operand2}/{operator} resource you just created.

From the Actions drop-down menu, choose Create Method.

From the method drop-down menu, choose GET.

In the Setup pane, choose Lambda Function for Integration type, to use the streamlined setup process enabled by the console.

Choose a region (e.g., us-west-2) for Lambda Region. This is the region where the Lambda function is hosted.

Choose your existing Lambda function (Calc) for Lambda Function.

Choose Save and then choose OK to consent to Add Permissions to Lambda Function.

Choose Integration Request.

Set up the mapping template as follows:

Expand the Mapping Templates section.

Leave set to When no template matches the requested Content-Type header.

Choose Add mapping template.

Type application/json for Content-Type and then choose the check mark icon to open the template editor.

Choose Yes, secure this integration to proceed.

Copy the following mapping script into the template editor:


{
   "a": "$input.params('operand1')",
   "b": "$input.params('operand2')",
   "op": #if($input.params('operator')=='%2F')"/"#{else}"$input.params('operator')"#end
   
}
This template maps the three URL path parameters, declared when the /calc/{operand1}/{operand2}/{operator} resource was created, into designated property values in the JSON object. Because URL paths must be URL-encoded, the division operator must be specified as %2F instead of /. This template translates the %2F into '/' before passing it to the Lambda function.

Choose Save.

When the method is set up correctly, the settings should look more or less like this:


                        Set up the GET method with path parameters to invoke
                            the Lambda function
                    
To test your GET function, do the following:

Choose Method Execution.

Choose Test.

Type 1, 1 and + in the {operand1}, {operand2} and {operator} fields, respectively.

Choose Test.

The result should look like this:


                                Map a method request URL path parameters to the integration
                                    request payload to call the Lambda function
                            
This test result shows the original output from the backend Lambda function, as passed through the integration response without mapping, because there is no mapping template. Next, you'll model the data structure of the method response payload after the Result schema.

By default, the method response body is assigned an empty model. This will cause the integration response body to be passed through without mapping. However, when you generate an SDK for one of the strongly-type languages, such as Java or Objective-C, your SDK users will receive an empty object as the result. To ensure that both the REST client and SDK clients receive the desired result, you must model the response data using a predefined schema. Here you'll define a model for the method response body and to construct a mapping template to translate the integration response body into the method response body.

Choose /calc/{operand1}/{operand2}/{operator}.

Choose GET.

Choose Method Execution.

Choose Method Response.

Expand the 200 response,

Under Response Body for 200, choose the pencil icon next to the model for the application/json content type.

From the Models drop-down list, choose Result.

Choose the checkmark icon to save your choice.

Setting the model for the method response body ensures that the response data will be cast into the Result object of a given SDK. To make sure that the integration response data is mapped accordingly, you'll need a mapping template.

To create the mapping template, do the following:

Choose Method Execution.

Choose Integration Response and expand the 200 method response entry.

Expand the Mapping Templates section.

Choose application/json from the Content-Type list.

Choose Result from the Generate template drop-down list to bring up the Result template blueprint.

Change the template blueprint to the following:


#set($inputRoot = $input.path('$'))
{
  "input" : {
    "a" : $inputRoot.a,
    "b" : $inputRoot.b,
    "op" : "$inputRoot.op"
  },
  "output" : {
    "c" : $inputRoot.c
  }
}
Choose Save.

To test the mapping template, do the following:

Choose Method Execution.

Choose Test.

Type 1 2 and + in the operand1, operand2 and operator input fields, respectively.

The integration response from the Lambda function is now mapped to a Result object.

Choose Test, and you'll see the following under Response Body in the console:


{
  "input": {
    "a": 1,
    "b": 2,
    "op": "+"
  },
  "output": {
    "c": 3
  }
}
At this point the API can be called only via Test Invoke in the API Gateway console. To make it available to clients, you'll need to deploy it as follows:

Choose Deploy API from the Actions dropdown menu.

Choose [New Stage] from the Deployment Stage dropdown menu.

For Stage Name, enter test.

Choose Deploy.

Note the Invoke URL at the top of the console window. You can use this with tools such as Postman and cURL to test your API.


