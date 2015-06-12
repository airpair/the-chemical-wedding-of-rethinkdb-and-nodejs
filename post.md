### Before we dive in

> This article assumes you have some Javascript experience, already installed and configured RethinkDB and io/node.js. Otherwise, prepare for awesomeness with [these](http://rethinkdb.com/docs/install/) [instructions](http://rethinkdb.com/docs/quickstart/) and install io.js with [n](https://github.com/tj/n) or [nvm](https://github.com/creationix/nvm).
I won't explain every ReQL function used in the code but I hope you can understand what we're doing from the comments.
Also, as matter of preference, I will be using an alternative Javascript driver called [rethinkdbdash](https://github.com/neumino/rethinkdbdash). The API is the same, but I encourage you to use it because it provides nice defaults like a connection pool and stream support.

## RethinkDB && Node.js && ReQL == Awesome
RethinkDB has been on development for three years and it's getting more and more attention [because of its production-ready release](http://rethinkdb.com/blog/2.0-release/) and [new realtime features](http://rethinkdb.com/blog/rethinkdb-pubnub/).

Node.js excels at non-blocking I/O, so you want to delegate the heavy processing to a background worker as fast as possible. Turns out ReQL(RethinkDB Query Language) is growing into a nice option for this kind of job, it will distribute the group-map-reduce work and probably simplify your system architecture, executing the whole job on the same layer.

## Case study: a geolocation worker

We will develop a background job where we search for new sessions in the RethinkDB database, geolocate them and as a bonus get the current weather data. The scheduling will be written in Node.js and the geolocation in ReQL.

You will need to sign in for [OpenWeatherMap](http://openweathermap.org/api) to get an API key.


### ReQL = Stored Procedures? Didn't we learn from the past?

Yes, you need an open mind here. It will generate huge queries, it will hurt testability, good practices, yadda yadda.
But, like anything in life, there are tradeoffs and at least you're not just concatenating strings. ReQL is a delight to work with. Also, when working seriously with Node.js it's impossible that this thought never crossed your mind :)


### Database preparation

Open the RethinkDB Data Explorer and let's create some tables:

```javascript,linenums=true
// we need to wrap every value in r() or r.expr() to use ReQL functions
// forEach() is required to iterate a sequence doing writes
r(["sessions","pending_geo_sessions","cached_geolocations","cached_weathers"])
.forEach(function(table) { 
    return r.tableCreate(table); 
});
```

And insert fake data:

```javascript,linenums=true
// insert returns a object containing a property "generated_keys" with a array of UUIDs for each document id
r.table("sessions").insert([{
    ip: "143.204.121.1" //put some real IP here
},{
    ip: "143.204.121.2"
}])("generated_keys").do(function(session_ids) {
    // now let's insert these new session ids into our jobs table
    return session_ids.forEach(function(session_id) {
        return r.table("pending_geo_sessions").insert({id: session_id});
    });
});
```

### The background worker

Now we have sessions and pending jobs for our background worker. We will develop it in Node.js with the help of [node-cron](https://github.com/ncb000gt/node-cron):

```javascript,linenums=true
// cronjob.js

"use strict";

const CronJob = require('cron').CronJob,
    r = require('rethinkdbdash')(), // enter your RethinkDB config here
    geo_job = require("./jobs/pending_geo")(r);

// It will run every 30 seconds
new CronJob("*/30 * * * * *", geo_job , null, true);
```

Now let's write the the worker, first we will atomically remove and return all pending session jobs:

```javascript,linenums=true
// jobs/pending_geo.js

const pending_sessions = r.table("pending_geo_sessions").delete({
    returnChanges: true
})("changes").default([]); // .default() is like a monad for returning a value in case our query throws a error(when there are no changes) 
```

RethinkDB returns query changes in the format ```{new_val: ..., old_val: ...}```, so given a session job, we will geolocate it from a external service([Telize](http://www.telize.com/)) or cache:

```javascript,linenums=true
// jobs/pending_geo.js

const geolocate_session = function(session) {
    // this is the conditional syntax of RethinkDB, r.branch(condition, true, false)
    return r.branch(
        session("old_val").ne(null), // you may think that every change should return a old val, but if two workers are atomically deleting from the job table, there is a chance someone will get a {new_val: null, old_val: null} as of current RethinkDB version
        geo_from_cache(session),
        null // no geolocation was found
    ).do(function(geo) {
        return get_weather_from_cache(session, geo); // get weather data(and update session information)
    });
};
```

```javascript,linenums=true
// jobs/pending_geo.js

const geo_from_cache = function(session) {
    return r.table("cached_geolocations").get(session("old_val")("ip"))
    .do(function(geo) {
        return r.branch(
            geo.eq(null), // no cache
            geo_from_api(session), // get from Telize API
            {
                country: geo("country"),
                longitude: geo("longitude"),
                latitude: geo("latitude"),
                region: geo("region"),
                region_code: geo("region_code"),
                country_code: geo("country_code"),
                city: geo("city")
            } // otherwise return from our cache table
        );
    });
};
```

```javascript,linenums=true
// jobs/pending_geo.js

const geo_from_api = function(session) {
    const TELIZE_URL = r("http://www.telize.com/geoip/")
        .add(session("old_val")("ip")); // this will run server-side so we can't rely on Javascript native concatenation syntax
    // RethinkDB supports sending HTTP requests via r.http
    return r.http(TELIZE_URL).default(null).do(function(data) {
        return r.branch(
            data.ne(null).and(data.hasFields("latitude")), // validating API response
            r.table("cached_geolocations").insert({
                id: session("old_val")("ip"),
                country: data("country").default(""),
                longitude: data("longitude"),
                latitude: data("latitude"),
                region: data("region").default(""),
                region_code: data("region_code").default(""),
                country_code: data("country_code").default(""),
                city: data("city").default(""),
                expires_at: r.now().add(86400) // 1-day cache
            }, {
                returnChanges: true
            })("changes").nth(0)("new_val").without("expires_at"), // return the cached data without the expiration field 
            null // no valid response
        );
    });
};
```

Now if there was a geolocation, we want to get the current weather from the coordinates(and cache it for future queries):

```javascript,linenums=true
// jobs/pending_geo.js

const get_weather_from_cache = function(session, geo) {
    const weather_id = geo("longitude").coerceTo("string")
    .add(",").add(geo("latitude").coerceTo("string"));
    return r.branch(
        geo.ne(null), // get weather only if we got geolocation coordinates
        r.table("cached_weathers").get(weather_id).do(function(weather) {
            return r.branch(
                weather.eq(null), // no cached weather data
                get_weather_from_api(session, geo),
                update_session_data(session, geo, weather)
            );
        }),
        [] // you may ask why we are returning a empty array instead of null, well, forEach will be called on the pending sessions array and it will throw an error if we return null, as of RethinkDB 2.0
    );
};
```

```javascript,linenums=true
// jobs/pending_geo.js

const get_weather_from_api = function(session, geo) {
    const API_URL = r("http://api.openweathermap.org/data/2.5/weather?lat=")
        .add(geo("latitude").coerceTo("string")).add("&lon=")
        .add(geo("longitude").coerceTo("string"))
        .add("&units=metric&APPID=YOUR_API_ID"); // this will run server-side so we can't rely on Javascript native concatenation syntax
        
    return r.http(API_URL).default(null).do(function(weather) { // in case there is a error we will return null from the HTTP request
        return r.branch(
            weather.typeOf().eq("OBJECT").and(weather("weather").ne(null)), // validating the API response format
            r.table("cached_weathers").insert({
                id: geo("longitude").coerceTo("string").add(",").add(geo("latitude").coerceTo("string")),
                type: weather("weather").nth(0)("main"),
                temperature: weather("main")("temp"),
                weather_icon: weather("weather").nth(0)("icon"),
                expires_at: r.now().add(10800) // 3-hour cache
            }, {
                returnChanges: true
            })("changes").nth(0)("new_val").without("expires_at"), // return weather data without the expiration field
            null
        ).do(function(weather) {
            return update_session_data(session,geo,weather); // finally update session data with geolocation and weather
        });
    });
};
```

```javascript,linenums=true
// jobs/pending_geo.js

// the final step on each pending session iteration
const update_session_data = function(session, geo, weather) {
    return r.table("sessions").get(session("old_val")("id")).update({
        geo: geo,
        point: r.point(geo("longitude"), geo("latitude")),
        weather: r.branch(
            weather.ne(null), 
            {
                type: weather("type"),
                temperature: weather("temperature"),
                weather_icon: weather("weather_icon")
            }, 
            null
        )
    });
};
```

And this is the task runner(omitting the functions created above):

```javascript,linenums=true
// jobs/pending_geo.js

"use strict";

module.exports = function(r) {
    let busy = false;
    return function() {
        if (!busy) {
            busy = true;
            pending_sessions.do(function(data) {
                return data.forEach(geolocate_session);
            }).run();
            busy = false;
        }
    };
};

```

Then you can execute it with a simple ```node cronjob.js```. If you wanna set it as a daemon with systemd, [this is a nice article about it.](http://blog.carbonfive.com/2014/06/02/node-js-in-production/)

## Conclusion
This is a naive implementation, if it were for real we would setup a failover strategy, maybe even intercepting API errors with the [changefeed API](http://rethinkdb.com/docs/changefeeds/javascript/) and other creative measures.

You can see a sample project of this background job  [here](https://github.com/thelinuxlich/rethinkdb_geolocation_job).

We didn't touch advanced queries with group-map-reduce or realtime stuff but I hope you can get a glimpse of the power and flexibility that RethinkDB + Node.js can give you!

## Reference

- [r.expr](http://rethinkdb.com/api/javascript/expr/)
- [r.http](http://rethinkdb.com/docs/external-api-access/)
- [r.branch](http://rethinkdb.com/api/javascript/branch/)
- [r.do](http://rethinkdb.com/api/javascript/do/)
- [default](http://rethinkdb.com/api/javascript/default/)
- [forEach](http://rethinkdb.com/api/javascript/for_each/)
- [insert](http://rethinkdb.com/api/javascript/insert/)
- [update](http://rethinkdb.com/api/javascript/update/)
- [delete](http://rethinkdb.com/api/javascript/delete/)