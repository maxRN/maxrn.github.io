# Developing a local, multi-player Quiz App

Me and my friends enjoy hanging out with each other and doing quizzes that we prepare for each other.
One friend in particular likes preparing these a lot.
However up until now he has been using a rather rudimentary quiz-setup consisting of 
* Microsoft PowerPoint for displaying the questions, 
* an Excel spreadsheet were he keeps track of all the points of all the teams, 
* and a 3rd party website so we can use our phones as buzzers.

While this setup works okay, it has some major and minor shortcomings.

1. Microsoft PowerPoint allows him to easily and quickly create quizzes with different categories and question types.
It also allows for linking between pages creating a basic navigation between questions.
One problem however, is that we cannot keep track of what questions/categories we already answered and which ones are still to be completed.
I am sure there are PowerPoint wizards out there that know some way to mark whether a slide has been visited or not, however we don't know of any such technique.
(To be fair I haven't checked, but I am pretty sure that this should be possible, given that PowerPoint is [turing complete](https://www.youtube.com/watch?v=uNjxe8ShM-8).)
1. The score keeping is done in an Excel spreadsheet.
While this is a reasonable approach given that we are all familiar with the program and it's very easy to quickly setup a basic scorecard, it is annoying to do this manually and also prone to errors.
It's also tedious to switch from PowerPoint to Excel, find the correct column for the team, enter the correct point value (different questions give different amount of points) and then if the column is full you have to add up all the points accumulated so far and replace this value with the previous points to make room for new entries. <br>
In short, the process of keeping track of the points is tedious and error-prone.
1. The buzzer app we used, [Cosmobuzz](https://www.cosmobuzz.net/#/), was not reliable and working correctly for us.
We had problems with latency, UI issues (the buzz going through but the button was stuck on "sending"), and not all buzzers re-enabling after the quiz master reset everyone.

I think I can do better than that. Let's try it!

## What Makes A Great Quiz App

Two things in particular were bothering me with the current setup: The buzzer and the point tracking.

Since we are all in the same room, there is no reason why the quiz app would have to run on an external server.
This only creates unwanted latency (especially important for such a time-critical use case like a buzzer) and possible points of failure (server going down, server problems).
Thus, one of the main reasons I want to create this app is to write my own buzzer application that runs locally and is also **integrated with the quiz**.
That ties in to the second point: point tracking.
If the buzzer and quiz app are integrated, it's trivial to track the points of everyone.
When someone buzzes, the server knows who it is and the quiz master can then decided whether the answer is correct or not.
Then the app can automatically award points based on the question.
No need to do this manually and risk messing up points by accident.

## How to Build the Perfect Quiz App

The tech stack will consist of the following:

* NodeJS as the backend
* NextJS as the front-end framework
* coupled with React for the UI
* TailwindCSS for styling
* the "database" will be a simple folder with text documents
* and docker for deploying the app to my RaspberryPi


## What *Exactly* Can It Do?

I already mentioned some points regarding specifications but in this chapter I go into detail about what this app can and can't do.

Starting from the view of the participants...

* a responsive buzzer that is easy to use
    * the users can join the buzzer via a QR code displayed on the main display
    * who buzzed should be visible by everyone on the main display
* see their points and everyone else's points on their phone

Now to the requirements of the quiz master...
* it should be easy and simple to add questions of varying types
    * in version 1.0 that is questions with 4 pre-defined answers as well as "Guess-the-Song" questions, using Spotify links
* quiz master should have an admin dashboard that displays various information and **is not visible to the participants**
    * current question, next question, correct answer (not visibile to participants)
    * points of all teams
* in version 1.0, the questions will have to be added via a .yaml file
    ** in later versions, one should also be able to add questions via the GUI

In addition to that...

* I want to add some sort of metrics/statistics evaluation to the end of the game
    * team that buzzed the most/least amount of times
    * team that answered correctly the most/the least amount of times
    * team that gained the most points
    * team that lost the most points
    * team that buzzed fastest
    * answered the most music/League of Legends/geography/etc. questions
    * and more!

## Challenges Along The Way

Some random problems that I encountered along the way:

In order to allow the participants to easily join the game, I want the quiz master to be able to display a QR code that contains the local URL where the buzzer/participant page is located.
To do this, I need to a) get the IP address and port of the server and b) render and display this information as a QR code.

### Getting IP Address and Port

