---
layout: post
title: Create note API
date: 2016-12-31 00:00:00
---

### Create Function Code

Create a new file create.js and paste the following code

{% highlight javascript %}
'use strict';

const uuid = require('uuid');
const AWS = require('aws-sdk');
const dynamoDb = new AWS.DynamoDB.DocumentClient();

module.exports.create = (event, context, callback) => {
  // Request body is passed in as a JSON encoded string in 'event.body'
  const data = JSON.parse(event.body);

  const params = {
    TableName: 'notes',
    Item: {
      // Use the federated identity ID of the authenticated user for 'userId'
      // which is in 'event.requestContext.authorizer.claims.sub'
      userId: event.requestContext.authorizer.claims.sub,
      // Generate a unique uuid for 'noteId'
      noteId: uuid.v1(),
      content: data.content,
      attachment: data.attachment,
      createdAt: new Date().getTime(),
    },
  };

  dynamoDb.put(params, (error, data) => {
    // Set response headers to enable CORS (Cross-Origin Resource Sharing)
    const headers = {
      'Access-Control-Allow-Origin': '*',
      "Access-Control-Allow-Credentials" : true,
    };

    // Return status code 500 on error
    if (error) {
      const response = {
        statusCode: 500,
        headers: headers,
        body: JSON.stringify({status: false}),
      };
      callback(null, response);
      return;
    }

    // Return status code 200 and newly created item in response body
    const response = {
      statusCode: 200,
      headers: headers,
      body: JSON.stringify(params.Item),
    }
    callback(null, response);
  });
};
{% endhighlight %}

### Configure API Endpoint

Open **serverless.yml** file and replace the content with following code

{% highlight yaml %}
service: notes-app-api

provider:
  name: aws
  runtime: nodejs4.3
  stage: prod
  region: us-east-1

  # iamRoleStatement defines the permission policy for the Lambda function.
  # In this case Lambda functions are granted with permissions to access DynamoDB.
  iamRoleStatements:
    - Effect: Allow
      Action:
        - dynamodb:DescribeTable
        - dynamodb:Query
        - dynamodb:Scan
        - dynamodb:GetItem
        - dynamodb:PutItem
        - dynamodb:UpdateItem
        - dynamodb:DeleteItem
      Resource: "arn:aws:dynamodb:us-east-1:*:*"

functions:
  create:
    # HTTP endpoint will trigger the create function in create.js
    handler: create.create
    events:
      - http:
          # path is /notes
          path: notes
          # request type is POST
          method: post
          # CORS (Cross-Origin Resource Sharing) enabled for browswer cross domain api call.
          cors: true
          # the api is authencated via the user pool we created in the previous tutorial.
          # use the user pool arn in place for arn:aws:cognito-idp:us-east-1:632240853321:userpool/us-east-1_KLsuR0TMI
          authorizer:
            arn: arn:aws:cognito-idp:us-east-1:632240853321:userpool/us-east-1_KLsuR0TMI
{% endhighlight %}

### Deploy

{% highlight bash %}
$ serverless deploy
{% endhighlight %}

Near the bottom of the output, you will find **Service Information**

{% highlight bash %}
service: notes-app-api
stage: prod
region: us-east-1
endpoints:
  POST - https://ly55wbovq4.execute-api.us-east-1.amazonaws.com/prod/notes
functions:
  notes-app-api-prod-create: arn:aws:lambda:us-east-1:632240853321:function:notes-app-api-prod-create
{% endhighlight %}

### Test

Now let's test the API we just created. Because the API is authencated via Cognito User Pool, we need to obtain the necessary identity token to include in the request Authorization header.

First generate the identity token. Replace the following parameters with those of the Cognito User Pool created in the earlier chapter:

- **--user-pool-id**: User pool id
- **--client-id**: App client id
- **--auth-parameters**: username and password of the test user created in the pool

{% highlight bash %}
aws cognito-idp admin-initiate-auth \
  --region us-east-1 \
  --user-pool-id us-east-1_KLsuR0TMI \
  --client-id 1nbegql09uran4rd015kja3o5k \
  --auth-flow ADMIN_NO_SRP_AUTH \
  --auth-parameters USERNAME=admin,PASSWORD=Passw0rd!
{% endhighlight %}

The identity token can be found in the **IdToken** field of the response.

