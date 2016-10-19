<div id="table-of-contents">
<h2>Table of Contents</h2>
<div id="text-table-of-contents">
<ul>
<li><a href="#sec-1">1. How to get Okta's version of OIDC working with AWS Cognito</a>
<ul>
<li><a href="#sec-1-1">1.1. Create an OIDC app in Okta</a></li>
<li><a href="#sec-1-2">1.2. Create an IAM OpenID Connect Identity Provider in AWS</a></li>
<li><a href="#sec-1-3">1.3. Calculate a "thumprint" to give to AWS</a></li>
<li><a href="#sec-1-4">1.4. Create a Cognito "identity pool"</a></li>
<li><a href="#sec-1-5">1.5. Use the AWS JS SDK to try out Cognito</a></li>
<li><a href="#sec-1-6">1.6. A very simple Cognito app</a></li>
<li><a href="#sec-1-7">1.7. Next up</a></li>
<li><a href="#sec-1-8">1.8. I didn't have to do this</a></li>
</ul>
</li>
</ul>
</div>
</div>

# How to get Okta's version of OIDC working with AWS Cognito<a id="sec-1" name="sec-1"></a>

Scenario:

-   Use the OAuth 2.0 Implicit Flow to get an `id_token` from Okta,
    then pass that `id_token` to AWS in order to authenticate against Cognito.

## Create an OIDC app in Okta<a id="sec-1-1" name="sec-1-1"></a>

## Create an IAM OpenID Connect Identity Provider in AWS<a id="sec-1-2" name="sec-1-2"></a>

-   Provider URL
-   Audience

## Calculate a "thumprint" to give to AWS<a id="sec-1-3" name="sec-1-3"></a>

AWS validates JWTs using a "thumbprint" which is a (I think) a hash
of the public key used in an x509 certificate.

By default AWS assumes that the private key used by the TLS server
on the domain from which a JWT is issued is the same private key
that is used to sign the JWTs.

Here is how AWS generates a thumbprint:
<http://docs.aws.amazon.com/IAM/latest/UserGuide/id_roles_providers_create_oidc_verify-thumbprint.html>

To verify this, you can run this command:

    domain="jfranusic.oktapreview.com"
    (echo "" | openssl s_client -showcerts -connect $domain:443 2> /dev/null) | grep -A27 -- '-----BEGIN CERTIFICATE-----' | tail -28 | \
        openssl x509 -fingerprint -noout | \
        cut -d '=' -f 2 | \
        tr -d ':' | \
        tr '[:upper:]' '[:lower:]'

    a031c46782e6e6c662c2c87c76da9aa62ccabd8e

However, Okta does *not* sign our JWTs or `id_tokens` using the
same private key as we use for TLS, we generate a different keypair
to sign our JWTs, this keypair is different for each Okta Org, and
will eventually be rotated on a regular basis.

Below is how to configure an AWS IAM OIDC Identity Provider to with the
correct thumbprint for validating OIDC `id_token` s from Okta.

To do this, we do the following:

1.  Get the `jwks_uri` from the `/.well-known/openid-configuration`
          endpoint for our Okta org
2.  Pull out the first x509 certificate from the first key using the
    jq expression `.keys[0].x5c[0]`
3.  Remove the quote character from the string using `tr`
4.  Base64 decode the string and pass to `openssl`, asking `openssl`
    to give us the fingerprint for that key, telling
    `openssl` that the input is a DER encoded x509 certificate
5.  Use `cut` and `tr` to clean up the format of the fingerprint
    suitable for pasting into AWS.

    jwks_uri=`curl -s https://jfranusic.oktapreview.com/.well-known/openid-configuration | /Users/joel.franusic/.nix-profile/bin/jq -r .jwks_uri`;
    
    curl -s $jwks_uri | \
        /Users/joel.franusic/.nix-profile/bin/jq '.keys[0].x5c[0]' | \
        tr -d '"' | \
        base64 -D | \
        openssl x509 -inform DER -fingerprint -noout | \
        cut -d '=' -f 2 | \
        tr -d ':' | \
        tr '[:upper:]' '[:lower:]'

    3bf0b4923452feed3f14b8bbf1809c350cda8e04

Now we should have an IAM provider configured in AWS that can
validate an OIDC `id_token` from Okta.

## Create a Cognito "identity pool"<a id="sec-1-4" name="sec-1-4"></a>

Open Cognito in your AWS console, then:

-   Click "Manage Federated Identities"
-   Create a new Identity Pool
    -   If needed, click "Manage Federated Identities" again
    -   or, Click "Create new identity pool"
