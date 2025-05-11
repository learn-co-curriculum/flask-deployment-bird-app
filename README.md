# Technical Lesson: Deploying a Flask-React App to Render

## Introduction 

In the previous lesson, we deployed a small Flask API application to Render to learn how the deployment process works in general and what steps are required to take the code from our machine and get it to run on a server.

In this lesson, we'll be tackling a more complex app with a modern Flask-React stack and exploring some of the challenges of getting these two apps to run together on a single server.

## Tools and Resources 
- [GitHub Repo](https://github.com/learn-co-curriculum/flask-deployment-bird-app)

## Set Up

To follow along with this lesson, we have a pre-built Flask-React application that you'll be deploying to Render. To start, fork this GitHub repo.

After downloading the code, set up the repository locally:

```bash
$ npm install --prefix client
$ pipenv install && pipenv shell
```

Create a .env file at the root and add the following variable:

```
DATABASE_URI=postgresql://{retrieve this from from render}
```

We've installed a new package in this repository called python-dotenv. It allows us to set environment variables when we run our application using .env files. This is a nice midway point between setting invisible environment variables from the command line and writing hidden values into our code.

To generate these environment variables, we just need to run the following command at the beginning of the module:

```python
# server/app.py

from dotenv import load_dotenv
load_dotenv()
```

After this, we can import any of our .env variables with os.environ.get().

Run the following commands to install upgrade and seed our database:

```bash
$ cd server
$ flask db upgrade
$ python seed.py
```

This application has a RESTful Flask API, a React frontend using React Router for client-side routing, and PostgreSQL for the database.

You can now run the app locally with:

```bash
$ honcho start -f Procfile.dev
```

Spend some time familiarizing yourself with the code for the demo app before proceeding. We'll be walking through its setup and why certain choices were made through the course of this lesson.

## Instructions

### Task 1: Define the Problem

Deploying a single Flask app is relatively straightforward, but deploying a full-stack application that includes both a Flask API backend and a React frontend introduces new challenges. These two applications have different build processes and runtime requirements:

* The React app needs to be built into static files that can be served by a production server.
* The Flask app needs to be configured to serve both its API routes and the frontend files.
* The combined application must be deployed in such a way that:
    * Client-side routing still works (e.g., /birds/:id doesn’t break on refresh).
    * API endpoints (e.g., /api/birds) are accessible and don't conflict with frontend routes.
    * The build and deploy process can be automated so future updates require minimal manual work.

The goal is to unify the deployment of both apps into a single service on Render, while maintaining a modular structure and ensuring performance, routing, and database connectivity.

### Task 2: Determine the Design

To successfully deploy a full-stack Flask-React app to Render, we need a process that:

* Backend Setup
    * Uses Flask as the web server to serve both API endpoints and React static files.
    * Adjusts the Flask app’s configuration to:
    * Serve static files from the /client/build directory.
    * Route unhandled URLs (404s) to the React index.html.

* Frontend Setup
    * Builds the React app into a static production bundle.
    * Serves the compiled frontend from the Flask app.

* Render Build Configuration
    * The Build Command must:
        * Install Python dependencies
        * Install React dependencies
        * Build the React app

* The Start Command should run:
```bash
gunicorn --chdir server app:app
```

This tells Render to use gunicorn as the production-ready web server and point it to the Flask app.

* Environment Variables
    * Must include at minimum:
        * DATABASE_URI to connect to PostgreSQL
        * PYTHON_VERSION to specify the runtime

* Deployment Goals
    * After deployment, the app should:
        * Serve the frontend at /
        * Serve Flask API routes under /api/
        * Support frontend routing (e.g., page refreshes)
        * Automatically rebuild and redeploy when pushed to GitHub

### Task 3: Develop, Test, and Refine the Code

#### Step 1: Building a Static React App

When developing the frontend of a site using Create React App, our ultimate goal is to create a static site consisting of pre-built HTML, JavaScript, and CSS files, which can be served by Flask when a user makes a request to the server to view our frontend. To demonstrate this process of building the production version of our React app and serving it from the Flask app, follow these steps.

1. Build the production version of our React app:

```bash
$ npm run build --prefix client
```

This command will generate a bundled and minified version of our React app in the client/build folder.

Check out the files in that directory, and in particular the JavaScript files. You'll notice they have very little resemblance to the files in your src directory! This is because of that bundling and minification process: taking the source code you wrote, along with any external JavaScript libraries your code depends on, and squishing it as small as possible.

2. Add static routes to Flask:

If you check app.py, you will see that the following additions have been made since you last saw the bird API:

```python
app = Flask(
    __name__,
    static_url_path='',
    static_folder='../client/build',
    template_folder='../client/build'
)

...

@app.errorhandler(404)
def not_found(e):
    return render_template("index.html")
```

These configure our Flask app for where to search for static and template files- both in our client/build/ directory.

We also set up a catch-all here for any route that doesn't match those already defined on the server. This means that when Flask receives a request, it will render the index.html that was generated to run the client application. The client still handles its routing through clicks and form submissions, but with this configuration, Flask can find the resources by URL as well. This is important for when people refresh the page, or visit your site, either manually, from bookmarks, or from an external link.

NOTE: Often, you may be setting up RESTful client-side routes, allowing people to go to /birds or /birds/:id to see all of the birds, or one at a time, respectively. These routes wouldn't be accessible on the frontend if they're already set up on the server (like they are in this app). To solve this, it's common to rewrite the backend routes so they all start with `/api/`, like `/api/birds` and `/api/birds/<int:id>` to free up the non-api urls to be used for client-side routing. Just remember to also update your fetches to match the backend urls.

3. Run the Flask server:

```bash
$ gunicorn --chdir server app:app
```

Visit http://localhost:8000 in the browser. You should see the production version of the React application!

Explore the React app in the browser using the React dev tools. What differences do you see between this version of the app and what you're used to when running in development mode?

Now you've seen how to build a production version of the React application locally, and some of the differences between this version and the development version you're more familiar with.

Now that you've seen how to create a production version of our React app locally and integrate it with Flask, let's talk about how to deploy the application to Render.

#### Step 2: Render Build Process

Think about the steps to build our React application locally. What did we have to do to build the React application in such a way that it could be served by our Flask application? Well, we had to:

1. Run npm install --prefix client to install any dependencies.
2. Use npm run build --prefix client to create the production app.
3. Install pip dependencies for the Flask app.
4. Run gunicorn --chdir server app:app to run the Flask server.

We would also need to repeat these steps any time we made any changes to the React code, i.e., to anything in the client folder. Ideally, we'd like to be able to automate those steps when we deploy this app to Render, so we can just push up new versions of our code to Render and deploy them like we were able to do in the previous lesson.

Thankfully, Render lets us do just that! Let's get started with the deployment process and talk through how this automation works.

1. Commit all of your work to your fork on GitHub and copy the project URL.

2. Navigate to your Render dashboard and create a new web service. Find your forked repository through "Connect a repository" or search by the copied URL under "Public Git repository."

3. Change your "Build Command" to the following:

```bash
$ pip install -r requirements.txt && npm install --prefix client && npm run build --prefix client
```

4. Change your "Start Command" to the following:

```bash
$ gunicorn --chdir server app:app
```
These commands will:
* Install any Python dependencies with pip.
* Install any Node dependencies with npm.
* Build your static site with npm.
* Run your Flask server.

5. Once you have saved these changes, navigate to the "Environment" tab and make sure the following values are set:

```
DATABASE_URI=postgresql://{retrieve this from from render}
PYTHON_VERSION=3.8.13
```

6. Click "Save Changes" and wait for a while.

Render's free tier can take up to an hour to get everything set up, so you might want to grab a bagel and coffee while you wait. 

7. Navigate to "Events" to check for progress and errors. 

When Render tells you the site is "Live", navigate to your site URL and view Birdsy in all its glory!

### Task 4: Document and Maintain

Best Practice documentation steps:
* Add comments to the code to explain purpose and logic, clarifying intent and functionality of your code to other developers.
* Update README text to reflect the functionality of the application following https://makeareadme.com. 
  * Add screenshot of completed work included in Markdown in README.
* Delete any stale branches on GitHub
* Remove unnecessary/commented out code
* If needed, update git ignore to remove sensitive data

## Considerations 

### Build Automation

Challenge: Building the React app manually every time is error-prone and inefficient.

Solution: Automate this step in Render’s Build Command:
```bash
pip install -r requirements.txt && npm install --prefix client && npm run build --prefix client
```

### Frontend/Backend Route Conflicts

Challenge: Flask and React might both try to handle routes like /birds.

Solution: Namespace backend routes (e.g., /api/birds) so the React router can use clean client-side paths like /birds/:id.

Pros:
* Clear separation of concerns
* Easier debugging
* Fewer unexpected route collisions
Con:
* Requires updating fetch URLs in the frontend

### Environment Management

Challenge: Hardcoding database URLs or other secrets is unsafe.

Solution: Use .env for local development (with python-dotenv) and Render’s Environment tab for production.

Best Practice:
* Do not push .env files to GitHub
* Reference variables in code using os.environ.get("DATABASE_URI")
