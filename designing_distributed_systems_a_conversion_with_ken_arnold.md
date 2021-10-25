# Designing Distributed Systems -- A Conversation with Ken Arnold, Part III

> Summary
>
> In this third installment of Bill Venners' interview with Ken Arnold, the discussion centers around designing distributed systems, including the importance of designing for failure, avoiding the "hell" of state, and choosing recovery strategies.

Ken Arnold has done a lot of design in his day. While at Sun Microsystems, Arnold was one of the original architects of Jini technology and was the original lead architect of JavaSpaces. Prior to joining Sun, Arnold participated in the original Hewlett-Packard architectural team that designed CORBA. While at UC Berkeley, he created the Curses library for terminal-independent screen-oriented programs. In Part I of this interview, which is being published in six weekly installments, Arnold explains why there's no such thing as a perfect design, suggests questions you should ask yourself when you design, and proposes the radical notion that programmers are people. In Part II, Arnold discusses the role of taste and arrogance in design, the value of other people's problems, and the virtue of simplicity. In this third installment, Arnold discusses the concerns of distributed systems design, including the need to expect failure, avoid state, and plan for recovery.

Bill Venners: What is important to keep in mind when you are designing a distributed system?

Ken Arnold: We should start off with some notion of what we mean by distributed system. A distributed system, in the sense in which I take any interest, means a system in which the failure of an unknown computer can screw you.

Failure is not such an important factor for some multicomponent distributed systems. Those systems are tightly controlled; nobody ever adds anything unexpectedly; they are designed so that all components go up and down at the same time. You can create systems like that, but those systems are relatively uninteresting. They are also quite rare.

Failure is the defining difference between distributed and local programming, so you have to design distributed systems with the expectation of failure. Imagine asking people, "If the probability of something happening is one in ten to the thirteenth, how often would it happen?" Your natural human sense would be to answer, "Never." That is an infinitely large number in human terms. But if you ask a physicist, she would say, "All the time. In a cubic foot of air, those things happen all the time." When you design distributed systems, you have to say, "Failure happens all the time." So when you design, you design for failure. It is your number one concern.

Yes, you have to get done what you have to get done, but you have to do it in the context of failure. One reason it is easier to write systems with Jini and RMI (remote method invocation) is because they've taken the notion of failure so seriously. We gave up on the idea of local/remote transparency. It's a nice thought, but so is instantaneous faster-than-light travel. It is demonstrably true that at least so far transparency is not possible.

## Partial Failure

What does designing for failure mean? One classic problem is partial failure. If I send a message to you and then a network failure occurs, there are two possible outcomes. One is that the message got to you, and then the network broke, and I just couldn't get the response. The other is the message never got to you because the network broke before it arrived. So if I never receive a response, how do I know which of those two results happened? I cannot determine that without eventually finding you. The network has to be repaired or you have to come up, because maybe what happened was not a network failure but you died.

Now this is not a question you ask in local programming. You invoke a method and an object. You don't ask, "Did it get there?" The question doesn't make any sense. But it is the question of distributed computing.

So considering the fact that I can invoke a method on you and not know if it arrives, how does that change how I design things? For one thing, it puts a multiplier on the value of simplicity. The more things I can do with you, the more things I have to think about recovering from. That also means the conceptual cost of having more functionality has a big multiplier. In my nightmares, I'll tell you it's exponential, and not merely a multiplier. Because now I have to ask, "What is the recovery strategy for everything on which I interact with you?" That also implies that you want a limited number of possible recovery strategies.

## Transactions

So what are those recovery strategies? J2EE (Java 2 Platform, Enterprise Edition) and many distributed systems use transactions. Transactions say, "I don't know if you received it, so I am forcing the system to act as if you didn't." I will abort the transaction. Then if you are down, you'll come up a week from now and you'll be told, "Forget about that. It never happened." And you will.

Transactions are easy to understand: I don't know if things failed, so I make sure they failed and I start over again. That is a legitimate, straightforward way to deal with failure. It is not a cheap way however.

Transactions tend to require multiple players, usually at least one more player than the number of transaction participants, including the client. And even if you can optimize out the extra player, there are still more messages that say, "Am I ready to go forward? Do you want to go forward? Do you think we should go forward? Yes? Then I think it's time to go forward." All of those messages have to happen.

And even with a two-phase commit, there are some small windows that can leave you in ambiguous states. A human being eventually has to interrupt and say, "You know, that thing did go away and it's never coming back. So don't wait." Say you have three participants in a transaction. Two of them agree to go forward and are waiting to be told to go. But the third one crashes at an inopportune time before it has a chance to vote, so the first two are stuck. There is a small window there. I think it has been proven that it doesn't matter how many phases you add, you can't make that window go away. You can only narrow it slightly.

So the transactions approach isn't perfect, although those kinds of problems happen rarely enough. Maybe instead of ten to the thirteenth, the probability is ten to the thirtieth. Maybe you can ignore it, I don't know. But that window is certainly a worry.

The main point about transactions is that it has overhead. You have to create the transaction and you have to abort it. One of the things that a container like J2EE does for you is that it hides a lot of that from you. Most things just know that there's a transaction around them. If somebody thinks it should be aborted, it will be aborted. But most things don't have to participate very directly in aborting the transaction. That makes it simpler.

## Idempotency

I tend to prefer something called idempotency. You can't solve all problems with it, but basically idempotency says that reinvoking the method will be harmless. It will give you an equivalent result as having invoked it once.

If I want to manipulate a bank account, I send in an operation ID: "This is operation number 75. Please deduct $100 from this bank account." If I get a failure, I just keep sending it until it gets through. If you're the recipient, you say, "Oh, 75. I've seen that one, so I'll ignore it." It is a simple way to deal with partial failure. Basically, recovery is simple retry. Or, potentially, you give up by sending a cancel operation for the ID until that gets through. If you want to do that, though, you're more likely to use transactions so you can abort them.

