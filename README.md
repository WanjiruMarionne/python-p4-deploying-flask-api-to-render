# Deploying a Flask API to Render

## Learning Goals

- Set up your local environment for deploying with Render.
- Deploy a basic Flask application to Render.

---

## Key Vocab

- **Deployment**: the processes that make an application available for its
  intended use. For web applications, this means moving the application to a
  platform that supports requests from the internet.
- **Developer Operations (DevOps)**: the practices and tools that improve a
  team's ability to develop and deploy applications quickly.
- **PostgreSQL**: an open-source relational database system that provides more
  SQL functionality than SQLite. Unlike SQLite, its data is stored on a server
  rather than in files.
- **Platform as a Service (PaaS)**: a development and deployment platform that
  exists on a wide range of servers with different functionality. PaaS solutions
  reduce maintenance time for a software development team, but can increase
  cost. Some PaaS solutions, such as Render, provide free tiers for small
  applications.

---

## Introduction

In this lesson, we'll be deploying a basic, standalone Flask API application to
Render. We'll give instructions to generate the application from scratch and
talk through the steps to get the code running on a Render server.

In coming lessons, we'll learn how to add more complexity to the application
with a React frontend. Since the setup for a Flask-React application is a bit
trickier, it'll be beneficial to see the setup for Flask alone first. Let's get
started!

---

## Environment Setup

To make sure you're able to deploy your application, you'll need to do the
following:

### Sign Up for a Render Account

