.. _extending_locust:

==================================
Extending Locust using event hooks
==================================

Locust comes with a number of event hooks that can be used to extend Locust in different ways.

Event hooks live on the Environment instance under the :py:attr:`events <locust.env.Environment.events>` 
attribute. However, since the Environment instance hasn't been created when locustfiles are imported,  
the events object can also be accessed at the module level of the locustfile through the 
:py:obj:`locust.events` variable.

Here's an example on how to set up an event listener::

    from locust import events
    
    @events.request.add_listener
    def my_request_handler(request_type, name, response_time, response_length, response,
                           context, exception, **kw):
        if exception:
            print(f"Request to {name} failed with exception {exception}")
        else:
            print(f"Successfully made a request to: {name})

.. note::

    It's highly recommended that you add a wildcard keyword argument in your listeners
    (the \**kw in the code above), to prevent your code from breaking if new arguments are
    added in a future version.

    Note that it is entirely possible to implement a client that does not support all parameters 
    (some non-HTTP protocols might not have a concept of `response_length` or `response` object).

.. _request_context:

Request context
==================

By using the context parameter in the request method information you can attach data that will be forwarded by the 
request event. This should be a dictionary and can be set directly when calling request() or on a class level 
by overwriting the User.context() method. 

Context from request method::

    class MyUser(HttpUser):
        @task
        def t(self):
            self.client.post("/login", json={"username": "foo"}, context={"username": "foo"})

        @events.request.add_listener
        def on_request(context, **kwargs):
            print(context["username"])
    
Context from User class::

    class MyUser(HttpUser):
        def context(self):
            return {"username": self.username}

        @task
        def t(self):
            self.username = "foo"
            self.client.post("/login", json={"username": self.username})

        @events.request.add_listener
        def on_request(context, **kwargs):
            print(context["username"])


.. seealso::

    To see all available events, please see :ref:`events`.


Adding Web Routes
==================

Locust uses Flask to serve the web UI and therefore it is easy to add web end-points to the web UI.
By listening to the :py:attr:`init <locust.event.Events.init>` event, we can retrieve a reference 
to the Flask app instance and use that to set up a new route::

    from locust import events
    
    @events.init.add_listener
    def on_locust_init(web_ui, **kw):
        @web_ui.app.route("/added_page")
        def my_added_page():
            return "Another page"

You should now be able to start locust and browse to http://127.0.0.1:8089/added_page



Extending Web UI
================

As an alternative to adding simple web routes, you can use `Flask Blueprints 
<https://flask.palletsprojects.com/en/1.1.x/blueprints/>`_ and `templates 
<https://flask.palletsprojects.com/en/1.1.x/tutorial/templates/>`_ to not only add routes but also extend 
the web UI to allow you to show custom data along side the built-in Locust stats. This is more advanced 
as it involves also writing and including HTML and Javascript files to be served by routes but can 
greatly enhance the utility and customizability of the web UI.

A working example of extending the web UI, complete with HTML and Javascript example files, can be found 
in the `examples directory <https://github.com/locustio/locust/tree/master/examples>`_ of the Locust 
source code.



Run a background greenlet
=========================

Because a locust file is "just code", there is nothing preventing you from spawning your own greenlet to
run in parallel with your actual load/Users.

For example, you can monitor the fail ratio of your test and stop the run if it goes above some threshold:

.. code-block:: python

    from locust import events
    from locust.runners import STATE_STOPPING, STATE_STOPPED, STATE_CLEANUP, WorkerRunner

    def checker(environment):
        while not environment.runner.state in [STATE_STOPPING, STATE_STOPPED, STATE_CLEANUP]:
            time.sleep(1)
            if environment.runner.stats.total.fail_ratio > 0.2:
                print(f"fail ratio was {environment.runner.stats.total.fail_ratio}, quitting")
                environment.runner.quit()
                return


    @events.init.add_listener
    def on_locust_init(environment, **_kwargs):
        # only run this on master & standalone
        if not isinstance(environment.runner, WorkerRunner):
            gevent.spawn(checker, environment)


More examples
=============

See `locust-plugins <https://github.com/SvenskaSpel/locust-plugins#listeners>`_