Generally, with idempotency, everybody needs to know how to go forward. But people don't often need to know how to go back. I don't abort a transaction. I just repeatedly try again until I succeed. That means I need to know how to say to do this. I don't have to deal with all sorts of ugly recovery most of the time.

Now, what happens if failure increases on the network? You start sending messages more often. If that is a problem, for a long distance you can solve it by writing a check and buying more hardware. Hardware is much cheaper than programmers. Other ways of dealing with this tend to increase the system's complexity, requiring more programmers.

Bill Venners: Do you mean transactions?

Ken Arnold: Transactions on everything can increase complexity. I'm just talking about transactions and idempotency now, but other recovery mechanisms exist.

If I just have to try everything twice, if I can simply reject the second request if something has already been done, I can just buy another computer and a better network—up to some limit, obviously. At some point, that's no longer true. But a bigger computer is more reliable and cheaper than another programmer. I tend to like simple solutions and scaling problems that can be solved with checkbooks, even though I am a programmer myself.

## Wide-Area Distributed Systems

Bill Venners: Is there anything in particular about Internet- wide distributed systems or large wide area networks that is different from smaller ones? Dealing with increased latency, for example?

Ken Arnold: Yes, latency has a lot to do with it. When you design anything, local or remote, efficiency one of the things you think about. Latency is an important issue. Do you make many little calls or one big call? One of the great things about Jini is that, if you can use objects, you can present an API whose natural model underneath deals with latency by batching up requests where it can. It adapts to the latency that it is in. So you can get away from some of it, but latency is a big issue.

Another issue is of course security. Inside a corporate firewall you say, "We'll do something straightforward, and if somebody is mucking around with it, we'll take them to court." But that is clearly not possible on the Internet; it is a more hostile environment. So you either have to make things not care, which is fine when you don't care if somebody corrupts your data. Or, you better make it so they can't corrupt your data. So aside from latency, security is the other piece to think about in widely distributed systems.

## State is Hell

Bill Venners: What about state?

Ken Arnold: State is hell. You need to design systems under the assumption that state is hell. Everything that can be stateless should be stateless.

Bill Venners: Define what you mean by that.

Ken Arnold: In this sense, state is essentially something held in one place on behalf of somebody who is in another place, something that is not reconstructible by the other entity that wants it. If I can reconstruct it, it's called a cache. And caches are often OK. Caching strategies are their own branch of computer science, and you can screw them up. But as a general rule, I send you a bunch of information and you send me the shorthand for it. I start to interact with you using this shorthand. I pass the integer back to you to refer to the cached data: "I'm talking about interaction number 17." And then you go down. I can send that same state to some equivalent service to you, and build myself back up. That kind of state is essentially a cache.

Caching can get complex. It's the question that Jini solves with leasing. If one of us goes down, when can the person holding the cache throw this stuff away? Leasing is a good answer to that. There are other answers, but leasing is a pretty elegant one.

On the other hand, if you store information that I can't reconstruct, then a whole host of questions suddenly surface. One question is, "Are you now a single point of failure?" I have to talk to you now. I can't talk to anyone else. So what happens if you go down?

To deal with that, you could be replicated. But now you have to worry about replication strategies. What if I talk to one replicant and modify some data, then I talk to another? Is that modification guaranteed to have already arrived there? What is the replication strategy? What kind of consistency do you need—tight or loose? What happens if the network gets partitioned and the replicants can't talk to each other? Can anybody proceed?

There are answers to these questions. A whole branch of computer science is devoted to replication. But it is a nontrivial issue. You can almost always get into some state where you can't proceed. You no longer have a single point of failure. You have reduced the probability of not being able to proceed, but you haven't eliminated it.

If my interaction with you is stateless in the sense I've described—nothing more than a cache—then your failure can only slow me down if there's an equivalent service to you. And that equivalent service can come up after you go down. It doesn't necessarily have to be there in advance for failover.

So, generally, state introduces a whole host of problems and complications. People have solved them, but they are hell. So I follow the rule: make everything you can stateless. If there is one piece of the system you can't make stateless—it has to have state—to the extent possible make it hold all the state. Have as few stateful components as you can.

If you end up with a system of five components and two of them have state, ask yourself if only one can have state. Because, assuming all components are critical, if either of the two stateful components goes down you are screwed. So you might as well have just one stateful component. Then at least four, instead of three, components have this wonderful feature of robustability. There are limits to how hard you should try to do that. You probably don't want to put two completely unrelated things together just for that reason. You may want to implement them in the same place. But in terms of defining the interfaces and designing the systems, you should avoid state and then make as many components as possible stateless. The best answer is to make all components stateless. It is not always achievable, but it's always my goal.

Bill Venners: All these databases lying around are state. On the Web, every Website has a database behind it.

Ken Arnold: Sure, the file system underneath a Website, even if it's just HTML, is a database. And that is one case where state is necessary. How am I going to place an order with you if I don't trust you to hold onto it? So in that case, you have to live with state hell. But a lot of work goes into that dealing with that state. If you look at the reliable, high-performance sites, that is a very nontrivial problem to solve. It is probably the distributed state problem that people are most familiar with. Anyone who has dealt with any large scale, high availability, or high performance piece of the problem knows that state is hell because they've lived with that hell. So the question is, "Why have more hell than you need to have?" You have to try and avoid it. Avoid. Avoid. Avoid.

---
1. 见书 P344
2. 原文链接：https://www.artima.com/articles/designing-distributed-systems
