# -*- mode: ruby -*-
# vi: set ft=ruby :

# Update Ubuntu's package database and install the necessary requirements for
# the project. We're using PostgreSQL as our database backend and we're
# serving the site using gunicorn and ngingx.
$install_os_packages = <<SCRIPT
apt-get update
apt-get install -y python3 python3-dev postgresql postgresql-server-dev-all nginx
SCRIPT

# Create a new nginx configuration file, and reload the service.
$create_nginx_configuration = <<SCRIPT
echo '
# quick_django_vagrant.conf

upstream django {
    server 127.0.1.1:8000;
}

server {
    listen      80;
    server_name 127.0.0.1 127.0.1.1 localhost quick-django-vagrant quick-django-vagrant.local;
    charset     utf-8;

    client_max_body_size 75M;   # adjust to taste

    location /media  {
        alias /home/vagrant/quick_django_vagrant/mysite/media;
    }

    location /static {
        alias /home/vagrant/quick_django_vagrant/mysite/static;
    }

    location / {
        proxy_pass       http://django;
        proxy_redirect   off;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Host $server_name:8000;
    }
}
' > /etc/nginx/sites-available/quick_django_vagrant.conf
ln -sf /etc/nginx/sites-available/quick_django_vagrant.conf /etc/nginx/sites-enabled/

service nginx reload
SCRIPT

# Create the user and datanase, and grant all privileges on the database to
# the user.
$create_database = <<SCRIPT
sudo -u postgres psql --command="CREATE USER mysite WITH PASSWORD 'mysite';"
sudo -u postgres psql --command="CREATE DATABASE mysite WITH OWNER mysite;"
sudo -u postgres psql --command="GRANT ALL PRIVILEGES ON DATABASE mysite TO mysite;"
SCRIPT

# Create the virtualenv and install pip manually. Pip has to be installed
# manually due to an issue with Ubuntu 15.04's default virtualenv package.
$create_virtualenv = <<SCRIPT
pyvenv-3.4 --without-pip quick_django_vagrant_venv
source quick_django_vagrant_venv/bin/activate
curl --silent --show-error --retry 5 https://bootstrap.pypa.io/get-pip.py | python
SCRIPT

# Install the site's requirements
$install_requirements = <<SCRIPT
source quick_django_vagrant_venv/bin/activate
pip install -r quick_django_vagrant/requirements.txt
SCRIPT

# Apply migrations
$apply_migrations = <<SCRIPT
source quick_django_vagrant_venv/bin/activate
cd quick_django_vagrant/mysite/
python manage.py migrate
SCRIPT

# Collect static resources
$collect_static_resources = <<SCRIPT
source quick_django_vagrant_venv/bin/activate
cd quick_django_vagrant/mysite/
python manage.py collectstatic --noinput
SCRIPT

# Load test data
$load_test_data = <<SCRIPT
source quick_django_vagrant_venv/bin/activate
cd quick_django_vagrant/mysite/
echo '
[
{
    "model": "auth.user",
    "pk": 1,
    "fields": {
        "last_name": "",
        "is_active": true,
        "groups": [],
        "is_superuser": true,
        "date_joined": "2015-07-19T00:56:30.151Z",
        "last_login": "2015-07-19T00:58:27.282Z",
        "email": "admin@quick-django-vagrant.local",
        "first_name": "",
        "password": "pbkdf2_sha256$20000$XaNO735y9YmE$3vaHYxRGS7mBPrIZS7nmQCFGuP8urI9Ru54vkXTGN50=",
        "is_staff": true,
        "user_permissions": [],
        "username": "admin"
    }
},
{
    "model": "polls.question",
    "pk": 1,
    "fields": {
        "question_text": "Is Django awesome?",
        "pub_date": "2015-07-19T00:58:48Z"
    }
},
{
    "model": "polls.choice",
    "pk": 1,
    "fields": {
        "question": 1,
        "votes": 0,
        "choice_text": "Yup"
    }
},
{
    "model": "polls.choice",
    "pk": 2,
    "fields": {
        "question": 1,
        "votes": 0,
        "choice_text": "Of course!"
    }
},
{
    "model": "polls.choice",
    "pk": 3,
    "fields": {
        "question": 1,
        "votes": 0,
        "choice_text": "What kind of a question is that?"
    }
}
]
' > data.json
python manage.py loaddata data.json
SCRIPT

# Run the application with Gunicorn
$run_application = <<SCRIPT
source /home/vagrant/quick_django_vagrant_venv/bin/activate
cd /home/vagrant/quick_django_vagrant/mysite/
gunicorn --bind 127.0.1.1:8000 --daemon --workers 4 mysite.wsgi
SCRIPT

Vagrant.configure(2) do |config|
  config.vm.box = "ubuntu/trusty64"

  config.vm.hostname = "quick-django-vagrant.local"

  config.vm.network "forwarded_port", guest: 80, host: 8000

  config.vm.synced_folder ".", "/home/vagrant/quick_django_vagrant/"

  config.vm.provider "virtualbox" do |vb|
    vb.memory = "1024"
  end

  # Execute provisioning scripts

  config.vm.provision "shell", inline: $install_os_packages
  config.vm.provision "shell", inline: $create_nginx_configuration
  config.vm.provision "shell", inline: $create_database
  config.vm.provision "shell", privileged: false, inline: $create_virtualenv
  config.vm.provision "shell", privileged: false, inline: $install_requirements
  config.vm.provision "shell", privileged: false, inline: $apply_migrations
  config.vm.provision "shell", privileged: false, inline: $collect_static_resources
  config.vm.provision "shell", privileged: false, inline: $load_test_data

  # Run application

  config.vm.provision "shell", run: "always", privileged: false, inline: $run_application
end
