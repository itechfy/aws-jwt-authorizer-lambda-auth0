# AWS API Gateway Custom Authorizer for RS256 JWTs

An AWS API Gateway [Custom Authorizer](http://docs.aws.amazon.com/apigateway/latest/developerguide/use-custom-authorizer.html) that authorizes API requests by requiring
that the OAuth2 [bearer token](https://tools.ietf.org/html/rfc6750) is a JWT that can be validated using the RS256 (asymmetric) algorithm with a public key that is obtained from a [JWKS](https://tools.ietf.org/html/rfc7517) endpoint.

## About

### What is AWS API Gateway?

API Gateway is an AWS service that allows for the definition, configuration and deployment of REST API interfaces.
These interfaces can connect to a number of back-end systems.
One popular use case is to provide an interface to AWS Lambda functions to deliver a so-called 'serverless' architecture.

### What are "Custom Authorizers"?

In February 2016 Amazon
[announced](https://aws.amazon.com/blogs/compute/introducing-custom-authorizers-in-amazon-api-gateway/)
a new feature for API Gateway -
[Custom Authorizers](http://docs.aws.amazon.com/apigateway/latest/developerguide/use-custom-authorizer.html). This allows a Lambda function to be invoked prior to an API Gateway execution to perform custom authorization of the request, rather than using AWS's built-in authorization. This code can then be isolated to a single centralized Lambda function rather than replicated across every backend Lambda function.

### What does this Custom Authorizer do?

This package gives you the code for a custom authorizer that will, with a little configuration, perform authorization on AWS API Gateway requests via the following:

- It confirms that an OAuth2 bearer token has been passed via the `Authorization` header.
- It confirms that the token is a JWT that has been signed using the RS256 algorithm with a specific public key
- It obtains the public key by inspecting the configuration returned by a configured JWKS endpoint
- It also ensures that the JWT has the required Issuer (`iss` claim) and Audience (`aud` claim)

## Setup

Install Node Packages:

```bash
npm install
```

This is a prerequisite for deployment as AWS Lambda requires these files to be included in a bundle (a special ZIP file).

## Local testing

Configure the local environment by creating a `.env` file with following contents:

```bash
TOKEN_ISSUER=https://your-tenant.auth0.com/
JWKS_URI=https://your-tenant.auth0.com/.well-known/jwks.json
AUDIENCE=audience-of-token
```

### Environment Variables

Modify the `.env`:

- `TOKEN_ISSUER`: The issuer of the token. If you're using Auth0 as the token issuer, this would be: `https://your-tenant.auth0.com/`
- `JWKS_URI`: This is the URL of the associated JWKS endpoint. If you are using Auth0 as the token issuer, this would be: `https://your-tenant.auth0.com/.well-known/jwks.json`
- `AUDIENCE`: This is the required audience of the token. If you are using Auth0 as the Authorization Server, the audience value is the same thing as your API **Identifier** for the specific API in your [APIs section](<(https://manage.auth0.com/#/apis)>).

You can test the custom authorizer locally. You just need to obtain a valid JWT access token to perform the test. If you're using Auth0, see [these instructions](https://auth0.com/docs/tokens/access-token#how-to-get-an-access-token) on how to obtain one.

With a valid token, now you just need to create a local `event.json` file that contains it. Start by copying the sample file:

```bash
cp event.json.sample event.json
```

Then replace the `ACCESS_TOKEN` text in that file with the JWT you obtained in the previous step.

Finally, perform the test:

```bash
npm test
```

This uses the [lambda-local](https://www.npmjs.com/package/lambda-local) package to test the authorizer with your token. A successful test run will look something like this:

```
> lambda-local --timeout 300 --lambdapath index.js --eventpath event.json

Logs
----
START RequestId: fe210d1c-12de-0bff-dd0a-c3ac3e959520
{ type: 'TOKEN',
    authorizationToken: 'Bearer eyJ0eXA...M2pdKi79742x4xtkLm6qNSdDYDEub37AI2h_86ifdIimY4dAOQ',
    methodArn: 'arn:aws:execute-api:us-east-1:1234567890:apiId/stage/method/resourcePath' }
END


Message
------
{
    "principalId": "user_id",
    "policyDocument": {
        "Version": "2012-10-17",
        "Statement": [
            {
                "Sid": "Stmt1459758003000",
                "Effect": "Allow",
                "Action": [
                    "execute-api:Invoke"
                ],
                "Resource": [
                    "arn:aws:execute-api:*"
                ]
            }
        ]
    }
}
```

An `Action` of `Allow` means the authorizer would have allowed the associated API call to the API Gateway if it contained your token.

## Deployment

Now we're ready to deploy the custom authorizer to an AWS API Gateway.

### Create the lambda bundle

First we need to create a bundle file that we can upload to AWS:

```bash
npm run bundle
```

This will generate a local `custom-authorizer.zip` bundle (ZIP file) containing all the source, configuration and node modules an AWS Lambda needs.


### Test your API Gateway endpoint remotely

In the examples below:

- `INVOKE_URL` is the **Invoke URL** from the [Deploy the API Gateway](#deploy-the-api-gateway) section above
- `ACCESS_TOKEN` is the token in the `event.json` file
- `/your/resource` is the resource you secured in AWS API Gateway

#### With Postman

You can use Postman to test the REST API

- Method: (matching the Method in API Gateway, eg. `GET`)
- URL: `INVOKE_URL/your/resource`
- Headers tab: Add an `Authorization` header with the value `Bearer ACCESS_TOKEN`

#### With curl from the command line

```
curl -i "INVOKE_URL/your/resource" \
  -X GET \
  -H 'Authorization: Bearer ACCESS_TOKEN'
```

The above command is performed using the `GET` method.

#### In (modern) browsers console with fetch

```
fetch( 'INVOKE_URL/your/resource', { method: 'GET', headers: { Authorization : 'Bearer ACCESS_TOKEN' }}).then(response => { console.log( response );});
```

---
