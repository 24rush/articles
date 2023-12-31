A while ago, I decided to learn **Rust** and thought it would be a great opportunity to combine it with my second biggest interest besides coding: sports! I came up with the idea of displaying all my Strava activities on a single map. The purpose was to visualize all the tracks I've been on, allowing me to identify unexplored areas and focus on reaching them in the future, particularly for cycling.

I knew that some processing would be required on this data so I sold the idea to myself that Rust would be a good fit for such a task. Coming from a background in C and C++ where my mind got accustomed to thinking in bits, I also told myself that this would be a good opportunity to learn '*big data*' as my sports activities are the most amount of data I've gotten my hands on so far.

Before diving into the details, here is the app I've managed to build. I called it "[The World Covered](https://the-world-covered.vercel.app/)" because my intention with it was to see how much of the world I've covered so far by cycling. It's a good thing I did because I saw how little I actually did.

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1687523621275/4b0505b7-b82c-451b-bf39-f5ed8707a3e3.png align="center")

The first steps I took were to figure out how to download all my activities from Strava and to see if displaying them on a single map (through a web app) would crash the webpage or make it very sluggish so let's see how I tackled these two topics.

If you are curious about the code, you can find it all in the repos below:

Backend ▶️ [https://github.com/24rush/the-world-covered](https://github.com/24rush/the-world-covered)

Frontend ▶️ [https://github.com/24rush/the-world-covered-web](https://github.com/24rush/the-world-covered-web)

## Handling the data

### Strava API

All my activities are stored in Strava so naturally, I had to figure out how to get them out of there. Conveniently, Strava offers an API for such tasks, the only problem was that I wanted to do it from Rust which was something not that well documented.

Get the breakdown of how I accessed Strava's API from Rust in this separate article below:

%[https://hashnode.com/post/clji6pkzx000j0al8acuqdh61] 

### MongoDB

Once I had my data source set up, I went on to figuring how to persist it. After an unfortunate attempt of putting the data into a **Redis** database, I moved to a more appropriate **MongoDB** instance.

I had used MongoDB some years ago and didn't think much of it at that time but man was I impressed this time around! The richness of features and ease of use in all API realms (server side, web clients, Atlas web portal, tools, etc.) is second to none. Hats off to them! Not giving them ideas but this is one of the few services I would happily pay money for - it so too happens that they have a very good free tier so I didn't have to do it.

Get the breakdown of how I used Rust to store the Strava data in MongoDB in this separate article below:

%[https://hashnode.com/post/clji6r62w001p09mi0j2i8ns6] 

I also knew that the web app would be using MongoDB's **WebSDK** for retrieving the data in production but for development purposes, I wanted to use data from a local MongoDB database so I had to create a Rust app that would act as my MongoDB stub during development (more details on this later on in the article). As MongoDB queries are plain JSONs, my Rust server would just need to handle these JSONs by running them against the local mongod instance and return the documents to the client as JSONs as well.

A quick design diagram can be seen below:

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1687527465130/ad9e175e-12bf-422e-984f-d0cf3c5305d2.png align="center")

You can see in the gist below how the switching between the two of them was implemented. Spoiler alert: it uses an interface :)

%[https://gist.github.com/24rush/ae24beb1e6230e24a28c086d81dad10f] 

Worth noting the amazingness of MongoDB's WebSDK which with around 5 lines of code, allows you to read from the cloud database. *Lines 17-19* specify the application ID you want to connect to and your credentials, *lines 24-25* issue the request for data.

## Handling the map

Now that I had my data stored locally, I could start thinking about how to visualize it in a web app. At first, all I wanted was to try to display as many of them on a single map and see how the browser behaves. I didn't know the amount of data required but not seeing many apps like that in the wild, I was concerned that it might be for a reason.

### Leaflet.js

One of the best decisions I've made at this point was to use Leaflet.js for handling the OSM maps in the browser.

<details data-node-type="hn-details-summary"><summary>What is Leaflet.js</summary><div data-type="detailsContent">Leaflet.js is an open-source JavaScript library for mobile-friendly interactive maps (not limited to OSM) which makes it particularly easy to overlay stuff (polylines, markers) on a map and also interact with the map itself (like zoom, pan, center, etc.) plus it offers a multitude of facilities for handling GPS coordinates.</div></details>

Luckily, both Leaflet and Strava API use [Google Encoded](https://developers.google.com/maps/documentation/utilities/polylinealgorithm) polylines for describing GPS tracks so I could directly use the data coming in from Strava to display it on the map. What I created initially was just a wrapper over Leaflet.js which allowed me to:

1. display polyline (GPS track of the activity) on the map
    
2. highlight polyline when hovering it
    
3. show/hide all polylines
    
4. get notified when the user clicked on a polyline
    
5. center the map, pan to a GPS location, perform zoom at a particular GPS position
    

This resulted in the: [src / leaflet / LeafletMap](https://github.com/24rush/the-world-covered-web/blob/develop/src/leaflet/map.ts) class.

Having these pieces together resulted in a web app that could display the routes I've been on during my activities. What I could immediately find out is that if I were to request the data for displaying all my activities in one go, it would amount to around 6MB of data which did not create the best user experience during its retrieval but once it was loaded onto the map, the page was smooth - which was a huge relief. I will later discuss how I optimized this step by using a worker thread and also an incremental approach to downloading the activities.

Having fewer concerns about the feasibility of the app, I went on to think about its main **features** and put in place an **overall architecture**. I also researched where I could host it (the cheapest) and how I would get around the **security issues** of it being mostly a serverless app.

Let me first state the list of features I came up with:

1. Display the complete list of tracks I've been on (while running, cycling, etc.) on the map
    
2. Merge all my tracks into unique ones as I usually take the same track multiple times
    
3. Extract meaningful information from:
    
    * the telemetry data (altitude, distance, time, GPS coordinate, etc) like longest climbs or descents, fastest 12 min runs (for Cooper test), etc.
        
    * the activities themselves like the number of km per year, number of hours, days I've ridden most often, etc.
        
4. Query the data using natural language (with the help of OpenAI's SDK)
    
5. Extract even more meaningful information from the data by using ML models
    

## Architecture

Considering the requirements above, the overall architecture diagram of the project looked like this:

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1687585697010/27e7b35b-7847-47e2-b20a-47bdce265b72.png align="center")

*Before I go into details, I must emphasize that some design decisions I've taken might look cumbersome and not production worthy but this was intentional as the sole purpose of this project was for me to learn Rust through something that looks exciting to me. The decisions I'm referring to relate to the Rust code that is used solely on localhost.*

As you can spot from the diagram, the project consists of two parts: a **localhost** infrastructure for processing data coming in from Strava and the **cloud** (user-facing) one which relates to the website.

Let's delve into each box and see what they mean.

## Rust backend (data pipeline) `1`

This is the main chunk of the app and the reason I created this project - *to learn Rust.* It contains the processing pipeline for the Strava data and the handling of MongoDB, which are all done on a Linux *localhost*.

The details of how I processed all this data can be read in detail in the article below:

%[https://hashnode.com/post/clji6gvm8000b09mfg54q45ct] 

## MongoDB databases `2, 5`

You might have spotted that there are two instances of MongoDB in the project. I did it like that because I wanted to have a local instance of the data so I could manipulate it easier & faster plus I didn't want to use up my MongoDB free-tier quota on development.

There are two databases to work with, one (*strava\_db*) for mostly static (read-only) data coming in through the Strava API and another database (*gc\_db*) for the data I am creating through the Rust pipeline (unique activities, statistics, etc.).

The content of the databases is as follows:

| strava\_db ***collections*** |  | gc\_db ***collections*** |  |
| --- | --- | --- | --- |
| *Athletes* | data about the athlete we are storing activities for | *Routes* | all the activities merged into unique ones based on their GPS tracks |
| *Activities* | the complete set of activities for the athletes | *Statistics* | different statistics on the data |
| *Telemetry* | telemetry data for each activity (GPS points, altitude data, etc.) |  |  |

To synchronize the local databases with the cloud ones, I am using two very nifty tools that MongoDB provides and these are: *mongodump* & *mongorestore*. The first one dumps all the data from your *mongod* instance into a local folder and the second one restores the target database (in my case the cloud database) with the data found in this folder.

The sync procedure looks like this:

> mongodump
> 
> mongorestore --uri mongodb+srv://&lt;username&gt;:&lt;password&gt;@&lt;db\_url&gt; --nsInclude=***strava\_db.activities***

where *db\_url* is the URL of the cloud MongoDB and the value of *\--nsInclude* parameter is the collection I want to synchronize.

The sync procedure needs to be run in a few instances:

* initially when the remote database is created
    
* every time a new activity comes in from Strava API
    

## Rust local server `3`

Because of the local database I am using, I had to provide a data endpoint to the web app during development and this is where the Rust local server comes into play. This is a plain [**Rocket.rs**](https://rocket.rs/) web server that listens for incoming query requests from the local web app and returns the corresponding JSON data from the database.

### CORS & Fairings

If there is someone to spoil a party, that is *CORS* and obviously, it had to spoil mine. Luckily, Rocket.rs had all the pieces to make this fix a breeze at least if you knew what you had to do and in my case, it was to implement a CORS fairing for all my responses that would inform the browser that, as a server, I'm allowing requests from http://localhost:5173 which is the address of my Vite local webserver hosting the development web app.

%[https://gist.github.com/24rush/a3ac5025d3cfcbc908c0eb643f80ab19] 

In *line 28* you can see how the fairing is attached to the rocket instance. Moreover, in *line 6*, I tell Rocket that this fairing is for *Response* which will make it call my *on\_response* method after each request is handled but just before passing it back to the client. This way, when the browser issues the pre-flight **OPTIONS** request, I can attach the *Access-Control-Allow-XYZ* response headers required for CORS. Without these headers, the browsers just block the request.

### Data requests

After handling CORS, the actual request are handled like so:

%[https://gist.github.com/24rush/270c2d108bec48f89957b15e3c6b34c5] 

My web app, through the mechanism I mentioned earlier in the article, issues a POST request to *http://localhost:8000* which contains a preformatted JSON query to be issued to MongoDB. The Rust local server then parses this JSON string into the bson format needed by the Rust mongo driver and proceeds to run it against the local database. The response is then formatted as JSON and sent back to the client.

This is how in the development environment I can use the local mongo database and in production the cloud version. I later figured that through this setup I could also target the remote database by changing the connection string the Rust mongo driver is using. Double win.

## Web application `4`

As I am a bit of a fanboy of **Typescript**, I had to use it to build the web app and because I also had some experience with **Vue.js**, I went ahead with the two of them plus **Vite** as my bundler. This is another instance where I was pleasantly surprised at how good these frameworks have become. It seems the web ecosystem is settling on some standards which are getting more reliable and polished as time passes.

I am not going to go into much detail on the web part because it's the usual Vue.Js & Typescript website but I want to touch on how I managed to make it work smoothly considering the amount of data required to display.

The challenge was the main page which had to contain all my unique (deduplicated) routes - 320 of them and worth around 6MB of data. Downloading them in one go was not feasible and also not needed as the viewport doesn't even contain them when the page first loads.

The solution came in two parts:

1. using a worker thread for requesting & processing any data from the database
    
    Worker threads in web applications have become the standard nowadays so no need to explain how they work. In my app, the implementation consists of two files:
    
    ▶️ [src / webworker\_handler.ts](https://github.com/24rush/the-world-covered-web/blob/develop/src/webworker_handler.ts) containing the 'frontend' for the thread and the class that the rest of the application is using when needing data. Its job is to register the data requests, spawn them to the thread using *postMessage* and then proxy back the results incrementally to their corresponding requesters as they are processed by the thread.
    
    %[https://gist.github.com/24rush/9388a52ff4ae9bfe0724313d2d1bae52] 
    
    ▶️ [src / worker.js](https://github.com/24rush/the-world-covered-web/blob/develop/src/worker.js) containing the actual thread function which has the responsibility of taking the query and calling the data endpoint with it. After the result is received, it does post-processing on each item (track) and calls the requester back with the outcome. This way, the resulting activities are being returned item by item, not all in a bunch.
    
    %[https://gist.github.com/24rush/2e79c814c89c4b204841cda8b20259dd] 
    
2. requesting activities incrementally as the user panned/zoomed the map and they came into view
    
    This was accomplished by doing some server-side processing on the activities and determining their distance in intervals of 100km to the capital city (Bucharest). Therefore, every activity has a number allocated: 0 if it's less than 100km away from Bucharest, 1 if it's between 100 and 200km away and so on. The blue circles represent these regions and you can see that for the first ring, there aren't that many activities (around 170kb of data actually if the viewport can only display a corner that is less than 100km away from Bucharest).
    
    ![](https://cdn.hashnode.com/res/hashnode/image/upload/v1687791055564/876b51ba-2548-4f44-8348-c88a78332a8c.png align="center")
    
    As the user pans/zooms the map, I get notified of a new bounding box of the map and for each, I compute the distance of the NW corner to the capital city. If it's more than the last ring I retrieved, I make a call for the activities on the next ring.
    
    %[https://gist.github.com/24rush/340eecf08172e3955783afb5848ee5e8] 
    
    This is how I managed to go from 6MB of data to 170kb and make the page move smoothly. I got to tell you that I find it quite mesmerizing how the page keeps popping new routes as I move it around - I guess it's also because it brings back memories from those places 😊
    

## Hosting & Security

From the beginning, I wanted the website to offer the possibility of searching through the activities using natural language. After doing some research I concluded that I could use OpenAI's SQL Translate API but this requires an *api\_key* that is recommended to be kept **private** as it's associated with your account's quota so anyone that has it could potentially eat into your (paid) quota. This meant that if you wanted to safely benefit from this feature, you would need a server component that would hide your API key so the clients would just call your public functions exposed by this server which then internally would use this *api\_key* to query OpenAI.

Till this point, I've managed to keep everything serverless, meaning that all my code was meant to be run on the client but with the introduction of this constraint from OpenAI's *api\_key*, I had to improvise. *I didn't have a particular reason for keeping everything on the client side just that I didn't want to pay for hosting and also I was not particularly keen on delving into these technologies as they were not the focus of this project.*

## Vercel `6`

This would be around the third instance where I was to be extremely impressed by how good some technologies have become - at the same level as I've been impressed by MongoDB.

<details data-node-type="hn-details-summary"><summary>What is Vercel?</summary><div data-type="detailsContent">Vercel is a <em>platform that provides the tools, workflows, and infrastructure you need to build and deploy your web apps faster, without the need for additional configuration</em></div></details>

Initially, I wanted to use Vercel just for API hosting meaning that I wanted it to be a proxy between the web page and Open AI's SDK so I could hide my *api\_key* but after some investigation, I found out that it could also host my web application in a very neat setup (before that I was using Github Pages which was an ok way to do it but not even close to how amazing Vercel would prove to be).

### Web application hosting

I was mentioning earlier how the industry tends to settle for some web technologies and they become broadly adopted and standardized. This is what Vercel saw as well and it offers out-the-box automatic configuration for deploying web applications based on popular frameworks among which is **Vue.js** as well.

One other neat aspect of Vercel is that it connects to **GitHub** and listens for new commits to automatically redeploy your app. After each new commit you push to your repo, Vercel will build your app and redeploy it - all happening in about 15 seconds (for my app). Amazing!

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1687594103591/f6fa5788-9ea5-4c8b-98fc-0e19141311ca.png align="center")

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1687594187715/96037490-6835-42f3-92ab-4ebe2582b1dc.png align="center")

How I configured it is so straightforward that it's not worth writing about but you can find all the details [here](https://vercel.com/docs/getting-started-with-vercel/import) on Vercel's website.

Having the hosting covered now, I could proceed with the OpenAI proxy part.

### Serverless functions

<details data-node-type="hn-details-summary"><summary>What are Serverless functions?</summary><div data-type="detailsContent">Vercel Serverless Functions enable running code on-demand without needing to manage your own infrastructure, provision servers, or upgrade hardware. Serverless Functions enable developers to write functions in JavaScript and other languages to handle user authentication, form submissions, database queries, custom Slack commands, and more.</div></details>

In a nutshell, this is what the Vercel setup looks like:

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1687616409462/93fbc1aa-9e0e-4555-9a3b-1cfbe0498360.png align="center")

As I was mentioning in the previous chapter, the deployment of the app starts with a push in the Github repo which triggers Vercel to build and redeploy the app. The part we are interested in now is related to OpenAI and more precisely, how to call their API when the user triggers a search request from the webpage.

The file below shows the serverless function responsible for calling OpenAI's API using the environment variable *process.env.OPENAI\_API\_KEY (line 32).* By using this environment variable which is part of the server's configuration, I could hide it from the client.

This is the thing of beauty #1 but check out #2: the file below is stored in my repo under the ***/api*** folder with the name *genq.ts* (generate query). This instructs Vercel to create a route ***/api/genq*** for my domain which when issuing an HTTP request to it, will call the *handler* (serverless) function listed below 👏

Notice our old friend CORS again but this time around we configure it to allow requests only from my origin *(line 3)* which is the address of the website.

%[https://gist.github.com/24rush/600862171547dfd4b3258c82b8d04592] 

Having these in place, my web page will now need to issue requests to *https://the-world-covered.vercel.app/api/genq* to trigger calls to OpenAI. Notice how the request does not contain any private information, just the search query itself.

%[https://gist.github.com/24rush/44e7967e24eeda2a14b74f0215ba5ab1] 

I am going into detail on how I integrated Open AI into my project in this separate article below:

%[https://hashnode.com/post/clji6stcu001r09mi31udc5vt] 

This concludes my work thus far on this project; however, there is one more important feature I wish to incorporate as part of my skills development initiative. I aim to use **machine learning** to extract meaningful information from my data. At the moment, I am researching this topic, but please feel free to share any pointers or ideas in the comments.

Thank you for reading and let's keep in touch ✌️