-   We suggest naming the pool after your okta org, for example: `example_okta_com`
-   In the "Authentication providers" section, select the "OpenID" tab
-   Select the Open ID Connect provider that you configured above

**Notes from Joel:**
Creating Cognito "identity pool":

-   Identity pool name: `okta_example`
-   [ ] Enable access to unauthenticated identities
-   [X] jfranusic.oktapreview.com

Using the defaults for new IAM roles

## Use the AWS JS SDK to try out Cognito<a id="sec-1-5" name="sec-1-5"></a>

Downloaded this:
<https://github.com/aws/aws-sdk-js/releases/tag/v2.3.7>

    {"AccountId":"832503333592","IdentityPoolId":"us-east-1:4c2fb031-039f-45e2-8cf4-df80dc5bf377","Logins":{"jfranusic.oktapreview.com":"eyJhbGciOiJSUzI1NiIsImtpZCI6IjFXbEJNc2dNRnFBS2VwbnVjVUpvZ0JHSWFoMkFxS1dwU25iNngzVjVpSW8ifQ.eyJzdWIiOiIwMHU2NmgwaWg1ak1kTEI5cDBoNyIsIm5hbWUiOiJKb2UgVXNlciIsImVtYWlsIjoiam9lLnVzZXJAZXhhbXBsZS5jb20iLCJ2ZXIiOjEsImlzcyI6Imh0dHBzOi8vamZyYW51c2ljLm9rdGFwcmV2aWV3LmNvbSIsImF1ZCI6InVPS21RcjdOd3lKbUNNb2k4anIwIiwiaWF0IjoxNDYyMjEyODIwLCJleHAiOjE0NjIyMTY0MjAsImp0aSI6ImZiaUxVVjBYYU5BMlZhOHd3R0hxIiwiYW1yIjpbInB3ZCJdLCJpZHAiOiIwMG8ya2pqYmk2VktNR1JVRExKWiIsIm5vbmNlIjoic3RhdGljTm9uY2UiLCJwcmVmZXJyZWRfdXNlcm5hbWUiOiJqb2UudXNlckBleGFtcGxlLmNvbSIsImdpdmVuX25hbWUiOiJKb2UiLCJmYW1pbHlfbmFtZSI6IlVzZXIiLCJ6b25laW5mbyI6IkFtZXJpY2EvTG9zX0FuZ2VsZXMiLCJ1cGRhdGVkX2F0IjoxNDYxOTc1MjgwLCJlbWFpbF92ZXJpZmllZCI6dHJ1ZSwiYXV0aF90aW1lIjoxNDYyMjEyODIwLCJzaGVsbCI6Ii91c3IvYmluL2VtYWNzIiwicGl6emEiOiJDaGVlc2UiLCJncm91cHMiOlsiYnJhbmQ6Qmx1ZSIsImJyYW5kOkdyZWVuIl19.Gu8qWoZ4MvkgbAqNMBjaPF2vaPC8MYa0oFjiqNZt7Ik7o7bgJ0AnO4nEyNoJsaO9wx7ntv00MeUPSeT_t7_9QhuW0Je9Zjg2yxVGIrAhkLXGvkkMrY6Lm0RmCbBCrtqt7aDFk-4XG0r7R54g2DI5QlK41uOSigqHEdQ-eVlnGRgvLEkXqjaHVsE0FScaBVHhEbMvjMAGrZu5H2kMVGr4g_JcvtpmvpNgdHOTmLTHbdrS9Xe4icWomTn8ZMpFkMJ2SpS_ZR_lxBhCXT7_YPJQ8K01ZjBg_cFxWI2dhnf-uNUFkD11JbIFUc0NABAnS77OJMjFLMsJ6aAnkoJFWwGmtA"}}

