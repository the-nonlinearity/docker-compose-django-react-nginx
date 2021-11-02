# docker-compose-django-react-nginx

## Summary

This repository serves as an example of a basic Django backend with React frontend for a development 
setup. Server routing is accomplished via an NGINX server to keep the development of the applications as 
simple as possible.


We start by creating a `create-react-app` in the frontend directory to keep things separated from the backend 
application. If you are using a Node environment tool like myself you can activate that first.  

```bash
npx create-react-app frontend
```

You should now have the following directory structure inside your `frontend` directory:

```text

```

And you can change into the `frontend` directory and run `yarn start`. And then going to `localhost:3000` you should 
see the React Start page loaded.

Now lets create the Dockerfile to run the frontend application as a docker image.

```dockerfile
FROM node:16.6

WORKDIR /app

# Install dependencies
COPY package.json /app

RUN yarn install

COPY . .

EXPOSE 3000

CMD [ "yarn", "start" ]
```
You should now be able to build the container with `docker build . -t frontend`, and to start it 
with `docker run -p 3000:3000 frontend`. 
The `-p 3000:3000` option says to bind the port 3000 of the host to the port 3000 of the 
container, allowing you to go to [http://localhost:3000](http://localhost:3000) and see your 
application running as if you were inside the container.

If you receive a build error, you might need to update `eslintConfig` to `"eslintConfig": { "extends": [ "react-app" ] },`

## Compose: add a container for React

Let's build the frontend app using the compose file we already have and then connect it up. We can start with setting 
up the basic configs, so add the `frontend` service to your `docker-compose.yaml` file in your project root:

```yaml 
  frontend:
    build: ./frontend
    container_name: react-frontend
    volumes:
      - /app/node_modules
      - ./frontend:/app
    ports:
      - "3000:3000"
    depends_on:
      - backend
```

Build and run the service with `docker-compose up --build`. The name of the image will be created as the 
container name `react-frontend`. You can name your service accordingly. You should be able to navigate to
[http://localhost:3000](http://localhost:3000) and see your application running as before.

Ok, let's add the NginX service now. Remember that we need a bridge to make our services communicate.

Update the `docker-compose.yml` as follows:

```yaml 

  frontend:
    build: ./frontend
    container_name: react-frontend
    volumes:
      - /app/node_modules
      - ./frontend:/app
    depends_on:
      - backend
    networks:
      - nginx_network
```

Note that we removed the ports directive from our frontend service. We will now communicate with 
NginX instead of the React server. We still want to access our app at `http://localhost:3000`, 
and we want NginX to listen to the port 80 in the container, so we use `ports: - 3000:80`.

We also bind a local directory to the `/etc/nginx/conf.d` container directory. It should already exist, so let's add 
some code to it.

```text
# first we declare our upstream server, which is our Django application
upstream backend_server {
    # docker will automatically resolve this to the correct address
    # because we use the same name as the service: "backend"
    server backend:8000;
}

# here we declare our upstream server, which is our React application
upstream frontend_server {
    # docker will automatically resolve this to the correct address
    # because we use the same name as the service: "frontend"
    server frontend:3000;
}

# now we declare our main server
server {

    listen 80;
    server_name localhost;

    location / {
        # everything is passed to Gunicorn
        proxy_pass http://backend_server;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header Host $host;
        proxy_redirect off;
    }

    location /static/ {
        alias /app/src/static/;
    }

    location /media/ {
        alias /app/src/media/;
    }
}

server {
    listen 8080;

    # ignore cache frontend
    location ~* (service-worker\.js)$ {
        add_header 'Cache-Control' 'no-store, no-cache, must-revalidate, proxy-revalidate, max-age=0';
        expires off;
        proxy_no_cache 1;
    }

    location / {
        proxy_pass http://frontend_server;
    }

}
```

This time we listen to the frontend service on port 8080 and expose it on 3000 to the client. 

Before diving into what the Nginx server does, let’s look at a proxy and a reverse proxy:

A proxy server is a server that retrieves data from the Internet on behalf of a user, such as a web page. 
It acts as a bridge between two entities (the client and the server) and provides a service (requests and responses).

The proxy server will operate as a go-between, retrieving that web page for you. Now, when you go to a website, 
the proxy server receives the request from your computer, and the proxy server directly retrieves the web page on 
your behalf and sends it to your computer.

There are two primary proxy servers:

Forward proxy server - handles the request to the server from different clients.
A reverse proxy server functions in the opposite way that a forward proxy server works.
In a forward proxy server, a client connects to the server, but in reverse proxy, the server connects to the client. 
A forward proxy is thus for clients, while a reverse proxy is for servers.

In this case, the reverse proxy server makes requests from one or more destination servers on behalf of the client.

In this application, we will access both the client and the server using the Nginx proxy. Also, when using Nginx, 
changes made to the application will reflect immediately since the whole application is being served on a reverse 
proxy architecture. This is exactly what we want for our development setup.

Now you also need to update your `docker-compose.yaml`:

```yaml
version: '3'

services:

  backend:
    build: ./backend
    container_name: django-backend
    stdin_open: true
    tty: true
    volumes:
      - ./backend/:/app/src # maps host directory to internal container directory
      - static_volume:/app/src/static
      - media_volume:/app/src/media
    networks:
      - nginx_network
      - postgres_network
    depends_on:
      - postgres

  frontend:
    build: ./frontend
    container_name: react-frontend
    volumes:
      - /app/node_modules
      - ./frontend:/app
    depends_on:
      - backend
    networks:
      - nginx_network  <-- Connect the nginx network

  nginx:
    image: nginx:1.13
    volumes:
      - ./nginx/conf.d:/etc/nginx/conf.d
      - static_volume:/app/src/static
      - media_volume:/app/src/media
    ports:
      - 8000:80
      - 3000:8080  <-- Here we connect the frontend to port 3000 and listen to the reverse proxy on port 8080
    depends_on:
      - backend
      - frontend
    networks:
      - nginx_network

  postgres:
    image: postgres:12
    env_file:
      - env.db
    networks:
      - postgres_network
    volumes:
      - postgres_volume:/var/lib/postgresql/data

networks:
  nginx_network:
    driver: bridge
  postgres_network:
    driver: bridge

volumes:
  postgres_volume:
  static_volume:
  media_volume:
```

Now you likely also want to modify your compose file to load updates you make to the source code, so we add the 
following to the yaml definition.

```yaml 

  frontend:
    build: ./frontend
    container_name: react-frontend
    stdin_open: true  <-- Add here
    tty: true         <-- Add here
    volumes:
      - /app/node_modules
      - ./frontend:/app
    depends_on:
      - backend
    networks:
      - nginx_network
```

This allows your source code to be reloaded in the volume you created for the frontend. All you need to do is hit
refresh in your browser after saving your code changes. 

## NginX: Hook up the backend with your frontend

The last thing we would like to do is hook up our backend api with our frontend React application.

Todo:


## Conclusion

I created the setup in this way in order to have a reuseable development environment in which you can easily swap 
out the frontend framework or backend framework without modifying the infrastructure. You can also easily add multiple 
backend or database services if you would like to. Most examples I have found usually only cover a single service, 
either on the frontend or the backend, therefore I hope this will be helpful to others. 




References:

[https://reactjs.org/docs/create-a-new-react-app.html](https://reactjs.org/docs/create-a-new-react-app.html)
[https://www.section.io/engineering-education/build-and-dockerize-a-full-stack-react-app-with-nodejs-and-nginx/](https://www.section.io/engineering-education/build-and-dockerize-a-full-stack-react-app-with-nodejs-and-nginx/)
