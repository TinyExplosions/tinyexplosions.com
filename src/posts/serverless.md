---
title: Serverless on OpenShift
date: '2020-08-26'
tags:
  - homelab
  - tech
  - blog
  - knative
  - OpenShift
  - serverless
---

Serverless is big these days. Everyone wants to use the likes of [AWS's Lambda](https://aws.amazon.com/lambda/) to save some costs thanks to not paying for idle processes, to help with scaling, and cos it's the cool thing to do. Thankfully, with knative, it's now come to OpenShift, so you can get the benefits of serverless wherever OpenShift is running. Given the popularty in this functionality, I thought I'd dive in and see how quick it was to set up on my lab, and was surprised at how quick I got up and running.


### Enabling serverless on OpenShift
The first step in going serverless on OpenShift is installing the Serverless Operator through operatorhub, making sure to install for all namespaces, on the relavant channel, with Automatic approval. [The official documentation](https://docs.openshift.com/container-platform/4.5/serverless/installing_serverless/installing-openshift-serverless.html#serverless-install-web-console_installing-openshift-serverless) is very clear, and takes one through everything required. Once the operator is installed, I then installed knative serving and knative eventing (once again following the docs), and in just a few minutes it looked like everything was installed and working.


### My first service


Once the Serverless Operator was installed, it was time to look at deploying an *actual* application. [Our Documentation](https://docs.openshift.com/container-platform/4.5/serverless/serving-creating-managing-apps.html#creating-serverless-applications-using-the-developer-perspective) is a good starter, and will get you up and running with the sample 'Hello World' service in short order. However, being an awkward cuss, I wanted to try something different, and a good candidate is [my first app](https://tinyexplosions.com/posts/my-first-app/) - it's a simple, self contained API, and is a great candidate.

To get up and running, I created an application is the same manner outlined in the previous post, and checked the 'Knative Service' option under the resource type to generate. Then it was build, and..... nothing. The service didn't start up correctly, even though the build had completed. It was then I remembered that my application starts up on port 3000 rather than 8080. Thankfully, this can be defined in the YAML for the Serverless Service by adding a `containerPort` declaration.

```yaml
spec:
  containers:
    ports:
      - containerPort: 3000
```

Once this was added, the service completed creation, and all conditions on the service page showed up as 'True'. From there, it was a matter of hitting the URL, and seeing an Adventure Time quote returned!

### Performance

We know that performance suffers with serverless and cold starts, so I thought I'd log a couple of tests to quantify it. These are individual runs, and absolutely not scientific (there may be future posts in making things perform better), but it's interesting to see that we go from 6 seconds to 21 milliseconds!

[![Postman screen capture showing cold retrieval of data from service is 6.12s](/images/serverless-stats-cold.png "Cold start, so not expecting fantastic performance on this request.")](/images/serverless-stats-cold.png)

[![Postman screen capture showing warm retrieval of data from service is 21ms](/images/serverless-stats-warm.png "Performance is greatly improved when the pod is running ;)")](/images/serverless-stats-warm.png)

All in all this was a a lot more straightforward to get this up and running, and an application deployed. Kudos must go to the OpenShift team for integrating knative so well into the product, and I'll certainly be using it frequently in future.
