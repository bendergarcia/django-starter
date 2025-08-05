# Overview
Starting django projects can be a hassle, especially if you don't want to deal with environments and packages. This is a document for simply using commands to initialize a django project. After that you can build your own container for running that django image. I'll document some of that here as well.

I welcome any and all pull requests, but the goal should be this and only this:
- start a django project without having to install any libraries on your host machine
- make it straight forward to then launch and manage your project
- keep it real simple and barebones to let the developer then begin to modify the project to their needs. At the end of this there will be a list of actions to consider.

# Settings to consider:
- As you are setting up your project you will have to name your project. In this document the project is called myproject. See where this is referenced and update it according. This will be true in the initialize django project command, in the Docerfile in the DJANG_SETTINGS_MODULE,
- When you run the second command, that is how you start an app, you can create a name other than new_app.
- As your project matures you'll update the requirements.txt file to include the necessary django and python libraries for your project


# Initializing a django project
## Run the command to initialize the django project
- replace my project with whatever you want your project to be called.
`docker run --rm -v "$PWD:/app" -w /app python:3.11-slim sh -c "pip install django && django-admin startproject myproject ."`

- this creates a django project in your current directory. it will create the manage.py and myproject directories.
- you are now set up with your django project. Soon you can run everything within the container

## Create a docker to now run your project
- Now you have a project and an app. You don't HAVE to create an app, you can create an app after you run the project in your django-container.
- copy this to a `requirements.txt` file, and add any additional requirements you think are necessary:
```
django==5.2
django-allauth==0.52.0
django-cors-headers

djangorestframework
djangorestframework-simplejwt

drf-spectacular
drf-spectacular[swagger-ui]
```
- copy this `Dockerfile` file:
```
# Dockerfile
FROM python:3.11-slim

ENV PYTHONDONTWRITEBYTECODE 1
ENV PYTHONUNBUFFERED 1

ENV DJANGO_SETTINGS_MODULE=myproject.settings

RUN apt-get update && apt-get install -y \
    vim \
    && apt-get clean

WORKDIR /app

COPY requirements.txt .
RUN pip install --upgrade pip
RUN pip install --no-cache-dir -r requirements.txt
RUN pip install debugpy

COPY . .

CMD ["bash"]
# when using a .env file you can expore a port this way
# EXPOSE ${APP_API_PORT}
EXPOSE 7777 
```
- copy this `docker-compose.yaml` file to run your project
```
version: '3.8'

services:
  db:
    image: postgres:latest
    container_name: app-db
    environment:
      POSTGRES_DB: project_db # ${DB_NAME}
      POSTGRES_USER: root # ${DB_USER}
      POSTGRES_PASSWORD: password # ${DB_PASSWORD}
    networks:
      - app-network # rename to something related to your project
    volumes:
      - postgres_data:/var/lib/postgresql/data

  app:
    build: 
      context: ./
      dockerfile: Dockerfile
    container_name: app
    environment:
      - DB_NAME=project_db # ${DB_NAME}
      - DB_USER=root # ${DB_USER}
      - DB_PASSWORD=password # ${DB_PASSWORD}
      - DB_HOST=db # the name of your db service, in this case db
      - DB_PORT=5432 # ${DB_PORT}
      - SECRET_KEY=secret_key # ${SECRET_KEY}
      - PROJECT_ENV=dev # ${PROJECT_ENV}
      - DOMAIN=example.com # ${DOMAIN}
    command: python manage.py runserver 0.0.0.0:8000 # ${CMD}

    volumes:
      - .:/app/
    ports:
      - 7777:8000 # ${KLAXON_API_PORT}:8000
    depends_on:
      - db
    networks:
      - app-network

networks:
  app-network:
    driver: bridge

volumes:
  postgres_data:
```
# Managing your django project
Now that you've created your project and your container you'll use the running container to run the django commands. like this:
- `docker exec -it app python manage.py migrate` you'll run this to set up the database
- `docker exec -it app python manage.py createsuperuser` you'll runthis to create your first superuser
- `docker exec -it app python manage.py startapp new_app` you'll runthis to create your a django app with your project

# Things to do or remember to do:
- You should create a .env file and then comment out all the places where its referenced.
- If you create an app, be sure to add it to your settings.py file
- You are running a database service but are not using it. By default django projects use sqlite. Be sure to update your settings.py file to point to your database.
  

