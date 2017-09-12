If you've read through Working with private modules, you'll know that in order to use private modules, you need to be logged in to npm via the npm CLI.

If you're using npm private modules in an environment where you're not directly able to log in, such as inside a CI Server or a Docker container, you'll need to get and export an npm token as an environment variable. That token should look like.NPM_TOKEN=00000000-0000-0000-0000-000000000000

The Getting an Authentication Token should help you generate that token.

If this is the workflow you need, please read the CI Server Config doc. If that works with your system than perfect.

If it doesn't, here we'll look at the problems with this workflow when running inside npm install a Docker container.

Runtime Variables

If you had the following Dockerfile:

FROM risingstack/alpine:3.3-v4.3.1-3.0.1
 
COPY package.json package.json  
RUN npm install
 
# Add your source files
COPY . .  
CMD npm start  
Which will use the RisingStack Alpine Node.JS Docker image, copy the package.json into our container, installs dependencies, copies the source files and runs the start command as specified in the package.json

In order to install private packages, you may think that we could just add a line before we run,npm install using the ENV parameter:

ENV NPM_TOKEN=00000000-0000-0000-0000-000000000000
However this doesn't work as you would expect because you want the npm install to occur when you run,docker build and in this instance, ENV variables aren't used, they are set for runtime only.

Build-time variables

We have to take advantage of a different way of passing environment variables to Docker, available since Docker 1.9. We'll use the slightly confusingly named ARG parameter.

A complete example that will allow us to use to--build-arg pass in our NPM_TOKEN requires adding a .npmrc file to the project. That file should contain the following content:

//registry.npmjs.org/:_authToken=${NPM_TOKEN}
The Dockerfile that takes advantage of this has a few more lines in it than our example earlier that allows us to use the file .npmrc and the parameter ARG.

FROM risingstack/alpine:3.3-v4.3.1-3.0.1
 
ARG NPM_TOKEN  
COPY .npmrc .npmrc  
COPY package.json package.json  
RUN npm install  
RUN rm -f .npmrc
 
# Add your source files
COPY . .  
CMD npm start
This adds the expected,ARG NPM_TOKEN but also copies the file .npmrc, and removes it when npm install completes.

To build the image using this Dockerfile and the token, you can run the following (note the . at the end to give Docker build the current directory as an argument):
```
docker build --build-arg NPM_TOKEN=${NPM_TOKEN} .
```
This will take your current environment NPM_TOKEN variable, and will build the Docker image using it, so you can run inside npm install your container as the current logged in user!

Note: Even if you delete the file .npmrc, it'll be kept in the commit history - to clean your secret up entirely make sure to squash them.
