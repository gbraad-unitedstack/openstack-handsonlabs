# Deployment using OpenStack Heat

_Gerard Braad <me@gbraad.nl>_


## Prerequisites

```
$ yum install -y python-virtualenv
$ virtualenv venv
$ source venv/bin/activate
(venv) $ pip install python-heatclient
```

## Example deployment
Use the file `create-instance.yml` and create the stack.

```
(venv) $ heat stack-create my_stack -f create-instance.yaml -P "key=gbraad;image=Fedora-23"
```
