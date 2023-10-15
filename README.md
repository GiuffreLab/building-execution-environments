# Create Ansible Execution Environments using Ansible Builder

With `ansible-builder` you can configure and build portable, consistent, customized Ansible control nodes that are packaged as containers by Podman or Docker. These containers are known as execution environments. You can use them on AWX or Ansible Controller, with Ansible Navigator, for local playbook development and testing, in your CI pipelines, and anywhere else you run automation.

You can design and distribute specialized execution environments for your Ansible content, choosing the versions of Python and ansible-core you want, and installing only the Python packages, system packages, and Ansible collections you need for your tasks.


# Getting started

I run most of my testing devices on Debian OS. Because of this I need to use Virtual Environments to use python tools/packages as Debian uses only APT in the standard linux OS to prevent cross-pollination issues with various package managers.

If you do not use Debian, you can skip this part.

**Installation of the Python vENV**

``` bash
sudo apt update && sudo apt install python3 python3-venv
```

**Create the environment**

We'll create a purpose built environment solely for `ansible-builder`

``` bash
python3 -m venv ansible-builder-env
```

you should now see a folder called `ansible-builder-env`. You can only launch this environment being a level above this folder.

Activate the virtual environment

``` bash
source ansible-builder-env/bin/activate
```

We can build our files in advance before compiling with the `ansible-builder-env`

Install `ansible-builder`

``` bash
pip install ansible-builder
```

Leave the virtual environment for now

``` bash
deactivate
```

# Core concepts of Execution Environments

Ansible Builder depends on more generalized containerization tools like Podman or Docker.

Before you start using Ansible Builder, you should understand the following concepts and terms relevant to any use of containers:

**Build instruction file** (called a `Containerfile` in Podman and a `Dockerfile` in Docker): an instruction file for creating a container image by installing and configuring the code and dependencies.

**Container:** a package of code and dependencies that runs a service or an application across a variety of computing environments.

**Image:** a complete but inactive version of a container - you can distribute images and create one or more containers based on each image.


To put simply, on a CLI Ansible CORE environment, you would do your dependency installs for python packages like `results` or `proxmoxer` using the `pip` python package manager.

If you were using any Ansible Galaxy collections or roles, you would install those as well.

Because AWX and AAP spin up containers to perform their Template tasks, you have no way of adding these extra dependencies to make your playbooks run. Enter the Execution Environment, which are at their core, just additional containers.


# Example building an Execution Environment

We'll use the Ansible Galaxy collection for `Community.General` as our example project. As this collection has a very large amount of roles in it, we'll continue to focus on what we need to use the `Proxmox` specific ones.


# Understanding build files

You will need to create a few different files for `ansible-builder` to use to compile your Execution Environment container image.

`bindep.txt` = System level dependencies for what your image needs

`requirements.txt` = Python packages you need for your playbook to run

`requirements.yaml` = Ansible Galaxy Collections and Roles you wish to use

`execution-environment.yaml` = The equivalent of the `Dockerfile` or `Containerfile` that `ansible-builder` will use to compile your container image.

# The build files we'll need

Create the following files for our build in a folder called `awx-community-general-ee` or whatever you wish. All further will use that for examples.

`bindep.txt` contents

``` text
subversion [platform:rpm]
subversion [platform:dpkg]
```

`requirements.txt` contents

``` text
ansible
proxmoxer
requests
```

`execution-environment.yaml` contents

``` yaml
---
version: 1

build_arg_defaults:
  EE_BASE_IMAGE: 'quay.io/ansible/ansible-runner:latest'

dependencies:
  python: requirements.txt
  system: bindep.txt

additional_build_steps:
  prepend:
    - RUN /usr/bin/python3 -m pip install --upgrade pip
  append:
    - RUN pip3 list installed
    - RUN ansible-galaxy collection install community.general -p /usr/share/ansible/collections
```

You'll notice we aren't using a `requirements.yaml` file. Unfortunately for some reason, the `community.general` collection does not play well with `ansible-builder` right now, so we build it right in the container file. Until I figure that issue out, the above compiles correctly.

our `requirements.yaml` file would look like this if we could use it.

`requirements.yaml` contents

``` yaml
---
collections:
  - name: community.general

roles:
  - name: 
```

# Compiling the Image

Now that we have our needed files we can go back to the folder level of our python `ansible-builder-env`.

Activate the virtual environment again 

``` bash
source ansible-builder-env/bin/activate
```

Return back to our Execution Environment folder `awx-community-general-ee`

We can now run `ansible-builder`.

We will run the following command to compile the image. the -v 3 at the end is for the verbosity. If you do not care about this run it without it. Levels of verbosity are 1-3.

You will need to login to your docker hub account so that you can push the image when complete. This will go on the example of `giuffrelab` as the account going forward as that is my docker hub name.

``` bash
ansible-builder build --tag giuffrelab/awx-community-general-ee -v 3
```

Leave the virtual environment

``` bash
deactivate
```

You should now have a new folder called `context` containing a `Dockerfile` and a `_build` folder in the `awx-community-general-ee` folder. You can ignore these for now. 

``` bash
docker images
```

next you will need to upload the compiled image to Docker Hub. change `giuffrelab` and the image name you chose to whatever you used

``` bash
docker push giuffrelab/awx-community-general-ee
```

# Using the new Execution Environment in AWX-Operator

to make use of the new EE in AWX you need to do the following settings changes

**Add docker hub account**

Under `Resources` and `Credentials` 
- Select `Add`
- Fill out the `name` field
- Under `Credential Type` select `Container Registry`
- Under `Authentication URL` enter `docker.io`
- Fill in `Username`
- Fill in `Password`
- `Save`

**Add the new Execution Environment**

change `giuffrelab` and the image name you chose to whatever you used

Under `Administration` and `Execution Environments`
- Select `Add`
- Fill out the `name` field
- Under Pull select `Always Pull`
- Under image put `giuffrelab/awx-community-general-ee:latest`
- Under `Registry credential` select the one created above
- `Save`