## A very simple Cognito app<a id="sec-1-6" name="sec-1-6"></a>

    // From the node.js standard library:
    var fs = require('fs');
    // npm install aws-sdk
    var AWS = require('aws-sdk');
    
    //FIXME: These names should be improved
    var MY_OIDC_PROVIDER_URL = 'jfranusic.oktapreview.com';
    // An example JWT
    var MY_JWT_RETURNED_FROM_OIDC_PROVIDER = 'eyJhbGciOiJSUzI1NiIsImtpZCI6IjFXbEJNc2dNRnFBS2VwbnVjVUpvZ0JHSWFoMkFxS1dwU25iNngzVjVpSW8ifQ.eyJzdWIiOiIwMHU2NmgwaWg1ak1kTEI5cDBoNyIsIm5hbWUiOiJKb2UgVXNlciIsImVtYWlsIjoiam9lLnVzZXJAZXhhbXBsZS5jb20iLCJ2ZXIiOjEsImlzcyI6Imh0dHBzOi8vamZyYW51c2ljLm9rdGFwcmV2aWV3LmNvbSIsImF1ZCI6InVPS21RcjdOd3lKbUNNb2k4anIwIiwiaWF0IjoxNDYyMjE1NzQ5LCJleHAiOjE0NjIyMTkzNDksImp0aSI6IldvVEszSWhuRm9idXdZUDJHUEpqIiwiYW1yIjpbInB3ZCJdLCJpZHAiOiIwMG8ya2pqYmk2VktNR1JVRExKWiIsIm5vbmNlIjoic3RhdGljTm9uY2UiLCJwcmVmZXJyZWRfdXNlcm5hbWUiOiJqb2UudXNlckBleGFtcGxlLmNvbSIsImdpdmVuX25hbWUiOiJKb2UiLCJmYW1pbHlfbmFtZSI6IlVzZXIiLCJ6b25laW5mbyI6IkFtZXJpY2EvTG9zX0FuZ2VsZXMiLCJ1cGRhdGVkX2F0IjoxNDYxOTc1MjgwLCJlbWFpbF92ZXJpZmllZCI6dHJ1ZSwiYXV0aF90aW1lIjoxNDYyMjE1NzQ5LCJzaGVsbCI6Ii91c3IvYmluL2VtYWNzIiwicGl6emEiOiJDaGVlc2UiLCJncm91cHMiOlsiYnJhbmQ6Qmx1ZSIsImJyYW5kOkdyZWVuIl19.CB4deqt1RbN38iI5-ZKfWfPdikAVYqmtKn4maMw0H-yyDbIhoDGtCZ-iwkgSsmaGhMzt18o_KvuCEhSK4uGC_yoisR6ybnWuw6RTnYy0XfKoXl5v-76K17N9vDnMLUDA4XRs41z906EOFQqPmujkQYg5XDQbAUDGKTwsRzNE5OgMDsICoYE5lhoeLpxGmPzx1vGveZqXsB00koqxFwkWq819GfdicOTnJz75bWotcI7rCs1_9BuFLKEXSGCqzsn2nw9BgmWk2mesebfDOX6n6JWe4XB1WySx0VlUU6ZZMLaxvQVo9Os3zBggUOca29UvlJYJxMNVFXptfcWLNYRJ7w';
    // Found in: https://console.aws.amazon.com/billing/home?#/account
    var MY_ACCOUNT_ID = '832503333592';
    // Found in your identity pool, in the "Sample code" section:
    // https://console.aws.amazon.com/cognito/federated/
    var MY_ID_POOL_ID = 'us-east-1:4c2fb031-039f-45e2-8cf4-df80dc5bf377';
    
    // FIXME: Shouldn't this just be parsed out of the "MY_ID_POOL_ID" varable? 
    AWS.config.region = 'us-east-1';
    
    var cognitoidentity = new AWS.CognitoIdentity();
    var provider_url = MY_OIDC_PROVIDER_URL
    var logins = {};
    logins[provider_url] = MY_JWT_RETURNED_FROM_OIDC_PROVIDER;
    var idparams = {
        AccountId: MY_ACCOUNT_ID,
        IdentityPoolId: MY_ID_POOL_ID,
        Logins: logins
    };
    cognitoidentity.getId(idparams, function(err, data) {
        if (err) console.log(err, err.stack);
        else     console.log(data);
    });

