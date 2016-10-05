# smokeless.github.io
If there's one thing web developers love, it's knowing better than conventional wisdom, but conventional wisdom is conventional for a reason: that shit works. Something's been bothering me for a while about this node.js nonsense, but I never took the time to figure it out until I read this butthurt post from Ryan Dahl, Node's creator. I was going to shrug it off as just another jackass who whines because Unix is hard. But, like a police officer who senses that something isn't quite right about the family in a minivan he just pulled over and discovers fifty kilos of black horse heroin in the back, I thought that something wasn't quite right about this guy's aw-shucks sob story, and that maybe, just maybe, he has no idea what he is doing, and has been writing code unchecked for years.

Since you're reading about it here, you probably know how my hunch turned out.

Node.js is a tumor on the programming community, in that not only is it completely braindead, but the people who use it go on to infect other people who can't think for themselves, until eventually, every asshole I run into wants to tell me the gospel of event loops. Have you accepted epoll into your heart?

A Scalability Disaster Waiting to Happen

Let's start with the most horrifying lie: that node.js is scalable because it "never blocks" (Radiation is good for you! We'll put it in your toothpaste!). On the Node home page, they say this:

Almost no function in Node directly performs I/O, so the process never blocks. Because nothing blocks, less-than-expert programmers are able to develop fast systems.
This statement is enticing, encouraging, and completely fucking wrong.
Let's start with a definition, because you Reddit know-it-alls keep your specifics in the pedantry. A function call is said to block when the current thread of execution's flow waits until that function is finished before continuing. Typically, we think of I/O as "blocking", for example, if you are calling socket.read(), the program will wait for that call to finish before continuing, as you need to do something with the return value.

Here's a fun fact: every function call that does CPU work also blocks. This function, which calculates the n'th Fibonacci number, will block the current thread of execution because it's using the CPU.
code:
function fibonacci(n) {
  if (n < 2)
    return 1;
  else
    return fibonacci(n-2) + fibonacci(n-1);
}
(Yes, I know there's a closed form solution. Shouldn't you be in front of a mirror somewhere, figuring out how to introduce yourself to her?.)

Let's see what happens to a node.js program that has this little gem as its request handler:

code:
http.createServer(function (req, res) {
  res.writeHead(200, {'Content-Type': 'text/plain'});
  res.end(fibonacci(40));
}).listen(1337, "127.0.0.1");
On my older laptop, this is the result:
code:
ted@lorenz:~$ time curl [url]http://localhost:1337/[/url]
165580141
real0m5.676s
user0m0.010s
sys0m0.000s
5 second response time. Cool. So we all know JavaScript isn't a terribly fast language, but why is this such an indictment? It's because Node's evented model and brain damaged fanboys make you think everything is OK. In really abusive pseudocode, this is how an event loop works:
code:
while(1) {
  ready_file_descriptor = event_library->poll();
  handle_request(ready_file_descriptor);
}
That's all well and good if you know what you're doing, but when you apply this to a server problem, you've pluralized that shit. If this loop is running in the same thread that handle_request is in, any programmer with a pulse will notice that the request handler can hold up the event loop, no matter how asynchronous your library is.

So, given that, let's see how my little node server behaves under the most modest load, 10 requests, 5 concurrent:
code:
ted@lorenz:~$ ab -n 10 -c 5 [url]http://localhost:1337/[/url]
...
Requests per second:    0.17 [#/sec] (mean)
...
0.17 queries per second. Diesel. Sure, Node allows you to fork child processes, but at that point your threading/event model is so tightly coupled that you've got bigger problems than scalability.
Considering Node's original selling point, I'm God Damned terrified of any "fast systems" that "less-than-expert programmers" bring into this world.

Node Punishes Developers Because it Disobeys the Unix Way
A long time ago, the original neckbeards decided that it was a good idea to chain together small programs that each performed a specific task, and that the universal interface between them should be text.

If you develop on a Unix platform and you abide by this principle, the operating system will reward you with simplicity and prosperity. As an example, when web applications first began, the web application was just a program that printed text to standard output. The web server was responsible for taking incoming requests, executing this program, and returning the result to the requester. We called this CGI, and it was a good way to do business until the micro-optimizers sank their grubby meathooks into it.

Conceptually, this is how any web application architecture that's not cancer still works today: you have a web server program that's job is to accept incoming requests, parse them, and figure out the appropriate action to take. That can be either serving a static file, running a CGI script, proxying the connection somewhere else, whatever. The point is that the HTTP server isn't the same entity doing the application work. Developers who have been around the block call this separation of responsibility, and it exists for a reason: loosely coupled architectures are very easy to maintain.

And yet, Node seems oblivious to this. Node has (and don't laugh, I am not making this shit up) its own HTTP server, and that's what you're supposed use to serve production traffic. Yeah, that example above when I called http.createServer(), that's the preferred setup.

If you search around for "node.js deployment", you find a bunch of people putting Nginx in front of Node, and some people use a thing called Fugue, which is another JavaScript HTTP server that forks a bunch of processes to handle incoming requests, as if somebody maybe thought that this "nonblocking" snake oil might have an issue with CPU-bound performance.

If you're using Node, there's a 99% probability that you are both the developer and the system administrator, because any system administrator would have talked you out of using Node in the first place. So you, the developer, must face the punishment of setting up this HTTP proxying orgy if you want to put a real web server in front of Node for things like serving statics, query rewriting, rate limiting, load balancing, SSL, or any of the other futuristic things that modern HTTP servers can do. That, and it's another layer of health checks that your system will need.

Although, let's be honest with ourselves here, if you're a Node developer, you are probably serving the application directly from Node, running in a screen session under your account.

It's Fucking JavaScript

This is probably the worst thing any server-side framework can do: be written in JavaScript.
code:
if (typeof my_var !== "undefined" && my_var !== null) {
  // you idiots put Rasmus Lerdorf to shame
}
What is this I don't even...
tl;dr

Node.js is an unpleasant software library and I will not use it.
