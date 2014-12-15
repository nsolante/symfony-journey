Interrupt Symfony with an Event Subscriber
==========================================

Hey guys! Welcome to a series that we're calling: Journey to the Center of
Symfony! In this first part, we'll be talking about the deep, dark core piece
called the HttpKernel, a wondrous component that not only sits at the heart
of Symfony, but also at the heart of Silex, Drupal 8, PhpBB and a lot of
other stuff. How is that possible? We'll find out! And this stuff is really
nerdy, so we're going to have some fun.

Getting the Project Running
---------------------------

I already have the starting point of our app ready on my computer. You can
download this right on the screencast page. I've already run ``composer install``,
I've already created my database, I've already created my schema: I won't
show those things here because you guys are a bit more of experts. We do
have fixtures, so let's load those.

Now let's use the built-in PHP web server to get our site running.

Perfect!

So in true Journey to the Center of the "Symfony" theme, we're going to talk
about dinosaurs. I've already created an app, which has 2 pages. We can list
dinosaurs - these are coming out of the database - and if we click on one
of them, we go to the show page for that dinosaur.

Big Picture: Request-Route-Controller-Response
----------------------------------------------

No matter what technology or framework we're using, our goal is always to
start with a request and use that to create a response. Everything in between
those 2 steps will be different based on your tech or framework. In our app,
and in almost every framework, two things that are going to be between the
request and response are the route and controller. In this case, you can see
our homepage has a route, our function is a controller, and our controller
returns a Response object::

    // src/AppBundle/Controller/DinosaurController.php
    // ...

    /**
     * @Route("/", name="dinosaur_list")
     */
    public function indexAction()
    {
        $dinos = $this->getDoctrine()
            ->getRepository('AppBundle:Dinosaur')
            ->findAll();

        return $this->render('dinosaurs/index.html.twig', [
            'dinos' => $dinos,
        ]);
    }

And we have the same thing down here with the other page: it has a route,
a controller, and that returns a response::

    // src/AppBundle/Controller/DinosaurController.php
    // ...

    /**
     * @Route("/dinosaurs/{id}", name="dinosaur_show")
     */
    public function showAction($id)
    {
        $dino = $this->getDoctrine()
            ->getRepository('AppBundle:Dinosaur')
            ->find($id);

        if (!$dino) {
            throw $this->createNotFoundException('That dino is extinct!');
        }

        return $this->render('dinosaurs/show.html.twig', [
            'dino' => $dino,
        ]);
    }

So what we're going to look at is *how* that all works. Who actually runs
the router? Who calls the controller? How do events work in between the
request-response flow?

But before we dive into that, what we're going to do first is create an event
listener and hook into that process. Then we'll be able to play with that
event listener as we dive into the core of things.

The Best Parts of the Web Profiler
----------------------------------

I'm going to open up the profiler and go to the timeline. This is going to
be our guide to this whole process. This shows everything that happens between
the request and the response. Even if you don't understand what's happening
yet, after we go through everything, this is going to be a lot more interesting.
You can already see where our controller is called, and under the controller
you can see the Twig template and even some Doctrine calls being made.

Before and after that, there are a lot of event listeners - you notice a
lot of things that end in the word Listener. That's because most of the things
that happen between the request and the response in Symfony are events: you
have the chance to hook into them with event listeners.

In fact, one other tab I really like on here is the Events tab. You can see
there's some event called ``kernel.request``. Maybe you already understand
what that means, maybe you don't, but you will soon. There's another event
called ``kernel.controller`` with listeners and several other events. We're
going to see where these events are dispatched and why you would add a hook
to one versus another.

Creating an Event Subscriber/Listener
-------------------------------------

Let's create a listener on that ``kernel.request`` event! In my ``AppBundle``,
I'll create a new directory called ``EventListener`` and a new class. Inside
this event listener, we're going to read the ``User-Agent`` header off the
request and do some things with that. So I'll call this ``UserAgentSubscriber``::

    // src/AppBundle/EventListener/UserAgentSubscriber.php

    namespace AppBundle\EventListener;

    class UserAgentSubscriber
    {
    }

If you want to hook into Symfony, there are 2 ways to do it: with a listener
or a subscriber. They're actually exactly the same, the only difference is
where you configure *which* events you want to listen to.

I'm going to create a subscriber here because it's a little more flexible.
So ``UserAgentSubscriber`` needs to implement ``EventSubscriberInterface``::

    // src/AppBundle/EventListener/UserAgentSubscriber.php
    namespace AppBundle\EventListener;

    use Symfony\Component\EventDispatcher\EventSubscriberInterface;

    class UserAgentSubscriber implements EventSubscriberInterface
    {
    }

Notice that it added the ``use`` statement up there. And we're going to need
to implement 1 method which is ``getSubscribedEvents``. What this is going
to return is a simple array that says: Hey, apparently there's some event
whose name is ``kernel.request`` - we don't necessary know why it's called
or what it does yet - but when that event happens, I want Symfony to call
this  ``onKernelRequest`` function, which we're going to put inside of this
class. For now, let's just put a ``die('it works');``::

    // src/AppBundle/EventListener/UserAgentSubscriber.php
    // ...

    class UserAgentSubscriber implements EventSubscriberInterface
    {
        public function onKernelRequest()
        {
            die('it works');
        }

        public static function getSubscribedEvents()
        {
            return array(
                'kernel.request' => 'onKernelRequest'
            );
        }
    }