The port can be easily got through `const port = process.env.PORT || 3000`.
When looking for solutions for getting the IP address, on stumbles over recommendations for the [ip](https://www.npmjs.com/package/ip) NPM package. However this did not work for me, because it returned the localhost address (127.0.0.1) when I really wanted the IP address of my server in the local network (192.168.xxx.xxx).

I found a different [solution](https://stackoverflow.com/a/43491534/16727853) on the same StackOverflow thread: 
```javascript
require('os').networkInterfaces().en0.find(elm => elm.family=='IPv4').address
```
However this approach is specifically tailored towards MacOS (which is my OS of choice for development) and will likely have to be adjusted to work in Docker/on my RaspberryPi. 
Something like [this](https://stackoverflow.com/a/67522718/16727853) approach is probably platform-agnostic.
```javascript
Object.values(require("os").networkInterfaces())
    .flat()
    .filter(({ family, internal }) => family === "IPv4" && !internal)
    .map(({ address }) => address)
```

### Rendering as QR Code

For rendering QR codes, there, of course, also exist NPM packages.
The one I'm using is [node-qrcode](https://www.npmjs.com/package/qrcode).

I tried to use the `.toDataURL()` function at first and while this worked perfectly, it created a data URL of fixed size, which was too small for my needs (displaying on a large TV or projector).
After some googling I found that the `.toString()` function offers the option to provide a `type` parameter.
If I know specify `{type: 'svg'}`, the function returns a string representing an SVG file of the QR code:
```xml
<svg xmlns="http://www.w3.org/2000/svg" viewBox="0  ... 0h1m1 0h2m2 0h1m1
 3m1 0h1"/></svg>
```
The naive approach of just passing this string to an `<img>` tag or just outputting this string itself did not work (which is obvious in hindsight, since, as the name implies, this really is just a string).
So I now need a way to parse this SVG string as an HTML element and display it to my site.

I used the [html-react-parser](https://www.npmjs.com/package/html-react-parser) node package to do this.
I initially tried to encapsulate all the QR code logic in a separate component, but I had problems rendering this because the `getString` method was async and this did not seem to work with the way things are rendered in React and Next JS.
What I did instead was do the SVG string generation in the `getServerSideProps` function of my page and pass the string as a parameter to my page.
I could then parse the string and display the QR code.

I want to note here that these problems stem from my limited understanding of React and NextJS and I am fairly certain that there is a better way to do this.
I think encapsulation the entire QR code logic in a separate component would be a better approach.
This is definitely something I want to tackle later and even if it does not work, I think it will teach me a lot about how React does processing under the hood.


### Buzz, Buzz, Buzz

On my quest to create a better buzzer for our quiz I (re)discovered the WebSocket technology.
This is a protocol that enables real-time, duplex communication between multiple hosts and is the technology used in applications such as live editing of documents with multiple users and also other streaming related use cases.

After some research and trial-and-error I decided on not hosting the WebSocket server with Next JS but instead host a separate server instance using Go.
I used the [gorilla](https://github.com/gorilla/websocket) package which has all I need to set-up a basic WebSocket server.

For the client side I used the [isomorphic-ws](https://www.npmjs.com/package/isomorphic-ws) node package. I had some problems connecting to the server though. 
I just got a rather unhelpful error message in the browser console so the first troubleshooting step I tried was using a 3rd party WebSocket server so I could rule out that the Go server was the problem.
I used [Pie Socket](https://www.piesocket.com/websocket-tester) to test the client frontend, and: It worked!
So if the frontend was working fine, there had to be some problem with the backend.
However the server seemed to work find and as far as I could tell this also wasn't a configuration issue.

"Fixing" this problem took me longer than I care to admit, but the error eventually turned out to be a wrong server address.
This is the line which specifies the port and IP address of the server
```Go
var addr = flag.String("addr", "localhost:8080", "http service address")
```
From my basic understanding I wrongly pointed my frontend to `ws://localhost:8080` however that did not work.
After looking at the example provided by the `gorilla` library again I tried connecting to `wss://localhost:8080/echo`.
This got me a step closer but I was now getting the following error 
```javascript
uncaughtException: Error: write EPROTO 8754423296:error:1408F10B:SSL routines:ssl3_get_record:wrong version number:../deps/openssl/openssl/ssl/record/ssl3_record.c:332:
``` 
I suspect there is some kind of problem with security and how localhost works with https.
So next I tried to connect not via the secure `wss` protocol but via the insecure `ws` and that actually worked!
Sort of.
I was now getting another error message, however this time on the server, meaning my requests actually went through!
The error reported
```Go
upgrade:websocket: request origin not allowed by Upgrader.CheckOrigin
```
was related to the origin checking.
The server seems to block any request coming from a different machine/process by default.
So after some Googling I came across the solution of allowing connections from every host.
This was done by putting this line
```Go
upgrader.CheckOrigin = func (r *http.Request) bool {return true}
```
above the `upgrade.Upgrade()` call in the Go file.
Usually you would assign a function to `upgrader.CheckOrigin` that only allows hosts from certain known origins to connect to the server, however as this application is only intended to run locally anyway, I have no problem with allowing all connections since connections from a different network can't reach the server.

And now finally, I could talk to the Go WebSocket server from my NextJS client frontend.
However I was faced with one more challenge:
When I tried to connect to the Go server from my phone, I was getting a connection not found error.
I was puzzled until I tried removing the `localhost` prefix in the Go server address settings (`:8080` instead of `localhost:8080`).

This resolved my issue and now I was able to connect and talk to my Go server from every device in my local network.
Success!

The next step was implementing the logic on the Go backend to enable the phones and the server to register there, and to pass information between them.

## React Learnings

`setState` in `useState` is async... kind of.
This discussion has a lot of helpful information: https://stackoverflow.com/questions/54069253/usestate-set-method-not-reflecting-change-immediately
What I learned is that the state of a `useState` hook is always the same in one render instance.
So if you update with `setState` and then immediately read `state`, you will read the old value, because the new is only reflected after a re-render has happened.
 
 This has been a very fun experience so far and I'm looking forward to more interesting experiences and learning along the way; until then: thank you for reading and see you next time!
