---
title:  "Develop locally for Slack using tunnels"
date:   2018-02-05 19:15:43 +1000
categories: web-dev
permalink: "/:title"
---

Slack is a messaging app that has been rising to stardom among all sorts of teams. One of the many things that makes Slack so compelling is that it integrates nicely with all kinds of other applications and services making it a perfect fit within the API space.

Slack comes with a bunch of standard integrations, but it’s also really easy to build your own. Imagine how awesome it would be if development could be simplified even more by allowing you to test your requests directly from your machine without constantly redeploying your work to a hosting platform/webserver. Is that even possible? Certainly is.

Behold ngrok. This convenient tool allows you to setup a secure tunnel to your localhost, or in plain English, opening your local app access to your local app from the internet. The only drawback is you will have to be on a paid version to enable custom domains and other features. But we won’t be using any of those for now.


## This is what we'll do in this tutorial:

Download and setup ngrok
Setup a basic HTTP server with Node.js
Develop a Slack app
Create a slash command that sends a request to your ngrok address

### Step 1: Download ngrok
Go to https://ngrok.com/ and download the version that corresponds to your platform. In our case, we'll be downloading the Windows (32-bit).

### Step 2: Install ngrok
Installing ngrok is basically extracting the file. There are two ways of doing this, depending how you want to run it:

a) You can extract ngrok into the folder of your preference and run ngrok from there.

b) The way I like to do it is by extracting ngrok to my preferred folder and adding it to my system's $PATH. The advantage of going with this option is that you'll be able to run ngrok from any path on the command line.

### Step 3: Tunneling your server
If you went for option A on Step 2, fire up the command line, navigate to the directory where you extracted ngrok and start it by telling it which port we want to expose to the public internet. You can do this by typing:

./ngrok http 3000
Or if ngrok is on your $PATH, then simply type the following from any directory:

ngrok http 3000
If all goes well you should see the following:


The online is a good sign which means the tunnel is working and your app is now accessible at that particular 3000 port.

The address to access it from the internet would be the one next to Forwarding, with ngrok.io domain. In my case it’s https://eacea434.ngrok.io.

One of ngrok's niftiest features is a UI to inspect requests. Let's access it by following the Web Interface URL.


While examining the Web Interface, you’ll notice that there are no requests being displayed. This is because we haven't made any requests to our new ngrok address. Let's just click on the first HTTP url there to make a simple GET request from our browser.


Something’s not right. Fear not, comrade!

While we've tunneled our app to the internet through that port; we still don't have any app running yet on that specific port. We'll need to create a web service that points to that port to get rid of that nasty error screen.

### Step 4: Create a simple HTTP server
Let's set up a simple web server to processes all incoming HTTP requests.

For this part you'll need a code editor such as Sublime Text or Atom. We'll be using Node.js to develop our app, so you'll need to make sure you've installed it on your machine as well. Now create an empty folder for your project, let's name it testApp. Create an index.js file inside that folder and open it up in your editor. Write down the following code. I've commented it so you can understand what's going on with each line:

// This imported module contains all the logic for dealing with HTTP requests.

var http = require('http');



// We define the port we want to listen to and it should be the same port than we specified on ngrok.

const PORT=3000;



// We create a function which handles any requests and sends a simple response

function handleRequest(request, response){

  response.end('Ngrok is working!');

}



// Passing our request function onto createServer guarantees the function is called once for every HTTP request that's made against the server

var server = http.createServer(handleRequest);



// Finally we start the server

server.listen(PORT, function(){

  // Callback triggered when server is successfully listening.

  console.log("Server listening on: http://localhost:%s", PORT);

});

To run the app type the following from your testApp directory using the Terminal:

node index.js
Now try visiting your ngrok address once more and you should be pleasantly greeted with a Ngrok is working! message. Well done!


### Step 5: Take a breather
Now that your local server is exposed to the internet, let's create a simple app with a slash command that gets a message from your server to confirm it's operational.

First, we'll need to modify the code so it can interact correctly with Slack. To keep our script concise, we'll use a couple of essential third-party modules - express and request.