Cool! The event subscriber is ready to go. No, Symfony doesn't automatically
know this class is here or automatically scan the codebase. So to get Symfony
to know that there's a new ``UserEventSubscriber`` that wants to listen on
the ``kernel.request`` event, we're going to need to register this as a
service.

Registering the Subscriber/Listener
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

So I'm going to go into ``app/config/services.yml`` and clear the comments
out. And we'll give it a short, but descriptive name - ``user_agent_subscriber``,
the name of the service doesn't really matter in this case. There are no
arguments yet, so I'll just put an empty array. Now in order for Symfony
to know this is an event subscriber, we'll use something called a tag, and
set its name to ``kernel.event_subscriber``:

.. code-block:: yaml

    # app/config/services.yml
    # ...

    services:
        user_agent_subscriber:
            class: AppBundle\EventListener\UserAgentSubscriber
            tags:
                - { name: kernel.event_subscriber }

Now, that tag is called a `dependency injection tag`_, which is really awesome,
really advanced and really fun to work with inside of Symfony. And we're
going to talk about it in a different part of this series. With just this
configuration, Symfony will boot, it'll know about our subscriber, and when
that ``kernel.request`` event happens, it *should* call our function.

Sweet!

Logging Something in the Subscriber
-----------------------------------

Now inside of ``onKernelRequest``, let's do some real work. For now, I want
to log a message. I'm going to need the logger so I'll add a constructor
and even type hint the argument with the PSR LoggerInterface. And I'll use
a little PHPStorm shortcut to create and set that property for me::

    // src/AppBundle/EventListener/UserAgentSubscriber.php
    // ...

    use Psr\Log\LoggerInterface;

    class UserAgentSubscriber implements EventSubscriberInterface
    {
        private $logger;

        public function __construct(LoggerInterface $logger)
        {
            $this->logger = $logger;
        }

        // ...
    }

Now in our function, we'll log a very important message::

    // src/AppBundle/EventListener/UserAgentSubscriber.php
    // ...

    public function onKernelRequest()
    {
        $this->logger->info('Yea, it totally works!');
    }

And of course this isn't going to work unless we go back to ``services.yml``
and tell Symfony: Hey, we need the ``@logger`` service:

.. code-block:: yaml

    # app/config/config.yml
    # ...

    services:
        user_agent_subscriber:
            class: AppBundle\EventListener\UserAgentSubscriber
            arguments: ["@logger"]
            tags:
                - { name: kernel.event_subscriber }

Cool!

Let's refresh! It works, and if we click into the profiler, one of the
tabs is called "Logs", and under "info" we can see the message. So this is
already working, and if we go back to the Timeline and look closely, we should
see our ``UserAgentSubscriber``. And it's right there. Also, if we go back
to the events tab, we see the ``kernel.request`` with all of its listeners.
And if you look at the bottom, you see our ``UserAgentSubscriber`` on that
list too.

So we're hooking into that process already, even if we don't understand what's
going on with it.

Every Listener Gets an Event Object
-----------------------------------

Whenever you listen to *any* event - whether it's one of Symfony's core events
or it's an event from a third-party bundle you installed, your function is
passed an ``$event`` argument. So, we'll add ``$event``. The only trick is
that you don't automatically know what type of object that is, because every
event you listen to is going to pass you a different type of event object.

But no worries! I'm going to use the new ``dump()`` function from Symfony 2.6::

    // src/AppBundle/EventListener/UserAgentSubscriber.php
    // ...

    public function onKernelRequest($event)
    {
        dump($event);
    }

Let's go back a few pages, refresh, and the dump function prints that out
right in the web debug toolbar. And we can see it's dumping a ``GetResponseEvent``
object. So that's awesome - now we know what type of object is being passed
to us. And that's important because every event object will have different
methods and different information on it.

Let's type-hint the argument. Notice I'm using PHPStorm, so that added a
nice ``use`` statement to the top - don't forget that::

    // src/AppBundle/EventListener/UserAgentSubscriber.php
    // ...
    use Symfony\Component\HttpKernel\Event\GetResponseEvent;

    public function onKernelRequest(GetResponseEvent $event)
    {
        dump($event);
    }

What I want to do is get the ``User-Agent`` header and print that out in a
log message. Fortunately, this ``getResponseEvent`` object gives us access
to the request object. And again, every event you listen to will give you
a different event object, and every event object will have different methods
and information on it. It just *happens* to be that this one has a ``getRequest``
method, which is really handy for what we want to do. Now I'll just read
the ``User-Agent`` off of the headers, and log a message::

    // src/AppBundle/EventListener/UserAgentSubscriber.php
    // ...

    public function onKernelRequest(GetResponseEvent $event)
    {
        $request = $event->getRequest();
        $userAgent = $request->headers->get('User-Agent');

        $this->logger->info('Hello there browser: '.$userAgent);
    }

Let's try it! I'll get back into the profiler, then to the Logs... and it's
working perfectly.

Even if we don't understand everything that's happening between the request
and response, we already know that there are these listeners that happen.
But next, we're going to walk through the code that handles *all* of this.

.. _`dependency injection tag`: http://symfony.com/doc/current/reference/dic_tags.html