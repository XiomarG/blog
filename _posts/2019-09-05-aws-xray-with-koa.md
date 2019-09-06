---
layout: post
title: Enable AWS X-Ray in koa project written in async/await
description: "because x-ray sdk plugin doesn't work with async/await"
modified: 2019-09-05
tags: [nodejs, aws, xray, koa]
image:
  path: https://github.com/XiomarG/blog/blob/gh-pages/images/beta-by-laptop.jpeg
  feature: beta-by-laptop.jpeg
---

Recently at work I was required to implement aws x-ray in our web service. There is official document
on [AWS website] (https://docs.aws.amazon.com/xray/latest/devguide/xray-sdk-nodejs.html). However it doens't
help much because the official plugin only supports `Express`. What we are using is `Koa`.

## Set up local X-Ray daemon
This is the first step for development and it's quite friendly. After starting the [daemon](https://docs.aws.amazon.com/xray/latest/devguide/xray-daemon-local.html),
I can configure xray sdk to publish packet to local udp port (127.0.0.1:2000) and the daemon will submit segment
to aws using credentials in `~/.aws/credentials`.

## Use x-ray to trace koa request
The root cause of the lack of support on Koa is that the xray sdk cannot work with async/await.
it uses `cls` for segment storage, which doesn't support async/await. When you call `AWSXRay.setSegment()`, it will
save it to a cls namespace. Later when you try to `getSegment()`, it returns empty.
As a result we cannot use automatic mode for x-ray. We have to manually create and close `segment` following this
[github repo](https://github.com/hongymagic/aws-xray-sdk-koa/blob/master/index.js).
By using koa middleware, we can start a segment at the beggining of processing the request and close it at the end.

```
const segment = new Segment(name, root, parent)
segment.addIncomingRequestData(new IncomingRequestData(ctx.req))
segment.close()
```
In the x-ray console, in order to see the correct url for the request, we must pass in correct `IncomingRequestData`.

## How to work with SQL?
The xray sdk supports SQL query subsegment as shown [here](https://docs.aws.amazon.com/xray/latest/devguide/xray-sdk-nodejs-sqlclients.html).
The first problem I faced was that we are using `sequelize` as an ORM for Postgres. I found a [github repo](https://github.com/jhairau/sequelize-aws-xray-pg)
But it doesn't work. I looked into their code and found that essentially it was simply calling `AWSXRay.captureMySQL(require('pg'));`. No wonder it won't work.
Even if i changed it to `AWSXRay.capturePostgres(require('pg'));` it didn't work because it was using cls in the lower level. I have to manually capture subsegment.

## Capture subsegment
According to [document](https://docs.aws.amazon.com/xray/latest/devguide/xray-sdk-nodejs-subsegments.html) I should use
`AWSXRay.captureAsyncFunc` to create subsegment for a function. Unfortunately it doens't work either because it uses cls as well.
From `aws-xray-sdk-core/lib/capture.js line 30` I know it's actually very easy to do it manually
```
const subsegment = segment.addNewSubsegment(name)
subsegment.close()
```
Just do these two lines before and after the function execution. If I want to do this for many functions, I'd better do it in a
decorator way. Suprisingly nodejs is going to support decorator in the future. It's only in stage II now. Our project is not using babel
so I have to wrap things myself.
```
async wrappedFn(...arg) {
    const subsegment = segment.addNewSubsegment(name)
    let res = await originalFunction.apply(this, arg)
    subsegment.close()
    return res
}
```

## Repeate this for the whole class
Apperently I cannot wrap all functions in our project. I decided to write a helper to wrap all functions in a class.
After some trial and error, I got this
```
function xrayObject(obj) {
    const className = obj.constructor.name;
    for(const method of Object.getOwnPropertyNames( Object.getPrototypeOf(obj))) {
        const temp = obj[method];    *1
        const subsegmentName = `${className}.${method}`;
        if(temp.constructor.name === 'AsyncFunction') {  *2
            obj[method] = async (...args) => {
                const subsegment = currentSegment.addNewSubsegment(name);
                const res = await temp.apply(obj, args);
                subsegment.close();
                return res;
            };
        } else {
            obj[method] = (...args) => {
                const subsegment = currentSegment.addNewSubsegment(name);
                const res = temp.apply(obj, args);
                subsegment.close();
                return res;
            };
        }
    }
    return obj;
}
```
*1 If I want to wrap the function and keep the function name, I have to save it to temp first. Otherwise the wrap
will create infinite loop.
*2 I didn't have this check before. Without it this wrapper will turn every function into async function. That is not what we want.

With this wrapper I can trace all function executions within a class, including database related ones.
