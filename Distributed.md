<!DOCTYPE html>
<html>
<head>
<meta charset="UTF-8">
<title>Distributed Systems in Haskell :: Will Yager</title>
<link href='http://fonts.googleapis.com/css?family=Lato' rel='stylesheet' type='text/css'>
<link href='/post.css' rel='stylesheet' type="text/css">
</head>
<body>
<!-- 2016-05 -->

<br>


### [Blog](http://yager.io)

## Distributed Systems in Haskell

I recently completed Lorenzo Alvisi's Distributed Computing class at the University of Texas at Austin. This class had three substantial project components, which were to implement the Chandy-Lamport snapshot protocol, Paxos, and the Bayou distributed database algorithm. The class allowed for arbitrary language choice, so long as we adhered to the provided API. My project partner [Pato](http://plankenau.com) and I decided to give it a go in Haskell.

This turned out to be an excellent idea. We put in a fraction of the time most implementations (in Java or Python) required. (One classmate used Erlang to great success.)

This article represents a summary of what I learned over the course of the class. Some of this is Haskell-specific, and some is good advice for distributed programming in general.

## Haskell-nonspecific Advice

### 1: Do Not Block

Every single time we used blocking reads from the network, it came back to bite us in the ass. For example, we would send a `Ping` and wait for a `Pong` before continuing. This leads to all sorts of bad behavior. What happens if both servers `Ping` each other at the same time? Deadlock. Even blocking reads that seemed innocuous at first usually led to confusion and race conditions later on.

