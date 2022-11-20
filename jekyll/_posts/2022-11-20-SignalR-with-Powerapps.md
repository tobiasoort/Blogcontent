---
layout: post
title:  "SignalR with Azure, PowerApps and PCF"
date:   2022-11-20 19:01:03 +0100
categories: general
---
I'm trying some new things. One of those things is PowerApps. Another is SignalR and live communication. Using live push can really lower the load on systems as you're actively preventing 'brick on F5' behaviour.

The awesome [Greg Hurlman](https://github.com/ghurlman) did a talk about using PCF, which is basically an extension framework for PowerApps built in TypeScript, to add support for SignalR to PowerApps. The talk is available [here](https://www.youtube.com/watch?v=oVDrdr7Gxzk) and the code is in Greg's GitHub. Since I've not seen a lot about this online, I decided to blog about it here.

## How it works

The setup works because PCF is basically TypeScript, so it gets run anywhere the PowerApps client runs (which in turn is mainly built upon open web technology). Long story short, you're shipping Javascript to the client. And we can do SignalR in JS very well, thank you.
You could choose to run your own SignalR service, but with pricing for most setups is [free or nearly free](https://azure.microsoft.com/pricing/details/signalr-service/) I don't really know why you would go through the hassle. Just set up a SignalR hub in Azure, and all you need is an app to be your 'first hop' and negotiate for you with Azure SignalR. This function can also be serverless (so on-demand) so it will literally only cost you a few cents of storage and scale upon demand. That's great use of the cloud!

For this app, Greg uses a very very very simple Azure Function. The trick here is that using the Function Bindings, all the heavy lifting is done by the Azure infrastructure. The function to negotiate a connection is only a few lines:

```javascript
module.exports = async function (context, req, connectionInfo) {
  context.res.json(connectionInfo);
};
```

This works because you can tell Azure what the function input should be bound to. With this bit of json in `function.json`:

```json
{
  "disabled": false,
  "bindings": [
    {
      "authLevel": "anonymous",
      "type": "httpTrigger",
      "direction": "in",
      "name": "req"
    },
    {
      "type": "http",
      "direction": "out",
      "name": "res"
    },
    {
      "type": "signalRConnectionInfo",
      "name": "connectionInfo",
      "hubName": "chat",
      "direction": "in"
    }
  ]
}
```

Then finally all you need is to add the `AzureSignalRConnectionString` key to the `host.json` file in your function app root. Voila! You can also stick it in as an app config parameter in the portal.

## Sending data

Since it's hard for PowerApps to send data into a Websocket, Greg created an Azure Function called 'SendMessage' which does exactly what it says on the tin. A tiny bit of code allows the client to simply send a HTTPS POST which then gets passed on to SignalR. It even fishes out the loggedin users name from the headers and passes it as part of the SignalR message. Neat!

```javascript
module.exports = function (context, req) {
    const message = req.body;
    if (req.headers && req.headers['x-ms-client-principal-name']) {
        message.sender = req.headers['x-ms-client-principal-name'];
    }
        
    context.bindings.signalRMessages = [{
        "target": "newMessage",
        "arguments": [ message ]
    }];
    context.done();
};
```

## Securing it

The thing Greg didn't mention was security. I'm going to try to use Azure AD authentication for the Function, which should allow me to limit usage of the function, and with that, to the SignalR hub. It should work similar to other PowerApps connectors that require the user to sign in. When I got that going, i'll blog about it!

I hope you learned something about how TypeScript is a great way to extend and augment PowerApps, and how we can leverage Azure Functions in a Serverless environment to set up these ridiculously performant and complex platforms, for literally a few cents per month. 
