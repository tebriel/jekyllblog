---
layout: post
title:  "Lambda and Using OAuth Refresh Tokens"
date:   2018-08-08 22:47:45 -0400
categories: aws lambda oauth
---

## Problem

Recently I have been working on a project for [Online Great Books](https://onlinegreatbooks.com/) stitching
together Slack Events and [Infusionsoft](https://www.infusionsoft.com/). The Infusionsoft API uses OAuth and
the token expires after 24 hours, and then you follow the "standard" flow to refresh it. The API token has a
refresh token in it, you use this to generate a new API token, which is again valid for 24h (and includes a
new refresh token).

## Solution Description

As "serverless" is the new thing, I thought I'd do this entire project with lambdas, but storage and
refreshing of the OAuth presented a problem I didn't know how to solve. I'm not exactly sure where I got the
idea, but someone suggested storing the token in [Parameter
Store](https://docs.aws.amazon.com/systems-manager/latest/userguide/systems-manager-paramstore.html) and then
using a CloudWatch Trigger to schedule a lambda which refreshes the lambda and updates ParamStore.

I gave my lambda the IAM policy for both get and put param:

```
Allow: ssm:GetParameter*
Allow: ssm:PutParameter*
```

And scheduled it to run every 20 hours (gives some breathing room). Now every time it runs, it fetches the
token from SSM, fetches a new token using the refresh flow, and then stores the updated token in SSM.

## Possible Issues

There is a possible race condition where I fetch a token in a different lambda before the refresh has started,
but try to use it after the refresh has finished. InfusionSoft tokens seem to be valid for the full 24 hours,
not getting blacklisted once the refresh token is used, so this doesn't seem to be a problem. If this were the
case, I'd likely make my other lambdas fetch the token at the very last moment and then retry the fetch if the
token comes back as expired.

Good luck, let me know if you have questions about this workflow, it's all written in terraform and python, so
parts of it are easily shared.
