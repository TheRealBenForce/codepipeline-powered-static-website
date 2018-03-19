
# Code-pipeline powered static website serverless, and powered entirely by AWS

This is a fork of Eric Hammond's great implementation for infrastructure to build static websites on s3: [aws-git-backed-static-website](https://github.com/alestic/aws-git-backed-static-website).
It has the following modifications:

1. The approach uses lambda functions rather than AWS's CodeBuild service to generate the site and push it to production. We found that one of the lambda functions provided in the template did not work (see [this github issue](https://github.com/alestic/aws-git-backed-static-website/issues/9), which seems to have since been resolved for more details).
We replaced the lambda functions with a CodeBuild stage.

2.  We also found that the permissions for S3 buckets created in the stack did not seem to allow the contents to be served publicly. We  therefore added a bucket policy to the S3 bucket.

3.  CodeBuild seemed to more naturally fit the process of building these sites than lambda functions. 

4. We also wanted to allow users to connect their GitHub repositories with the Source step of our pipeline. 


[![Launch CloudFormation stack][launching]][launch]

[launching]: https://s3.amazonaws.com/cloudformation-examples/cloudformation-launch-stack.png

[launch]: https://console.aws.amazon.com/cloudformation/home?region=us-east-1#/stacks/new?templateURL=https:%2F%2Fs3.amazonaws.com%2Felementryx-open-source%2Fcodepipeline-powered-static-website-cloudformation.yml&stackName=codepipeline-powered-static-website

## Overview

This project contains a YAML CloudFormation template that creates a
Git repository and a static https website, along with the necessary
AWS infrastructure glue so that every change to content in the Git
repository is automatically deployed to the static web site.

The website can serve the exact contents of the Git repository, or 
a static site generator plugin (e.g., Hugo) can be specified to automatically generate the site content from the source in the Git
repository. The codebuild.yml file instructs the created CodeBuild
project how to build the static file in question.

The required stack parameters are a domain name and an email address.

The primary output values are a list of nameservers to set in your
domain's registrar and a CodeCommit git URL if you are not connecting an
existing GitHub repository for adding and updating the website content.

Git repository event notifications are sent to an SNS topic and your
provided email address is initially subscribed.

Access logs for the website are stored in an S3 bucket.

Benefits of this architecture include:

 - Trivial to launch - Can use aws-cli or AWS console (click "launch"
   above).

 - Maintenance-free - Amazon is responsible for managing all the
   services used.

 - Negligible cost at substantial traffic volume - Starts off as low
   as $0.51 per month if running alone in a new AWS account (Route 53
   has no free tier). Your cost may vary over time and if other
   resources are running in your AWS account.

 - Scales forever - No action is needed to support more web site
   traffic, though the costs for network traffic and DNS lookups will
   start to add up to more than a penny per month.

## Create CloudFormation stack for static website

This CloudFormation stack has an CodeBuild buildspec.yml plugin
architecture that supports arbitrary static site generators. The 
generator buildspec.yml's currently available are in the buildspecs/ directory.
Ensure that the buildspec in use is compatible with the BuildEnvironmentImage
specified when creating the stack. Default Ruby environment is used to support
Ruby based static site generators like Jekyll. 

1. Identity transformation plugin - This copies the entire Git
   repository content to the static website with no
   modifications.

2. Subdirectory plugin - This plugin is useful if your
   Git repository has files that should not be included as part of the
   static site. It publishes a specified subdirectory (e.g., "htdocs"
   or "public-html") as the static website, keeping the rest of your
   repository private.

3. Hugo Plugin - This plugin runs the popular
   [Hugo][hugo] static site generator. The Git repository should
   include all source templates, content, theme, and config.

4. Jekyll plugin - This plugin runs the popular
   [Jekyll][jekyll] static site generator. The Git repository should
   include all source templates, content, theme, and config.

You are welcome to make your own buildspec.yml based on these
of these and place it at the root of your static site project. If you feel like other people might like to use it, feel free to contribute it back.

[jekyll]: https://jekyllrb.com/
[hugo]: https://gohugo.io/


