##Building a Flask app with Docker and MySQL

In an effort to be more in line with the spirit of DevOps
and CI/CD, I decided to try and use Docker with one of my 
current projects. There are a fair amount of tutorials out on how to create a 
Docker container running MySQL or on how to use Docker to deploy a Flask
application. However, finding a quality tutorial that covers using Docker
to deploy a Flask application that has a MySQL backend was a little more challenging for me.
Miguel Grinberg has a section in his Flask mega-tutorial that covers both 
of these topics [here](https://blog.miguelgrinberg.com/post/the-flask-mega-tutorial-part-xix-deployment-on-docker-containers).
However, he doesn't cover creating a docker-compose.yml file for the project and the use of environment variables 
within the docker-compose file. Another issue I found is that since Flask is such a barebones framework, some of the 
tutorials I found were using a different project layout than I was used to.

So, since I finally have this working, I want to document what I had to do in order to more easily replicate this in 
the future. 

The first thing I had to do was to creat a schema.sql file of the database that I had been using.

    mysqldump -u root -p --no-data eplApi > schema.sql
    
Where eplApi was the name of the database that I was working with. When I did this on my Mac there was an issue with
mysqldump not being found on the command line. This can be resolved by going into /usr/local/msyql/bin and then running 
the mysqldump command. See [this](https://stackoverflow.com/questions/36893507/mysqldump-isnt-working-command-not-found)
for reference.

Once you have the schema.sql file saved it's time to set up the docker-compose file. 

    version: '2'
    services:
      web:
        build: .
        image: epl-api
        links:
          - db
        ports:
          - "5000:5000"
        volumes:
          - .:/app
        container_name: epl-api
      db:
        image: mysql:5.7
        env_file: .env
        ports:
          - 3306
        environment:
          MYSQL_ROOT_PASSWORD: root
          MYSQL_DATABASE: eplApi
          MYSQL_USER: ${epl_db_user}
          MYSQL_PASSWORD: ${epl_db_password}
        volumes:
          - ../db:/docker-entrypoint-initdb.d/:ro
          
From what I can understand of docker-compose each section in services is defining two different
docker containers, in this case _web_ and _db_. Then in the web section we put a link to the 
db service so that it can be accessed from our flask application running in that container.

In the db section I specified a env_file which contains the username and password for connecting
to the database in the docker container. This is the same .env file that I am using in my flask 
application which means that I am consistent in my use of login credentials to the db. The format
of the .env file should be as follows

    epl_db_user="someuser"
    epl_db_password="somepassword"

Make sure you have the docker-compose file saved in the same directory as your 
flask application. My current project layout is the following. Also note that I saved the schema.sql file in the db directory.


    .
    |-- db
    |   `-- schema.sql
    |-- public_api
    |   |-- __pycache__
    |   |   `-- app.cpython-35.pyc
    |   |-- app.py
    |   |-- docker-compose.yml
    |   |-- dockerfile
    |   |-- models.py
    |   |-- requirements.txt
    |   |-- useful_commands.sh
    |   `-- util.py
        


My understanding is that when you run the ```docker-compose up```, 
docker will copy the schema.sql file over to ```/docker-entrypoint-initdb.d/``` and that will
automatically create the database schema.

Within my app.py file there was one change I needed to make so that the flask app could read
from the MySQL database running in the docker container, instead of the db running on my local host.
In order to do that I changed 

    app.config['MYSQL_DATABASE_HOST'] = os.getenv('epl_db_host')

to be

    app.config['MYSQL_DATABASE_HOST'] = 'db'
    
For clarity my app.py file looks like this

    from flask import Flask, jsonify, make_response
    from flask_httpauth import HTTPBasicAuth
    from flaskext.mysql import MySQL
    from flask_restful import Api, Resource, reqparse
    from dotenv import load_dotenv
    import os
    
    #Loading environment variables
    load_dotenv()
    load_dotenv(dotenv_path='../.env')
    
    app = Flask(__name__)
    
    auth = HTTPBasicAuth()
    
    mysql = MySQL()
    
    app.config['MYSQL_DATABASE_USER'] = os.getenv('epl_db_user')
    app.config['MYSQL_DATABASE_PASSWORD'] = os.getenv('epl_db_password')
    app.config['MYSQL_DATABASE_DB'] = os.getenv('epl_db_name')
    app.config['MYSQL_DATABASE_HOST'] = 'db'
    
    mysql.init_app(app)
    
    
    
    @app.route('/api/v1.0/get_db_info')
    def test_db_conn():
        cursor = mysql.connect().cursor()
        cursor.execute("SELECT * from Players")
        data = cursor.fetchall()
        #Want to get the
        print(data[0][3])
        return make_response(jsonify({'success': 'It worked!'}), 200)
    
    
    if __name__=="__main__":
        #Need to set host='0.0.0.0' so that this app can run on a docker container
        app.run(debug=True, host='0.0.0.0')

And now, running ```docker-compose up``` should launch both of the docker containers. Once 
you see that the mysql container is ready to start accepting connections, you should be able to 
verify that everything is running through the use of curl

    curl http://0.0.0.0:5000/api/v1.0/get_db_info
    
Quite a bit of set up, but this did take me a few hours to figure out so I wanted to make sure 
that my time wasn't completely wasted.

    