{% highlight json %}
{
    "AuthenticationResult": {
        "ExpiresIn": 3600, 
        "IdToken": "eyJraWQiOiIxeVVnNXQ3NWY3YzlzYlpnNURZZWFDVWhGMVhEOEdUUEpNXC9zQVhDZEhFbz0iLCJhbGciOiJSUzI1NiJ9.eyJzdWIiOiI3MDM4MDg1Mi1iZGNiLTQ5NzAtOTU2Zi1kZTZkMGFjODBjODUiLCJhdWQiOiIxMnNyNTBwZzF1ZjAwNDRhajYzZTRoc2g2aSIsImVtYWlsX3ZlcmlmaWVkIjpmYWxzZSwidG9rZW5fdXNlIjoiaWQiLCJhdXRoX3RpbWUiOjE0ODc1NDUzNzUsImlzcyI6Imh0dHBzOlwvXC9jb2duaXRvLWlkcC51cy1lYXN0LTEuYW1hem9uYXdzLmNvbVwvdXMtZWFzdC0xX1dkSEVHQWk4TyIsImNvZ25pdG86dXNlcm5hbWUiOiJmcmFuayIsImV4cCI6MTQ4NzU0ODk3NSwiaWF0IjoxNDg3NTQ1Mzc1LCJlbWFpbCI6IndhbmdmYW5qaWVAZ21haWwuY29tIn0.d7HRBs2QegvQsGwQhJfpJBWYdh9N6CwoQFhmC91ugJ0YFxVdRhHUFQl4uoLplrOJO90PjTrjmxR7az17MfRlfu8v-ij3s31oaQqz8IdWECuhWW63xCNfGMN8lAbnUBwlHISer9CIGmdf8iF-xar2uyHeH8WHhIjI3gbJw15ORCC6Fo43CuKJ6k2zWaOywMkNr7oT2U7Etk93b2pDwIgeZ4V6uGbHgv3IRJYXYvMdIqsemoF8tLpx3XD58Iq8hNJlw_gOpOp8dlpDA3AK9-vjyXYDjJ_0zZa6alf6j0XEgwCVm08IIcYhF8ntg7ju0ZVBbQwYrdgzBCBhxtfzz1elVg", 
        "RefreshToken": "eyJjdHkiOiJKV1QiLCJlbmMiOiJBMjU2R0NNIiwiYWxnIjoiUlNBLU9BRVAifQ.LHH4qcesrzfF5mRi3wuykgn_1Kw-rwYjiaP459Lcingf86LkYX-zL9ZZ-jKMFELbyjnXr83P7VDdNIlZCMIFOX4djp4rVU22N3tu02-xunaACSaS6oa_5j-UTcH_2dTFN5yWYkk8VyG5UQUvDYRsmblrTrLshuG9gGhwkQRVYqTP631Zt3N5TqE-YseC211_JcEcGtN_UTxq4Rjc_b2Hh7lQVZrPIKX-7ZTfcQB1QrCTseNRI2aXl6DZFqdBGSRsfpZG4lVyjCWGELT23MreX5kp8rbRhIJzPJMGSD41GdvRjzD24fOqWAp4hg1lYsJKvN1NmCjPUfsrQOohZooOiQ.FXMEC5WtcKW1Sa2h.gfvDrbzFhUboXLhKnqjIBmoTb3YkqdRc8VJuCRsNTMXGr_R5_IgTlwagesd_2ARA50DibEEX5HdfOy8sugE0QnLiGc-DLhSPmvlgTKUiz8Dbm158vBZpK8-Ps5iSp7wiZvkZvW0GzhR8v3Toyp6I_gapDlIqV3RTj34AXTbX3-3jNPitB374Pvy3yVibkhO9WPuUmFDw3AP0x1xcWjw3j8gDY_l3Hs_HyVf4con0gk17DYOqNJIgCV5dR38n2MNNY718MXmivqpFTevg4Kx0AaFPNBbixRNLlIhGbKURo3KPirUGdS_bmU4fC3p_y1xPT8qs8l-2mXT_t1XEpMQDyAF_uRKGQwNifyz-GyeuXE8hNr_32zMJEDDKRD6cP5JvfCAt--gGKIWlYfbt2e3KbG7KMnbflCTdHFvGWNa0G49Y7LUU7IebfTbuX2R8XJLi5uE3GkSSuSp3FL4aqdA1qnBOQnN7ui37BMI9vsZMRQvyYTVynQJk5wBAD59QPVPiKQocknGqeEBTKhg84vNemL9ArZYTQcxnOg-kN6Wsi9wlWoU5Q8kpsHnuEEIqRyTROcXZ4z-6Fx_S3nFVA2VBcNKA4gH9ZzsWz1N1hswFmTaeDR12PkKNVgZgXdepGoT7D8Xe3AmLtEK4Szaen1PeYEcK0VjVpglLFYMOv49a25JxU-PjcT4rA35FQ-vrSau4FHYZRDoaUi_vcZL87pjwd1OLo7pFTzJf45k_sVTl3KPasOGaHdxdC1Q6aGr9m1vTNrgy2_unqH1u4Zgrv_vyj3KlcwWkUvNlBohE-GBh4LCgeq9Piz-rq0pUOuIRheCHKgLWOu64u128pXvjvtPu-uvFwHZ9dsRQNOYEwABTI9uKgZd4hpYLVzrTSm0Y3-DS0MCnfCq5fq25PBFUfTu5XdbDt1hkCubKyw-MRHalVgZ7xlQ8HO-z7UdqHrc-JGJU40cUv6MUAq4UFdZcXkSeZdEtFj3Ww7Ck1fawNIDULzoc4ioBXJIHa7ibIVdNICU5Vv6I6d2GgAtbx5imxXXxqfvW3jqAdsTxBc9Y5c13MhTLmCrBUH33_Hya85pMqZak1uY9DQ8jgzbhSvlTZxp5rRf9p2dhPUSdr2MphN7VFOMm7cD5Fn2dseaQSjZI2wDw7A.sDAeL_Qo_PygnfFZsrk7JQ", 
        "TokenType": "Bearer", 
        "AccessToken": "eyJraWQiOiJlXC9DY3dwSjdOeWZFdk9OWXZhMmttQzdycENqRG9Rb1NKYXFaNURraldMdz0iLCJhbGciOiJSUzI1NiJ9.eyJzdWIiOiI3MDM4MDg1Mi1iZGNiLTQ5NzAtOTU2Zi1kZTZkMGFjODBjODUiLCJ0b2tlbl91c2UiOiJhY2Nlc3MiLCJzY29wZSI6ImF3cy5jb2duaXRvLnNpZ25pbi51c2VyLmFkbWluIiwiaXNzIjoiaHR0cHM6XC9cL2NvZ25pdG8taWRwLnVzLWVhc3QtMS5hbWF6b25hd3MuY29tXC91cy1lYXN0LTFfV2RIRUdBaThPIiwiZXhwIjoxNDg3NTQ4OTc1LCJpYXQiOjE0ODc1NDUzNzUsImp0aSI6ImVjNjcxNzBlLTUwNzUtNDg4Yi04ZDhkLTdlNGYwYzkzMjcyOSIsImNsaWVudF9pZCI6IjEyc3I1MHBnMXVmMDA0NGFqNjNlNGhzaDZpIiwidXNlcm5hbWUiOiJmcmFuayJ9.GOcqDC2PMJdoIdCcvaG8a7GinZWGM-LwRKs98Ck-iLGkdxx3hfHK7AfaxTAE8QeP3MXoLJ0A-EwhNUofEJRhHA-R0cAsTBCmHUuIP2VLoBKSnUBFLnFojCkBoQDHE30aJ-HwIlxM9ExACDAnt6c58T3t8ALihdevUxstjRutBGJgYc-xQhXBJAqEZ0Ov7gu6-js4i070pnIEaS-NxfDIGNDqfE5tvQkglXN_RBezsnufrwFKYTqTRMeCweJE287X6-UCcTgZY16GZw8SVqik9LqbXfO9lufo3W6vkDU-fEwNat1Q-S2iKXwK-Ew2e6mQZHOHxHcw2RQ709Z_iDv3mw"
    }, 
    "ChallengeParameters": {}
}
{% endhighlight %}

