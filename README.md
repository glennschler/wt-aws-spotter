# wt-aws-spotter
Manage Amazon Elastic Compute Cloud (Amazon EC2) spot instances using webtask.io

####  * warning * 
This readme is outdated. The quick instuctions are to `npm -g install webpack` and then run `webpack` from the root of this repository. The resulting `build\wt-spotter-packed.js` is what should be used when calling `wt create`. Below is for historical reference.

#### Get Started
As a proof of concept, create a webtask which requests the current spot price of a given machine instance type in a given region. This will show that AWS Identity and Access Management (IAM) user credentials can be securely stored for use in a webtask. Once the webtask is proven and understood through these steps, a larger goal to fully manage EC2 spot instance will be possible.

1. Create an EC2 IAM user following this [aws guide](http://docs.aws.amazon.com/IAM/latest/UserGuide/Using_SettingUpUser.html#Using_CreateUser_console). Here are the quick steps:
  * Sign in to the [AWS Management Console](https://console.aws.amazon.com/iam/) and open the IAM console.
  * In the navigation pane, choose **Users**, and then choose **Create New Users**.
  * Enter a user name. Check **Generate an access key**, and choose **Create**.
  * Once created, choose **Show User Security Credentials**. Save the credentials for the webtask. You will not have access to *this* secret access key again after you close.

2. Attach a policy to limit the user permissions to specific AWS resources. For more information, see [Attaching Managed Policies](http://docs.aws.amazon.com/IAM/latest/UserGuide/policies_using-managed.html#attach-managed-policy-console). Assign a policy which only allows the spot price history action. The following shows a good policy for this first goal:
  ```json
  {
    "Version": "2012-10-17",
    "Statement": [
    {
      "Action": [ "ec2:DescribeSpotPriceHistory" ],
      "Effect": "Allow",
      "Resource": "*",
      "Condition": {
        "Bool": { "aws:SecureTransport": "true" },
        "StringEquals": {
          "ec2:Region": [ "us-east-1", "us-west-2" ]
        }
      }
    }]
  }
  ```

3. If missed earlier or using an existing IAM user, create new [access credentials](http://docs.aws.amazon.com/IAM/latest/UserGuide/ManagingCredentials.html#Using_CreateAccessKey). Save the credentials for the webtask.

4. Install and initialize the webtask.io CLI. There are detailed instructions at https://webtask.io/cli.

5. To create a webtask token, the webtask.io CLI command ```wt create``` will upload code along with the EC2 credentials. Both are encrypted and stored. This create command will return a url which represents the new webtask token. Even though the code and secrets are cryptographically protected, the webtask token url still needs be well protected.
  * The EC2 IAM policy specified above is an additional level of protection, since it does not allow actions which might incur AWS cost.
  * The code uploaded in this example is part of this repository, so is publicly available for your review. Understand that any code that is used to create a webtask token needs to be trusted with **your** user's IAM credentials.
  ```bash
  # Set to where the webtask code exists. This is a file in this repository
  # change to WT_GITHUB=. if reading this as a local repo
  $ export WT_GITHUB=https://raw.githubusercontent.com/glennschler/wt-aws-spotter/master
  $ export WT_CODE=$WT_GITHUB/test/wt-spotter.js
  ```

  * In this JSON string replace the enclosed {secret} in both the accessKeyId and secretAccessKey with the real IAM user's credentials.
  ```bash
  $ export WT_SECRET='{"accessKeyId":"{secret}","secretAccessKey":"{secret}"}'
  ```

  * Call the webtask.io CLI command ```wt create```.
  * The optional exp=+10 parameter instructs the webtask token to expire in 10 minutes.
  * The above JSON $WT_SECRET is sent using the wt --secret parameter.
  ```bash
  $ export WT_OPTS='--exp=+10'
  $ export WT_URL=$(wt create $WT_CODE $WT_OPTS --secret wtData=$WT_SECRET)
  ```
  ```bash
  # Echo the previous output to view the created webtask token url
  $ echo $WT_URL
  ```
  >
  ```bash
  https://webtask.it.auth0.com/api/run/{container}/{jt-name}?webtask_no_cache=1
  ```

6. Now the webtask request is available to execute remotely as a microservice.

  * Replace the post data JSON arguments "region" and "type" as needed.
  * Request the $WT_URL which was created during the previous ```wt create``` step.
  * To format the output, optionally pipe the output to a python command as demonstrated here.
  ```bash
  $ curl -s $WT_URL \
  -H "Content-Type: application/json" \
  -X POST -d '{"construct":{"keys":{"accessKeyId":"","secretAccessKey":"","region":"us-west-1"},"upgrades":{"serialNumber":"","tokenCode":""}},'\
  '"attributes":{"type":"m3.medium","dryRun":"false","isLogging":"true"}}' | python -mjson.tool
  ```
  >
  ```bash
  [
    {
        "AvailabilityZone": "us-west-2b",
        "InstanceType": "m3.medium",
        "ProductDescription": "Linux/UNIX",
        "SpotPrice": "0.008400",
        "Timestamp": "2015-08-02T22:49:09.000Z"
    },
    {
        "AvailabilityZone": "us-west-2a",
        "InstanceType": "m3.medium",
        "ProductDescription": "Linux/UNIX",
        "SpotPrice": "0.008100",
        "Timestamp": "2015-08-02T22:30:01.000Z"
    },
    {
        "AvailabilityZone": "us-west-2c",
        "InstanceType": "m3.medium",
        "ProductDescription": "Linux/UNIX",
        "SpotPrice": "0.008400",
        "Timestamp": "2015-08-02T12:33:39.000Z"
    }
  ]
  ```

  * Try again with a region that is not defined in the IAM policy. It should fail.
  ```bash
  $ curl -s $WT_URL \
  -H "Content-Type: application/json" \
  -X POST -d '{"construct":{"keys":{"accessKeyId":"","secretAccessKey":"","region":"us-west-1"},"upgrades":{"serialNumber":"","tokenCode":""}},'\
  '"attributes":{"type":"m3.medium","dryRun":"false","isLogging":"true"}}' | python -mjson.tool
  ```
  >
  ```bash
  {
    "code": 400,
    "details": "UnauthorizedOperation: You are not authorized to perform this operation.",
    "error": "Script returned an error.",
    "message": "You are not authorized to perform this operation.",
    "name": "UnauthorizedOperation",
    "stack": "UnauthorizedOperation: You are not authorized to perform this operation.\n    at Request.extractError...
  }
  ```

7. Execute all the above steps using one helper bash script.
```bash
# Change to WT_GITHUB=. if reading this as a local repo
$ export WT_GITHUB=https://raw.githubusercontent.com/glennschler/wt-aws-spotter/master
$ curl -O $WT_GITHUB/test/wt-aws-spotter.sh
```
```bash
# replace {accessKeyId} {secretAccessKey} with the real secrets
$ sh wt-aws-spotter.sh {accessKeyId} {secretAccessKey} us-west-2 m3.large
```
