

.. |stream| raw:: html

   <i class='fas fa-stream'></i>

.. |sort_up| raw:: html

   <i class='fas fa-sort-alpha-up'></i>

.. |color_palette| raw:: html

   <i class='fas fa-palette'></i>

.. |save| raw:: html

   <i class='fas fa-save'></i>

.. |camera| raw:: html

   <i class='fas fa-camera-retro'></i>

.. |cog| raw:: html

   <i class='fas fa-cog'></i>

.. |filter| raw:: html

   <i class='fas fa-filter'></i>

.. |table| raw:: html

   <i class='fas fa-table'></i>

.. |tags| raw:: html

   <i class='fas fa-tags'></i>

.. |images| raw:: html

   <i class='far fa-images'></i>


MLV Developer Documentation
############################



Installing the Application
==============================


Clone from Git
---------------

.. code-block:: python

    git clone https://github.com/Hughes-Genome-Group/mlv.git


Setting up a Python Environment
--------------------------------
MLV requires python 3.6 or later. One way to manage the python environment is to use use virtualenv and virtualenvwrapper.

.. code-block:: python

   pip install -U virtualenv
   pip install -U virtualenvwrapper

Then add the following to your .bashrc (paths may vary)

.. code-block:: python

    export WORKON_HOME=$HOME/envs
    export VIRTUALENVWRAPPER_PYHTON=/usr/bin/python3.6
    . /usr/local/bin/virtualenvwrapper.sh

Next create a virtual environment and install all the required python modules:-

.. code-block:: python

    mkvirtualenv mlv
    workon mlv
    pip install -r requirements.txt



Installing Dependencies
-----------------------

The following  programs need to installed and avialable in the path:-

