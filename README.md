# Pulumi Fundamentals 

use Pulumi to build, configure, and deploy a real-life, modern application.

we’re going to learn more about cloud computing by exploring how to use Pulumi to build, configure, and deploy a real-life, modern application using Docker. 
We will create a frontend, a backend, and a database to deploy the Pulumipus Boba Tea Shop. 
Along the way, we’ll learn more about how Pulumi works.

## Sample app

The sample app we’re building, the Pulumipus Boba Tea Shop, is a progressive web application (PWA) built 
with MongoDB, ExpressJS, React, and NodeJS (the MERN stack). 
It’s a fairly common implementation found in eCommerce applications. 
We have adapted this application from this repository. 
The app consists of a frontend client, a backend REST server to manage transactions, 
and a MongoDB instance for storing product data.

## Topics

### Creating a Pulumi Project

Infrastructure in Pulumi is organized into projects. 
In the Pulumi ecosystem, a project represents a Pulumi program that, when run, declares the desired infrastructure for Pulumi to manage. 
The program has corresponding stacks, or isolated, independently configurable instances of your Pulumi program. 
We’ll talk more about stacks later in the Building with Pulumi pathway.

#### Initialize the project

Since a Pulumi project is just a directory with some files in it, it is possible for you to create a new one by hand. 
The pulumi new command-line interface (CLI) command, however, automates the process and ensures you have everything you need, so let’s use that command. 
The -y flag answers “yes” to the prompts to create a default project:

```
pulumi new typescript -y
```

#### Inspect the new project

The basic project created by pulumi new is comprised of multiple files:

* Pulumi.yaml: your project's metadata, containing its name and language
* index.ts : your program's main entrypoint file
* package.json: your project's Node.js dependency information

### Creating Docker Images

In this part, we’ll create our first Pulumi resource. 
Resources in Pulumi are the basic building blocks of your infrastructure, whether that’s a database instance or a compute instance or a specific storage bucket. 
In Pulumi, resource providers manage your resources. 
You can group those resources to abstract them (such as a group of compute instances that all have the same configuration and implementation) via component resources.

In this case, our resources are going to be Docker containers and images that we build locally using infrastructure as code. 
Our resource provider is Docker, and we’re using Node.js as our language host, or the executor that compiles the code we write and interprets it for Pulumi.

```
import * as pulumi from "@pulumi/pulumi";
import * as docker from "@pulumi/docker";

const stack = pulumi.getStack();

// Pull the backend image
const backendImageName = "backend";
const backend = new docker.RemoteImage(`${backendImageName}Image`, {
    name: "pulumi/tutorial-pulumi-fundamentals-backend:latest",
});

// Pull the frontend image
const frontendImageName = "frontend";
const frontend = new docker.RemoteImage(`${frontendImageName}Image`, {
    name: "pulumi/tutorial-pulumi-fundamentals-frontend:latest",
});

// Pull the MongoDB image
const mongoImage = new docker.RemoteImage("mongoImage", {
    name: "pulumi/tutorial-pulumi-fundamentals-database-local:latest",
});
```

### Configuring and Provisioning Containers

Now that we’ve created our images, we can provision our application with a network and containers. 
First, we’re going to add configuration to our Pulumi program. Pulumi is a tool to configure your infrastructure, and that includes being able to configure the different stacks with different values. 
As a result, it makes sense to include the basic configurations as variables at the top of your program.

```
import * as pulumi from "@pulumi/pulumi";
import * as docker from "@pulumi/docker";

// Get configuration values
const config = new pulumi.Config();
const frontendPort = config.requireNumber("frontendPort");
const backendPort = config.requireNumber("backendPort");
const mongoPort = config.requireNumber("mongoPort");
const mongoHost = config.require("mongoHost"); // Note that strings are the default, so it's not `config.requireString`, just `config.require`.
const database = config.require("database");
const nodeEnvironment = config.require("nodeEnvironment");
const protocol = config.require("protocol")

const stack = pulumi.getStack();

// Pull the backend image
const backendImageName = "backend";
const backend = new docker.RemoteImage(`${backendImageName}Image`, {
    name: "pulumi/tutorial-pulumi-fundamentals-backend:latest",
});

// Pull the frontend image
const frontendImageName = "frontend";
const frontend = new docker.RemoteImage(`${frontendImageName}Image`, {
    name: "pulumi/tutorial-pulumi-fundamentals-frontend:latest",
});

// Pull the MongoDB image
const mongoImage = new docker.RemoteImage("mongoImage", {
    name: "pulumi/tutorial-pulumi-fundamentals-database-local:latest",
});

// Create a Docker network
const network = new docker.Network("network", {
    name: `services-${stack}`,
});

// Create the MongoDB container
const mongoContainer = new docker.Container("mongoContainer", {
    image: mongoImage.repoDigest,
    name: `mongo-${stack}`,
    ports: [
        {
            internal: mongoPort,
            external: mongoPort,
        },
    ],
    networksAdvanced: [
        {
            name: network.name,
            aliases: ["mongo"],
        },
    ],
});

// Create the backend container
const backendContainer = new docker.Container("backendContainer", {
    name: `backend-${stack}`,
    image: backend.repoDigest,
    ports: [
        {
            internal: backendPort,
            external: backendPort,
        },
    ],
    envs: [
        `DATABASE_HOST=${mongoHost}`,
        `DATABASE_NAME=${database}`,
        `NODE_ENV=${nodeEnvironment}`,
    ],
    networksAdvanced: [
        {
            name: network.name,
        },
    ],
}, { dependsOn: [ mongoContainer ]});

// Create the frontend container
const frontendContainer = new docker.Container("frontendContainer", {
    image: frontend.repoDigest,
    name: `frontend-${stack}`,
    ports: [
        {
            internal: frontendPort,
            external: frontendPort,
        },
    ],
    envs: [
        `PORT=${frontendPort}`,
        `HTTP_PROXY=backend-${stack}:${backendPort}`,
        `PROXY_PROTOCOL=${protocol}`
    ],
    networksAdvanced: [
        {
            name: network.name,
        },
    ],
});
```

Run pulumi up to get the application running. Open a browser to http://localhost:3001, and our application is now deployed.

#### Update the database

What if we want to add to the products on the page? We can POST to the API just as we would any API. Generally speaking, you would typically wire the database to an API and update it that way with any cloud, so we’re going to do exactly that here.

Open a terminal and run the following command.

```
curl --location --request POST 'http://localhost:3000/api/products' \
--header 'Content-Type: application/json' \
--data-raw '{
    "ratings": {
        "reviews": [],
        "total": 63,
        "avg": 5
    },
    "created": 1600979464567,
    "currency": {
        "id": "USD",
        "format": "$"
    },
    "sizes": [
        "M",
        "L"
    ],
    "category": "boba",
    "teaType": 2,
    "status": 1,
    "_id": "5f6d025008a1b6f0e5636bc7",
    "images": [
        {
            "src": "classic_boba.png"
        }
    ],
    "name": "My New Milk Tea",
    "price": 5,
    "description": "none",
    "productCode": "852542-107"
}'
```

#### Cleaning up

Whenever you’re working on learning something new with Pulumi, 
it’s always a good idea to clean up any resources you’ve created so you don’t get charged on a free tier or otherwise leave behind resources you’ll never use. 
Let’s clean up.

Run the pulumi destroy command to remove all of the resources:

```
pulumi destroy
```