Express is a web application framework that allows us to set up a simple web server. It makes it easy to set up the routing logic we need for the requests we'll receive from Slack. Request is a handy module to make HTTP calls. We'll use it to interact with Slack's web API.

To install the two modules, open the command line from the root directory of your folder and run the following commands separately:

npm install express

npm install request
Step 6: Create your Slack App
Go to https://api.slack.com/apps
Click on Create new App on the top-right hand-side.
Name your app and select the workspace you'd like the app to be associated with its creator.
Click on OAuth & Permission from the left menu bar and scroll down till you reach Scopes. Select commands and channels:history from the permission scopes. Save your changes.

The Install App to Workspace button should be enabled now. Click on it and install your app to your workspace.
In the redirect URI field, paste your ngrok forwarding address and add the /oauth endpoint at the end of the address that we opened up in our script. In this example it would look something like this:
https://eacea434.ngrok.io/oauth
Now let's add a slash command with our app. Click on Slash Commands on the left menu -> Create New Command and fill the form as shown below. Just make sure that the Request URL matches your ngrok route and is pointing to the /command endpoint.



Last but not the least we need the Client information for our app. Click on Basic Information on the left menu, copy your Client ID and Client Secret and pass them as the corresponding variables on the next scipt in Step 2.

### Step 7: Refactor code
// Import express and request modules

var express = require('express');

var request = require('request');



// For this tutorial, we'll keep your API credentials right here

var clientId = '123456789.123456789';

var clientSecret = '123456789abcdefghijkl';



// Instantiates Express and assigns our app variable to it

var app = express();


// Define a port we want to listen to

const PORT=3000;


// Lets start our server

app.listen(PORT, function () {

    //Callback triggered when server is successfully listening.

    console.log("Example app listening on port " + PORT);

});


// Route the endpoint that our slash command will point to and send back a simple response to indicate that ngrok is working

app.post('/command', function(req, res) {

    res.send('Your ngrok tunnel is up and running!');

});


// This route handles GET requests to our root ngrok address and responds with the same "Ngrok is working message" we used before

app.get('/', function(req, res) {

    res.send('Ngrok is working!');

});



// We'll use this endpoint for handling the logic of the Slack oAuth process behind our app.

app.get('/oauth', function(req, res) {

    // When a user authorizes an app, a query parameter is passed on the oAuth endpoint. If that code is not there, we respond with an error message

    if (!req.query.code) {

        res.status(500);

        res.send({"Error": "Looks like we're not getting code."});

        console.log("Looks like we're not getting code.");

    } else {

        // GET call to Slack's `oauth.access` endpoint

        request({

            url: 'https://slack.com/api/oauth.access',

            qs: {code: req.query.code, client_id: clientId, client_secret: clientSecret},

            method: 'GET',



        }, function (error, response, body) {

            if (error) {

                console.log(error);

            } else {

                res.json(body);



            }

        })

    }

});
Since this file contains your Client ID and Secret, I'd suggest not to upload it to Github. You can separate these and store them as environment variables before doing so.

### Step 8: Spread the love
Before we can run the slash command, we'll need to authenticate against our app to add it to our team. There's a simple way to generate the authentication link.

Go to https://api.slack.com/docs/slack-button, scroll down to the Add the Slack button section where you will find a generator for a button that would allow anyone to add your new app. Make sure the right app is selected on the generator and that you're selecting the commands scope, which is necessary to add our bundled slash command:


Click on the Add to Slack button to go through the OAuth process and add the app to the team of your preference (it doesn't have to be the team used to create the app).


If all goes well, you should be able to send the ngrok command from any channel on the team and it should invoke the following response:



## Farewell
Great job — you've built an app that's running on your local server. I hope now you’re more comfortable creating, working and running Slack apps locally before putting your app out there. Now you can get started on building more complex apps and use your new tunnel-creating skills to test and play around with other nifty Slack features. Next week I’ll post about getting notifications via SMS to your mobile directly from Slack. Stay tuned!
