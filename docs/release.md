# Init Release
First we need to create the release using the following command. It will create a directory called `staticsite-boshrelease` and will set up a few directories and files for us to help get started.
```bash
bosh init-release --dir=staticsite-boshrelease
```
This will create some directories
```
config/
jobs/
packages/
src/
```
* config contains files that describe our release and where our release stores its blobs externally.
* jobs contains all the jobs that this release will have available to it, a job is a way to run a package or packages
* packages contains the packages that will be used in this release, it describes how to build/compile the packages
* src is where you can store sourcecode and files if you don't want to utilise an external blobstore

## Create Source/Blob Files
Now that we have our release skeleton, we need to create the HTML that will deployed in this release, and all the resources we are using in our HTML (css/js).

First we need to make a directory in the src folder to store our files that we need.
```bash
mkdir src/static
```
Now we need to get the bootstrap archive that contains the css/js we will be using and put it into the `src/static` directory too.
```bash
wget -O src/static/bootstrap-4.1.3-dist.zip https://github.com/twbs/bootstrap/releases/download/v4.1.3/bootstrap-4.1.3-dist.zip
```
And finally, generate the index.html file we will use.
```bash
cat << "EOF" > src/static/index.html
<html>
  <head>
    <title>Samuel L Ipsum</title>
    <link rel="stylesheet" href="/css/bootstrap.min.css">
    <script src="/js/bootstrap.min.js"></script>
  </head>
  <body>
    <nav class="navbar navbar-expand-lg navbar-dark bg-dark">
      <button class="navbar-toggler" type="button" data-toggle="collapse" data-target="#navbarSupportedContent" aria-controls="navbarSupportedContent" aria-expanded="false" aria-label="Toggle navigation">
        <span class="navbar-toggler-icon"></span>
      </button>
      <div class="collapse navbar-collapse" id="navbarSupportedContent">
        <ul class="navbar-nav mr-auto">
          <li class="nav-item active">
            <a class="nav-link" href="/">Home <span class="sr-only">(current)</span></a>
          </li>
        </ul>
      </div>
    </nav>
    <div class="container">
      <h1>Samuel L Ipsum</h1>
      <!-- start slipsum code -->
      <h2>Are you ready for the truth?</h2>
      Well, the way they make shows is, they make one show. That show's called a pilot. Then they show that show to the people who make shows, and on the strength of that one show they decide if they're going to make more shows. Some pilots get picked and become television programs. Some don't, become nothing. She starred in one of the ones that became nothing.
      <h2>We happy?</h2>
      Now that we know who you are, I know who I am. I'm not a mistake! It all makes sense! In a comic, you know how you can tell who the arch-villain's going to be? He's the exact opposite of the hero. And most times they're friends, like you and me! I should've known way back when... You know why, David? Because of the kids. They called me Mr Glass.
      <!-- end slipsum code -->
    </div>
  </body>
</html>
EOF
```

# Generate Package
Next up we need to create the package that will copy our static site into the right location on the deployed VM.

First generate the package skeleton for our package that we will call `staticsite`
```bash
bosh generate-package staticsite
```
Now we have a skeleton directory, it contains 2 files

* `packages/staticsite/packaging`
* `packages/staticsite/spec`

!!! info
    * Packaging is where we write how to compile or build our software (our static website)
    * Spec is where we define things that our static site will need to create the package, files etc..

Lets update the spec file first, it will tell the package what we need to build with.

!!! note
    BOSH will look in the src directory first for any resources, if they aren't there, it will look in the blobstore.

Earlier we downloaded a zip file, and created an index.html file. We need to tell our spec file these 2 files exist, and where to find them.
```bash
cat << "EOF" > packages/staticsite/spec
---
name: staticsite

dependencies: []

files:
  - static/index.html
  - static/bootstrap-4.1.3-dist.zip
EOF
```

Great, now we need to tell BOSH how to package these 2 things together.
```bash
cat << "EOF" > packages/staticsite/packaging
set -e

echo "Moving index.html to install target"
cp -a static/index.html ${BOSH_INSTALL_TARGET}

echo "Extract bootsrap resources into target"
unzip static/bootstrap-4.1.3-dist.zip -d ${BOSH_INSTALL_TARGET}
EOF
```
Since the bootstrap ZIP file contains our css/js, we need to unzip it into the install target directory to actually be able to use it.

!!! note
    `${BOSH_INSTALL_TARGET}` is a location that will be created on the deployed VM. In this case it will reference the final directory of `/var/vcap/packages/staticsite/`

Great, we have a script that tells BOSH how to put all our bits together. Now we need a job that knows what to do with our package.

# Generate Job
Now we need to create a job that will tell BOSH how to use this package. Since it is a simple static website, our job will be relatively simple.

First we need to create the job structure.
```bash
bosh generate-job staticsite
```
This creates a few files
```
jobs/staticsite/spec
jobs/staticsite/monit
jobs/staticsite/templates/
```
* spec is a file that contains properties and package requirements for this job
* monit should know how to start/stop any services this job may have
* templates is a place to store templates, like start and stop scripts, drain scripts, etc.

Now we have the job skeleton, we need to tell it about our package.

We will need to set up a default docroot property, and tell it about a pre-start script we need.
```bash
cat << "EOF" > jobs/staticsite/spec
---
name: staticsite

templates:
  pre-start.sh: bin/pre-start

packages:
  - staticsite

properties:
  docroot:
    description: Nginx docroot
    default: /var/vcap/store/nginx/www/document_root
EOF
```

We aren't going to use any services in this release or job, so leave the monit file blank.

But we do need to tell BOSH how to set up our site on the server, for this we need a pre-start script (which has already been defined in the spec file)

This contains some embedded ruby in it, which BOSH knows how to interpret.
```bash
cat << "EOF" > jobs/staticsite/templates/pre-start.sh
#!/bin/bash
echo "Create the docroot base directory"
mkdir -p $(dirname <%= p('docroot') %>)
echo "If it our docroot already exists within it, delete it"
if [[ -d "<%= p('docroot') %>" ]]
then
  rm -rf <%= p('docroot') %>
fi
echo "Copy our new docroot into the base directory"
cp -a /var/vcap/packages/staticsite <%= p('docroot') %>
echo "Make sure vcap owns it"
chown -R vcap:vcap <%= p('docroot') %>
EOF
```

# Create Release
Great, we have the skeleton for a release done, now we need to create a release archive in preparation for uploading it to our director.

This will create a release archive in `/tmp/static-site.tgz`.
```bash
bosh create-release --force --tarball=/tmp/static-site.tgz
```

!!! info
    A release archive contains EVERYTHING needed to build the release, it will contain all sourcecode for your packages and scripts. This is part of the modern release engineering principles.

Now upload the release to the director
```bash
bosh upload-release /tmp/static-site.tgz
```
