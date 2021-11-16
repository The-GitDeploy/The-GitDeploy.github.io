# Setup

<!-- vscode-markdown-toc -->
* [Setting up the repository](#Settinguptherepository)
	* [Create the repository](#Createtherepository)
	* [Set up a webhook](#Setupawebhook)
	* [(For private repos:) Generate a PAT](#Forprivaterepos:GenerateaPAT)
* [Installing GitDeploy](#InstallingGitDeploy)
	* [Install Docker](#InstallDocker)
	* [Launch GitDeploy](#LaunchGitDeploy)
* [Additional steps](#Additionalsteps)
	* [Changing the admin password](#Changingtheadminpassword)
* [Adding applications](#Addingapplications)
	* [Specify app in `docker-compose.yml`](#Specifyappindocker-compose.yml)
	* [Connecting an app to a subdomain](#Connectinganapptoasubdomain)
		* [Connecting to the GitDeploy network](#ConnectingtotheGitDeploynetwork)
		* [Seting up the nginx-routing](#Setingupthenginx-routing)
	* [Accessing other files in the repository](#Accessingotherfilesintherepository)
* [Troubleshooting](#Troubleshooting)

<!-- vscode-markdown-toc-config
	numbering=false
	autoSave=true
	/vscode-markdown-toc-config -->
<!-- /vscode-markdown-toc -->

## <a name='Settinguptherepository'></a>Setting up the repository

### <a name='Createtherepository'></a>Create the repository

Log in to GitHub and navigate to the following repository: [The-GitDeploy/core-config](https://github.com/The-GitDeploy/core-config)

This repository is a template, so you can use it to create your own repository, using this button:

![The "Use this template" button is on the top right of the code section](/img/use_as_template.png "Use this 'Use this template' button")

You'll be asked to provide a name for your repository; you can name it however you want.  
It shouldn't matter, whether the repository is private or not, you probably want to make it private though, to provide anoter layer of security.

### <a name='Setupawebhook'></a>Set up a webhook
Navigate to the settings and add a new webhook.

![Settings > Webhooks > Add Webhook](/img/webhook.png "Add a new webhook")

Set the Payload URL to `https://<your server>:5555/webhook`.

Keep the content type as `application/x-www-form-urlencoded`.

For now, you need to disable SSL verification, as the admin interface is "only" encrypted using a self-signed certificate.

### <a name='Forprivaterepos:GenerateaPAT'></a>(For private repos:) Generate a PAT

For the server to be able to clone a private repo, it needs a Personal Auth Token of your account.

You can generate one in [Githubs developer settings](https://github.com/settings/tokens/new?scopes=repo). Set a name, set it to not expire, and keep the repo scope.

Once generated, copy the token. You need to provide it to the container containing the service to update the containers.  
In the file `/compose/management/docker-compose.yml` find the line that says `- PAT=""` and paste your PAT between the quotes.  
You will also need the PAT at a later stage, so keep it nearby.

## <a name='InstallingGitDeploy'></a>Installing GitDeploy

### <a name='InstallDocker'></a>Install Docker

[Get Docker](https://docs.docker.com/get-docker/) and follow the installation instructions.

### <a name='LaunchGitDeploy'></a>Launch GitDeploy

Execute the following command:
```
bash <(curl -s https://github.com/The-GitDeploy/core-config/blob/main/setup.sh)
```

If your repo is private, you will be asked to enter your PAT here.

After a couple seconds the basic config should be up and running and update, whenever you push to a new config to your repo.

The monitoring panel is available under `https://username:password@<your server>:5555`. The default username is `admin` with the password `admin`

## <a name='Additionalsteps'></a>Additional steps

### <a name='Changingtheadminpassword'></a>Changing the admin password

The usernames/passwords for the monitoring panel are defined in the file `/compose/management/build_nginx/.htpasswd` in the form:
```
# comment
username:password
username:password:comment
```
Usernames are supplied in plaintext and passwords can be generated with the following command:
```
openssl passwd -5 <plaintext password goes here>
```

## <a name='Addingapplications'></a>Adding applications

### <a name='Specifyappindocker-compose.yml'></a>Specify app in `docker-compose.yml`

All containers are specified in a file `/compose/<project name>/docker-compose.yml`, so to add a new app you need to create a new folder and the `docker-compose.yml`-file in there.

You can either look up the syntax of the file [here](https://docs.docker.com/compose/compose-file/compose-file-v3/) or look at the templates in [this repositoy](https://github.com/The-GitDeploy/templates).

A very basic file might look like this:
```yaml
version: "3.8"
services:
  <service name>:
    image: <docker image>
    restart: always
```

While this file is enough to launch the image, you probably want to do more:

### <a name='Connectinganapptoasubdomain'></a>Connecting an app to a subdomain

To expose a container to the internet to steps are necessary.

#### <a name='ConnectingtotheGitDeploynetwork'></a>Connecting to the GitDeploy network

The setup command automatically created a network, that containers on your server can use to communicate with each other. To connect to it, you need to modify your `docker-compose.yml`:

```yaml
version: "3.8"
services:
  <service name>:
    image: <docker image>
    ports:
      -"<port of the container to connect to>"
    restart: always

networks:
  default:
    external: true
    name: gitdeploy
```

This itself does not expose the container to the internet yet, it only allows other containers to connect to it.

#### <a name='Setingupthenginx-routing'></a>Seting up the nginx-routing

The folder `/compose/nginx` contains a nginx instance that will control all routings. The file `/compose/nginx/build/servers.conf` is a regular nginx configuration file.

To now add a route to connect to your container you need to add the following in that file:
```conf
server {
  listen 443 ssl;

  # if this block is included, a certificate will be issued from Let's Encrypt
  # if you comment it out, a default self-signed certificate will be used
  ssl_certificate_by_lua_block {
    auto_ssl:ssl_certificate()
  }

  # specify the (sub)domain here.
  server_name <domain>;

  # tells nginx to proxy all requests to the given container
  # replace hostname with the name of the service (specified in the docker-compose file)
  # replace port with the port to connect to
  location / {
    proxy_pass http://hostname:port/;
    proxy_read_timeout 1800;
  }
}

```

Now your container can be accessed from anywhere.

### <a name='Accessingotherfilesintherepository'></a>Accessing other files in the repository

Do NOT link to files using relative paths.
Instead use the docker volume `gitdeploy` like this:
```yaml
version: "3.8"
services:
  <service name>:
    image: <docker image>
    restart: always
    volumes:
      - gitdeploy:/gitdeploy:ro

volumes:
  gitdeploy:
    external: true
```

All files of the repository can be accessed from within the container under the path `/gitdeploy`

## <a name='Troubleshooting'></a>Troubleshooting

If you ever pushed a faulty configuration to the degree that the server won't update itself, you can always just connect over SSH and execute the following commands:

Stop all running containers:
```
docker kill $(docker ps -q)
```

Re-run the init script:
```
bash <(curl -s https://github.com/The-GitDeploy/core-config/blob/main/setup.sh)
```
Don't worry, if an error like `Can't create network, because it already exists!`, this is no problem.