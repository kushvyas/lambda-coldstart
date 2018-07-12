## How to keep desired amount of AWS Lambda function containers warm ?

This being my first article this may not be up to mark , that being said I am writing this as I did not find comprehensive solution to the problem mentioned anywhere.

We all are aware of the so called **Hype** created by serverless in todays time and advantages is has over traditional approach of servers. AWS Lambda is a compute service that lets you run code without provisioning or managing servers. AWS Lambda executes your code only when needed and scales automatically, from a few requests per day to thousands per second. 

## What is Cold Start ?

There are many blog posts where you can read about what is cold start is and here I would try to explain it in short.

When a function is executed for the first time or after having the functions code or resource configuration updated, a container will be spun up to execute this function. All the code and libraries will be loaded into the container for it to be able to execute. The code will then run, starting with the initialisation code. The initialisation code is the code written outside the handler. This code is only run when the container is created for the first time. Finally, the Lambda handler is executed. **This set-up process is what is considered a cold start.**

For performance, Lambda has the ability to re-use containers created by previous invocations. This will avoid the initialisation of a new container and loading of code. Only the handler code will be executed. However, you cannot depend on a container from a previous invocation to be reused. if you haven’t changed the code and not too much time has gone by, Lambda may reuse the previous container. If you change the code, resource configuration or some time has passed since the previous invocation, a new container will be initialized and you will experience a ***cold start.***

#### Examples to further understand Lambda Execution

Consider the Lambda function, in the example, is invoked for the first time. Lambda will create a container, load the code into the container and run the initialisation code. The function handler will then be executed. This invocation **will have experienced a cold start**. If function takes 15 seconds to complete. After a minute, the function is invoked again. Lambda will most likely re-use the container from the previous invocation. This invocation **will not experience a cold start.**

Now **consider the second scenario**, where the second invocation is executed 5 seconds after the first invocation. Since the previous function takes 15 seconds to complete and has not finished executing, the new invocation will have to create a new container for this function to execute. Therefore **this invocation will experience a cold start.**

### The Solution that everyone knows !
There are many blogs and solutions that are already written in detail to solve the of problem of warming single container in Lambda. I will try to explain it in short:

Regarding preventing cold starts, this is a possibility, however, it is not guaranteed, the common workaround will only keep warm one container of the Lambda function. To do, you would run a CloudWatch event using a schedule event (cron expression) that will invoke your Lambda function every couple of minutes to keep it warm.

### The Next Problem !
The issue - such approach allow to keep warm only one container of Lambda function while the actual number of parallel calls from different users can be much larger in some case that's hundreds and sometimes even thousands of users. **Is there any way to implement warm up functionality for Lambda function which could warm not only single container, but some desired number of them?**

### The Workaround:
For this use-case, your Lambda function **will be invoked very frequently with a very high concurrency rate.** To avoid as many cold starts as possible, you will need to keep warm as many containers as you expect your highest concurrency to reach. To do this you will **need to invoke the functions with a delay to allow the concurrency of this function to build and reach the desired amount of concurrent executions**. This will force Lambda to spin up the number of containers you desire. *That being said, here is a break down on how you can keep multiple containers for your function warm at one time:*
- You should have a **CloudWatch Events Rule that is triggered on a schedule**. This schedule can be a fixed rate or a cron expression. for example, You can set this rule to trigger every 5 minutes. You will **then specify a Lambda function (Controller function) as the target of this rule.**

- Your **Controller Lambda function** will then **invoke the Lambda function (Function that you want to be kept warm) for as many concurrent running containers as you desire**. There are a few things to consider here:

1.  You will have to build concurrency because if the first invocation is finished before another invocation starts then this invocation may reuse the previous invocations container and not create a new one. To do this you will need to add some sort of delay on the Lambda function if the function is invoked by the controller function. **This can be done by passing in a specific payload to the function with these invocations. The lambda function that you want to be kept warm will then check if this payload exists. If it does then the function will wait (to build concurrent invocations), if it does not then the function can execute as expected.**

2. You will also need to ensure you are not getting throttled on the Invoke Lambda API call if you are calling it repeatedly. Your Lambda function should be written to handle this throttling if it occurs and consider adding a delay between API calls to avoid throttling.

## Bonus : What is Concurrency in Lambda ?
There has been lot of confusion regarding this and many are too lazy to read the AWS Documentation where it is explained beautifully, nevertheless concurrent executions refers to the number of executions of your function code that are happening at any given time. The number of concurrent functions can be estimated by the following: 

`Invocations per second * function duration.`

For example, let's say you invoke your function 100 times in a second. However, your average function duration is 100 ms. You can estimate that your function will have a concurrency of 10 (100*0.1).

AWS Lambda will dynamically scale capacity in response to increased traffic, subject to your account's Account Level Concurrent Execution Limit (default 1000). Your function will scale to its Immediate Concurrency Increase value. This value is different per each region. A table displaying these values can be found here. If this limit is reached, Lambda will begin adding 500 concurrent invocations per minute. All requests made before this increase will get throttled and will need to be retried.

### The Conclusion
This solution can reduce cold starts but it will increase costs and will not guarantee that **cold starts will occur as they are inevitable when working with Lambda**. If your application needs faster response times then what occurs with a Lambda cold start, I would recommend looking into having your server on a EC2 instance.
