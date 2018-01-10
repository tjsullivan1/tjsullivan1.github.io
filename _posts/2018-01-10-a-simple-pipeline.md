# The Challenge

As I have started building Elastic Beanstalk environments for developers in my organization, I have grown tired of redeploying files with their source code. We don't have any CI/CD pipelines or automated deployment methods in the cloud (we have a homegrown tool on-prem that will copy a war file to production after a developer submits a page in our TWiki), so this is very likely not the best method of doing this. Also, because we have never done CI/CD, we don't have great testing (if any), and this pipeline will reflect that. Since AWS is our cloud provider of choice, I decided to look at the native tooling and built myself a simple pipeline that will rebuild a PHP Elastic Beanstalk environment upon a commit to the master branch of an AWS CodeComit repository.

# The Solution

I broke down the solution into its component parts: CodeCommit, Elastic Beanstalk, and CodePipeline with CodeBuild.

## CodeCommit

For CodeCommit, a simple repository with a single branch will do. The basic process is:
1. Login to your AWS account.
1. Confirm that you are in the desired region by looking at the top right section of the header. Select a different region if desired.
1. Click Services.
1. Click CodeCommit under "Developer Tools" or search for it in the search box.
   * If you have never created a repository in this region, you will see the splash screen. Click get started.
   * If you have created a repository in this region, you will see a list of your repositories. Click "Create repository" above the Filter box.
1. Enter a repository name for your project. Click "Create Repository" again.
1. The wizard prompts you to configure email notifications. Click Skip.
1. Congratulations, you've created a Codecommit repository! Now you must clone it and commit at least once to get a branch created.

## Elastic Beanstalk

If you haven't deployed Elastic Beanstalk before, please note that it has many more configuration options than what I am going to show you.

1. Confirm that you are in the same region as your CodeCommit repository. This is not necessary, but I like to be consistent.
1. Click Services.
1. Click "Elastic Beanstalk" under Compute or search for it in the serach box.
   * If you have never created an environment in this region, you will see the splash screen. Select PHP from the dropdown in the center of the screen and click "Launch Now". This will launch a configured sample app and you have completed this section.
   * If you have used Elastic Beanstalk in this region before, you will see a list of your existing applications. Click "Create New Application" on the upper right hand side of the page.
1. Provide a name for the application and click "Next".
1. We are going to "Create web server" for this example.
1. Use the predefined PHP configuration with the "Load balancing, auto scaling" environment type.
1. We will use the sample application and all default settings for the Application Version. Click "Next".
1. Check that your URL is available by clicking "Check availability". If it is, click "Next".
1. Check the box to "Create this environment inside a VPC", then click "Next".
1. Under configuration details:
   * Select an appropriate instance type. Note: t2.micro is the only instance type in the free tier, but this application deployment will cost you something regardless because of the ELB.
   * Select a pre-existing EC2 key pair.
   * Add an email address for yourself -- this is pretty cool so you can watch your Pipeline destroy and rebuild the Elastic Beanstalk environment.
   * Click "Next".
1. Add any tags you want, then click "Next".
1. Under VPC Configuration:
   * Select the availability zones where you would put load balancers and web servers.
   * Select a security group that you would use for web servers.
   * Click "Next".
1. Select an appropriate "Instance profile" and "Service role". I allowed AWS to create the defaults, and clicked "Next".
1. On the next page, review for accuracy, then click "Launch".
1. Congratulations, you've created an Elastic Beanstalk environment!

## CodePipeline

1. Confirm that you are in the same region as your CodeCommit repository.
1. Click Services.
1. Click "CodePipeline" under "Developer Tools" or search for it in the serach box.
   * If you have never created a pipeline in this region before, you will see the splash screen. Click "Get Started".
   * If you have used created a pipeline in this region before, you will see a list of your existing pipelines. Click "Create pipeline" above the filter box.
1. Enter a pipeline name for your project. Click "Next step".
1. Select "AWS CodeCommit" as your source provider.
1. Select your previously created repository and the branch name that you want to contain your deployed code. Click "Next step".
1. Select "AWS CodeBuild" as your "Build provider".
1. Select "Create a new build project" and add a project name in the proper field.
1. Select Ubuntu as your OS. Select Base as the runtime. There should only be one version to select in the version directory. Select that.
1. If you have an existing role, feel free to choose it, otherwise create a service role.
1. Leave VPC ID as "No VPC". Click "Save build project".
1. Click "Next step".
1. Select "AWS Elastic Beanstalk" as your deployment provider.
1. Choose our previously created application as your application name. Choose its environment in the environment name section. Click "Next step".
1. Click "Create role" if you do not have a previously created CodePipeline Service role. Click "Next step".
1. Review, and then click "Create pipeline".

# Testing it out

Okay, you have the pipeline, but now what? Well, you could try committing your source code to your CodeCommit repository, but the build will fail. Why? Because we never created the buildspec.yml file! I will be honest, I don't completely understand the nuances of this file, but it is pretty powerful.

The simplest version I could get away with (to remove all *actual* build phases) was this:

{% highlight yaml %}
version: 0.2

artifacts:
  files:
    - '**/*'
  base-directory: 'source'
{% endhighlight %}

In order for this to work, we must move the source code for our PHP application to a subfolder in our git repository called "source". From there, you can save this file as buildspec.yml to the root of the repository and commit. Then, push the repository and you are off to the races!

If you watch your pipeline (by navigating back to CodePipeline and clicking on the name of your pipeline), you will be able to see -- without refreshing -- the source push get recognized and processed, the source passing onto the build, and the build redeploying to Elastic Beanstalk!

# Conclusion

As I mentioned, this is a quick and dirty pipeline, and it is not the ideal setup for important production applications. Nevertheless, for the applications that have minimal SLA requiremetns, this is a quick solution to remove roadblocks from getting new features into the hands of your users!
