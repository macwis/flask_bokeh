# Bokeh + panel + Flask + asyncio + gunicorn + SQLAlchemy (boilerplate)

This repo summarized the research on combining the solutions above.


# Configuration

Set environmental variable for your database connection:

```
SQLALCHEMY_DATABASE_URI=postgres+psycopg2://postgres:postgres@localhost:5432/bokehfun
```


# Running

`gunicorn -w 4 flask_gunicorn_embed:app`

What it does it serves a bokeh server app embedded in Flask with a wait for SQL query result.

There is 5s sleep in an SQL query, blocking, which after execution pushes the chart render to the client - it imiatates a long runnign SQL query.

The service is available concurrently to multiple users at the same time.



# Description

Basic reading material:

- https://docs.bokeh.org/en/latest/docs/user_guide/server.html
    Important quote: "There is currently no locking around adding next tick callbacks to documents. It is recommended that at most one thread adds callbacks to the document. It is planned to add more fine-grained locking to callback methods in the future."

- https://panel.holoviz.org/user_guide/Server_Deployment.html
    Important quote: "Once you have deployed the app you might find that if your app is visited by more than one user at a time it will become unresponsive. In this case you can use the Heroku CLI to scale your deployment."


# Observations:

1. The scaling happens on the thread level - where each thread opens up a communication port for a websocket. Once thread - one user :(

2. When integrating with a Flask app and rasing server instances we can specify the port via `sockets, port = bind_sockets("localhost", 5100)`, but there is no port range option so while running `-w 4` we need to use `0` - eventually not knowing which ports are going to be randomly picked.

3. To handle websockets we need to make an nginx reverse proxy (as the documentation suggests) and appoint every running instance via `upstream` declaration. It requires some custom mechanism to control the random port selection from point 2 and manage the pool:

```
upstream myapp {
    least_conn;                 # Use Least Connections strategy
    server 127.0.0.1:5100;      # Bokeh Server 0
    server 127.0.0.1:5101;      # Bokeh Server 1
    server 127.0.0.1:5102;      # Bokeh Server 2
    server 127.0.0.1:5103;      # Bokeh Server 3
    server 127.0.0.1:5104;      # Bokeh Server 4
    server 127.0.0.1:5105;      # Bokeh Server 5
}
```

4. Alternatively to the above we could run multiple instances with stand-alone servers (as they suggest for the Heroku deployment), but is a very low resource utilization strategy.

6. There is special care needed in passing on the sessions and stick sessions for user-instance via nginx proxy.

7. In larger deployments topic 6. would have to be addressed by higher layers of the reverse proxy and they all would have to properly work.

8. For development nginx would have to be built in the Dockerfile together with gunicorn running multiple threads (via e.g. systemd).
