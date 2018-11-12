<div align="center">
<a href="https://github.com/eddiejibson/chae-limitrr"><img alt="chae" src="https://cdn.oxro.io/chae/img/limitrr.png" width="432.8" height="114.2"></a>
<br>
<br>
<a href="https://discord.gg/C8MtjUm"><img src="https://img.shields.io/discord/498178136517640194.svg"></a>
<a href="https://paypal.me/eddiejibson/5"><img src="https://img.shields.io/badge/donate-PayPal-brightgreen.svg"></a>
<img src="https://requires.io/github/eddiejibson/chae-limitrr/requirements.svg?branch=master">
<img src="https://img.shields.io/npm/dw/limitrr.svg">
<p>Light Express rate limiting using Redis.</p>
</div>

Limitrr assists with the rate-limiting of various routes within your Express application. Unlike other similar packages, this utility allows the user to limit not only by the number of requests but also the number of completed actions (e.g allowing a certain amount of accounts to be successfully created within a timespan) and have such restricted with custom options. As well as this, custom discriminators are possible - you no longer have to limit by just the user's IP.

**Pull Requests are welcomed**

# Roadmap

[![Codacy Badge](https://api.codacy.com/project/badge/Grade/ac214dc580fb46f3bbf8735abda4a4ef)](https://app.codacy.com/app/jibson/chae-limitrr?utm_source=github.com&utm_medium=referral&utm_content=eddiejibson/chae-limitrr&utm_campaign=Badge_Grade_Dashboard)

- [x]   Create multiple route limits (separate expiry times e.t.c) without having to initialize the Limitrr class multiple times
- [ ]   Pass parameters into functions via an object

# Install

``` bash
npm install limitrr --save
```

# Quick Guide

## Basic Usage

``` javascript
const express = require("express");
const app = express();
const Limitrr = require("limitrr");
const limitrr = new Limitrr({
    //Redis keystore connection information
    "redis": {
        "password": "supersecret",
        "host": "666.chae.sh",
        "port": 6379,
        "family": 4
        //There are many more options all of which can be seen further into the documentation under the "configuration" title
    },
    //General limitrr options
    "options": {
        "expiry": 900,
        //There are many more options all of which can be seen further into the documentation under the "configuration" title
    },
    "routes": {
        //Overwrite the default options
        "default": {
            //You do not have to overwrite all the options, only the
            //ones needed.
            "requestsPerExpiry": 500
        },
        "createUser": {
            //Not all keys need to be set here either, they will resolve
            //to the default if not set.
            "requestsPerExpiry": 300
        }
    }
});

app.use(limitrr.getIp()); //Middleware to get a user's IP - even if the application is behind Cloudflare
//This is assigned to req.realIp

//Simple rate limiting
app.get('/', limitrr.limit(), (req, res, next) => {
    res.send('Hello World!')
});

app.get("/registerUser/:user", limitrr.limit(), (req, res, next) => {
    //Non intensive actions like simple verification will have a different limit to intensive ones.
    //and will only be measured in terms of each request via the middleware.
    //No further action is required.
    if (req.params.user.length < 5) {
        //Dummy function creating user
        someRandomModule.registerUser().then((result) => {
            //Intensive actions like actually registering a user should have a
            //different limit to normal requests, hence the completedActionsPerExpiry option.
            //and should only be added to once this task has been completed fully
            //In this example, we will be limiting the amount of completed actions a certain IP can make.
            //Anything can be passed in here, however. For example, a email address or user ID.
            //req.realIp was determined by calling the middleware earlier - limitrr.getIp()
            limitrr.complete(req.realIp); //Calling will add to the completed count
            //Bearing in mind this a promise.
            limitrr.complete(req.realIp).then((result) => {
                //In this example, we will be returning a
                //success message as the action has been completed.
                return res.status(200).json({
                    "message": "Success!"
                });
            }).catch((err) => {
                //Handle error
            });
        });
    }
});

app.listen(3000, () => console.log(`Limitrr example app listening on port 3000!`))
```

## Get the value of a certain key

### limitrr.get()

**Returns**: Promise

``` javascript
limitrr.get(discriminator, route)
```

#### Parameters

- **discriminator**: **Required** Where discriminator is the thing being limited (e.g x amount of completed actions/requests per discriminator)
- **route**: *String* When route should the values be retrieved from? If this is not set, it will get the counts from the default route.

```javascript
limitrr.get(discriminator, route) //Besides discriminator, all parameters are optional.
//If type is not passed into the function, it will
//return both the amount of requests and completed actions


//Where discriminator is the thing being limited
//e.g x amount of completed actions/requests per discriminator
limitrr.get(discriminator)

//This tends to be the user's IP.
limitrr.get(req.realIp)
//This will return both the amount of requests and completed actions stored under the
//discriminator provided in an object. You can handle like this:
limitrr.get(req.realIp).then((res) => {
    console.log(`${res.requests} Requests`);
    console.log(`${res.completed} Completed Tasks`);
}).catch((err) => {
    //Handle error
});

//The above example would get the request and completed task count from the default
//route. If you would like to retrieve values from a different route, you can specify
//this as well. It can be done like this:
limitrr.get(req.realIp, "exampleRouteName").then((res) => {
    console.log(`${res.requests} Requests made through the route exampleRouteName`);
    console.log(`${res.completed} Completed Tasks made through the route exampleRouteName`);
}).catch((err) => {
    //Handle error
});

```

## Removal of values from certain request/completed keys

### limitrr.reset()

**Returns**: Promise

``` javascript
limitrr.reset(discriminator, type, route)
```

#### Parameters

- **discriminator**: **Required** Where discriminator is the thing being limited (e.g x amount of completed actions/requests per discriminator)
- **type**: Which count do you wish to be reset? `requests` or `completed`? If you want both removed, set this to false. Or, don't pass this value in at all.
- **route**: *String* When route should the values be retrieved from? If this is not set, it will get the counts from the `default` route.

``` javascript
//Where discriminator is the thing being limited
//e.g x amount of completed actions/requests per discriminator
//This will remove both the amount of requests and completed action count
limitrr.reset(discriminator);

//This tends to be the user's IP.
limitrr.reset(req.realIp);

//If you want to remove either one of the amount of requests or completed actions.
//but not the other, this can be done too.
//The value passed in can either be "requests" or "completed".
//In this example, we will be removing the request count for a certain IP
limitrr.reset(req.realIp, "requests").then((res) => {
    if (res) {
        console.log("Requests removed");
    }
}).catch((err) => {
    //Handle error
});

//If you wish to reset counts from a particular route, this can be done as well.
//The second parameter for the type has been set to false as we want both
//the request and completed count removed - not just one.
limitrr.reset(req.realIp, false, "exampleRouteName").then((res) => {
     if (res) {
        console.log("Requests removed from the route exampleRouteName");
    }
}).catch((err) => {
    //Handle error
});
```

# Configuration

## redis

**Required**: false

**Type**: Object

**Description**: Redis connection information.

- **port**: *Integer* Redis port. Defaults to: `6379`

- **host**: *String* Redis hostname. Defaults to: `"127.0.0.1"`

- **family**: *Integer* If Redis hostname is IPv4 (4) or IPv6 (6). Defaults to: `4` (IPv4)

- **password**: *String* Redis password. Defaults to: `""`

- **db**: *Integer* Redis DB. Defaults to: `0`

### Example of the redis object that could be passed into Limitrr

``` javascript
"redis": {
  "port": 6379, //Redis Port. Required: false. Defaults to 6379
  "host": "127.0.0.1", //Redis hostname. fequired: False. Defaults to "127.0.0.1".
  "family": 4, //Is redis hostname IPv4 (4) or IPv6 (6)? Required: false. Defaults to 4 (IPv4).
  "password": "mysecretpassword1234", //Redis password. Required: false. Defaults to "" (empty).
  "db": 0 //Redis DB. Required: false. Defaults to 0.
}
```

## options

**Required**: false

**Type**: Object

**Description**: Various options to do with Limitrr.

- **keyName**: *String* The keyname all of the requests will be stored under. This is mainly for aesthetic purposes and does not affect much. However, this should be changed on each initialization of the class to prevent conflict. Defaults to: `"limitrr"`
- **errorStatusCode**: *Integer* Status code to return when the user is being rate limited. Defaults to `429` (Too Many Requests)
- **catchErrors**: *Boolean* Should important errors such as failure to connect to the Redis keystore be caught and displayed? If this is set to false, it will throw an error instead. Defaults to `true`.

### Example of the options object that could be passed into Limitrr

``` javascript
"options": {
  "keyName": "myApp", //The keyname all of the requests will be stored under. Required: false. Defaults to "limitrr"
  "errorStatusCode": 429, //Status code to return when the user is being rate limited. Defaults to: 429 (Too many requests)
  "catchErrors": true //Should important errors such as failure to connect to the Redis keystore be caught and displayed?
}
```

## routes

**Required**: false

**Type**: Object

**Description**: Define route restrictions.

Inside the routes object, you can define many seperate routes and set custom rules within them. The custom rules you can set are:

- **requestsPerExpiry**: *Integer* How many requests can be accepted until user is rate limited? Defaults to: `100`
- **completedActionsPerExpiry**: *Integer* How many completed actions can be accepted until the user is rate limited? This is useful for certain actions such as registering a user - they can have a certain amount of requests but a different (obviously smaller) amount of "completed actions". So if users have recently been successfully registered multiple times under the same IP (or other discriminator), they can be rate limited. They may be allowed 100 requests per certain expiry for general validation and the like, but only a small fraction of that for intensive procedures. Defaults to the value in `requestsPerExpiry` or `5` if not set.
- **expiry**: *Integer* How long should the requests be stored (in seconds) before they are set back to 0? If set to -1, values will never expire and will stay that way indefinitely or must be manually removed. Defaults to: `900` (15 minutes)
- **completedExpiry**: *Integer* How long should the "completed actions" (such as the amount of users registered from a particular IP or other discriminator) be stored for (in seconds) before it is set back to 0? If set to -1, such values will never expire and will stay that way indefinitely or must be manually removed. Defaults to the value in `expiry` or `900` (15 minutes) if not set.
- **errorMsgs**: *Object* Seperate error messages for too many requests and too many completed actions. They have been given the respective key names "requests" and "actions". This will be returned to the user when they are being rate limited. If no string was set in `requests`, it will default to `"As you have made too many requests, you are being rate limited."`. Furthermore, if a value has not been set in `completed`, it will resolve to the string found in `requests`. Or, if that wasn't set either, `"As you performed too many successful actions, you have been rate limited."` will be it's value.

## Example of the routes object

``` javascript
"routes": {
    //Overwrite default route rules - not all of the keys must be set,
    //only the ones you wish to overwrite
    "default": {
        "expiry": 1000
    },
    "exampleRoute": {
        "requestsPerExpiry": 100,
        "completedActionsPerExpiry": 5,
        "expiry": 900,
        "completedExpiry": 900,
        "errorMsgs": {
            "requests": "As you have made too many requests, you are being rate limited.",
            "completed": "As you performed too many successful actions, you have been rate limited."
        }
    },
    //If not all keys are set, they will revert to
    //the default values
    "exampleRoute2": {
        "requestsPerExpiry": 500
    }
}

```
