---
layout: post
title:  "How to setup basic webpush notification with flask"
date:   2020-02-08 00:26:37 +0530
categories: webpush flask
canonical_url: 'https://suryasankar.medium.com/how-to-setup-basic-web-push-notification-functionality-using-a-flask-backend-1251a5413bbe'
---

Web browsers have become increasingly very powerful over the past few years. Many features which were earlier offered only by mobile apps are now available on web as well. This has primarily been due to Google's investments in Chrome browser (and Mozilla's work on Firefox as well) and some allied technologies like Progressive Web Apps
Service Workers are the core piece of the technology that enabled PWAs to compete with native apps in terms of the functionality offered. You can get a good introduction to how these service workers work from this tutorial created by google and this MDN article. As mentioned in those articles, service workers can do a bunch of great things like
Execute an action which needs to talk to the server on the background - thus removing the pesky loading spinners
Act as a network proxy - caching various assets required by the site and letting a basic site be loaded even without network connectivity
Receive push notifications from the server and display them to the user.

In this post we will just stick to the last mentioned feature - Web Push Notifications. It is easy to add support for the same by integrating third party tools like WebEngage, OneSignal, Clevertap etc.
While these third party tools are sufficient for most purposes, if we intend to incorporate push notifications as a significant part of the user experience, it will be worthwhile to implement it ourselves. Google web fundamentals course has a detailed tutorial here on this topic. There is also a codelabs project which provides a demo project which can be cloned and experimented with. And then there is this Web Push Book which is probably the most extensive tutorial available on the subject.
While these tutorials are quite detailed, there is also a need for a step by step tutorial that starts from scratch and progressively builds the features. This series intends to be that. In the first post of the series, we will just set up a bare minimal application with Web Push support just to demonstrate how to get it up and running. In the subsequent parts, we will improvise the user experience and add more features.
Before we proceed further, you should read through this post to get an understanding of how the technology works. If you are short of time, here is a concise version -
Javascript is single threaded. This means that the when a website is loaded, the main thread will always be occupied in rendering and handling user interactions. But this limits the user experience that we can provide to the users. There was a need for threads which can run in the background without disturbing the user interface.
Web workers were developed to satisfy this requirement.
Service workers are a type of web workers, via which we can instruct the browser to do certain specific background tasks.
One of the tasks that they can do is to listen for Push notifications sent by the server which can then be displayed to the user.
Chrome pioneered support for this feature. Firefox incorporates it as well now.

So, with this basic understanding of WebPush, let's dive into the code now.

### Our hero - The Service Worker
Let's first see what a service worker js file looks like. A side-note - I have taken the scripts used in the Google Codelabs project and modified them to keep only what is essential for a simple demo.

{% gist e9d0adbadda0a9ee7cc909610778873d %}

The code should be easy to understand for anyone with a basic javascript knowledge. The main unusual feature in the above code is the keyword self. It is a reference to the ServiceWorkerGlobalScope which is an interface representing the scope of the service worker. Since the service worker does not have a browsing context, this self reference provides the interface for various functionalities which are usually provided by the window object in the browsing context. Just like we would do window.addEventListener, here the event handlers of the service worker are defined using self.addEventListener

In the above code, we have listed 3 event handlers. The ones handling install and activate are just logging the events for our debugging purposes. The push event handler is also simple to understand. It reads the text data sent with the event, checks if it is a JSON format, and if so extracts a title and body from it. If it is not in JSON format, it just assumes that the entire string is to be used as the body, with title set as "Untitled". As we will see below, this will help us debug the service worker from the client's chrome developer tools itself.

### Registering the service worker
The above service worker needs to be registered by the website so that the browser knows about it. Here is the registration script which we are going to use - register_service_worker.js

{% gist 0d8781f453cb1e7a284e564a8e65fc18 %}

The last function - `registerServiceWorker` is the main function which will be invoked to register the service worker. It accepts 3 arguments
1. __serviceWorkerUrl__ - The url which can be used by the browser to load the service worker. In our case, we will place the service worker js in the static folder of our flask app and the url will hence be /static/service_worker.js
2. __The application server public key__ - The second argument is the vapid public key. This is the public key component of a public key - private key pair which is used to securely authorize the communication between the push server and the push service running on the client. The mechanism is explained in more detail here.
3. __apiEndpoint__ - Finally the third argument is the api end-point which will be used to save the push subscriptions generated by the clients. In our example we will define an endpoint /api/push-subscriptions which will do this job.

