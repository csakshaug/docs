# Dealing with CORS

> No 'Access-Control-Allow-Origin' header is present on the requested resource

If you have seen the error message above in your web application, this is the correct page to have a look at. This error is related to a topic called Cross-Origin Resource Sharing (CORS).

## Understanding the Same-origin policy

First of all, you need to know that browsers have a policy called the **Same-origin policy** for security reasons. Under this policy, a script inside a web page can access other web page's data only if they both have the same **origin**. An **origin** consists of URI scheme, host name, and port number. This means, a script inside `http://example.com` can only access data within the same origin such as `http://example.com/users`. Trying to access data in other origins such as `http://sample.com/users` will lead to a violation error. You don't want arbitrary scripts sending to or retrieving data from malicious web sites so the **Same-origin policy** plays a very important role in order to protect you from these threats. 

## Problems in Developing Web Applications Locally

Despite the pros, there are situations where the **Same-origin policy** gets in the way. What if you wanted to develop a Single Page Application locally and access data via OpenFaaS functions? Often, developers get stuck with an error telling you that you don't have a `Access-Control-Allow-Origin` header present. This is because your local web server and the OpenFaaS gateway have **different origins** as illustrated below.

<p align="center">
  <img src="/images/cors/cors1.png">
</p>

## Using Reverse Proxies as a Workaround

Mainly there are two ways to deal with this problem. Either set a CORS header (which is not covered in this page) or make the browser **think** it is talking to the same origin using reverse proxies. If you place a reverse proxy inside or in front of your web server, you can route specific requests to specific targets. For example, you can route all ajax requests in your web application which start with `/api` to the OpenFaaS gateway by proxying it from the web server as illustrated below.

<p align="center">
  <img src="/images/cors/proxy1.png">
</p>

This way, the request from the browser doesn't violate the **Same-origin policy** and under the hood, would be proxied to the OpenFaaS gateway.

## An Example with a Vue.js + Webpack Application

Let's look at a very minimal Vue.js Single Page application setup with webpack to actually see how it works. Typically, a web developer will run `npm run dev` to spin up a local webpack dev server. Suppose this was served on `http://localhost:8081`. By default, the OpenFaaS gateway will be exposed on `http://localhost:8080` which means the local dev server and the OpenFaaS gateway have **different origins**. Therefore, if the Vue.js application requests data from the OpenFaaS gateway (`http://localhost:8080`), it will fail because it violates the **Same-origin policy**. The following diagram shows the relation.

<p align="center">
  <img src="/images/cors/cors2.png">
</p>

Here's a quick view of the code trying to call the `echo` function from the Vue.js application:

```js
// URL to the echo function
const url = 'http://localhost:8080/function/echo';

// Sample payload
const data = {
  test: 'test'
};

// AJAX request
fetch(url, {
  body: JSON.stringify(data),
  method: 'POST'
}).then(res => res.json())
```

and ends up with the following error:

> Failed to load http://localhost:8080/function/echo: No 'Access-Control-Allow-Origin' header is present on the requested resource. Origin 'http://localhost:8081' is therefore not allowed access. If an opaque response serves your needs, set the request's mode to 'no-cors' to fetch the resource with CORS disabled.

In order to solve this problem, we'll need a reverse proxy to proxy the request from the webpack dev server to the OpenFaas gateway. Luckily, the webpack dev server has a built-in feature for this by taking advantage of the [devServer.proxy](https://webpack.js.org/configuration/dev-server/#devserver-proxy) feature. You can simply write a definition as follows in your webpack configuration:

```js
proxy: {
  '/function': 'http://localhost:8080'
}
```

With this, all your requests starting with `/function` will be proxied to the gateway (http://localhost:8080) which is exactly the behavior we want. The following diagram shows how the data flows.

<p align="center">
  <img src="/images/cors/proxy2.png">
</p>

Now that the requests would be proxied, the previous code will look like this:

```js
// URL to the echo function.
// Note that the origin isn't needed since we request to the same server.
const url = '/function/echo';

// Sample payload
const data = {
  test: 'test'
};

// AJAX request
fetch(url, {
  body: JSON.stringify(data),
  method: 'POST'
}).then(res => res.json())
```

Which will end as a successful request.