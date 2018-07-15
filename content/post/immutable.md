+++
author = "Brandon Nicoll"
date = 2015-10-18T13:34:18Z
description = ""
draft = false
slug = "immutable"
image = "/Immutable.jpg"
tags = ["AWS", "Automation"]
title = "Automating with the AWS API: Enabling Immutable Infrastructure"

+++

<img src="/Immutable.jpg" style="max-width: 100%" />
One of the biggest advantages to cloud computing is the ability to quickly react to an unknown system load. A well-known pattern within AWS is to make use of the [Auto Scaling](https://aws.amazon.com/autoscaling/) feature and adjust total computing capacity accordingly. That sounds great, but how do you go from this concept to an actual, working application? How do you deploy changes? Most important of all, how do you automate this process once you understand the architecture? Many of those questions can be answered with clever use of [Amazon Machine Images (AMIs)](http://docs.aws.amazon.com/AWSEC2/latest/UserGuide/AMIs.html). Here, I'll provide an example of a (rudimentary) system that exposes an endpoint to automate creation of a new, pre-configured AMI that will be ready for deployment into an Auto Scaled group.


### Permissions Changes
Before starting, you'll want to make sure the [key](http://nicoll.io/keys) used to interface with AWS has the following permissions enabled:
<li>[Run Instances](http://docs.aws.amazon.com/AWSEC2/latest/APIReference/API_RunInstances.html)
<li>[Describe Instances](http://docs.aws.amazon.com/AWSEC2/latest/APIReference/API_DescribeInstances.html)
<li>[Create Image](http://docs.aws.amazon.com/AWSEC2/latest/APIReference/API_CreateImage.html)
<li>[Describe Images](http://docs.aws.amazon.com/AWSEC2/latest/APIReference/API_DescribeImages.html)
<li>[Terminate Instances](http://docs.aws.amazon.com/AWSEC2/latest/APIReference/API_TerminateInstances.html)


Your [policy](http://docs.aws.amazon.com/IAM/latest/UserGuide/access_policies.html) will look something like this:
{{< highlight json >}}
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Action": "ec2:RunInstances",
            "Effect": "Allow",
            "Resource": "arn:aws:ec2"
        },
        {
            "Action": "ec2:DescribeInstances",
            "Effect": "Allow",
            "Resource": "arn:aws:ec2"
        },
        {
            "Action": "ec2:CreateImage",
            "Effect": "Allow",
            "Resource": "arn:aws:ec2"
        },
        {
            "Action": "ec2:DescribeImages",
            "Effect": "Allow",
            "Resource": "arn:aws:ec2"
        },
        {
            "Action": "ec2:TerminateInstances",
            "Effect": "Allow",
            "Resource": "arn:aws:ec2"
        }
    ]
}
{{< / highlight >}}

### Making Endpoints Meet
I set out to make a simple system in which users would be able to automate deployments of their projects to an Auto Scaled group. The first step in this process would be to allow users to expose an endpoint that, when fired, will do the following:
<li>Provision and configure a new EC2 instance
<li>Freeze the instance's state in an AMI
<li>Terminate the instance

This endpoint can be utilized as a [GitHub webhook](https://developer.github.com/webhooks/), [Git hook](http://git-scm.com/book/en/v2/Customizing-Git-Git-Hooks), or even as part of a larger deployment pipeline after a number of build and test steps have successfully executed. I decided to call this system [Quadraxis](https://github.com/bgnicoll/quadraxis); I created a new [Laravel](http://laravel.com/) project, added the [AWS SDK for PHP](https://packagist.org/packages/aws/aws-sdk-php) to the composer.json file, then added the [keys](http://nicoll.io/keys) to the [.env](https://github.com/vlucas/phpdotenv) file. If you're interested, [Quadraxis](https://github.com/bgnicoll/quadraxis) is on GitHub, but the actual implementation of the system is less important than the concepts behind automating AMI creation. 

### Using User Data
A crucial piece of this puzzle is solved with utilization of the [User Data](http://docs.aws.amazon.com/AWSEC2/latest/UserGuide/user-data.html) feature when creating an instance. User data can be passed to the API when launching an instance to provide an initialization script that will run once the instance is ready. I saved the exact script I used for this example in [this Gist](https://gist.github.com/bgnicoll/0c3cd95dd6d91d2bcbb7) for reference. Note that when trying to execute a bash script in User Data, it is important to include "#!/bin/sh" (or wherever shell might reside on your flavor of OS) as the first line. Also note this value needs to be base 64 encoded when submitting to the API. Of course, this step is made easier in this example by having the project in a public GitHub repository and written in an interpreted language. The endpoint created by Quadraxis will find the saved initialization script for the specific project and apply it when launching the instance.
{{< highlight php >}}
    public function webhook(Request $request, $name)
    {
        //Find the project
        $project = Project::where('name', $name)
                        ->get()
                        ->first();
        if (is_null($project)){
            abort(404);
        }
        
        //Instantiate the EC2 client
        $ec2Client = Ec2Client::factory(array(
            'region'  => getenv('AWS_REGION'),
            'version' => '2012-10-17'
        ));

        //Launch the instance from a pre-determined AMI with saved
        //user data and into a pre-configured security group
        $result = $ec2Client->runInstances(array(
            'ImageId'        => $project->base_ami_id,
            'MinCount'       => 1,
            'MaxCount'       => 1,
            'InstanceType'   => 't2.medium',
            'UserData'       => base64_encode(
                                $project->init_script),
            'SecurityGroups' => array('torvus-sec-group')
        ));
        
        //Save the new instance ID for later
        $instanceId = $result->search('Instances[0].InstanceId');
    }
{{< / highlight >}}

### Waiter, There's a Fly in My Soup
Before trying to create the image, the instance needs to be in a "running" state. We have a couple of options, such as polling [DescribeInstances](http://docs.aws.amazon.com/AWSEC2/latest/APIReference/API_DescribeInstances.html) until the instance is ready, or we could try using a "waiter". [Waiters](http://docs.aws.amazon.com/aws-sdk-php/v3/guide/guide/waiters.html) are a useful feature in many of the AWS SDKs that will allow you to wait for a resource to be in a specific state. We can extend the endpoint code further to wait until the instance is in a running state with a waiter.
{{< highlight php >}}
        $ec2Client->waitUntil('InstanceRunning', [
            'InstanceIds' => array($instanceId)
        ]);
{{< / highlight >}}

### A Positive Self-Image
Once the instance is in a running state, we can issue the command to [create an image](http://docs.aws.amazon.com/AWSEC2/latest/APIReference/API_CreateImage.html). 
{{< highlight php >}}
        $result = $ec2Client->createImage(array(
            'InstanceId' => $instanceId,
            'Name' => $project->name . time()
        ));

        $imageId = $result->search('ImageId');
{{< / highlight >}}

### Custom "Waiter"
Now we need to wait for the image creation to finish so we can terminate the instance, but there isn't a "wait for this image to be available" waiter out-of-the-box for the AWS SDK for PHP(at least I don't think there is). I simply wrote a loop that checks every thirty seconds for the image to be available. 

{{< highlight php >}}
        while (1) { 
            $result = $ec2Client->describeImages(array(
                'ImageIds' => array($imageId)
            ));
            $imageState = $result->search('Images[0].State');
            if ($imageState == 'available') {
                break;
            }
            sleep(30);
        }
{{< / highlight >}}

### Terminate the Instance
We have our deployable artifact, so we can terminate the instance used to create the AMI with the [TerminateInstances](http://docs.aws.amazon.com/AWSEC2/latest/APIReference/API_TerminateInstances.html) command:
{{< highlight php >}}
        $result = $ec2Client->terminateInstances(array(
            'InstanceIds' => array($instanceId)
        ));
{{< / highlight >}}

### Wrapping Up
With this AMI, we now have what we need to start changes to an application down a deployment pipeline. You may be wondering about how to solve problems like environment configuration, managing DNS, or cycling out old instances in an automated fashion. I'm planning on continuing writing more functionality in Quadraxis to address these problems and using it as a backdrop for more posts like this one. Stay tuned! In the meantime, let me know what you think on Twitter, [@bgnicoll](https://twitter.com/bgnicoll).