# netlify-serverless-oauth2-backend

This is an AWS Lambda based service to help perform authentication to Github via an OAuth2 authentication process.

## Install the serverless npm module globally

`npm install serverless -g`

## Clone the netlify-serverless-oauth2-backend repo and install deps

    git clone https://github.com/marksteele/netlify-serverless-oauth2-backend.git
    cd netlify-serverless-oauth2-backend
    npm install

## Install the AWS CLI v2

[AWS docs](https://docs.aws.amazon.com/cli/latest/userguide/install-cliv2.html)

## Create a root account on AWS.

## Create an IAM Admin user

See the AWS docs [AWS docs here](https://docs.aws.amazon.com/IAM/latest/UserGuide/getting-started_create-admin-group.html). 

Be sure to save the ACCESS and SECRET keys that are shown with the new user. If you don't do this, you can [create new ones](https://docs.aws.amazon.com/IAM/latest/UserGuide/id_credentials_access-keys.html).

Also take note of the new user's account ID, username and password, as all three will be needed to log in as this user.

## Check the default region of your AWS IAM Admin user

Log in to the AWS console as the admin user created above at [https://console.aws.amazon.com](https://console.aws.amazon.com). The default region is in the url after `?region=`. For example, if the url after logging into the console is `https://eu-west-1.console.aws.amazon.com/console/home?region=eu-west-1`, then the default region is `eu-west-1`.

## Create a custom AWS CLI profile

This is simply a collection of settings that you assign to a named profile, making it easier to apply those settings to cli requets and commands. See profiles [here](https://docs.aws.amazon.com/cli/latest/userguide/cli-configure-quickstart.html).

`aws configure --profile <YOUR_AWS_PROFILE_NAME>`

This will result in 4 prompts:

`AWS Access Key ID [None]:`

This is the access key of the IAM Admin user created above.

`AWS Secret Access Key [None]:`

This is the secret access key of the IAM Admin user created above.

`Default region name [None]: `

Use the default region the IAM Admin user (See above in how to check this) 

`Default output format [None]:` 

Enter `text`

## Get your KMS key for the parameter store in AWS

### Option 1 - AWS Console

Log in to the AWS console as the IAM Admin user created above. From the *Services* dropdown, click *Key Manangement Service* and then click on *AWS managed keys*.

Your KMS key Key ID for the *aws/ssm* key.

### Option 2 - AWS CLI

`aws kms describe-key --key-id alias/aws/ssm --profile <YOUR_AWS_PROFILE_NAME> --region <YOUR_DEFAULT_REGION>`

`<YOUR_DEFAULT_REGION>` is the default region of the IAM user whose access and secret keys were used to create the user profile above.

In the response you will see something like `arn:aws:kms:eu-west-1:891873474258:key/234f35c3-b45e-9976-09a7-ea2343467567`. Your kms key is what comes after `:key/`. In the above example it would be `234f35c3-b45e-9976-09a7-ea2343467567`.

## Update serverless.yml

This file is at the root of the `netlify-serverless-oauth2-backend` repo, which you cloned above.

Replace the contents of this file with the following:

```
service: serverless-oauth2
provider:
  name: aws
  runtime: nodejs12.x
  stage: ${opt:stage, self:custom.defaultStage}
  environment:
    GIT_HOSTNAME: "/<YOUR_AWS_PROFILE_NAME>/oauth/${opt:stage, self:provider.stage}/GIT_HOSTNAME"
    OAUTH_TOKEN_PATH: "/<YOUR_AWS_PROFILE_NAME>/oauth/${opt:stage, self:provider.stage}/OAUTH_TOKEN_PATH"
    OAUTH_AUTHORIZE_PATH: "/<YOUR_AWS_PROFILE_NAME>/oauth/${opt:stage, self:provider.stage}/OAUTH_AUTHORIZE_PATH"
    OAUTH_CLIENT_ID: "/<YOUR_AWS_PROFILE_NAME>/oauth/${opt:stage, self:provider.stage}/OAUTH_CLIENT_ID"
    OAUTH_CLIENT_SECRET:  "/<YOUR_AWS_PROFILE_NAME>/oauth/${opt:stage, self:provider.stage}/OAUTH_CLIENT_SECRET"
    REDIRECT_URL: "/<YOUR_AWS_PROFILE_NAME>/oauth/${opt:stage, self:provider.stage}/REDIRECT_URL"
    OAUTH_SCOPES: "/<YOUR_AWS_PROFILE_NAME>/oauth/${opt:stage, self:provider.stage}/OAUTH_SCOPES"
    TZ: "utc"
  iamRoleStatements:
    - Effect: Allow
      Action:
        - ssm:DescribeParameters
        - ssm:GetParameters
      Resource: "arn:aws:ssm:${opt:region, self:provider.region}:*:parameter/<YOUR_AWS_PROFILE_NAME>/oauth/${opt:stage, self:provider.stage}/*"
    - Effect: Allow
      Action:
        - kms:Decrypt
      Resource: "arn:aws:kms:${opt:region, self:provider.region}:*:key/${self:custom.kms_key.${opt:region, self:provider.region}.${self:provider.stage}}"

custom:
  defaultStage: dev
  kms_key:
    "<YOUR_DEFAULT_REGION>":
      prod: "<YOUR_KMS_KEY>"
      dev: "foo"

functions:
  auth:
    handler: auth.auth
    memorySize: 128
    timeout: 5
    events:
      - http:
          path: /auth
          method: get
          cors: true               
  callback:
    handler: auth.callback
    memorySize: 128
    timeout: 5
    events:
      - http:
          path: /callback
          method: get
          cors: true
  success:
    handler: auth.success
    memorySize: 128
    timeout: 5
    events:
      - http:
          path: /success
          method: get
          cors: true  
  default:
    handler: auth.default
    memorySize: 128
    timeout: 5
    events:
      - http:
          path: /
          method: get
          cors: true 

plugins:
  - serverless-plugin-optimize
  - serverless-offline

package:
  individually: true
```

The replace each of `<YOUR_DEFAULT_REGION>`, `<YOUR_AWS_PROFILE_NAME>` and `<YOUR_KMS_KEY>` witht eh relavant values obtained above.

## Deploy the Lambda function

`sls deploy -s <STAGE> --aws-profile <YOUR_AWS_PROFILE_NAME> --region <YOUR_DEFAULT_REGION>`

`<STAGE>` above chooses which stage listed under `kms_key` in `serverless.yml` to use. Most likely `prod` as this is where uypou have just addded your AWS KMS key id.

This command will gebnerate your serverless application in your AWS account. This will include:

* 1 AWS Lambda application
* 4 AWS Lamnda functions
* 1 CloudFormation stack
* 1 S3 bucket

If successful, the response to the `sls deploy` command will look something like this.

```
Serverless: Stack update finished...
Service Information
service: serverless-oauth2
stage: prod
region: eu-west-1
stack: serverless-oauth2-prod
resources: 32
api keys:
  None
endpoints:
  GET - https://34h346jhj4.execute-api.eu-west-1.amazonaws.com/prod/auth <= <SERVERLESS_APP_AUTH_ENDPOINT>
  GET - https://34h346jhj4.execute-api.eu-west-1.amazonaws.com/prod/callback <= <SERVERLESS_APP_CALLBACK_ENDPOINT>
  GET - https://34h346jhj4.execute-api.eu-west-1.amazonaws.com/prod/success
  GET - https://34h346jhj4.execute-api.eu-west-1.amazonaws.com/prod/
functions:
  auth: serverless-oauth2-prod-auth
  callback: serverless-oauth2-prod-callback
  success: serverless-oauth2-prod-success
  default: serverless-oauth2-prod-default
layers:
  None
```

## Create an OAuth app in Github

Go to go to [https://github.com/settings/applications/new](https://github.com/settings/applications/new), or in your github.com account, go to `Settings > Developers > OAuth Apps` and click `New OAuth App`.

**Application name**, **Homepage url** and **Application description** can be anything.

**Authorization callback URL** must be the `<SERVERLESS_APP_CALLBACK_ENDPOINT>` listed in the response to the `sls deploy` command above. In the above example example it would be `https://34h346jhj4.execute-api.eu-west-1.amazonaws.com/prod/callback`.

Click **Register application**.

Thke note of the **Client ID** and **Client Secret**. You will need these below.

## Add parameters to the AWS parameter store

Finally, once the code is deployed you need to add some parameters to the AWS parameter store.

Head on over to the AWS console, find the **Systems manager**, and go to the **Parameter store**.

In there, you'll want to create the following parameters/values (as SecureStrings), making sure to replace `<STAGE>` with your stage (eg: prod), and `<YOUR_AWS_PROFILE_NAME>` with the profile name used in `serverless.yml`.

| Name                                                          | Type              | Value |
| --------------------------------------------------------------| ----------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
|`/<YOUR_AWS_PROFILE_NAME>/oauth/<STAGE>/GIT_HOSTNAME`          | **SecureString**  | `https://github.com`|
|`/<YOUR_AWS_PROFILE_NAME>/oauth/<STAGE>/OAUTH_TOKEN_PATH`      | **SecureString**  | `/login/oauth/access_token`|
|`/<YOUR_AWS_PROFILE_NAME>/oauth/<STAGE>/OAUTH_AUTHORIZE_PATH`  | **SecureString**  | `/login/oauth/authorize`|
|`/<YOUR_AWS_PROFILE_NAME>/oauth/<STAGE>/OAUTH_CLIENT_ID`       | **SecureString**  | The **Client ID** of your Githib OAuth app created above.|
|`/<YOUR_AWS_PROFILE_NAME>/oauth/<STAGE>/OAUTH_CLIENT_SECRET`   | **SecureString**  | The **Client Secret** of your Githib OAuth app created above.|
|`/<YOUR_AWS_PROFILE_NAME>/oauth/<STAGE>/REDIRECT_URL`          | **SecureString**  | `<SERVERLESS_APP_CALLBACK_ENDPOINT>` <br>_This is the callback endpoint listed in the response to the deployment of the Lamda function above. In the above example example it would be `https://34h346jhj4.execute-api.eu-west-1.amazonaws.com/prod/callback`_|
| `/<YOUR_AWS_PROFILE_NAME>/oauth/<STAGE>/OAUTH_SCOPES`         | **SecureString**  | `repo` <br>_Note that this scope will request full read and write access to all of the user's Github repos. This is required for Netlify CMS to work._|

## Update your Netlify CMS config file

Find the `config.yml` file at the root of your netlify CSM `admin` directory.

```
backend:
  name: github
  repo: <SITE_REPO_PATH>
  base_url: <SERVERLESS_APP_CALLBACK_ENDPOINT>
  auth_endpoint: <SERVERLESS_APP_AUTH_ENDPOINT>
```

For `base_url` use the base URL of your serverless application. This is the base url if the endpioints listed in the response top the the response to the `sls deploy` command. In the above example, the base url would be `https://9s9ttkn75e.execute-api.eu-west-1.amazonaws.com`.

For `auth_endpoint` add the pathname of your `<SERVERLESS_APP_AUTH_ENDPOINT>` - ie the part after the base irl. In the above example this would be `/prod/auth`.