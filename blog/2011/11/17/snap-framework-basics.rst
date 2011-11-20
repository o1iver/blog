public: no
tags: [haskell, snap-framwork, web]

Snap Framework Basics
=====================

I usually develop in Python, although that has more to do with the languages look-and-feel than anything else. After almost ten years of progamming I have come to the conclusion that that is one of the most important aspects of a language. If you like static-typing use something like Java/C#, if you don't use Python or Ruby or whatever else you like. Anyway that doesn't really matter right now (although I might try to write down my thoughts about this some time). The thing is that a few months ago I first had a look at `Haskell <http://haskell.org>`_.

Haskell is a beauty. If Java is a Volswagen Golf and Python a BMW M3, then Haskell is a Porsche Carrera GT, incredible, but hard to drive. The second I finally understood what Haskell was about, I knew that it was going to become my go-to language in the future (once I get it :-). So after dabbling around for a while I thought it was time to have a look at how suited Haskell is for web development and that is where the `Snap Framework <http://snapframework.com/>`_ comes into play. From the site:

*Snap is a simple web development framework for unix systems, written in the Haskell programming language. Snap has a high level of test coverage and is well-documented.*

Installation was very easy:

.. sourcecode:: bash
    
    cabal install snap

So below are a few simple examples that I am playing with to get a hang of the framework before I can build a real project with it (already have an idea for that).

Getting the request method
--------------------------

The first thing I tried to figure out was how to get information about the request. Here is a request handler that simply prints out a request's HTTP method. With this as a basis it becomes pretty obvious how to extract other information such as the request URI, headers, etc.

.. sourcecode:: haskell

    methodHandler = do
        methodStr <- (show . rqMethod) <$> getRequest
        writeBS $ B.pack (methodStr ++ ['\n'])

Using different handlers with the same route (depending on request parameters)
------------------------------------------------------------------------------

Although you could use the above mentioned method of getting a request's method to handle these differently using cases or ifs/thens, there is an idiom for returning different responses depending on the request headers. It is based on the "<|>" function. Here is an example:

.. sourcecode:: haskell
    
    indexHandler =  method GET  indexGet
                <|> method POST indexPost
                <|> errorHandler

So what is going on here? Well it's actually pretty simple and entirely based on Snap's implementation of the `Alternatives' "associative binary operator" (<|>) <http://hackage.haskell.org/packages/archive/base/4.4.1.0/doc/html/Control-Applicative.html#v:-60--124--62->`_. The *<|>* operator allows you to *try* a certain handler, but specify following ones in case the former fails. This is what we are doing here. We are saying if the method is *GET* use the *indexGet* handler, if it is *POST* use the *indexPost* handler. If it is neither then use the *errorHandler*. How is this done? Well here is the description of the method function:

.. sourcecode:: haskell
    
    method :: MonadSnap m => Method -> m a -> m aSource
    -- Runs a Snap monad action only if the request's HTTP method matches the given method.

So you give the method function a `Method <http://hackage.haskell.org/packages/archive/snap-core/0.6.0.1/doc/html/Snap-Core.html#t:Method>`_ and a handler, then the handler action will be completed if and only if the request's method is the same as the given one. If the methods do not matche the method function will call `pass <http://hackage.haskell.org/packages/archive/snap-core/0.6.0.1/doc/html/Snap-Core.html#v:pass>`_ and the handler will fail, resulting in the next handler being tried (*method POST indexPost*). If you want to see how exactly this happens have a look the `method function source <http://hackage.haskell.org/packages/archive/snap-core/0.6.0.1/doc/html/src/Snap-Internal-Types.html#method>`_:

.. sourcecode:: haskell

    ------------------------------------------------------------------------------
    -- | Runs a 'Snap' monad action only if the request's HTTP method matches
    -- the given method.
    method :: MonadSnap m => Method -> m a -> m a
    method m action = do
        req <- getRequest
        unless (rqMethod req == m) pass
        action

Note that you can chain these *checks* to create more advanced request handlers. See this extended example:

.. sourcecode:: haskell
    
    indexHandler =  ifTop (method GET  indexGet) 
                <|> ifTop (method POST indexPost)
                <|> errorHandler  

Here we are not only checking the request method but also whether or not the request URI is *top* (yes, I agree that this function name is not ideal). Here is the ifTop method description and type:

.. sourcecode:: haskell

    ifTop :: MonadSnap m => m a -> m aSource
    -- Runs a Snap monad action only when rqPathInfo is empty.

What does this mean? Well *rqPathInfo* return the request's URI's path part that is not covered by the route declaration. If your route is "/posts/2011/" a request for "/posts/2011/11/17/" may still be routed to the specified request handler. The difference will be that in the former case *rqPathInfo* will return an empty string and in the latter case it will return *11/17/*. So using *ifTop* allows you to say *the request URI may not be longer than the one specified in the routing scheme*.

But to get back to the point of how to handle different request headers: what this is meant to show is that you can chain different types of request checks to route a request to the correct handler.

Complete routing example
------------------------

Here is a complete routing and handling example for an application that I am currently developing:

.. sourcecode:: haskell
    
    indexHandler = ifTop ( method GET  indexHandler'
                       <|> genericError 405 "Method Not Allowed"
                   )
               <|> error404 -- will catch any routing error (even for other request
                             -- URIs as this is the fallback route "/")
    
    generateHandler = ifTop ( method GET generateHandler'
                          <|> error405
                      )
                      
    registeredHandler = ifTop ( method GET registeredHandler'
                            <|> error405
                        )
    
    indexHandler' = do
      -- application logic
      writeBS $ B.pack "index page"
    
    generateHandler'  = do
      expr <- fromMaybe "" <$> getParam "expr"\
      -- application logic
      writeBS $ append (B.pack "API.generate: ") expr
    
    registeredHandler' = do
      domain <- fromMaybe "" <$> getParam "domain"
      -- application logic
      writeBS $ append (B.pack "API.registered: ") domain
    
    error404 = genericError 404 "Not Found"
    error405 = genericError 405 "Method Not Allowed"
    
    genericError c s = do
      modifyResponse $ setResponseStatus c $ B.pack s
      writeBS $ B.pack ((show c) ++ " - " ++ s)
      r <- getResponse
      finishWith r
    
    ------------------------------------------------------------------------------
    -- | The main entry point handler.
    site :: Application ()
    site = route [ ("/"                           , indexHandler)
                 , ("/api/generate/:expr/"        , generateHandler)
                 , ("/api/registered/:domain/"    , registeredHandler)
                 ]
           <|> serveDirectory "resources/static"

As you can see these handlers combine both *method* and *ifTop* to check whether a request's HTTP method is right and whether or not the request URI contains additional unwanted path segements. Here are a few examples of requests and the server's response:

::
    request:  GET /
    response: HTTP/1.1 200 OK
              index page

    request:  POST /
    response: HTTP/1.1 405 Method Not Allowed
              405 - Method Not Allowed

    request:  GET /api/
    response: HTTP/1.1 404 Not Found
              404 - Not Found

    request:  POST /api/
    response: HTTP/1.1 404 Not Found
              404 - Not Found

    request:  GET /api/generate/abc
    response: HTTP/1.1 200 OK
              API.generate: abc

    request:  PUT /api/registered/google.com
    response: HTTP/1.1 405 Method Not Allowed
              405 - Method Not Allowed