* tabix and bgzip (https://github.com/samtools/htslib)
* rabbitmq-server (https://www.rabbitmq.com/download.html)
* bedtools (https://github.com/arq5x/bedtools2/releases)
* bedToBigBed,bigBedToBed and bigWigInfo (http://hgdownload.cse.ucsc.edu/admin/exe/)
* nodejs (https://nodejs.org/en/download/)
* nodejs modules najax, jquery,xhr2,extend and canvas
* NGINX (https://www.nginx.com/) or another http server (for production)
* PostgreSQL 9.5 or above (https://www.postgresql.org/) 





Creating the Database
---------------------------------
MLV requires PostgreSQL 9.5 or above, which can be runnning on the same or a separate server. The first thing to do  
is to create the system database and associated tables by running *create_system_db.sql* in *app/dyatabases/*. To do this via the
the psql cosole, log in as a user with the correct permissions and run the following:-

.. code-block:: python

   createdb mlv_user; 
   \c mlv_user;
   \i /path/to/app/databases/create_system_db.sql;

If your PostgreSQL instance does not have a suitable user, you need to create one and grant access to tables generated in the previous step.

.. code-block:: python

    CREATE USER mlv WITH PASSWORD 'pword'; 
    GRANT SELECT, INSERT, UPDATE, DELETE ON ALL  TABLES IN SCHEMA public TO mlv;
    GRANT SELECT UPDATE, DELETE ON ALL SEQUENCES IN SCHEMA public TO mlv;

You also need to change the DB parameters in settings.py (or a custom config)

.. code-block:: python

    DB_USER="mlv"
    SYSTEM_DATABASE="mlv_user"
    DB_HOST="localhost" #or host name

Make sure you allow the user to connect to the database in PostgreSQL's *hb_config*:-

.. code-block:: python

    #allow connection from another host
    host   mlv_user,generic_genome   mlv    x.x.x.x/32 md5
    #allow local connection
    local  mlv_user,generic_genom mlv   md5 

If the database is hosted on a different server, you may have to update the firewall settings on this server to allow the mlv server to connect to it on port 5432 

Creating The First Users
++++++++++++++++++++++++++++

To create a user you will need to run the appropriate script (see `Executing Scripts`_) In order to do this, 
the app needs to know the location of the scripts module and also the password to the database.
These can be stored in a secure file and added to the environment variables:-

.. code-block:: python

    export FLASK_APP=/path/to/install/app/commands/cli_commands.py
    export DB_PASS=pword


Then you can add the guest user and an admin:-

.. code-block:: guess

    flask find_or_create_user --firstname John --last_name Doe --email guest@somewhere.com
    flask find_or_create_user --first_name Mr --last_name admin --email me@gmailcom \
                              --password password --admin True


Adding Genomes
+++++++++++++++++
Genomes are housed in separate databases to the system database. A single databases can hold many genomes
or separate databases can be created for each genome (not recommended). A fallback genome called 'other' initially needs to be 
added. The following code creates a database 'generic_genome' and then adds the fallback 'other' and the C. Elegains(cel1) genome:-

.. code-block:: python

    flask create_new_genome_database -db_name generic_genome
    flask add_new_genome --name other --label Other --database generic_genome
    flask add_new_genome --name ce11 --label C. Elegans(ce11) --database generic_genome




Running and Serving the Application
--------------------------------------
For testing porposes the application can be run using the *run_app* script.

.. code-block:: python

    flask run_app --port 5000

This will run the application locally on port 5000. For production purposes it should be run  through a web server gateway interface
such a Gunicorn. For example, the following code will run the app locally on port 5000 using 3 threads.

.. code-block:: python

    /path/to/virtualenv/gunicorn "app:create_app('lanceotron_config')"\
    -b 127.0.0.1:5000  \
    --workers 3 \
    --error-logfile /path/to/gunicorn.log


MLV should also be run through a webserver such as Nginx or Apache. Although static files such as images and tracks can be served
through the app via flask, this is not recommended in a production setting. It is more efficient to serve such files from the webserver.
An Nginx config to allow this is given below:-

.. code-block:: javascript

    #redirect to the flask app
    location / {
        proxy_pass http://localhost:5000;
        proxy_read_timeout 180s;
        proxy_set_header   Host                 $host;
        proxy_set_header   X-Real-IP            $remote_addr;
        proxy_set_header   X-Forwarded-For      $proxy_add_x_forwarded_for;
        proxy_set_header   X-Forwarded-Proto    $scheme;


    }
    #directly serve any static files
    location /static/ {
           alias /home/sergeant/mlv_dev/app/static/;
    }
    #temporary images
    location /data/temp/ {
            alias /data/mlv/temp/;
    }
    #static files belonging to a module
    location ~ /(.*)/static/(.*) {
                 alias /home/sergeant/mlv_dev/app/modules/$1/static/$2;
    }
    #genome tracks
    location /tracks/ {
        alias /path/to/tracks;
    }
    #any images belonging to projects
    location ~ /data/(.+\.(?:jpg|jpeg|gif|png))$  {
        alias /data/mlv/$1;
    }


Running the Message Queue
--------------------------------------
MLV uses celery and rabbit-mq to queue time consuming tasks and run simple pipelines. Two queues need to initialised,
a default queue for jobs whuch should return quickly and are required for the user to continue and a 'slow queue' 
for longer tasks and pipelines. To start the queues use the following commands:-

.. code-block:: python

    /path/to/venv/bin/flask runcelery 
    /path/to/venv/bin/flask runcelery --queue slow_queue

The default number of threads is 3, although this can be changed using the *--threads parameter* .
For debugging purposes celery can be disabled by changing the config setting *USE_CELERY=False*, which will
cause jobs to run directly in the flask thread and hence would be impractible for a production environment.


Using Supervisord 
--------------------------------------
Supervisor (http://supervisord.org/) can be used to populate environment variables, and run the server and job queues
as daemon threads. An example of a suitable config would be:-

.. code-block:: guess

    [unix_http_server]
    file=/path/to/supervisor.sock   ; (the path to the socket file)

    [supervisord]
    logfile=/path/to/supervisord.log  ; (main log file;default $CWD/supervisord.log)
    user=username                     ; (default is current user, required if root)
    directory=/path/to/root_dir       ; (default is not to cd during start)
    environment=FLASK_APP="/path/to/root_dir/app/commands/cli_commands.py",\
            FLASK_CONFIG="my_config",\
            DATABASE_PASS="pword"

    [supervisorctl]
    serverurl=unix:///var/run/supervisor/supervisor.sock ; use a unix:// URL  for a unix socket

    [program:mlv]
    command=/path/to/venv/bin/gunicorn "app:create_app('lanceotron_config')"\
                                        -b 127.0.0.1:5000 \
                                        --workers 3 \
                                        --error-logfile /path/to/gunicorn.log
    autostart=true
    autorestart=true

    [program:celery]
    command=/path/to/venv/bin/flask runcelery
    autostart=true
    autorestart=true

    [program:celery_slow]
    command=/path/to/venv/bin/flask runcelery --queue slow_queue
    autostart=true
    autorestart=true


Config Settings
==================
The main config is *settings.py* in the main app directory. Another config, which can add or override variables in *settings.py*
can be specified either in the *create_app* method or in the environment variable FLASK_CONFIG. The following shows the config
variables which may need to be changed.

Database Settings
------------------

* **DB_HOST** The database host, either a server name or an IP address. By default, the environment variable DB_HOST will be used or localhost if it is not set.
* **DB_USER** The name of the database user, mlv by default.
* **SYSTEM_DATABASE** The name of the system/user database, mlv_user by default.
* **DB_PASS** The database password, set to the environment variable DATABASE_PASS by default (passwords shouldn't be stored in the config)


Folder Locations
------------------

* **DATA_FOLDER** The location to store all the data, can be a mapped drive
* **TEMP_FOLDER** The location to store temporary data. Data here can be deleted
* **TRACKS_FOLDER** The location to store and serve genome tracks (BigWigs,BigBed files etc)


Message Queue (Celery) Settings
--------------------------------
* **BROKER_URL** The message queue url. By default is the localhost but potentially it could be a different server
* **USE_CELERY** The default is True, only set to False when debugging


App Settings
--------------
* **HOME_PAGE** the url of the home page of the application
* **HOST_NAME** the name of the machine hosting the app
* **APPLICATION_NAME** The name of the application
* **APPLICATION_LOGO** The url of the application logo
* **MODULES** A list of modules to load. ["multi_locus_view"] by default

Email Settings
---------------
* **MAIL_SERVER** The email server, smtp.gmail.com by default
* **MAIL_PORT**  The email server port, 587 by default
* **MAIL_USE_SSL**  False by default
* **MAIL_USE_TLS**  True by default
* **MAIL_USERNAME**  The mail username . The dummy entry mlv@gmail.com is the default.
* **MAIL_PASSWORD**  password by default
* **HELP_EMAIL_RECIPIENTS** A list of email addresees to which user questions are sent (an empty list bt default)
* **MAIL_DEFAULT_SENDER** The name to attach to emails that are sent out. 'The MLV Team' by default.


Misc. Setting
----------------
* **SECRET_KEY** Enables secure password hashing, should be sent to large random string
* **JS_VERSION** Should be changed each time a new version is rolled out as this will cause existing js and css caches on the user's computers to be refreshed.



Executing Scripts
==============================

To run any script requires the environment variable FLASK_APP to point to the module *cli_commands.py*.
Other environment variables that may be required include FLASK_CONFIG, which points to a custom config
and DATABASE_PASS, which holds the database password.

.. code-block:: guess

    export FLASK_APP=/path/to/install/app/commands/cli_commands.py
    export FLASK_CONFIG=linux_test_config
    export DATABASE_PASS="pword"


Writing scripts
-----------------

To write a script, simply import app from *cli_commands.py* and wrap your code in the app context.

.. code-block:: python

    from app.commands.cli_commands import app
    from app.jobs.jobs import get_job
    from appp.ngs.project import get_project

    with app.app_context():
        j=get_job(1234)
        j.process()
        p=get_project(4321)
        p.delete(True)


Built in Scripts
-------------------

There are a number of utility scripts in *cli_commands.py* that can be run with

.. code-block:: python

    flask name_of_script --param value


create_new_genome_database
+++++++++++++++++++++++++++
creates a new empty genome database.

* *- -db_name* - The name of the database


add_new_genome
+++++++++++++++++++++++++++
Adds a genome to the specified database. If the name matches a public genome in the UCSC genome browser, the RefSeq genes and 
chromosome file will automatically be added. Oterwise, the chromosome file (tab delimited chromosome to length) can be added manually to
*data_root/<genome_name>/<genome_name>/chrom.sizes*. 

* *- -name* - The name of the database (required)
* *- -label* - The label e.g. Human(hg19) (required)
* *- -icon* - The url of an icon to represent the database (24px x 24px). If not supplied a default icon will be used
* *- -database* - The name of the database to store the genome in (required)
* *- -connections* - The numner of connections optional - default 5


run_app
+++++++++++++++++++++++++++
Runs the app on the local host.

* *- -port* - The port


runcelery
+++++++++++++++++++++++++++
Runs the message queue

* *- -queue* - The name of the queue . the default is celery, the other oprion is slow_queue
* *- -threads* - The number of threads to give the queue - optional (default is 3)


remove_deleted_projects
+++++++++++++++++++++++++++
Removes all projects (and associated jobs) that are tagged as deleted. All data associated with the project is 
permanantly deleted.


check_all_jobs
+++++++++++++++++++++++++++
Calls *check_status* on all running jobs. This is not required for jobs in the local queue, only those running
on remote servers that have their *check_process* method overwritten. In which case this script should be run at
frequent time intervals e.g. in crontab


find_or_create_user
+++++++++++++++++++++++++++
Manually adds a user to the database

* *- -first_name* - The user's first name (required)
* *- -last_name* - The user's last name(required)
* *- -email* - The user's email(required)
* *- -password* - The user's password (required)
* *- -admin* - If True,true or TRUE,the user will habe admin rights (default False)


Modules
==================

Modules are a way of creating independent applications with discrete templates (html), static files (js,css and images) and python modules.
A module can be added to a system by simply adding the module folder to the *app/modules* directory and 
then adding  the name of the folder(module) to the MODULES list in the app's config.

Folder Structure
-----------------

.. code-block:: guess

    app
    |--modules
       |--module_name
          |--jobs
          |--projects
          |--static
          |--templates
          |--__init__.py
          |--config.json


Templates
----------------
The templates folder should contain subfolders named after each project in the module. Each subfolder should contain 
*home.html* (see `Project Home Page`_) as well as any other templates required by the project. Templates are referenced
in the normal way. e.g the  file *template.html* in the subfolder *project1* of the *templates* directory


.. code-block:: guess

    /app/modules/<module_name>/templates/project1/template.html

would be referenced in the view method as:-

.. code-block:: python

    get_template(self,args):
        return "project1/template.html"



Static Files
----------------
Any static files (js,css,images) go in the static subfolder of the module and can be referenced from template files
by prefixing the module name before static e.g. the js file 

.. code-block:: guess

    /app/modules/static/myjsfile.js 

would be accessed by:-

.. code-block:: guess

    <script src="/<module_name>/static/myjsfile.js?version={{config['JS_VERSION']}}"></script>


Projects
----------------
Any projects need to be specified in a dictionary in the projects list of the module's config
see  `App Settings`_. This will ensure the project is imported and registered when the app is initialised.
The actual project code needs to be in a python file named after the project in the module's project folder -
see `Project Class`_.

Jobs
-----------------
Jobs need to be specified in the 'jobs' list of the module's config. The job's code should then be 
in a module named after the job in the module's jobs folder - see `Jobs`.

Config
---------
The config should have three keys:- jobs, projects and config. The jobs and projects specify the jobs and projects that the
module contains and the config will update the app's config with any extra variables required. For example:-

.. code-block:: guess

    "jobs":[
        {"name":"peak_search_job"}
    ],
    "projects":[
        {
            "name":"peak_search",
            "label":"Peak Search",
            "large_icon":"/lanceotron/static/img/peak_search.png",
            "can_create":true,
            "is_public":true,
            "main_project":true,
         }
    ],
    "config":{
    
    "NEW_APP_PARAMETER":"value"
    }



Projects
==================

A Project represents an analysis or pipeline and is represented by a JSON config, a python class,
html (Jinja) templates and JavaScript files. The metadata for each project is kept in the 'projects' table of
the system database.

Config
---------
Each project needs to described by an entry in the project's list of a module's config.
The config should contain the following:-

* **name** - The name of the project type (that will be stored in the database as type)
* **label** - The name shown to user.
* **large_icon** -  The url of the icon which is displayed in the panels on the main page.
* **can_create** If True then this type of project can be directly created from the home page.
* **description** A short description which is displayed in the create panel on the main page.
* **is_public** If False, then the user must have the permission 'view_project_type' with the value of the project's type.
* **main_project** If True, then individual projects of this type will be accessible to view from the home page.
* **enter_genome** (optional) - If True then the genome can entered during the initial creation page, usually only name and description can be entered.
* **anonymous_create** (optional) If True, then projects of this type can be created by an anonymous user that is not logged in.


Users have to be given a specific permission to create a project If the project type is public,
new users will automatically get permission to create that project type.

HTML Templates
--------------------------

Project Home Page
+++++++++++++++++++
Projects that can be created directly will have the following url:

.. code-block:: guess

    http://<server_name>/projects/<project_type>/home


This page allows the user to enter a name, description and genome and then creates an empty project. The
actual template file should be located at 

.. code-block:: guess

    app/modules/<module_name>/templates/<project_name>/home.html

and contain the following html :-

.. code-block:: guess

    {% extends "projects/home_base.html" %}
    {% block project_explanation %}
       Information about the project here
    {% endblock %}

Individual Project Page
++++++++++++++++++++++++++++
A second url points to a specific instance of the project:-

.. code-block:: guess

    http://<server_name>/projects/<project_type>/<project_id>

The flask view behind this url checks the user has permission to view/edit the project and
calls the project's get_template method. The template is then rendered with the following kwargs 
(plus any extra returned by the get_template method)

* project_id
* project_type
* project_name
* project_description

The project's *get_template* method receives the args from the request and should return the location of the template
and a dictionary containing any key word arguments (in addition to the above) required by the template. Different
templates can be returned depending on the state of the project.

.. code-block:: python

    def get_template(self,args):
        kwgs={}
        template="<project_name>/<template_name>.html"
        #alter template and kwgs based on args and the project's current state
        return template,kwgs


The html templates for each project need to be located in the directory 

.. code-block:: guess

    app/modules/<module_name>/<project_name>/

with the following template:-

.. code-block:: guess

    {% extends "common/page_base.html" %}

    {% block stylesheets %}
        {{super() }}
        <!-- extra styles -->
    {% endblock %}

    {% block outercontent %}
        <!-- main content -->
    {% endblock %}

    {% block scripts %}
        {{ super() }}
        <!-- scripts -->  
    {% endblock %}


Project Class
---------------

Projects should inherit from the GenericObject in *app.ngs.project*, which supplies methods
amongst others for sharing, making public, deleting projects. In addidion the *projects* member of *app.ngs.project*
should be updated with the class name


To expose a project's method to an HTML Client use the static member 'methods', which is a dictionary with
the exposed method as the key and a dictionary containing the following parameters:-

* **permission**  required, either view or edit
    * view - allow all users with view permission to access the method. This will include all users if the object is public
    * edit - allow only users with edit permission to access the method
* **async**  optional, either True or False (False by default)
    * False - The method will be run in the browser thread
    * True - The method will be processed asynchronously
* **running_flag** optional, The projects's data will have this parameter set to the supplied value immediately before the method is sent to any queue. Should be a list containing the parameter and value to set

As an example a project *new_project* with a single method *get_data* that can be accessed by all users
(including anonymous ones) would be written as follows:-


.. code-block:: python
    
    from app.ngs.project import GenericObject,projects

    class CaptureCompareProject(GenericObject):
        def get_template(self,args):
            return "new_project/temp.html",kwgs
        def get_data(self,param1="default",param2="default):
		 #get the data using params
            return data

    projects["new_project"]= NewProject
   
    NewProjects.methods=
        {
            "filter_peaks":
                {
                   "permission":"view",
                   "running_flag":["filter_status","filtering"]
                 }
        }


The method can then be called from JavaScript using:-

.. code-block:: guess


    $.ajax({		
        url:"/meths/execute_project_action/<project_id>",
        type:"POST",
        dataType:"json",
        contentType:"application/json",
        data:JSON.stringify({
            method:"get_data",
            args:{
                param1:"value1",
                param2:"value2"
            }
        })
    })

The 'args' parameter contains the key word arguments sent to the project's method.
However, if 'project_data' is present in the args then this will be used to update the project's 
data immediately and not passed to the method. This is useful if the method is beng run asynchronously
with the async flag.


Jobs
==========================

Jobs run tasks asynchronously, either locally through celery or remotely on another server. 
Local jobs should extend *LocalJob* from *app.jobs.jobs*, whilst remote jobs should extend *BaseJob*

Constructor
-------------
To construct a job, user_id, inputs (a dictionary of key/value pairs) and  genome should be 
specified although default values of 0, an empty dictionary and 'other' will be used. 
A type, however must be specified. Inputs should contain all the information required to send or resend the job.
Once a job is constructed in this way, it will be added to the database and can be retrieved in the future 
with *app.jobs.jobs.get_job(<job_id>)*.

Registering a job
--------------------
In a module, each job should be in separate python file in the jobs folder of the module with the name of the job 
type e.g. new_job.py and registered in the config:-

.. code-block:: guess

    {  .....,
       "jobs":[
          {"name":"new_job"}
       ],
       .....
    }


The job's type needs to be linked to its class using *job_types* from *app.jobs.jobs*.
For example new_job.py could contain:-

.. code-block:: python

    from app.jobs.jobs import BaseJob,job_types
    import traceback

    class NewJob(LocalJob):
        def __init__(self,job=None,inputs={},user_id=0,genome="other"):
            if (job):
                super().__init__(job=job)
            else:
                super().__init__(inputs=data,genome=genome,type="new_job",user_id=user_id)

        def process(self):
		try:
             #run the job
           except:
             self.failed(traceback.format_exc())


    job_types["type"] = JobClass


Methods to Override
----------------------

send(data)
+++++++++++
For local jobs this simply calls *process*  asynchronously in the celery thread.
For remote jobs, this method needs to be over-ridden in the subclass to call a pipeline
on a remote server add it to a remote queue etc. Usually no parameters need to be passed as 
they should all be stored in the database when the job is created.
However, data can be passed for example a password, which you would not want to store.

resend()
+++++++++++
The default implementation is just to call *send*. However it should be over-ridden to clean up any mess that
was created when the job was first sent.

check_status()
+++++++++++++++
The default is to return the job's status (the field in the underlying database).
This is sufficient for local jobs, as the status is updated by the celery thread.
For Remote jobs this method should just return the status if it is 'complete', 'new', 'processing' or 'failed'.
Otherwise (i.e. the job is still running) it should actually check on the status of the
remote script/pipeline (query the remote server, check the queue etc.) and if the job 
has failed or is complete, call *failed* or *process* respectively.

process()
++++++++++++
For local jobs this should be the meat of the job and do all the heavy lifting ,storing the results to the database.
For remote jobs this should take the results from the remote pipeline and process/store them in the database.
Exceptions should be caught and if catastrophic, call *failed* passing the description of the exception.

delete()
++++++++++++
This method just removes the job from the database. Sub-classes should over-ride this method and delete
any files or other resources before removing the database entry.

failed(msg)
+++++++++++++ 
sets the status and time finished, writing the error message to the database.
If there is more cleaning up required then this method should be over-ridden with the relevent code.

kill()
+++++++
The default implementation of this method is simply to set the job's status as failed and add a message to the outputs.
For remote jobs it should send a message to the server/queue to kill the job.
For local jobs - if possible the job'status should be checked periodically and processing stopped if
the status is 'failed'.


Utility Methods
-------------------
Job objects have the instance variable 'job', which is just an SQLAlchemy object referring to the database entry.
This can be manipulated directly or there are the following convenience methods:-

* **complete** Sets date finished and status fields in the database
* **has_permission(user)** Returns whether the user has permission for this job
* **set_input_parameter(param,value)** Sets the input parameter
* **set_output_parameter(param,value)** Sets the output parameter
* **get_input_parameter(param,value)** Gets the input parameter
* **get_output_parameter(param,value)** Gets the output parameter
* **get_user_name()** Gets the full name of the job's owner
* **set_status(self,value)** Sets the status in the database
* **get_info()** Returns a dictionary containing the job's inputs and outputs


















        







 










