# Hello world web app using APC

This is taken from http://docs.apcera.com/tutorials/staticsite/ but improved to use github and provide the template index.html file.

You can quickly deploy a static web site (no backend dependencies) to Apcera using the built-in static-site staging pipeline. The static-site staging pipeline creates an application package containing your web site files and an NGINX package to serve the site files.

To create and deploy a static web site:
* From the ./tutorials/mywebsite directory
* The file named [index.html](index.html) contains the following HTML:
```
<!-- index.html -->
<!DOCTYPE html>
<html lang="en">
  <head>
    <meta charset="utf-8">
    <title>Hello, Apcera!</title>
  </head>
  <body>
    <h1>Hello, Apcera!</h1>    
  </body>
</html>
```

Open a terminal and change to the mywebsite directory.
```
 cd ./tutorials/mywebsite
```

Run the following command to create and start an application using the web site files, note that APC prompts you to confirm information about the application to deploy, such as the directory that contains the app's source files, the number of application instances to create, and the amount of memory to allocate to each instance. Press Enter at each prompt to accept the defaults:

```
$  apc app create mywebsite --start
Deploy path [/Users/admin/dev/mywebsite]:
Instances [1]:
Memory [256MB]:
╭───────────────────────────────────────────────────────────────────────────────╮
│                             Application Settings                              │
├───────────────────┬───────────────────────────────────────────────────────────┤
│              FQN: │ job::/sandbox/admin::mywebsite                            │
│        Directory: │ /local/git/github/UnofficialApceraDoc/tutorials/mywebsite │
│        Instances: │ 1                                                         │
│          Restart: │ always                                                    │
│ Staging Pipeline: │ (will be auto-detected)                                   │
│              CPU: │ 0ms/s (uncapped)                                          │
│           Memory: │ 256MB                                                     │
│             Disk: │ 1024MB                                                    │
│           NetMin: │ 5Mbps                                                     │
│           Netmax: │ 0Mbps (uncapped)                                          │
│         Route(s): │ auto                                                      │
│  Startup Timeout: │ 30 (seconds)                                              │
│     Stop Timeout: │ 5 (seconds)                                               │
╰───────────────────┴───────────────────────────────────────────────────────────╯

Is this correct? [Y/n]: Y


A summary table displays the properties for the new application, a few of which are shown below. All of these properties can be set on the command line when calling apc app create or apc app update (see )
* FQN – The application package's fully-qualified name.
* Staging Pipeline – The staging pipeline to use to stage the application for deployment. In this case, because we did not specify a staging pipeline in the command string, APC will detect which staging pipeline to use based on the contents of our application directory.
* Route – Indicates the URL where the web site will be accessible. In this case, we let the cluster generate a route for us that consists of the application name concatenated with six random characters. You can specify a specific route with the --routes command line parameter.

After pressing "Y" to confirm, the staging process will start.
```
Is this correct? [Y/n]:
Packaging... done
Creating package "mywebsite"... done
Uploading package contents... done!
[staging] Subscribing to the staging process...
[staging] Beginning staging with 'stagpipe::/apcera::static-site' pipeline, 1 stager(s) defined.
[staging] Launching instance of stager 'static-site'...
[staging] Downloading package for processing...
[staging] Validating an index.htm or index.html file exists
[staging] Staging is complete.
Creating app "mywebsite"... done
Start Command: ./start_nginx
Waiting for the application to start...
App should be accessible at "http://mywebsite-k6b68x.tutorial.apcera-platform.io"
```
If the application has been deployed successfully, the output indicates the URL where you can view the web site ("App should be accessible at <URL>"). 

Open this URL in your browser to see the running web site.