Note, the code above is equivalent to the `curl` command below:

    curl \
        -H "User-Agent: aws-sdk-nodejs/2.3.7 darwin/v0.10.35" \
        -H "Content-Type: application/x-amz-json-1.1" \
        -H "X-Amz-Target: AWSCognitoIdentityService.GetId" \
        -H "X-Amz-Content-Sha256: 3a3dae86c63289d6c8ef539db4786d568dabcded59beae2615a618e33a12afb6" \
        -H "Host: cognito-identity.us-east-1.amazonaws.com" \
        --data-binary '{"AccountId":"832503333592","IdentityPoolId":"us-east-1:4c2fb031-039f-45e2-8cf4-df80dc5bf377","Logins":{"jfranusic.oktapreview.com":"eyJhbGciOiJSUzI1NiIsImtpZCI6IjFXbEJNc2dNRnFBS2VwbnVjVUpvZ0JHSWFoMkFxS1dwU25iNngzVjVpSW8ifQ.eyJzdWIiOiIwMHU2NmgwaWg1ak1kTEI5cDBoNyIsIm5hbWUiOiJKb2UgVXNlciIsImVtYWlsIjoiam9lLnVzZXJAZXhhbXBsZS5jb20iLCJ2ZXIiOjEsImlzcyI6Imh0dHBzOi8vamZyYW51c2ljLm9rdGFwcmV2aWV3LmNvbSIsImF1ZCI6InVPS21RcjdOd3lKbUNNb2k4anIwIiwiaWF0IjoxNDYyMjEyODIwLCJleHAiOjE0NjIyMTY0MjAsImp0aSI6ImZiaUxVVjBYYU5BMlZhOHd3R0hxIiwiYW1yIjpbInB3ZCJdLCJpZHAiOiIwMG8ya2pqYmk2VktNR1JVRExKWiIsIm5vbmNlIjoic3RhdGljTm9uY2UiLCJwcmVmZXJyZWRfdXNlcm5hbWUiOiJqb2UudXNlckBleGFtcGxlLmNvbSIsImdpdmVuX25hbWUiOiJKb2UiLCJmYW1pbHlfbmFtZSI6IlVzZXIiLCJ6b25laW5mbyI6IkFtZXJpY2EvTG9zX0FuZ2VsZXMiLCJ1cGRhdGVkX2F0IjoxNDYxOTc1MjgwLCJlbWFpbF92ZXJpZmllZCI6dHJ1ZSwiYXV0aF90aW1lIjoxNDYyMjEyODIwLCJzaGVsbCI6Ii91c3IvYmluL2VtYWNzIiwicGl6emEiOiJDaGVlc2UiLCJncm91cHMiOlsiYnJhbmQ6Qmx1ZSIsImJyYW5kOkdyZWVuIl19.Gu8qWoZ4MvkgbAqNMBjaPF2vaPC8MYa0oFjiqNZt7Ik7o7bgJ0AnO4nEyNoJsaO9wx7ntv00MeUPSeT_t7_9QhuW0Je9Zjg2yxVGIrAhkLXGvkkMrY6Lm0RmCbBCrtqt7aDFk-4XG0r7R54g2DI5QlK41uOSigqHEdQ-eVlnGRgvLEkXqjaHVsE0FScaBVHhEbMvjMAGrZu5H2kMVGr4g_JcvtpmvpNgdHOTmLTHbdrS9Xe4icWomTn8ZMpFkMJ2SpS_ZR_lxBhCXT7_YPJQ8K01ZjBg_cFxWI2dhnf-uNUFkD11JbIFUc0NABAnS77OJMjFLMsJ6aAnkoJFWwGmtA"}}' \
        https://cognito-identity.us-east-1.amazonaws.com/

    {"__type":"NotAuthorizedException","message":"Invalid login token. Token expired: 1464384591 >= 1462216420"}

Assuming that the command above worked, we would now have an active
Cognito session available to us!

## Next up<a id="sec-1-7" name="sec-1-7"></a>

1.  GetID
2.  Get OpenIDToken
3.  Init STS (role, role session name, web id token (token I got
    back from cognito: data.token))
4.  Call AssumeRoleWithWebIdentity (role arn, token, length of validity)

access<sub>key</sub>
secret<sub>access</sub><sub>key</sub>
session<sub>token</sub>

## I didn't have to do this<a id="sec-1-8" name="sec-1-8"></a>

<http://docs.aws.amazon.com/IAM/latest/UserGuide/id_roles_create_for-idp_oidc.html#oidc-prereqs>

From the doc above:

    {
      "Version": "2012-10-17",
      "Statement": {
        "Effect": "Allow",
        "Principal": {"Federated": "cognito-identity.amazonaws.com"},
        "Action": "sts:AssumeRoleWithWebIdentity",
        "Condition": {
          "StringEquals": {"cognito-identity.amazonaws.com:aud": "us-east-1:12345678-abcd-abcd-abcd-123456"},
          "ForAnyValue:StringLike": {"cognito-identity.amazonaws.com:amr": "unauthenticated"}
        }
      }
    }

Updated for Okta

    {
      "Version": "2012-10-17",
      "Statement": {
        "Effect": "Allow",
        "Principal": {"Federated": "jfranusic.oktapreview.com"},
        "Action": "sts:AssumeRoleWithWebIdentity",
        "Condition": {
          "StringEquals": {"jfranusic.oktapreview.com:aud": "uOKmQr7NwyJmCMoi8jr0"}
        }
      }
    }

    {
      "Version": "2012-10-17",
      "Statement": {
        "Effect": "Allow",
        "Principal": {"Federated": "jfranusic.oktapreview.com"},
        "Action": "sts:AssumeRoleWithWebIdentity",
        "Condition": {
          "StringEquals": {"cognito-identity.amazonaws.com:aud": "us-east-1:4c2fb031-039f-45e2-8cf4-df80dc5bf377"}
        }
      }
    }