Instead, use an asynchronous architecture. Your program should block in exactly one place. Each node in the system should have a "superloop" that performs blocking reads from the network using an `epoll`-like mechanism. Many concurrency-oriented libraries and languages (like Haskell or Erlang) will make this very easy (you may not even realize it's happening).

#### Why?

It may seem like this architecture introduces uneccesary logical complexity compared to a bit of blocking sprinkled throughout the code, but in every instance we came across, blocking in exactly one place turned out to be much easier. We eliminated all race conditions we came across and absolutely maximized performance by eliminating any unneccesary delays. A server that only blocks in one place is guaranteed to process any waiting message the moment it has free CPU cycles.

### 2: Use Asynchronous Timing

Some algorithms (especially probabilistic ones) rely on things like timeouts.

In our experience, implementing timeout and other time-based behavior as a blocking read with a timeout is a recipe for confusion.

Instead, we found that the best approach was to spawn, for each node, a separate "tick generator" thread. This "tick generator" simply sends its parent thread an empty `Tick` message at a given frequency. By counting `Tick` messages, one can implement arbitrary timeout behavior without actually needing to deal with timeout reads. For example, one could have a `ticks_since_last_client_msg` counter. Every time the node gets a message from the client, it resets this counter to zero. Every time the node gets a tick, it increments the counter. If the counter reaches some pre-determined value, the node can assume the client has timed out. 

#### Why?

There are several advantages to this approach.

For one, it's often simpler than dealing with timeouts around blocking reads. Instead of having to write a separate exception handler that deals with timeouts, one just incorporates timeout logic into the rest of the server logic. `Tick`s are just plain old messages that we handle in the same way we handle any other message.

Where this approach really shines is if you have multiple independent timeouts or different timeout behavior for different peers. Wrapping your reads in a 10-second timeout won't help you if every server except one dies, and that one living server keeps sending you a message every couple seconds. Now you have two different kinds of timeouts: never receiving a message from some server in particular, and never receiving a message from any server at all.

If you use a tick-based approach, you can just keep a map from server names to the number of ticks it's been since you've heard from that server. If one of those numbers gets too large, you handle it in your server logic. Very easy!

### 3: Separate Networking and Logic

Distributed systems papers are often as poorly written as they are clever. The included code rarely works properly, if at all. One bad habit that these papers tend to have is the thorough mixing of network operations and algorithm logic. It's pretty common to see things along the lines of

```
send(server,msg);
x = receive_msg();
y = process(x);
send(server,y);
```

but with a lot more junk thrown in.

It turns out that this is not conducive to clean, understandable code. There's a lot of implicit state being offloaded to the network when you structure things like this, and it makes it a lot harder to recover from things like network interruptions or servers going offline. You end up using timeouts and all sorts of ugly constructs to make things work.

Instead, you should completely separate your server logic and your network functionality. Again, this might sound like a lot of work, but it's almost guaranteed to save you more time in the long run.

In Haskell terms, your server logic will (ideally) have a type like this:

```haskell
serverStep :: Config -> State -> Message -> (State, [Message])
```

In prose, the server logic takes three arguments:

* The server's configuration, which does not change (Hostname, directory, etc.)
* The server's previous state
* A message received from the network

The server logic then returns

* the new server state
* a list of messages to send

Then you just have to write a simple wrapper around this function that receives messages from the network, feeds them into the function, and sends the responses out to the network.

With a bit of work, any program requiring a sequence of sends and receives can be transformed into this form (a single receive followed by arbitrarily many sends), so even an the most stubbornly ugly distributed paper can be adapted to this form.

#### Why?

1. This form guarantees that you meet suggestion #1 and get all the advantages of doing so. In particular, your server will never waste time waiting for something to happen. It will process messages the moment they are available, because it never blocks unless absolutely necessary.
2. Network code is simpler. There's just one place you send and receive messages, and it's very straightforward to implement.
3. Testing is much easier. 
   When your server logic is a pure function as described above, server behavior is entirely deterministic and much more amenable to testing.
   It's easy to build a test harness that "simulates" the network. All you have to do is keep a list of all your servers' states and a queue of messages waiting to be delivered. A test harness looks like this:

            while queue is not empty:
               pop msg off queue
               (new_state, new_msgs) = serverStep configs[msg.dest] states[msg.dest] msg
               states[msg.dest] = new_state
               put new_msgs into queue

   If you want to test things like out-of-order message delivery, you just mix up your queue instead of putting messages in in order. You have complete control!


## Haskell-specific Advice

### 1. Monad It

Note that the suggested server logic type

```haskell
serverStep :: Config -> State -> Message -> (State, [Message])
```

is entirely pure and does not admit side effects. Unfortunately, this is not always realistic (as servers have to make database queries and such). However, as long as we avoid making network queries or using thread delays or anything silly like that, we still get most of the benefits described above if our server logic has the type e.g. 

```haskell
serverStep :: Config -> State -> Message -> IO (State, [Message])
```

We might not specifically need IO (we could be using `ST` or some other monad), but for simplicity I'll leave it at that.

We can do better, though. 

The `Config -> ...` behavior is described by the `MonadReader` typeclass.

The `... State -> ... -> IO (State, ...)` behavior is described by the `MonadState` typeclass.

The `... -> IO (..., [Message])` behavior is described by the `MonadWriter` typeclass.

Therefore, we can easily transform between a function of the above type and something of the type

```haskell
type ServerMonad m = (MonadIO m, MonadReader Config m, MonadState State m, MonadWriter [Message] m)
serverStep :: ServerMonad m => m ()
```

If we don't need `IO`, we can leave off the `MonadIO` constraint and still be 100% pure.

#### Why?

This one is a lot harder to explain if you're less familiar with Haskell. The question here is why

```haskell
pureServerStep :: Config -> State -> Message -> (State, [Message])
impureServerStep :: Config -> State -> Message -> IO (State, [Message])
```

is worse than

```haskell
type PureServerMonad m = (MonadReader Config m, MonadState State m, MonadWriter [Message] m)
pureServerStep :: PureServerMonad m => Message -> m ()
type ImpureServerMonad m = (MonadIO m, MonadReader Config m, MonadState State m, MonadWriter [Message] m)
impureServerStep :: ImpureServerMonad m => Message ->  m ()
```

These two forms are exactly semantically equivalent, but the monad-heavy form automates all the repetitive work of aplying the function to a state, pulling out the messages and the new state, applying the next step to the new state, pulling out the newer state and the new messages, concatenating the messages, etc. etc.

All of that work is very droll and predictable, and is already handled for us nicely by `MonadState` and `MonadWriter`. This means our code is shorter, cleaner, and less likely to contain mistakes (after a bit of up-front cost to define all our Monads and types and stuff).

Note that `RWST Config [Message] State IO` satisfies `ImpureServerMonad` and `RWS Config [Message] State` satisfies `PureServerMonad`, so all the work of creating a monad is already done for us.

Here's a quick bidirectional reduction proof showing that these two concepts (the function and the Monad) are semantically equivalent:

```haskell
logic :: Config -> State -> Message -> IO (State, [Message])
logic = \cfg state msg -> execRWST (logicM msg) cfg state
logicM :: Message -> RWST Config [Message] State IO ()
logicM = \msg -> RWST (\cfg state -> logic cfg state msg >>= (\(s,w) -> return ((),s,w)))
```

That is, the function form can always be implemented as the monad form and vice versa.


### 2. Use Cloud Haskell

[Cloud Haskell](http://haskell-distributed.github.io) is a library designed to make distributed systems development in Haskell obscenely easy. It takes advantage of Haskell's tremendous multithreading support and very powerful type system to allow for safe, fast, and simple message passing and control over a network. It does all the hard work of building a messaging layer, and it does it well.

### 3. Use Lenses

Many distributed systems algorithms are described in a very imperative way. Turns out that it's very easy to write imperative-style programs in Haskell in a very disciplined way.

Lenses are a Haskell concept that are reasonably well described as setters and getters on steroids. They are objects that describe how to pull information out of and put information into data structures. For example, I could make a lens that describes how to get the second value out of a tuple and how to put something else in its place. (Turns out that this exists and is called `_2`.)

Because Lenses are so well structured, one can programatically compose and manipulate lenses in very interesting ways.

One of the interesting things the `lens` package exports is a series of operators that interface with `MonadState` and allow us to write code that looks just like regular imperative code. For example, if our `State` had a field called `counter` that held an `Int`, we could write

```haskell
incMul :: PureServerMonad m => m ()
incMul = do
    counter += 1
    counter *= 2
```

## Example

Let's write a simple distributed application. We'll write some servers that send each other `Bing`s every once in a while, and if they get a `Bing` from someone, they send back a `Bong`. Each server counts the number of `Bing`s and `Bong`s it's received thus far. 

[Completed code](http://0.0.0.0/Distributed/Distributed.hs)

First, let's write out the types.

```haskell
data BingBong = Bing | Bong
    deriving (Show, Generic, Typeable)
instance Binary BingBong

data Message = Message {senderOf :: ProcessId, recipientOf :: ProcessId, msg :: BingBong}
               deriving (Show, Generic, Typeable)
instance Binary Message

data Tick = Tick deriving (Show, Generic, Typeable)
instance Binary Tick

data ServerState = ServerState {
    _bingCount :: Int,
    _bongCount :: Int,
    _randomGen :: StdGen
} deriving (Show)
makeLenses ''ServerState

data ServerConfig = ServerConfig {
    myId  :: ProcessId,
    peers :: [ProcessId]
} deriving (Show)

newtype ServerAction a = ServerAction {runAction :: RWS ServerConfig [Message] ServerState a}
    deriving (Functor, Applicative, Monad, MonadState ServerState,
              MonadWriter [Message], MonadReader ServerConfig)
```

* `BingBong` is the type of a `Bing` or a `Bong`. All that stuff about `Generic`, `Typeable`, and `Binary` is for automatic derivation of the correct code to safely send and receive these values over the network. If we wanted, we could do this manually, but this is rarely useful when the compiler does a good and guaranteed correct job for us.
* A `Message` is what gets sent from server to server.
* A `Tick` is what each server's tick generator sends it.
* The `ServerState` is what it says on the tin. It has the counts and a random number generator state. Notice how we put underscores before the field names and then used `makeLenses` to generate Lenses for `bingCount`, `bongCount`, and `randomGen`. Note that, for testing purposes, we can use pre-determined random generator seeds! This means that we only get "true" randomness (i.e. actual pseudorandomness) when we want it, but we're fully deterministic when we want to be (like for testing). If we'd used side-effectful randomness (like C's `random()` or reading from `/dev/urandom`), we wouldn't get that.
* `ServerConfig` is just the server's ID as well as a list of the IDs of all servers on the network.
* `ServerAction` is a custom Monad that gives us all the behavior we want (reading a config, sending messages, and updating state). It's really just a wrapper around `RWS`, so we don't really have to do anything. We just tell the compiler which features we want copied from `RWS`, such as its `Monad` behavior.

Now, let's write out our server logic.

```haskell
tickHandler :: Tick -> ServerAction ()
tickHandler Tick = do
    ServerConfig myPid peers <- ask
    random <- randomWithin (0, length peers - 1)
    let peer = peers !! random
    sendBingBongTo peer Bing

msgHandler :: Message -> ServerAction ()
msgHandler (Message sender recipient Bing) = do
    bingCount += 1
    sendBingBongTo sender Bong
msgHandler (Message sender recipient Bong) = do
    bongCount += 1

sendBingBongTo :: ProcessId -> BingBong -> ServerAction ()
sendBingBongTo recipient bingbong = do
    ServerConfig myId _ <- ask
    tell [Message myId recipient bingbong]

randomWithin :: Random r => (r,r) -> ServerAction r
randomWithin bounds = randomGen %%= randomR bounds
```

* `tickHandler` processes `Tick`s. It randomly chooses a peer and sends that peer a `Bing`.
* `msgHandler` processes `Message`s. It responds to `Bong`s and increments counters when appropriate.
* `sendBingBongTo` is a helper function that creates a message annotated with the sender and receiver and then outputs the message (using `tell`, which lets us write to the `MonadWriter` output).
* `randomWithin`, given an upper and lower bound, picks a random element in those bounds. It also updates the server's random number generator state.

Let's write out the network stack (i.e. the necessarily impure part of our code).

```haskell
runServer :: ServerConfig -> ServerState -> Process ()
runServer config state = do
    let run handler msg = return $ execRWS (runAction $ handler msg) config state
    (state', output) <- receiveWait [
            match $ run msgHandler,
            match $ run tickHandler]
    say $ "Current state: " ++ show state'
    mapM (\msg -> send (recipientOf msg) msg) output
    runServer config state'
```

This takes a server's config and initial state. It waits for an incoming message and processes it using the server logic. It sends any messages the server outputs, and then repeats the process with the server's new state. Note that we use a `match` construct to wait for either `Tick`s or `Message`s, since those have different types (and therefore different handlers).

We also use Cloud Haskell's `say` function, which sends debug messages to a process named `logger`. We do this instead of printing to standard output, because if `logger` is the only process printing things, we won't accidentally interleave two prints.

Now let's write the initialization code.

```haskell
spawnServer :: Process ProcessId
spawnServer = spawnLocal $ do
    myPid <- getSelfPid
    otherPids <- expect
    spawnLocal $ forever $ do 
        liftIO $ threadDelay (10^6)
        send myPid Tick
    randomGen <- liftIO newStdGen
    runServer (ServerConfig myPid otherPids) (ServerState 0 0 randomGen)

spawnServers :: Int -> Process ()
spawnServers count = do
    pids <- replicateM count spawnServer
    mapM_ (`send` pids) pids
```

* `spawnServer` spawns a new process which does the following:
    - Get my PID.
    - Wait for someone to send me everyone's PIDs.
    - Spawn a ticker process that sends me a `Tick` every second (1 million microseconds).
    - Create a random number generator seed.
    - Create the appropriate `ServerConfig` and initial `ServerState` and call `runServer`.
* `spawnServers` (plural) simply spawns `count` servers using `spawnServer`, collects all their PIDs, and sends the list of PIDs to each server.


And for our `main`:

```haskell
main = do
    Right transport <- createTransport "localhost" "0" defaultTCPParameters
    backendNode <- newLocalNode transport initRemoteTable
    runProcess backendNode (spawnServers 10)
    putStrLn "Push enter to exit"
    getLine
```

* First, we create a network transport endpoint. This is how Cloud Haskell actually talks over the network. (We don't use it here though.)
* Next, we create a local node (which manages all the processes on this machine) and attach it to the TCP transport.
* Next, we run `spawnServers 10` on our local node. If you recall, this spawns 10 communicating processes.
* Finally, we wait for the user to push enter before exiting.

[Completed code](http://0.0.0.0/Distributed/Distributed.hs)

<script>
  (function(i,s,o,g,r,a,m){i['GoogleAnalyticsObject']=r;i[r]=i[r]||function(){
  (i[r].q=i[r].q||[]).push(arguments)},i[r].l=1*new Date();a=s.createElement(o),
  m=s.getElementsByTagName(o)[0];a.async=1;a.src=g;m.parentNode.insertBefore(a,m)
  })(window,document,'script','//www.google-analytics.com/analytics.js','ga');

  ga('create', 'UA-21050351-2', 'auto');
  ga('send', 'pageview');

</script>

</body>
</html>
