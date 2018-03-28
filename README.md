# HASS-Google-Cloud-SQL
**Guide on using Google Cloud SQL as a database recorder for Home-assistant.**

[Home-assistant](https://home-assistant.io/) allows you to record and monitor significant volumes of data from sensors within your home. This data is worth keeping as it is a nice resource for [doing data analysis](http://nbviewer.jupyter.org/github/home-assistant/home-assistant-notebooks/tree/master/), where generally speaking the more data the better.

By default [this data is recorded ](https://home-assistant.io/docs/backend/database/)in an SQLite database on the computer running Home-assistant. The computer is typically a raspberry pi with an SD card for memory. Over time the database can become large enough to fill a low capacity SD card. Also pi's are know to occasionally [corrupt their SD card](https://community.home-assistant.io/t/rpi3-third-time-file-corruption/15716), in which case some or all of your data may be lost. Finally if you wish to access the data in a SQLite database you need [access](http://nbviewer.jupyter.org/github/home-assistant/home-assistant-notebooks/blob/master/database-examples.ipynb) to the .db file, which requires you to copy the .db file from the pi and onto the computer (your laptop) where you will do your data analysis.

A solution to this final problem of access to the .db is to setup a database server which you can connect to over the network. Hassio [offers MariaDB server](https://home-assistant.io/addons/mariadb/) for example, although in this case the database is still on the pi (with associated risks). Alternatively if you have access to network storage (NAS) you may wish to setup a database server. This is what I did, running [a MySQL server in Docker on my Synology NAS](https://community.home-assistant.io/t/setting-up-mysql-on-a-synology-nas-docker-container/16253). However if we are serious about long term data storage we should follow the [3-2-1 rule](https://www.acronis.com/en-us/blog/posts/3-2-1-simple-rule-complex-data-protection) and have a backup off-site. Indeed I lost several months worth of data from my MySQL database after a problem with Docker. Therefore I investigated cloud storage solutions, which additionally may provide better performance (read times from the db) than my own locally hosted solution. There are several companies offering databases in the cloud. However I already have an account with Google so decided to investigate their offering.

### Google Cloud SQL ###
Google provide an [interesting variety](https://cloud.google.com/products/) of cloud software solutions/products. Their [Cloud SQL](https://cloud.google.com/sql/) product appeared to meet my requirements, and is [reasonably priced](https://cloud.google.com/sql/pricing) (particularly compared to purchasing your own NAS!). Once configured, a Cloud SQL database can be accessed without fuss, and in my limited experience appears to offer faster query/response times than my own local solution.

### Configuring Cloud SQL ###
Both MySQL and PostgreSQL are offered, and I chose PostgreSQL since I found a [useful guide](https://github.com/naranjja/gcp-jupyter-sql) on Github (Googles own docs being horrendously confusing). Follow the [getting started](https://cloud.google.com/sql/docs/postgres/quickstart) exercise, which involves creating an instance on the cloud, and using the web terminal to create a database. Once you've been through that tutorial, navigate to the [SQL instances console](https://console.cloud.google.com/projectselector/sql/instances) and create a new instance. I created an instance with identification 'hass-2' and the following instance configuration which should be [free/lowest cost](http://diyfuturism.com/index.php/2017/12/11/self-hosting-how-to-get-free-and-cheap-linux-virtual-servers/):

<p align="center">
<img src="https://github.com/robmarkcole/HASS-Google-Cloud-SQL/blob/master/images/instance-config.png" width="500">
</p>

The console for this instance is shown below.

<img src="https://github.com/robmarkcole/HASS-Google-Cloud-SQL/blob/master/images/cloud_config.png">

I've hidden the IP address of my instance (required later) and put a red box around the tabs I will discuss here.

* **USERS :** here you add users that can access the database, I created a user called **hass**.
* **DATABASES :** I created one called **ha_db**
* **AUTHORISATION :** It is important that you add the IP address which you will connect from, i.e. your home IP. Just google 'whats my ip' and add this address here.

The other tabs cover SSL, automated backups, cloning the database (REPLICAS), and a log of activity (OPERATIONS). I wont cover those here, but as long as you setup USERS, DATABASES and AUTHORISATION then you are good to go using the cloud database instance with Home-assistant.

### Configuring Home-assistant ###
The Home-assistant docs cover the use of external databases as [recorders](https://home-assistant.io/components/recorder/). The minimum that is required to setup a recorder is the URL to the database. I used a little python script to generate the URL, shown below.

```
settings = {
   'user': 'hass',
   'pass': 'password',
   'host': '192.100.123.100',
     'db': 'ha_db'
}

url = 'postgresql+psycopg2://{user}:{pass}@{host}:5432/{db}'.format(**settings)
```
Here **password** is whatever you set for the hass user, and **host** us the IP address of the SQL server (greyed out earlier).
Settings is just a python dictionary and with the values shown the url is:

 **postgresql+psycopg2://hass:password@192.100.123.100:5432/ha_db**

Note that **5432** is the [default port](https://www.postgresql.org/docs/8.3/static/app-postgres.html) but you can change this if you wish. Now you need to enter the url into the [recorder component](https://home-assistant.io/components/recorder/) config. To configuration.yaml I've added:

```
recorder:
  db_url: postgresql+psycopg2://hass:password@192.100.123.100:5432/ha
```
Note that [the Home-assistant docs state](https://home-assistant.io/components/recorder/#postgresql) that when using a postgreSQL database with the recorder component, you may need to install the [psycopg2](https://pypi.python.org/pypi/psycopg2/2.7.3.2) package on the host machine (your pi). I didn't need to do this, but you may. Restart Home-assistant and check your logs for any errors. All being well, Home-assistant should now be ingesting data into your Google Cloud SQL instance. You can see this happening on the instance dashboard, with the graph indicating that data is linearly being added to the database.

### Data access ###
The advantage of using a cloud SQL instance is that we can access if from any computer with with internet access with minimal fuss. I've included a Jupyter notebook demonstrating how to access the cloud database using [SQLAlchemy](https://www.sqlalchemy.org/) and [Pandas](https://pandas.pydata.org/). There are also [several examples](http://nbviewer.jupyter.org/github/home-assistant/home-assistant-notebooks/tree/master/) on the Home-assistant repo to checkout.  

### Summary ###
We have seen how to setup a postgreSQL database on Google Cloud and use the Home-assistant recorder component to record data in the database. This allows low cost - long term storage of your data, and enables fuss free access to do data analysis using standard tools.