Make a curl call to the API endpoint
{% highlight bash %}
$ curl https://ly55wbovq4.execute-api.us-east-1.amazonaws.com/prod/notes \
  -H "Authorization:eyJraWQiOiIxeVVnNXQ3NWY3YzlzYlpnNURZZWFDVWhGMVhEOEdUUEpNXC9zQVhDZEhFbz0iLCJhbGciOiJSUzI1NiJ9.eyJzdWIiOiI3MDM4MDg1Mi1iZGNiLTQ5NzAtOTU2Zi1kZTZkMGFjODBjODUiLCJhdWQiOiIxMnNyNTBwZzF1ZjAwNDRhajYzZTRoc2g2aSIsImVtYWlsX3ZlcmlmaWVkIjpmYWxzZSwidG9rZW5fdXNlIjoiaWQiLCJhdXRoX3RpbWUiOjE0ODc1NDUzNzUsImlzcyI6Imh0dHBzOlwvXC9jb2duaXRvLWlkcC51cy1lYXN0LTEuYW1hem9uYXdzLmNvbVwvdXMtZWFzdC0xX1dkSEVHQWk4TyIsImNvZ25pdG86dXNlcm5hbWUiOiJmcmFuayIsImV4cCI6MTQ4NzU0ODk3NSwiaWF0IjoxNDg3NTQ1Mzc1LCJlbWFpbCI6IndhbmdmYW5qaWVAZ21haWwuY29tIn0.d7HRBs2QegvQsGwQhJfpJBWYdh9N6CwoQFhmC91ugJ0YFxVdRhHUFQl4uoLplrOJO90PjTrjmxR7az17MfRlfu8v-ij3s31oaQqz8IdWECuhWW63xCNfGMN8lAbnUBwlHISer9CIGmdf8iF-xar2uyHeH8WHhIjI3gbJw15ORCC6Fo43CuKJ6k2zWaOywMkNr7oT2U7Etk93b2pDwIgeZ4V6uGbHgv3IRJYXYvMdIqsemoF8tLpx3XD58Iq8hNJlw_gOpOp8dlpDA3AK9-vjyXYDjJ_0zZa6alf6j0XEgwCVm08IIcYhF8ntg7ju0ZVBbQwYrdgzBCBhxtfzz1elVg" \
  -d "{\"content\":\"hello world\",\"attachment\":\"earth.jpg\"}"
{% endhighlight %}

If curl is successful, the response will look similar to this
{% highlight bash %}
{
  "userId": "2aa71372-f926-451b-a05b-cf714e800c8e",
  "noteId": "578eb840-f70f-11e6-9d1a-1359b3b22944",
  "content": "hello world",
  "attachment": "earth.jpg",
  "createdAt": 1487555594691
}
{% endhighlight %}