You can sign up at for a free account at
[https://dashboard.render.com/][render dashboard]. We recommend signing up using
your GitHub account- this will streamline the process of connecting your
applications to Render later on. The instructions below assume you've done that.

Once you've completed the signup process, you will be taken to the Render
dashboard:

![Render dashboard](https://curriculum-content.s3.amazonaws.com/phase-4/deploying-rails-api/render-dashboard.png)

In order to connect Render to your GitHub account, you'll need to click the "New
Web Service" button in the "Web Services" box.

You'll then click the "Build and deploy from a Git repository" button.

![select build and deploy from git repo](https://curriculum-content.s3.amazonaws.com/6130/deploy_flask_app/build_deploy_render.png)

On the next page, you will see a GitHub heading on the right side and below that
a link labeled "Configure account". (If you didn't sign up using GitHub, it will
say "Connect account" instead.)

![Connect GitHub](https://curriculum-content.s3.amazonaws.com/phase-4/deploying-rails-api/configure-github.png)

Click that link; a modal will appear asking you for permission to install Render
on your GitHub account:

![Install Render](https://curriculum-content.s3.amazonaws.com/phase-4/deploying-rails-api/install-render.png)

Click "Install." You should then be taken back to the "Create a New Web Service"
page, which should now show a list of your GitHub repos. We won't create a web
service just yet so you are free to navigate back to the [Render dashboard][].

### Install PostgreSQL

Render requires that you use PostgreSQL for your database instead of SQLite.
PostgreSQL (or just Postgres for short) is an advanced database management
system with more features than SQLite. If you don't already have it installed,
you'll need to set it up.

#### PostgreSQL Installation for WSL

To install Postgres for WSL, run the following commands from your Ubuntu
terminal:

```console
$ sudo apt update
$ sudo apt install postgresql postgresql-contrib libpq-dev
```

Then confirm that Postgres was installed successfully:

```console
$ psql --version
```

Run this command to start the Postgres service:

```console
$ sudo service postgresql start
```

Finally, you'll also need to create a database user so that you are able to
connect to the database from Flask. First, check what your operating system
username is:

```console
$ whoami
```

If your username is "ian", for example, you'd need to create a Postgres user
with that same name. To do so, run this command to open the Postgres CLI:

```console
$ sudo -u postgres -i
```

From the Postgres CLI, run this command (replacing "ian" with your username):

```console
$ createuser -sr ian
```

Then enter `control + d` or type `logout` to exit.

[This guide][postgresql wsl] has more info on setting up Postgres on WSL if you
get stuck.

#### Postgresql Installation for OSX

To install Postgres for OSX, you can use Homebrew:

```console
$ brew install postgresql
```

Once Postgres has been installed, run this command to start the Postgres
service:

```console
$ brew services start postgresql
```

Check your Postgres version:

```console
$ psql --version
```

### Create the Database on Render

One limitation of Render is that it only allows one PostgreSQL instance to be
created per user account. With this instance, we can create an app, give it some
seed data, and deploy it, storing the data in the PostgreSQL instance's
database. But then what happens if you want to deploy additional apps to Render?
You can probably see how using a single database for multiple apps could get
complicated very quickly and potentially cause problems. Fortunately, Render
allows users to create [multiple databases within a single PostgreSQL
instance][multiple dbs] so you can have a separate database for each app you
deploy.

Let's start by creating the PostgreSQL instance, then we'll create the bird
database.

You need to check which version of Postgresql you have on your local machine.
Run `psql --version` in your terminal. The output should look something like
this, but with your version instead:

```console
$ psql --version
psql (PostgreSQL) 15.x
```

Go to the [Render dashboard][], click the "New +" button and select
"PostgreSQL". Enter a name for your database. This can be whatever you like —
we're using `my_database`.

![Creating a new database](https://curriculum-content.s3.amazonaws.com/phase-4/deploying-rails-api/create-database.png)

**Note: Hyphens may not be used in the database names; you should use
underscores instead (e.g., `my_db`, not `my-db`).**

Select the Postgresql version you have from the dropdown.

Render will randomly generate identifiers for the "Database" and "User" fields,
or you can create those yourself if you prefer.

For "Region", you can either select the location closest to you or you can use
the default selection. Make sure you use the same region when you are creating
the web service later in this lesson.

Scroll to the bottom of the page and click "Create Database". It may take a few
minutes to create the database. Leave the database page open — you'll need to
copy information from it as we proceed.

Next, let's create a database specifically for our bird app. We'll do this using
the PostgreSQL interactive terminal, [`psql`][psql].

The command to launch the interactive terminal is provided in the Render
database page. Scroll down to the "Connections" section and copy the PSQL
command.

![psql command link from render page](https://curriculum-content.s3.amazonaws.com/6130/deploy_flask_app/psql.png)

Paste it into your terminal window and press enter. This command connects you to
the remote database. You should now see the `psql` command prompt,
`my_database=>` (there may be some numbers appended to the database name).

```command
Type "help" for help.

my_database=>
```

If you type the `\l` command to list the databases, you'll see a table that
includes `my_database`.

```command
Type "help" for help.

my_database=> \l
```

You will need to hit the enter key to exit the database listing and return to
the PSQL prompt.

To create the database for our bird app, we'll run the `CREATE DATABASE` SQL
command. Again, you can name your database whatever you like; we're using
`bird_app_db`. Be sure to include the semi-colon at the end of the command:

```console
my_database=> CREATE DATABASE bird_app_db;
```

Now if you run the `\l` command again, you should see that `bird_app_db` has
been added to the list of databases.

You can now exit `psql` using the `\q` command.

> **Note**: The Render dashboard will not show the information about the
> `bird_app_db` database; it will only show the name you assigned when you
> created the PostgreSQL instance on Render (`my_database`). To see any other
> databases you have on your PostgreSQL instance, you'll need to use `psql`. For
> now, be sure to make a note of your new database's name as we'll need to use
> it in the next step.

---

## Creating a Flask App to Deploy

We'll be following the steps in Render's ["Deploy a Flask App"][render flask]
guide, so if you get stuck and are looking for more assistance, check that
guide.

The first thing we'll need to do is create our new Flask application. Make sure
you're in a non-lab directory, then run:

```console
$ mkdir bird-app && cd $_
$ pipenv install Flask gunicorn psycopg2-binary Flask-SQLAlchemy Flask-Migrate SQLAlchemy-Serializer Flask-RESTful
```

> \*\*NOTE: You may want to specify versions for these packages when you work on
> your Phase 4 and Phase 5 projects. This will prevent updates to these modules
> from breaking your code. You can check all versions with the command
> `pipenv requirements`.

This will set create an application directory and install some valuable
libraries for building a RESTful API. Let's also run the following command to
generate a `requirements.txt` file:

```console
$ pipenv requirements > requirements.txt
```

`requirements.txt` is very similar to a Pipfile- the primary difference here is
that instead of supporting a local virtual environment through pipenv, it
supports the creation of an application environment on a PaaS platform. (There
are some different tools that use this file as well.)

### Assigning DATABASE_URI

To test our Flask app locally, we need to assign the `DATABASE_URI` environment
variable to reference the bird database we created on Render. Return to the
[Render dashboard][] and click on the "my_database" link (or whatever you named
the Postgresql instance). Scroll down to `Connections` and copy the "External
Database URL".

![external database url](https://curriculum-content.s3.amazonaws.com/6130/deploy_flask_app/external_db.png)

You will assign this string to the `DATABASE_URI` environment variable, but you
need to make two modifications to the external database URL string:

![modifying database uri](https://curriculum-content.s3.amazonaws.com/6130/deploy_flask_app/database_uri.png)

- Modify the protocol at the beginning of the string to say `postgresql` instead
  of `postgres`.
- Modify the database name at the end of the string to say `bird_app_db` instead
  of `my_database`.

Do not modify any other part of the string. Assign the environment variable by
typing the following in a terminal:

```console
$ export DATABASE_URI=<Modified External Database URL goes here>
```

### Building the Demo App

Let's set up our app, models, and migrations to get things started. You can
create the files directly within the `bird-app` folder since we _won't_ be
needing a `server` sub-folder.

```py
# app.py

import os

from flask import Flask, jsonify, make_response
from flask_migrate import Migrate
from flask_restful import Api, Resource

from models import db, Bird

app = Flask(__name__)
app.config['SQLALCHEMY_DATABASE_URI'] = os.environ.get('DATABASE_URI')
app.config['SQLALCHEMY_TRACK_MODIFICATIONS'] = False
app.json.compact = False

migrate = Migrate(app, db)
db.init_app(app)

api = Api(app)

class Birds(Resource):

    def get(self):
        birds = [bird.to_dict() for bird in Bird.query.all()]
        return make_response(jsonify(birds), 200)

api.add_resource(Birds, '/birds')

```

```py
# models.py

from flask_sqlalchemy import SQLAlchemy
from sqlalchemy_serializer import SerializerMixin

db = SQLAlchemy()

class Bird(db.Model, SerializerMixin):
    __tablename__ = 'birds'

    id = db.Column(db.Integer, primary_key=True)
    name = db.Column(db.String)
    species = db.Column(db.String)

    def __repr__(self):
        return f'<Bird {self.name} | Species: {self.species}>'

```

Add this data to the `seed.py` file:

```py
# seed.py

from app import app
from models import db, Bird

with app.app_context():

    print('Deleting existing birds...')
    Bird.query.delete()

    print('Creating bird objects...')
    chickadee = Bird(name='Black-Capped Chickadee', species='Poecile Atricapillus')
    grackle = Bird(name='Grackle', species='Quiscalus Quiscula')
    starling = Bird(name='Common Starling', species='Sturnus Vulgaris')
    dove = Bird(name='Mourning Dove', species='Zenaida Macroura')

    print('Adding bird objects to transaction...')
    db.session.add_all([chickadee, grackle, starling, dove])

    print('Committing transaction...')
    db.session.commit()

    print('Complete.')

```

Then run these commands to generate the database and run the migrations and seed
file:

```console
$ flask db init
# => ...
$ flask db revision --autogenerate -m'create table birds'
# => ...
$ flask db upgrade
# => ...
$ python seed.py
# => Deleting existing birds...
# => Creating bird objects...
# => Adding bird objects to transaction...
# => Committing transaction...
# => Complete.
```

> `flask db upgrade` creates a new PostgreSQL database to be associated with
> your application based on the configuration on Render and in `models.py`.
> Unlike with SQLite, the actual database file isn't created in the
> application's folder; it lives on Render's servers and will not show up in
> your directory structure at all.

To make sure the app works locally before deploying, run `gunicorn app:app`.

```console
$ gunicorn app:app
```

Navigate to [http://localhost:8000/birds](http://localhost:8000/birds) and
confirm the app displays the bird data:

![screenprint of birds app on localhost](https://curriculum-content.s3.amazonaws.com/6130/deploy_flask_app/local_birds_app.png)

> **NOTE: gunicorn runs on port 8000 by default. Because this does not conflict
> with any system ports on MacOS, Windows, or Linux, we won't change it here.**

---

## Deploying

Now that we've got some working code, it's time to get that code to run on a
Render server! The process of uploading our code to Render is managed by Git.
This makes it easy to deploy new versions using a tool most developers,
including yourself, are already familiar with.

Make a commit to save your changes, then push them to a remote repo:

```console
$ git init
$ git add .
$ git commit -m 'Initial commit'
```

Next, you'll need to create a repository in GitHub:

![github create repo form. repo is set to public](https://curriculum-content.s3.amazonaws.com/python/python-p4-deployment-github-repo.png)

...and push your work to the new repo:

```console
$ git remote add <remote_name> <remote_repo_url>
$ git push -u <remote_name> <local_branch_name>
```

---

## Configuring the Environment and Deploying

Head back to [the Render dashboard][render dashboard] and click "New+" to make a
new Web Service. If you connected to GitHub when you signed up, you should see a
list of all your repos! Find your bird API and click "Connect".

Give your application a name. Render should be able to figure out the github
repo is for a Flask application and will fill in the Runtime, Build Command, and
Start Command. If not, configure the application as seen below. Try to specify
the same region as the database instance.

![configuration screen for a web service in render. the root directory is left
blank. environment is Python 3. Region is Oregon (US West). Branch is main. Build
command is "pip install -r requirements.txt". start command is "gunicorn
app:app"](https://curriculum-content.s3.amazonaws.com/6130/deploy_flask_app/new_web_service.png)

Before our first successful deployment, we need to click on the "Advanced"
button, then add two environment variables:

`PYTHON_VERSION` is `3.8.13`.

`DATABASE_URI` is the **Internal Database URL** from your PostgreSQL database
(not External Database URL). Don't forget to change the protocol to `postgresql`
and the database name to `bird_app_db`! It should look something like this:

```sh
postgresql://my_database_user:#################################################/bird_app_db
```

![change render environment variables](https://curriculum-content.s3.amazonaws.com/6130/deploy_flask_app/envir_vars.png)

NOTE: If your database instance and web service are in different regions, you
may need to use the External Database URL for the connection.

Scroll to the bottom of the page and click the "Create Web Service" button. It
may take a few minutes to build and deploy your application.

Your URL is located under your app name at the top of the screen.

Click on the link, navigate to `/birds`, and view your work in all its glory!

```json
[
  {
    "id": 1,
    "name": "Black-Capped Chickadee",
    "species": "Poecile Atricapillus"
  },
  {
    "id": 2,
    "name": "Grackle",
    "species": "Quiscalus Quiscula"
  },
  {
    "id": 3,
    "name": "Common Starling",
    "species": "Sturnus Vulgaris"
  },
  {
    "id": 4,
    "name": "Mourning Dove",
    "species": "Zenaida Macroura"
  }
]
```

---

## Adding New Features

Since Render integrates the deploying process with Git, it's straightforward to
add new features to your code and deploy them. Let's start by adding a new route
in `app.py`:

```py
class BirdByID(Resource):
    def get(self, id):
        bird = Bird.query.filter_by(id=id).first().to_dict()
        return make_response(jsonify(bird), 200)

api.add_resource(BirdByID, '/birds/<int:id>')

```

Test your code locally by running `gunicorn app:app` and visiting
[https://localhost:8000/birds/1](https://localhost:8000/birds/1).

After adding this code, make a commit:

```console
$ git add app.py
$ git commit -m 'Added get by ID route'
```

Then, to deploy the changes, push the new code up to GitHub:

```console
$ git push
```

After pushing new code, Render will run through the build process again and
deploy your changes. This may take a few minutes. Test to make sure the new
route works once the deployment is complete.

Note, you don't have to run your migrations again since the database already
exists on the server. You would have to run the migrations if you created a new
migration file.

---

## Notes on Render's Free Tier

### Free Web Services

According to Render's documentation on its
[Free Web Services](https://render.com/docs/free#free-web-services):

> Render spins down a Free web service that goes 15 minutes without receiving
> inbound traffic. Render spins the service back up whenever it next receives a
> request to process.

> Spinning up a service takes a few seconds, which causes a noticeable delay for
> incoming requests until the service is back up and running. For example, a
> browser page load will hang momentarily.

This means that, when you try to navigate to your app in the browser, it might
take a while to load.

### Free PostgreSQL Databases

With Render's free tier, databases expire after 90 days. This means that, before
the end of the 90 days, you will need to back up your databases, delete the
PostgreSQL instance from Render, create a new PostgreSQL instance, and populate
it from the database backups. Render should send an email warning you that your
database will be expiring soon.

We will go over the process for backing up and recreating your database — along
with some other tips for using databases with Render — in the next lesson.

## Conclusion

Congrats on deploying your first Flask app to the world wide web! Understanding
the deployment process and what it takes to run your application on another
computer is an important step toward becoming a full-stack developer. Like
anything new, this process can be daunting the first time you try it, but with
practice and exposure, you'll build confidence over time.

In the next lesson, we'll work on deploying a more complex application with a
Flask API backend and a React frontend, and talk through some of the challenges
of running these two applications together.

---

## Check For Understanding

Before you move on, make sure you can answer the following questions:

<details>
  <summary>
    <em>What familiar process is used for deploying code to Render?</code></em>
  </summary>

  <h3>Git</h3>
  <p>Render integrates natively with GitHub and GitLab. Your application needs
     to be added manually at first, but syncs automatically upon every new
     push.</p>
</details>

## Resources

- [Deploy a Flask App - Render][render flask]
- [PostgreSQL - Render](https://render.com/docs/databases)
- [Create a repo - GitHub](https://docs.github.com/en/get-started/quickstart/create-a-repo)

[render dashboard]: https://dashboard.render.com/
[postgresql wsl]:
  https://docs.microsoft.com/en-us/windows/wsl/tutorials/wsl-database#install-postgresql
[render flask]: https://render.com/docs/deploy-flask
[multiple dbs]:
  https://render.com/docs/databases#multiple-databases-in-a-single-postgresql-instance
[psql]: https://www.postgresql.org/docs/current/app-psql.html


Using Databases with Render
GitHub RepoCreate New Issue
Learning Goals
Review the process for creating a PostgreSQL instance on Render.
Use PSQL to execute database commands.
Create multiple databases in a PostgreSQL instance.
Use seed data with a database.
Back up and restore our apps' databases.
Reset an existing database.
Introduction
In the last lesson, we learned how to deploy a Flask API on Render, including a database. We touched on some issues with Render's free tier, and on using PSQL to run some of the database commands. In this lesson, we will review some of that material and go into greater detail about using databases with Render.

We recommend that as you go through this lesson, you pay attention to what you can do in Render and what you need to use PSQL to accomplish. In particular, PSQL will allow you to interact with any app-specific databases you add within your PostgreSQL instance. Your goal for this lesson should be to gain a basic understanding of the commands and utilities covered. You can return to this lesson and use it as a reference as needed moving forward.

Using PSQL to execute database commands
Render makes it pretty easy to accomplish certain database tasks, but there are a number of additional helpful tasks that can't be done through the browser interface. For these, we'll use the PostgreSQL interactive terminal, psqlLinks to an external site..

PSQL is a very helpful tool that allows us to connect to the remote database from our terminal and run certain "meta-commands"Links to an external site. that take the place of running a SQL query (e.g., list our databases; list the data tables in a database). It also includes some useful "utility" commands (e.g., backup a database; restore a database). Finally, we can run SQL commands directly from PSQL.

To get into PSQL, go to your PostgreSQL instance page in the Render dashboard, scroll down to the "Connections" section, and copy the "PSQL Command".

psql command link from render page

Alternatively, you can click "Connect" in the upper right corner, then click "External Connection" and copy the command from there.

Go to your terminal, paste in the command and press enter. You should now see the psql command prompt, my_database=>.

Note: As you're working with PSQL, be aware that if it's idle for a while, the connection will time out. If that happens, when you try to run a command, there will be a delay and then you'll see the following message:

could not receive data from server: Operation timed out
SSL SYSCALL error: Operation timed out
To reconnect, run the quit command (\q) then, from the terminal prompt, press the up arrow to access the PSQL command and hit enter.

Listing Databases
In the previous lesson, you created a PostgreSQL instance my_database to store all of your apps' databases. The free tier only allows you to create one database server instance, although you were able to create another database named bird_app_db within it by using the CREATE DATABASE command. Let's explore some other PSQL commands.

To list the databases on your PostgreSQL instance, run the \l meta-command. You should see something like this:

 \l
                                           List of databases
       Name       |         Owner         | Encoding |  Collate   |   Ctype    |   Access privileges
------------------+------------------+----------+------------+------------+-----------------------
 bird_app_db      | my_database_user | UTF8     | en_US.UTF8 | en_US.UTF8 |
 my_database      | my_database_user | UTF8     | en_US.UTF8 | en_US.UTF8 |
 postgres         | postgres              | UTF8     | en_US.UTF8 | en_US.UTF8 |
 template0        | postgres              | UTF8     | en_US.UTF8 | en_US.UTF8 | =c/postgres          +
                  |                       |          |            |            | postgres=CTc/postgres
 template1        | postgres              | UTF8     | en_US.UTF8 | en_US.UTF8 | =c/postgres          +
                  |                       |          |            |            | postgres=CTc/postgres
(5 rows)
Here you can see a list of databases, including some that were created automatically by Postgres (the ones with postgres as the owner), and some that were created by the user. The my_database database is the name of the PostgreSQL instance; you can tell because the associated user (my_database_user) is the owner of all of the user-created databases. The remaining user-created databases are associated with particular apps. (See the "Creating Multiple Databases within a PostgreSQL Instance" section below for instructions on how to create app-specific databases under the umbrella of your PostgreSQL instance.)

Listing Data Tables
To list the data tables in the currently selected database (in our case, my_db_instance), run:

\dt
Because we don't have any data tables associated with our PostgreSQL instance directly, we get the following:

Did not find any relations.
To see the tables associated with one of our apps, we first need to switch to the app's database.

Switching to a Different Database
To switch to the bird_app_db that we created in the last lesson, we'll type the command \c bird_app_db:

 \c bird_app_db
You should see something similar to:


SSL connection (protocol: TLSv1.3, cipher: TLS_AES_128_GCM_SHA256, bits: 128, compression: off)
You are now connected to database "bird_app_db" as user "my_database_user".

The c stands for connect. The PSQL command prompt will now show bird_app_db rather than my_database.

Now if we enter the command \dt again, we'll see something like this:

 \dt
                    List of relations
 Schema |      Name       | Type  |         Owner
--------+-----------------+-------+-----------------------
 public | alembic_version | table | my_database_user
 public | birds           | table | my_database_user
(2 rows)
Note that we can see our birds table in the second row.

Quitting PSQL
To exit PSQL and get back to your terminal prompt, run the \q command.

Using Seed Data with a Database
You may recall that in the last lesson, in order to deploy our app, we created the bulk of our data with a seed file. We ran python seed.py to seed the database. Whenever you add the seed data, just push up the changes to GitHub and re-deploy your app. Recall that seed scripts should begin with a deletion of existing data to avoid duplicating the data.

If you forget to do that, or if you need to start fresh for some other reason, you can reset the database. We'll learn how to do that a bit later in this lesson.

Backing Up and Recreating Your Databases on Render
We mentioned in the last lesson that, with Render's free tier, your PostgreSQL instance and any additional databases you've created on it will expire after 90 days. This means that, before the end of the 90 days, you will need to back up your databases, delete the PostgreSQL instance from Render, create a new PostgreSQL instance, and populate it from the database backups. Render should send an email warning you that your database will be expiring soon.

Before we launch into the instructions for backing up and recreating your Render PostgreSQL instance, let's take a closer look at the PSQL connection string. Go ahead and copy the "PSQL Command" from the Render dashboard then paste it into a new file in your text editor.

Warning: Be careful not to store any of the PSQL commands inside a project repo. Those commands contain secure information so you don't want them to be deployed to GitHub accidentally!

The connection string will look something like this:

PGPASSWORD=############# psql -h ################-postgres.render.com -U my_database_user my_database
The first element is the password for your database, which will be a 32-character string. Next is the command you're running, in this case, psql. The next component is the host (indicated by the -h flag), which will end with "-postgres.render.com". Next is the name of the database user (indicated by the -U flag), followed, finally, by the name of the database itself.

Note that the username and database name in the PSQL command above match the entry for the PostgreSQL instance in the list of databases we printed earlier in this lesson.

We discovered earlier that the main database in the PostgreSQL instance, my_database, does not contain any data. Therefore, we just need to back up the database we've created, bird_app_db.

Note: Technically, we don't need to back up our bird app either, because all the records come from our seed data — there is no way for users to add, change or delete records. This means all we need to do to restore the database is re-run the seed command. For purposes of illustration, however, we'll go ahead and go through the process of backing it up.

Backing Up Our Databases
To create a backup, we're going to modify the PSQL connection string to run PSQL's pg_dumpLinks to an external site. utility command. We'll do that once for each database we need to back up, starting with bird_app_db. We recommend making the edits to the string in your text editor then copy/pasting it into the terminal when you're done. (But remember not to push it up to Github!)

The first part of the string, the password, will remain the same. The psql command should be updated to pg_dump instead. The host and username should also stay the same. After that, we'll add the following options:

--format=custom --no-acl --no-owner
The final component of the original connection string is the name of the PostgreSQL instance, my_database. We'll replace that name with bird_app_db instead. After that, we'll add a > to indicate that we want the results of the command to be written to a file, followed by the name we want to use for the backup file, with the .sql extension:

bird_app_db > bird_app_db.sql
The updated string will look something like this:

PGPASSWORD=############# pg_dump -h ################-postgres.render.com -U my_database_user --format=custom --no-acl --no-owner bird_app_db > bird_app_db.sql
Now that we've created the command, we'll copy/paste it into the terminal (not PSQL) and run it. Make sure you're not inside a directory that's a git repo first to ensure the backup file doesn't get pushed to GitHub. The command will not print any output, but if we run ls, we'll see the newly-created .sql file in the current directory.

Now that we've backed up our database, the next step is to delete the current PostgreSQL instance and create a new one.

Replacing the Expiring PostgreSQL Instance
To delete the PostgreSQL instance, go to the database page on the Render dashboard. Scroll to the bottom of the page, then click "Delete Database" and follow the instructions.

Next, we'll create a new PostgreSQL instance by clicking the "New +" button and selecting PostgreSQL. Provide a name for the new instance (e.g., my_database), then scroll down to the bottom of the page, and click "Create Database."

You need to copy the PSQL connection string from this database instance, since it is different from the previous version.

Restoring the Databases to the New Instance
Once the new instance has been created, the next step is to create the app-specific databases within that instance.

First we need to execute the PSQL connection string for the new PostgreSQL instance to launch the interactive terminal. Next, we'll run the CREATE DATABASE commands for each of the databases (we're using the same database names but you can use different names if you prefer):

 CREATE DATABASE bird_app_db;
Once that's done, we can exit PSQL with the \q command.

Now we're ready to work on building the pg_restoreLinks to an external site. command. Once again, we recommend pasting the connection string into your text editor and editing it there.

The command will consist of the following:

The database password.
The pg_restore command.
The host.
The user.
The options: --verbose --clean --no-acl --no-owner.
The -d flag (for dbname) followed by the name of the new database you're restoring the data to.
The name of the .sql file you're restoring from.
The final string for the bird app will look something like this:

PGPASSWORD=################ pg_restore -h #################-postgres.render.com -U my_database_user --verbose --clean --no-acl --no-owner -d bird_app_db bird_app_db.sql
When we run the command in the terminal, we'll see a flurry of activity as it creates the database tables. You may see some error messages about dropping tables, which you can ignore.

Connecting the New Databases to Your Web Services
The final step in the process is to update each app's Web Service so that it points to the newly-restored database.

From the Render dashboard, we'll select the bird app, then click "Environment" in the nav on the left. Next, we delete the value associated with the DATABASE_URL key and replace it with the Internal URL for the new instance. Remember that we want to connect to the bird app database, not the instance itself, so you need to remove the name of the PostgreSQL instance from the end of the URL and replace it with the name of the bird app database. You also need to update the protocol to postgresql. It should look something like this:

postgresql://my_database_user:#################################################/bird_app_db
After we've saved the change, the web service will redeploy. When we click the app's URL for the /birds route, we should see the JSON for the list of birds.

Resetting an Existing Database
If you need to reset a database for some reason — say you have duplicate data — all you need to do is drop the database and recreate it.

There are two cases to consider. The first is if the database you need to reset is the PostgreSQL instance itself (i.e., if you only have a single app deployed). In this case, you would simply delete and recreate the instance in Render, then connect the new instance to the web service for the app and redeploy it.

If, on the other hand, you need to reset a database within your PostgreSQL instance and not the instance itself, you can do that using PSQL:

Execute the PSQL command for your PostgreSQL instance in the terminal.
Run the SQL command to drop the database: DROP DATABASE database_name;. You should see 'DROP DATABASE' echoed in the terminal. If you don't, make sure you included the semicolon and that your PSQL connection hasn't timed out.
Run the SQL command to create the new database: CREATE DATABASE database_name;
In Render, connect the web service for the app to the new database and redeploy. If you used the same name for the database, you'll just need to redeploy.
Optionally, with either approach, you can also re-seed your database if you choose.

That's it!

Conclusion
In this lesson, you learned some important database tasks that can be performed using the Render dashboard, and how to use PSQL to supplement those actions. In particular, familiarity with PSQL will be very helpful for you if you want to deploy multiple apps with databases to Render. You've also learned how to handle the situation when your Render PostgreSQL instance is approaching its expiration date.

In the next lesson, we'll work on deploying a more complex application with a Flask API backend and a React frontend, and talk through some of the challenges of running these two applications together.

Check For Understanding
Before you move on, make sure you can answer the following questions:

1. What database actions can be completed in the Render web app?


2. What database actions require the PSQL interactive terminal?


3. What issue do you need to be aware of when using seed data for your database? How can you address the situation?


Resources
Render Databases GuideLinks to an external site.
Multiple Databases In A Single PostgreSQL InstanceLinks to an external site.