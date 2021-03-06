# Deploy a Microservices Application on Openshift

### About
A demonstration on how to deploy a microservice application on Openshift. We will be deploying a voting application on openshift.

### Requirements
* An Openshift cluster. And the openshift-cli configured. The demo will be done through the openshift web-console. Options for the openshift cluster for development are [minishift](https://www.okd.io/minishift/), [Red Hat's CDK](https://developers.redhat.com/products/cdk/overview) or through [Red Hat's Interactive Learning Portal](https://learn.openshift.com/playgrounds)

* A clone of the [The Voting application](https://github.com/richardgreg/example-voting-application), or simply use the the repo as long you don't plan to add webhook triggers. This is a sample voting application which provides an interface for a user to vote and another interface to show the results. The application consists of various components such as the voting app which is a web application developed in Python to provide the user with an interface to choose between two options, a cat and a dog. When you make a selection the vote is stored in Redis. This vote is then processed by the worker which is an application written in .NET. The worker application takes the new vote and updates the persistent database which is PostgresSQL in this case. The PostgresSQL simply has a table with a number of votes for each category, cats and dogs. Finally the result of the vote is displayed in a web interface which is another web application developed in NodeJS. This resulting application rates the count of votes from the PostgresSQL database and display's it to the user.

### Steps
* Open your web console. If you have setup the openshift cluster, running `oc get routes -n openshift-console` will give show the link to your web console. Login with your username and password and create a new project. I called mine `voting-application`
![result screenshot](./assets/web-console.png)

* Start by deploying the front-end application for the client that will be used for the vote. Since that is dependent on the Redis, we deploy the Redis service first. If you don't find a redis service in the service catalog, you can add one yourself from [openshift examples](https://github.com/openshift/origin/blob/master/examples/db-templates/redis-ephemeral-template.json) on Github. Copy its content and select `import YAML/JSON` option in the UI. Paste it in the pop-up area. Click create. Select save template and continue. Refresh the browser to see a new Redis service. It can now be selected to deploy a Redis service. Click it and follow the prompt. Leave all default values and specify a password for Redis, `redis_passsword`. That's the same password used in the voting app. Click create to deploy the Redis service.

* The next step is to deploy the front-end application. In the [voting app](https://github.com/richardgreg/example-voting-application) it is under the 'vote' folder. It is written in python so add a new python application to the project from the catalog. You cannot simply specify the url because the python project is nested in a folder called '/_vote_'. So select advanced options to specify a sub-directory within git. Provide the name, git url and set the context directory to the sub-directory where the project is, _i.e_ '/vote'. Leave all other settings to default and create the application. This will autimatically create a build and deplpyment configuration. Once it is finished, simply select the link to access the app externally.
![vote screenshot](./assets/vote.png)
However, casting the vote creates an error. That is because we haven't specified the password used while deploying Redis anywhere in the voting app. Go back to the voting app front-end deployment config and add an environment variable `REDIS_PASSWORD`. And specify a password for Redis `redis_password`. Now casting a vote should work.

* Now for the second part of the application. The results service is dependent on the PostgreSQL service so we deploy that first. It is available as a catalog by default that will be used. The database service name must be the same name used by the applications to connect to the PostgreSQL database. In this case it is `db`. So in the '_Database Service Name_' field enter `db`. '_PostgreSQL Connection Username_' is `postgres`. '_PostgreSQL Connection Password_' is `postgres`. '_PostgreSQL Database Name_' is simply `postgres`. All of these values are used for simplicity and are not proper credentials. You can adjust the _Volume Capacity_ and _Memory Limit_ to something little like `100Mi` since we're not storing much data. Click create to create the database service.

* Now to deploy the results-app. It is based on NodeJS. Add a NodeJS app from the catalog. As with the Python setup, select advanced options. Provide a name, the git repository url. The context is '_/result_' From the applications code in github, the app listens on port 4000. That can be overriden in the Build Configuration section of the advanced options. Add an environment variable called '_PORT_', with value 8080 so the app is available on port 8080. Also add the same environment variable and value to the Deployment Configuration. Then click create. The progress can be monitored in the build page. Once deployed the app is available on the route url. Click on the link to access the results app. Modifying the vote count in the voting app doesn't affect the results yet. That's because the workers have not been deployed yet.
![result screenshot](./assets/result.png)

* The worker service will be deplyed using the Docker build strategy as opposed to Source to Image (S2I) strategy that we've been using. Select the Apache HTTP Server from the catalog. Select advanced options and provide a name, git url and context, '_/worker_'. Leave the rest to default and add the app. Ignore the automated build. Select the build configuration of the service. Click on actions in the top-right. Select edit YAML. Change the _strategy_ from 'sourceStrategy' to 'dockerStrategy'. Delete the '_from_' section. Change '_type_' to Docker. Click on save. Start build. The build goes through and pushes a new Docker image. Once the image is deployed, you can access the logs to see the votes being processed. Now changing the vote has an immediate effect.

### Configuring Automated Builds
You can configure automated web builds by configuring build triggers.

`oc describe buildConfig app-name` to get the GitHub webhook.
Or on the webconsole, navigate to builds. Select the app. And under configurations you'll
see the payload urls. Copy it. Go to projects repo and in the settings tab, select 'Webhooks'.
Paste the payload url. Change 'Content type' to `application/json`. Ignore 'secret' Disable SSL if you're in development. Add Webhook. Confirm that it has been successful as per the green tick. Now pushing changes to master branch will triger a build

### Todo
* Get Node app to communicate with Redis
