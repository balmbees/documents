# How we made Vingle's Front-end web server?
## Microservice web-server with AWS API Gateway and Lambda

### Writer  
*Front-End Developer*  
*Tylor Shin*

### Background
Original Vingle system was constituted with the Ruby on Rails as monolithic system. It means Rails served both Front-end logic and API server logic. this architecture has problem that the server logics and the client logics are handled in same logic and environment. So, when we tried to change subtle styles or add small admin feature, we had to wait server deploy and affected by server rollback.

Vingle is the one of the most user-friendly service and has many frequent client-side changes. But in the system we had, we can't deploy and manage our front-end server independently.

After all, we decide to make new architecture and reduce server cost and RoR's logic quantity. and we are inspired by the serverless architecture and it's frameworks like [Serverless](https://serverless.com), [APEX](https://github.com/apex/apex)

But there seemed to no needs to use framework yet, we made deploying and managing environment ourselves.

### Requirements
1. Ensure auto handling scalability. ( I don't want to think about scalability :-( )
2. Should server HTML. (if it is possible, server-side rendering(isomorphic-rendering) will be better)
2-1. Should serve SEO logic if SSR doesn't work.
3. Should serve proper target index.html following by user's device.(Mobile, Desktop)
4. Should handle logics that was in Rails.(ex: Authentication)
5. Should make the best use of the CDN such as Akamai.
6. Should do initial rendering in 1000ms.

### Architecture
[image]
1. Rails(it will be replaced with Route53 or Nginx) request index.html to API Gateway.
2. API Gateway invokes Lambda function with the previously set *stage variables* and *HTTP Header*, *active version of the each platform*, *User path*, *meta data*
3. Lambda requests index.html to S3 via AWS-SDK or CDN with each platform's deployed active version and metadata. (If lambda container survives)
