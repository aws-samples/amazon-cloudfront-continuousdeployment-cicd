# amazon-cloudfront-continuousdeployment-cicd

## Introduction

AWS CloudFront continuous deployment extends the functionality of your CloudFront distributions by allowing you to test and validate configuration changes to a small percentage of live traffic before extending to your wider audience. Integrating this functionality to your existing CloudFront distributions allows businesses and corporations to create a blue-green strategy to ensure a consistent experience throughout a users’ sessions without the worry of causing breaks to their production sites. 

As an engineer, extending this feature to a CI/CD pipeline allows for a simple, repeatable and automated way to release changes without doing the heavy lifting of changing DNS records or manually monitoring two different infrastructure environments. Integrating this allows for you or your team to make changes to your code, deploy those changes to your repository, and automatically promote changes to your CloudFront distributions. This provides a single source of truth without worry of causing downstream, unforeseen effects. 

This repoistory contains the code to use your existing CloudFront distributions to create a continuous deployment policy that is deployed with CloudFormation stacks through a CI/CD pipeline from staging to production environments. For this post, we are using an existing production CloudFront distribution created manually. 

## Overview of the solution

To implement this solution, let us walk through the high-level steps:

1. Deploy the pipeline.yml file from the Github repository within your account with appropriate parameters.
2. Edit the staging_cloudfront_distribution.yml file to reflect the desired changes for a staging distribution.
3. Upload the staging_cloudfront_distribution.yml file from the Github repository into the CodeCommit repository.
4. Browse to CloudFormation to view and copy Outputs tab values for next steps
5. Connect CloudFront continuous deployment policy to production CloudFront distribution through AWS CLI

After completing these steps, you will have an end-to-end, reusable CI/CD pipeline that integrates CloudFront continuous deployment with your current CloudFront production distribution. The pipeline starts automatically after you apply the intended changes on your yaml staging distribution file and commit into the CodeCommit repository. The pipeline will promote your changes from the staging distribution onto the production distribution once run the one-time CLI commands to connect your new continuous deployment policies and distributions.

The following diagram illustrates the solution architecture.

![alt text](https://github.com/jasonbrd/amazon-cloudfront-continuousdeployment-cicd/blob/main/CloudFront_ContinuousDeployment.png?raw=true)

The solution workflow is as follows:

- Developers integrate changes into a main branch hosted within a CodeCommit repository.
- CodePipeline polls the source code repository and triggers the pipeline to run when a new version is detected.
- CloudFormation deploys the yaml template with the desired changes for a staging CloudFront distribution.
- Once approved within the pipeline, a python lambda function promotes the staging distribution to the connected production distribution. 

To use this solution, you should have the following prerequisites:

 - An AWS account.
- A current CloudFront Distribution (we suggest a manually deployed CloudFront distribution).
- Basic knowledge of CloudFormation.

## Deploying the solution:

1. Once logged into your designated account, your first step is to browse to the CloudFormation console and deploy the pipeline.yml file found within the Github repository. These parameters can all be tailored to your desired naming strategy, but the ProductionCloudFrontID parameter is needed to eventually promote you staging distribution to production. 

2. Once deployed, browse to the CodeCommitt dashboard to find your newly created repository.

3. Create a new CloudFormation stack with the `pipeline.yml` file making sure to fill out the parameters. 
- Repository Name for CodeCommit such as: `CloudFront-ContinuousDeployment`
- Pipeline Name: `CloudFront-ContinuousDeployment-Pipeline`
- A name for the stack the pipeline later creates: `staging_cloudfront_distribution`
- The name of the file used for deploying the policy: `staging_cloudfront_distribution.yml`
- An email address CodePipeline uses to send approval emails.
- The CloudFront distribution ID for our existing production distribution.
Deploy the stack and wait for it to finish. Part of the infrastructure deployed from the stack is an Amazon Simple Notification Service topic that needs to be confirmed via the email you entered as a parameter. 

4. Before putting the `staging_cloudfront_distribution.yml` file within the newly created repository, you need to edit the file to reflect the continuous deployment policy information you desire for your setup. For our example, we will use the weight-based traffic shifting. Weight-based, more commonly referred to as a canary deployment, lets you define a percentage of your production traffic to push to your staging distribution. You can see that we are using a single weight configuration percentage of 10%.

```
  CFDistributionDeploymentPolicy:
    Type: AWS::CloudFront::ContinuousDeploymentPolicy
    Properties: 
      ContinuousDeploymentPolicyConfig: 
          Enabled: true
          StagingDistributionDnsNames: 
            - !GetAtt CFDistribution2.DomainName
          TrafficConfig: 
            SingleWeightConfig: 
              Weight: 0.10
            Type: SingleWeight
```

5. To create the needed `staging_cloudfront_distribution.yml` file within the newly created repository, clone the empty repository down via HTTPS or SSH to create and push the `staging_cloudfront_distribution.yml` file. 

6. Once the file is pushed to the repository, the CodePipeline will start its process which will deploy the new staging distribution with continuous deployment police via CloudFormation.

7. After a couple minutes, the CloudFormation stack will finish. Browse to the CloudFormation console and locate the new stack. Browse to the output tab and copy the information for our next step. 

8. Since we want to connect the current production CloudFront distribution with the newly created continuous deployment policy, we need to do a one-time deployment of the `cloudfront_continuous_deployment_connection.yml` that deploys the custom resource to connect the continuous deployment policy ID to the prodcution distribution. Make sure to update the parameters with the correct values as pulled from your environment. 

9. Once these one-time commands are run, you can browse to your production CloudFront distribution to see the newly connected policy.

10. At this point, you are ready to verify that the newly created staging distribution and continuous deployment policy are working as expected. For our example, we defined two different s3 back-ends for the primary and staging distributions. Our continuous deployment policy is configured to send 10% of our traffic to the staging environment s3 bucket back-end which hosts a different index.html file indicating the staging bucket. 

After you have verified that the staging distribution is providing the results as expected, you can approve the manual approval stage to promote the change to production. You can watch the progress of this promotion within the CodePipeline console or within CloudFront itself. 

After the promotion process is complete, you can verify the change was successful by browsing to the CloudFront production distribution endpoint to verify.

Part of the promotion process for CloudFront is that the continuous deployment policy gets disabled. The solution we are presenting is designed to follow the process again of making changes to your `staging_cloudfront_distribution.yaml` and commit to your repository. The pipeline process will kick-off again to deploy and enable the continuous deployment policy for the production distribution. You can run through your verification process again before promoting any change to production through the manual approval stage. 

## Cleaning Up

You may delete the resources that you created during this post in the following order:

- Delete the cfn-response CloudFormation stack.
- Empty the artifact Amazon S3 bucket created from the CloudFront-ContinuousDeployment-Pipeline stack.
- Delete the CloudFront Continuous Deployment policy from the production distribution.
- Delete the staging_cloudfront_distribution CloudFormation Stack.
- Delete the CloudFront-ContinuousDeployment-Pipeline CloudFormation Stack.

