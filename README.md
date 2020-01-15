# Serverless authentication example

This is a simple PoC using the [Serverless Framework](https://serverless.com/) to create an AWS Lambda solution which authenticates a user before allowing access.

## Requires

* NodeJS
* AWS account and AWS CLI with access token setup in `~/.aws`

## Deploy

1. Create the AWS resources and deploy the API Gateway/Lambda:

```
cd lambdas
serverless deploy -v
```

From the `Stack Outputs` get the `ServiceEndpoint`, `UserPoolClientId` and `UserPoolId`.

2. Test calling the endpoint unauthenticated to get error

```
curl -v SERVICE_ENDPOINT # should give 403 response `{"message":"Missing Authentication Token"}`
```

3. Create a test user in Cognito:

```
aws cognito-idp sign-up \
  --region COGNITO_REGION \
  --client-id USERPOOL_CLIENT_ID \
  --username admin@example.com \
  --password Passw0rd!
```

Record the `client-id`

4. Confirm the test user in Cognito to activate it:

```
aws cognito-idp admin-confirm-sign-up \
  --region COGNITO_REGION \
  --user-pool-id USER_POOL_ID \
  --username admin@example.com
```

5. Test calling the API as authenticated user

The command below uses [npx](https://www.npmjs.com/package/npx) and a module [aws-api-gateway-cli-test](https://www.npmjs.com/package/aws-api-gateway-cli-test) to authenticate with your test user to get a temporary token and calls the API with the token in the header. You can do this via AWS API and CLI but will need to make multiple calls and calculate values, see AWS docs [here](https://docs.aws.amazon.com/cognito/latest/developerguide/amazon-cognito-user-pools-authentication-flow.html) and [here](https://docs.aws.amazon.com/apigateway/latest/developerguide/apigateway-invoke-api-integrated-with-cognito-user-pool.html) for details.

```
npx aws-api-gateway-cli-test \
--username='admin@example.com' \
--password='Passw0rd!' \
--user-pool-id='USER_POOL_ID' \
--app-client-id='USER_POOL_CLIENT_ID' \
--cognito-region='COGNITO_REGION' \
--identity-pool-id='IDENTITY_POOL_ID' \
--invoke-url='API_GATEWAY_URL' \
--api-gateway-region='API_GATEWAY_REGION' \
--path-template='' \
--method='GET'
# should get response `{ status: 200, statusText: 'OK', data: 'Authenticated!' }`
```

The Lambda will have access to the details of the cognito user in the `event.requestContext.identity` object.

## Notes

This PoC uses the basic API Gateway Cognito User Pool Authorizer, see [here](https://docs.aws.amazon.com/apigateway/latest/developerguide/apigateway-control-access-to-api.html) for alternatives.

The advantage of this approach is that it is very quick and easy to setup, API Gateway manages authenticating requests and Cognito manages the users. You get the full set of Cognito functions for managing your users and only pay the cost for authentication calls and storage. The disadvantage is the authorization applies the whole API gateway and does not allow for role based access without customisation inside your Lambda code. You are also locked into Cognito, as it will be difficult to export user data and introduce a new authentication system.

The main supported alternative is using a Lambda as a custom authorizer, but this has other disadvantages, such as increased execution costs and performance (see the linked blog for details).

## Links

* [AWS documentation](https://docs.aws.amazon.com/apigateway/latest/developerguide/apigateway-control-access-to-api.html)
* Example guide [serverless stack](https://github.com/AnomalyInnovations/serverless-stack-com)
* [Blog - Guide to custom authorizers](https://www.alexdebrie.com/posts/lambda-custom-authorizers/)