The `registerServiceWorker` function calls the `subscribeUser` function which in turn calls the `swRegistration.pushManager.subscribe` method which generates the unique endpoint which will be used by the push server to send push notifications to the client. A side-effect of the .subscribe function call is that it will request permission from the user for showing push notifications, if the user has not yet allowed it.

This is sufficient for the sake of this simple example. But a good user experience requires that we control when and how this request alert is displayed by the browser. That can indeed be done using the `Notification.requestPermission()` method described [here](https://web.dev/articles/push-notifications-subscribing-a-user#requesting_permission). But for the sake of brevity, let's keep that out of this tutorial's scope as the primary aim of this first post in the series is just to get the push notifications working with the least amount of effort.
Now, the Push Subscription json generated by the `swRegistration.pushManager.subscribe` function call above looks like this

{% gist 6bfc71b6bb71f5f7071a8ef328def217 %}

This json should be stored so that the server can later send the desired push notification to this address. This is done by the updateSubscriptionOnServer function which posts the subscription json to the api endpoint which we will describe below.

### Structure of the app
Now that we have reviewed both the service worker script and the script required to register it, we can start discussing the structure of our server which will be used to serve the html pages and the api endpoints.
We are going to build a simple Flask server as the backend and configure it to serve a simple website with a service worker file which can provide Web Push support. The code is available in this repo. The app being used in this first post of the series is under the folder named basic_functionality
The app is structured like this

```
├── app.py
├── instance
│   └── application.cfg.py
├── requirements.txt
├── static
│   ├── register_service_worker.js
│   └── service_worker.js
├── templates
│   ├── admin.html
│   └── index.html
└── webpush_handler.py
```

The instance folder will be missing if you clone the github repo. It is a folder which is not meant to be version committed as it will contain the secret keys (to be explained below)
Before we review each of these files one by one, let's first see how to get the environment ready
Setting up the virtualenv and installing the requirements
We can use virtualenvwrapper to set up the python environment for running the app. Create a virtualenv like this
mkvirtualenv -p python3 simple_webpush
And with the virtualenv active, install the requirements using pip install -r requirements.txt. Refer the doc pages for virtualenv and virtualenvwrapper for more details if you are new to this.
With the environment set, we can start reviewing the server code.
Main server module - app.py
The full code of app.py is available here. We are using a simple Flask app pattern for this example. If you are new to Flask, going through this quickstart tutorial first will help. Note that this simple app structure is suitable only for development purposes. If we intend to deploy the application to production, we need to use a more scaleable architecture. But that is outside the scope of this first post.
Let's review the code of app.py section by section now.

```
from flask import Flask, render_template, request, jsonify
from flask_sqlalchemy import SQLAlchemy
from webpush_handler import trigger_push_notifications_for_subscriptions
app = Flask(__name__, instance_relative_config=True)
app.config.from_pyfile('application.cfg.py')
```

We are importing the essential flask and sqlalchemy modules. We are also importing a method which will be used for triggering the push notifications (to be discussed below)
The app is created using the flag instance_relative_config set to True. This lets us define some secret config information in a file inside a folder called instance which can be kept out of source control. The next line instructs the app to load the config from a file named application.cfg.py which is expected in the instance folder. So just create a folder named instance and create a file named application.cfg.py there. We will add the public and private keys required for sending web push notifications here.
Here is how the instance/application.cfg.py should look like
```
VAPID_PUBLIC_KEY = "BBBAASD#DFDFFFFFFFFFFFFFFFFFFFFFFF" 
VAPID_PRIVATE_KEY = "yyyyyyyyyyyyyyyyyyyyyyyyyyyyyyyyyyyyyyyyyy" 
VAPID_CLAIM_EMAIL = "[email protected]"
```
To generate this public key-private key pair, you can use this website - https://web-push-codelab.glitch.me/ Alternatively you can also use this npm command line utility - web-push.
The next section in app.py takes care of the database layer. The app needs a database connection to store the push subscription information generated by the browser. With Flask we generally use SQLAlchemy as the database ORM layer and FlaskSQLAlchemy as the linker library. You can refer their documentations here and here if you are unfamiliar.
```
app.config['SQLALCHEMY_DATABASE_URI'] = "sqlite://" 
db = SQLAlchemy(app) 
class PushSubscription(db.Model):    
  id = db.Column(db.Integer, primary_key=True, unique=True)
  subscription_json = db.Column(db.Text, nullable=False)
db.create_all()
```
In the above section, we have done the following
The database connection string sqlite:// instructs SQLAlchemy to use an in-memory sqlite database. Since it is in-memory it is non-persistent and the information will be lost if the server process is shut down. But this is sufficient for the purposes of this simple demo.
In the next line we have initialized the db handle and then we have defined a model named PushSubscription which will be used to store the subscription information. We will review the structure of this json shortly.
The next line db.create_all() takes care of creating the database tables - just one table in this case named push_subscription

With the database configured, we can now proceed to register the required url endpoints. The first route home_page renders the template index.html.
@app.route("/")
def home_page():
    return render_template("index.html")
The index.html rendered above can just be a bare minimum template which loads the register_service_worker.js script which we had described earlier and then invokes the registerServiceWorker function defined in the script. As required by Flask, we will place this in the `templates` folder.

templates/index.htmlWe have placed both the js files in the static folder and loaded the register_service_worker.js script directly in the html. That in turn takes care of loading the service worker using the url we have passed as the first argument. The second argument is supposed to be the public key as we discussed earlier when reviewing the registration script. Instead of hardcoding it, we are referring to it using a config variable. Flask will take care of loading this variable from the value we placed in instance/application.cfg.py earlier. And the third argument is supposed to be the api endpoint to which the registration script will upload the subscription_json. We will define it as follows in app.py

As it can be seen, the above api endpoint first checks if there is already a matching PushSubscription object with the same subscription_json, and if so returns it. So when we reload the page, even when the browser resends the same subscription_json due to the registerServiceWorker function being invoked again, it won't cause any error on the server. If there is no matching subscription_json on the other hand, a new entry is created and returned to the browser.
The setup done so far is enough to check that the service worker works as expected. Chrome developer tools has tools which let us test the service workers from the client itself. So let's do that first before seeing how to send the push notifications from the server.
Testing the client side functionality
We first start the server by running the flask run command
Once the server starts, open localhost:5000 in the browser. Keep the Developer tools open on the side. In case you haven't done this before, it is available under "More Tools" section in Chrome menu. Or you can press Ctrl + Shift + I. Once the site loads, you should see a dialog which asks you for notification permissions. Click on Allow.
When clicking on "Allow", if we inspect the Network tab in Developer tools, we can see a POST request being sent to the API endpoint.
In the response we can see the subscription details.
Now that the registration of service worker is done, let's test out its notification handler code. In the developer tools, click on the Application tab and click on the Application -> ServiceWorkers section in the side menu.
It shows the registered service worker on the localhost site. The status should show as activated and running. We can now test that this service worker is working as expected on the client side using this devtools interface itself. Type some string in the text-box labeled Push and click on the Push button. You should immediately see a push notification popup on the side. The title shows as "Untitled" because that is how we have coded (Check the service_worker.js code above)
We can also test how the service_worker will show the notification if we send it in the json format that we have coded the service worker to handle.
Thus we have verified that the notification handler in the service worker is working as expected. The next step is to go through the server setup to send the push notifications
Server code to send web push notifications
We will set up a simple admin page to send push notifications. A proper admin dashboard requires user authentication and authorization. But we will leave that for the next posts in this series. Here we will just focus on the code essential for sending out the notifications.
We first define the admin page like this in app.py
@app.route("/admin")
def admin_page():
    return render_template("admin.html")
The html is defined as follows with a simple form which allows you to enter a title and body of the push message and submit it.

templates/admin.htmlOn submission, it hits an api endpoint which is defined as follows. This endpoint uses the trigger_push_notifications_for_subscriptions function to send the notification to all the subscribers (only one in our case)

The trigger_push_notifications_for_subscriptions is defined in webpush_handler.py which we had already imported in app.py

It's a straightforward invocation of the function defined by the pywebpush library which we had already installed as part of the requirements. We are signing it with the vapid private key which we had copied to the instance/application.cfg.py. Since the client already knows the public key, it will be able to validate that the push message is coming from the authorized server only.
Testing out the admin form for sending push notifications
To test this functionality, let's visit localhost:5000/admin Here, if we fill the title and body and submit, we can see a push notification like this
If you are seeing a notification like above, congratulations you have successfully built a simple website with a very basic web-push notification support.
Conclusion
In the above tutorial we have reviewed how to build a simple app with support for push notifications. We can continue building features on top of this as follows
Improving the push notification handler on client side - by implementing the Notification.requestPermission() method, by adding more options for controlling the appearance and behavior of the notification, adding support for images etc
Modifying the app structure to make it more scaleable and ready to deploy
Use a persistent database for storing the push subscriptions
Implement user authentication and authorization so that the admin form can be opened only for certain users.
Implementing support for user segmentation on the backend, so that we can send the notifications to specific target segments

These features will all be done in the subsequent posts of this series.