## Setting up your first static site generator pipeline using Git
**Step 1**:
Fork  [our sample website source repository](https://github.com/elementryx/YAX-Coming-soon-Jekyll-codebuild-template) on GitHub or place the relevant buildspec.yml from the **buildspecs/** folder into your existing static website repository.

**Step 2**:
Follow the steps [provided by GitHub](https://help.github.com/articles/creating-a-personal-access-token-for-the-command-line/) to create a OAuth token from GitHub and make sure to include the scopes [*repo* and *admin:repo_hook*](https://docs.aws.amazon.com/codepipeline/latest/userguide/troubleshooting.html#troubleshooting-gs2) . Copy this OAuth token.

**Step 3**:
Open this [this wizard](https://console.aws.amazon.com/cloudformation/home?region=us-east-1#/stacks/new?templateURL=https:%2F%2Fs3.amazonaws.com%2Felementryx-open-source%2Fcodepipeline-powered-static-website-cloudformation.yml&stackName=codepipeline-powered-static-website) to enable you to launch your stack. Click next.

**Step 4**:
Set the following fields:

  DomainName = "your.blogdomain.com"
  SourceType = "GitHub"
  BranchName = "master"
  GithubRepoOwner = "yourgithubname"
  GithubRepoName = "codepipeline-powered-static-website"
  GithubOauthToken ="yourPastedOauthTokenFromStep2"
  NotificationEmail = "your@email-address.com"
  
If your domain is managed by AWS and you have an existing hosted zone, you should enter your pre-existing hosted zone.

  PreExistingHostedZoneDomain = "your.blogdomain.com"

**Step 5:**
Click **Next** > Click **Next** > check the box that reads **I acknowledge that AWS CloudFormation might create IAM resources.** > Click **Create**
 
**Step 6:**
When you receive emails from AWS requesting confirmation that you own the domain when CloudFormation is creating your SSL certificate, confirm that you want to allow AWS to create the certificate.
 
**Step 7:**
Wait, for about an hour, the cloudfront distributions will be ready to serve your website.  Simply copy the outputted DNS servers and set nameservers in your domain registrar to the above.


## Create CloudFormation stack for static website

Here is the basic approach to creating the stack with CloudFormation.

    domain=example.com
    email=yourrealemail@adomain.com

    template=aws-git-backed-static-website-cloudformation.yml
    stackname=${domain/./-}-$(date +%Y%m%d-%H%M%S)
    region=us-east-1

    aws cloudformation create-stack \
      --region "$region" \
      --stack-name "$stackname" \
      --capabilities CAPABILITY_IAM \
      --template-body "file://$template" \
      --tags "Key=Name,Value=$stackname" \
      --parameters \
        "ParameterKey=DomainName,ParameterValue=$domain" \
        "ParameterKey=NotificationEmail,ParameterValue=$email"
    echo region=$region stackname=$stackname

The above defaults to the Identity transformation plugin. You can
specify that a ruby environment is to be used like this:

        "ParameterKey=BuildEnvironmentImage,ParameterValue=aws/codebuild/ruby:2.3.1" 


When the stack starts up, two email messages will be sent to the
address associated with your domain's registration and one will be
sent to your AWS account address. Open each email and approve these:

 - ACM Certificate (2)
 - SNS topic subscription

The CloudFormation stack will be stuck until the ACM certificates are
approved. The CloudFront distributions are created afterwards and can
take well over 30 minutes to complete.

### Get the name servers for updating in the registrar

    hosted_zone_id=$(aws cloudformation describe-stacks \
      --region "$region" \
      --stack-name "$stackname" \
      --output text \
      --query 'Stacks[*].Outputs[?OutputKey==`HostedZoneId`].[OutputValue]')
    echo hosted_zone_id=$hosted_zone_id

    aws route53 get-hosted-zone \
      --id "$hosted_zone_id"    \
      --output text             \
      --query 'DelegationSet.NameServers'

Set nameservers in your domain registrar to the above.

### Get the Git clone URL (if you are using CodeCommit)

    git_clone_url_http=$(aws cloudformation describe-stacks \
      --region "$region" \
      --stack-name "$stackname" \
      --output text \
      --query 'Stacks[*].Outputs[?OutputKey==`GitCloneUrlHttp`].[OutputValue]')
    echo git_clone_url_http=$git_clone_url_http


### Pass your GitHub Repository parameters (if you are using a GitHub repository)

Follow the steps [provided by GitHub](https://help.github.com/articles/creating-a-personal-access-token-for-the-command-line/) to create a OAuth token from GitHub and make sure to include the scopes [*repo* and *admin:repo_hook*](https://docs.aws.amazon.com/codepipeline/latest/userguide/troubleshooting.html#troubleshooting-gs2) . Copy this OAuth token.


    SourceType = "GitHub"
    BranchName = "master"
    GithubRepoOwner = "yourgithubname"
    GithubRepoName = "codepipeline-powered-static-website"
    GithubOauthToken ="yourPastedOauthTokenFromStep2"
    NotificationEmail = "your@email-address.com"

### Use Git

    repository=$domain
    profile=$AWS_PROFILE   # The correct aws-cli profile name

    git clone \
      --config 'credential.helper=!aws codecommit --profile '$profile' --region '$region' credential-helper $@' \
      --config 'credential.UseHttpPath=true' \
      $git_clone_url_http

    cd $repository

## Clean up after testing

### Delete a stack (mostly)

    aws cloudformation delete-stack \
      --region "$region" \
      --stack-name "$stackname"

Leaves behind Route 53 hosted zone, S3 buckets, and Git repository.

### Clean up Route 53 hosted zone

    domain=...

    hosted_zone_id=$(
      aws route53 list-hosted-zones \
        --output text \
        --query 'HostedZones[?Name==`'$domain'.`].Id'
    )
    echo hosted_zone_id=$hosted_zone_id
    aws route53 delete-hosted-zone \
      --id "$hosted_zone_id"

### Clean up S3 buckets

    # WARNING! DESTROYS CONTENT AND LOGS!

    aws s3 rm --recursive s3://logs.$domain
    aws s3 rb s3://logs.$domain
    aws s3 rm --recursive s3://$domain
    aws s3 rb s3://$domain
    aws s3 rb s3://www.$domain
    aws s3 rm --recursive s3://codepipeline.$domain
    aws s3 rb s3://codepipeline.$domain

That last command will fail, so go use the web console to delete the
versioned codepipeline bucket.

### Delete CodeCommit Git repository

    # WARNING! DESTROYS CONTENT!

    #aws codecommit delete-repository \
      --region "$region" \
      --repository-name "$domain"
