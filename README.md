## Ref:

https://dev.to/dariansampare/setting-up-docker-typescript-node-hot-reloading-code-changes-in-a-running-container-2b2f

## Step 1: Creating a server with TypeScript & Express

Let's whip up a simple Express server with TypeScript and get it running locally (we'll dockerize it after!).

Make a directory for the project and cd in there:

```sh
mkdir ts-node-docker
cd ts-node-docker
```

Initialize a node project and add whatever values you want when prompted (I just skip everything by mashing enter...):

```sh
npm init
```

Next, install TypeScript as a dev dependancy:

```sh
npm i typescript --save-dev
```

Once that's downloaded, create a tsconfig.json file:

```sh
npx tsc --init
```

Now we should have a tsconfig.json in the root of out project directory, lets edit the following entries in there:

```sh
"baseUrl": "./src"
"target": "esnext"
"moduleResolution": "node"
"outdir": "./build"
```

The **baseUrl** tells TS that our .ts source code files will be in the ./src folder.

The **target** can be whatever version of JS you like, I go with esnext.

The **moduleResolution** must be set to node for node projects.

The **outdir** tells TS where to put the compiled JavaScript code when the TS files are compiled.

Next, let's install express, and then its typings as a dev dependancy:

```sh
npm i --save express
npm i -D @types/express
```

Cool, we are ready to code up our server. Let's make a src/ folder at the root of our project and add an index.ts file.

In index.ts, add the following code:

```sh
import express from 'express';

const app = express();
app.listen(4000, () => {
  console.log(`server running on port 4000`);
});

```

That's all we'll need to start our server, but now we need to get this thing running and watching for changes we make to the code.

For that, we'll use **ts-node** and **nodemon**, intall that now:

```sh
npm i -D ts-node nodemon
```

With nodemon, we can watch files while the code is running, and ts-node just lets us run node projects written in TS very easily.

I like to have my nodemon setup in a config file, so I'll add a nodemon.json file to the root of my project folder and add the following options:

```sh
{
  "verbose": true,
  "ignore": [],
  "watch": ["src/**/*.ts"],
  "execMap": {
    "ts": "node --inspect=0.0.0.0:9229 --nolazy -r ts-node/register"
  }
}
```

The key takeaways here are the watch command (which tells nodemon what files it should watch for), and the ts option in execMap.

This tells nodemon how to handle TS files. We run them with node, throw in some debugging flags, and register ts-node.

Okay, now we can add scripts to our package.json that uses nodemon to start our project. Go ahead and add the following to your package.json:

```sh
"scripts": {
    "start": "NODE_PATH=./build node build/index.js",
    "build": "tsc -p .",
    "dev": "nodemon src/index.ts",
}
```

The dev command starts our project with nodemon. The build command compiles our code into JavaScript, and the start command runs our built project.

We specify the **NODE_PATH** to tell our built application where the root of our project is.

You should now be able to run the application with hot reloading like so:
npm run dev 
Great! Now let's dockerize this thing 🐳



## Step 2: Docker Development & Production Step

If you haven't installed Docker, do that now. I also recommend their desktop app, both of which can be found on their website.

Next, let's add a Dockerfile to the root of our project directory and add the following code for the development step:

```sh
FROM node:14 as base

WORKDIR /home/node/app

COPY package*.json ./

RUN npm i

COPY . .
```

This pulls in a node image, sets a working directory for our container, copies our package.json and installs it, and then copies all of our project code into the container.

Now, in the same file, add the production step:

```sh
FROM base as production

ENV NODE_PATH=./build

RUN npm run build
```

This extends our development step, sets our environment variable, and builds the TS code ready to run in production.

Notice we haven't added any commands to run the development or production build, that's what our docker-compose files will be for!

Create a docker-compose.yml file at the root of our directory and add the following:

```sh
version: '3.7'

services:
  ts-node-docker:
    build:
      context: .
      dockerfile: Dockerfile
      target: base
    volumes:
      - ./src:/home/node/app/src
      - ./nodemon.json:/home/node/app/nodemon.json
    container_name: ts-node-docker
    expose:
      - '4000'
    ports:
      - '4000:4000'
    command: npm run dev
```

This creates a container called ts-node-docker, uses our dockerfile we created, and runs the build step (see the target).

It also creates volumes for our source code and nodemon config, you'll need this to enable hot-reloading!

Finally, it maps a port on our machine to the docker container (this has to be the same port we setup with express).

Once that's done, we can build our docker image:

```sh
docker-compose build
```

You should be able to see the build steps in your terminal.

Next, we can run the container as follows:

```sh
docker-compose up -d
```

Success! You should now have a container running that picks up on any changes you make to your TypeScript source code. I highly recommend using the docker desktop app to view the containers you have running.

Alt Text

You can stop the container like so:

```sh
docker-compose down
```

Now we are also going to want to run this thing in production, so let's create a separate docker-compose.prod.yml for that:

```sh
version: '3.7'

services:
  ts-node-docker:
    build:
      target: production
    command: node build/index.js
```

This file is going to work together with our first docker-compose file, but it will overwrite the commands we want to change in production.

So, in this case, we are just going to target the production step of our Dockerfile instead, and run node build/index.js instead of npm run dev so we can start our compiled project.

To start our container in production, run:

```sh
docker-compose -f docker-compose.yml -f docker-compose.prod.yml up -d   
```

This tells docker-compose which files to use, the later files will overwrite any steps in the prior files.

You should now have the built application running just how it would be in production, no hot reloading needed here!

Lastly, I hate typing out all these docker commands, so I'll create a Makefile in the root of my project and add the following commands that can be executed from the command line (eg make up):

```sh
up:
    docker-compose up -d

up-prod:
    docker-compose -f docker-compose.yml -f docker-compose.prod.yml up

down: 
    docker-compose down
```

If you made it all the way to the end, congrats, and thank you. Hopefully, this made somebody's day a lot easier while trying to integrate these two awesome technologies together.

If you liked this, I post tutorials and tech-related videos over on my YouTube channel as well.

We have a growing tech-related Discord Channel too, so feel free to pop by.

Happy coding! 👨‍💻 🎉