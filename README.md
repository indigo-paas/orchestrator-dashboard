# orchestrator-dashboard
INDIGO PaaS Orchestrator - Simple Graphical UI

Functionalities:
- IAM authentication
- Display user's deployments
- Display deployment details, template and log
- Delete deployment
- Create new deployment

The orchestrator-dashboard is a Python application built with the [Flask](http://flask.pocoo.org/) microframework; [Flask-Dance](https://flask-dance.readthedocs.io/en/latest/) is used for Openid-Connect/OAuth2 integration.


The docker image uses [Gunicorn](https://gunicorn.org/) as WSGI HTTP server to serve the Flask Application.

# How to deploy the dashboard

Register a client in IAM with the following properties:

- redirect uri: `https://<DASHBOARD_HOST>:<PORT>/login/iam/authorized`
- scopes: 'openid', 'email', 'profile', 'offline_access'
- introspection endpoint enabled

Create the folder `instance` to put the application configuration files:
 - (mandatory) `config.json` file (see the [example](app/config-sample.json))
 - (optional) `vault-config.json` file (see the [example]() needed to enable the integration with Vault 
 
````
instance
|____config.json
|____vaul-config.json [optional]

````

Clone the tosca-templates repository to get a set of tosca templates that the dashboard will load, e.g.:

````
git clone https://github.com/indigo-dc/tosca-templates -b stable/v3.0
````

You need to run the Orchestrator dashboard on HTTPS (otherwise you will get an error); you can choose between
- enabling the HTTPS support
- using an HTTPS proxy

Details are provided in the next paragraphs.

## TOSCA Template Metadata 

The Orchestrator dashboard can exploit some optional information provided in the TOSCA templates for rendering the cards describing the type of applications/services or virtual infrastructure that a user can deploy.


In particular, the following tags are supported:

| Tag name  | Description        | Type               |
| -------------- | ------------- | ------------------ |              
| description | Used for showing the card description  |  String |
| metadata.display_name | Used for the card title. If not pro  |    String |
| metadata.icon  |  Used for showing the card image. If no image URL is provided, the dashboard will load this [icon](https://cdn4.iconfinder.com/data/icons/mosaicon-04/512/websettings-512.png). | String |
| metadata.display_name | Used for the card title. If not provided, the template name will be used   | String |
| metadata.tag  | Used for the card ribbon (displayed on the right bottom corner)   |     String |
| metadata.allowed_groups | Used for showing the template only to members of specific groups |  String <br> - "*" == any group can see the template <br> - "group1,group2" == only members of _group1_ and _group2_ can see the template |


Example of template metadata:

```
tosca_definitions_version: tosca_simple_yaml_1_0

imports:
  - indigo_custom_types: https://raw.githubusercontent.com/indigo-dc/tosca-types/v4.0.0/custom_types.yaml

description: Deploy a Mesos Cluster (with Marathon and Chronos frameworks) on top of Virtual machines

metadata:
  display_name: Deploy a Mesos cluster
  icon: https://indigo-paas.cloud.ba.infn.it/public/images/apache-mesos-icon.png

topology_template:

....
```

### Enabling HTTPS

You would need to provide
- a pair certificate/key that the container will read from the container paths `/certs/cert.pem` and `/certs/key.pem`;
- the environment variable `ENABLE_HTTPS` set to `True`
 

Run the docker container:
```
docker run -d -p 443:5001 --name='orchestrator-dashboard' \
           -e ENABLE_HTTPS=True \
           -v $PWD/cert.pem:/certs/cert.pem \
           -v $PWD/key.pem:/certs/key.pem \
           -v $PWD/instance:/app/instance \
           -v $PWD/tosca-templates:/opt/tosca-templates \
           indigodatacloud/orchestrator-dashboard:latest
```
Access the dashboard at `https://<DASHBOARD_HOST>/`

### Using an HTTPS Proxy 

Example of configuration for nginx:
```
server {
      listen         80;
      server_name    YOUR_SERVER_NAME;
      return         301 https://$server_name$request_uri;
}

server {
  listen        443 ssl;
  server_name   YOUR_SERVER_NAME;
  access_log    /var/log/nginx/proxy-paas.access.log  combined;

  ssl on;
  ssl_protocols TLSv1 TLSv1.1 TLSv1.2;
  ssl_certificate           /etc/nginx/cert.pem;
  ssl_certificate_key       /etc/nginx/key.pem;
  ssl_trusted_certificate   /etc/nginx/trusted_ca_cert.pem;

  location / {
                # Pass the request to Gunicorn
                proxy_pass http://127.0.0.1:5001/;

                proxy_set_header        X-Real-IP $remote_addr;
                proxy_set_header        X-Forwarded-For $proxy_add_x_forwarded_for;
                proxy_set_header        X-Forwarded-Proto https;
                proxy_set_header        Host $http_host;
                proxy_redirect          http:// https://;
                proxy_buffering         off;
  }

}
```

Run the docker container:

```
docker run -d -p 5001:5001 --name='orchestrator-dashboard' \
           -v $PWD/instance:/app/instance \
           -v $PWD/tosca-templates:/opt/tosca-templates \
           indigodatacloud/orchestrator-dashboard:latest
```
:warning: Remember to update the redirect uri in the IAM client to `https://<PROXY_HOST>/login/iam/authorized`

Access the dashboard at `https://<PROXY_HOST>/`


### Add trusted certificates

If you need to install and trust certificates that are not included in the default CA bundle used by SLAT python application running in a docker container, you can mount the directory containing the cerficate(s) in PEM format (extensione .pem) in the container under the path /trusted_certs; e.g:

```
docker run -d -p 5001:5001 --name='orchestrator-dashboard' \
           -v $PWD/instance:/app/instance \
           -v $PWD/tosca-templates:/opt/tosca-templates \
           -v $PWD/trusted_certs:/trusted_certs \
           indigo-dc/orchestrator-dashboard:latest
```
The certificates provided in the directory will be automatically added to the python CA bundle.


### Performance tuning

You can change the number of gunicorn worker processes using the environment variable WORKERS.
E.g. if you want to use 2 workers, launch the container with the option `-e WORKERS=2`
Check the [documentation](http://docs.gunicorn.org/en/stable/design.html#how-many-workers) for ideas on tuning this parameter.

## How to build the docker image

```
git clone https://github.com/indigo-dc/orchestrator-dashboard.git
cd orchestrator-dashboard
docker build -f docker/Dockerfile -t orchestrator-dashboard .
```

## How to setup a development environment

```
git clone https://github.com/indigo-dc/orchestrator-dashboard.git
cd orchestrator-dashboard
python3 -m venv venv
source venv/bin/activate
pip3 install -r requirements.txt
```

Start the dashboard app:
```
FLASK_app=orchdashboard flask run --host=0.0.0.0 --cert cert.pem --key privkey.pem --port 443
```
## Troubleshooting

### SSL Cert Verification
If you see problems with the SLAM interaction, you would need to specify the certificate to be used to verify the SSL connection.
You can pass the path to a CA_BUNDLE file or directory with certificates of trusted CAs setting the parameter SLAM_CERT in the config.json file:

```
{
  ...
  "SLAM_URL": "https://indigo-slam.cloud.ba.infn.it:8443",
  "SLAM_CERT": "/path/to/certfile"
}
```

If you are running the docker container, you need to ensure that the cert file is available inside the container in the path set in the SLAM_CERT parameter, i.e. you would use a bind mount (`-v $PWD/certfile:/path/to/cerfile`)

#### References:

- https://2.python-requests.org/en/master/user/advanced/#ssl-cert-verification


# DB prerequisites:

```
You must have a running instance of MySql
You must have/create a user with full db administration rights to auto create/manage the database
```




