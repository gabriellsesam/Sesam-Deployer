[![Build Status](https://travis-ci.org/sesam-community/oracle-transform.svg?branch=master)](https://travis-ci.org/sesam-community/oracle-transform)

# Sesam Auto Deployer
## What do i do?
### Normal usage
I'm a docker container or python script which:
* Verifies a local node configuration by:
    * Checks all variables in config against specified variables files to verify from.
    * Checks all secrets in config against keyvault. (Path in vault must be same name as secret name)
* Uploads this to the specified master node:
    * Node configuration
    * Secrets
    * Variables

### Special usage
I can also update a git repo with pipes which match a pattern if "EXTRA_NODES" are specified. (This can be used to deploy from for the seperate node, I'll get back to this!)

Btw, i only update this repo if there are changes.

#### What do you mean by 'extra node'?

Well, in production and test environment it is really a completely seperate node, with some connection to the Master node.

An extra node is therefore a node config running on a seperate node, but it is connected to the master node somehow so you would actually like to do tests with both node configs!

Therefore, when the ENV is set to CI, this deployment script will run test based off of the whole node/… config

BUT! When it deploys, it automatically creates pipes (which will act as connections between the nodes) based off of templates you can define!

This means that during development, you can completely forget this pipe or that pipe exists on a seperate sesam node, because practically it doesn't make a difference :)

#### Ok, I'm both totally sold & confused. Please explain more
Let's say that you have two nodes: Master & Extra.

You want to run 'sesam test' with both node configs, but you also need to deploy the pipes belonging to Master to the real Master node! And also deploy the pipes belonging to Extra to the real Extra node!

Which pipes belong to which node can be found out simply by adding "metadata" : {"node": "extra"} and seperating based off of this.

But! You need more pipes to actually move data between nodes, and sometimes a node can only push or pull data because of firewall issues! 

Therefore you need to be able to dynamically create so called 'Replica pipes' between the nodes! This is the supercool thing about the script.

When CI is run, these replicas are not added to the node because it's all running on 1 node when CI runs. BUT! During deployment these replicas are deployed to actual sepereate nodes, along with the pipes which belong to the node.  

#### Alright, starting to make sense, so how do i set it up?
Well to start off you need to know the following:
if a pipe or system has this added to it:
````
{
    "metadata":{
        "node": "<extra_node_name>"        
    }
}
````
then it belongs to the extra node!

If this "node" key in metadata is not specified, it automatically belongs to the master node.

This extra node needs to be specified in the environment variable "EXTRA_NODES":{}, as follows:
```
EXTRA_NODES={
    "<extra_node_name>": {
        "EXTRA_NODE_GIT_URL": "github.com/<organization>/<repo_name>.git",
        "EXTRA_NODE_GIT_USERNAME": "<your-user>",
        "EXTRA_NODE_GIT_TOKEN": "<git_token>",
        "EXTRA_NODE_TEMPLATE_PATH": "extra_nodes/<extra_node_name>/"
    },
    "<another_extra_node>": {
        "EXTRA_NODE_GIT_URL": "github.com/<organization>/<repo_name>.git",
        "EXTRA_NODE_GIT_USERNAME": "<your-user>",
        "EXTRA_NODE_GIT_TOKEN": "<git_token>",
        "EXTRA_NODE_TEMPLATE_PATH": "extra_nodes/<extra_node_name>/"
    }
}
```
You could have many extra nodes specified, but each extra node would need it's own dictionary inside this environment variable!

You can also reuse the templates you're using for one extra node with another extra node, but you could also create different templates!

#### What are these templates?
Templates for pipes & systems are used to generate the connection between the master node and extra node. Node metadata is usually needed too.

Each extra node needs at minimum 3 templates, maximum 8.

Example: Extra node only writes to master node

1 Pipe on extra to send the data to master. 1 system on extra to connect to master. 1 pipe on master to receive the data.

Here are all the templates you can define. They must have this naming convention:

```
node-metadata.conf.json

pipe_on_extra_from_extra_to_master.json
pipe_on_extra_from_master_to_extra.json
system_on_extra_from_extra_to_master.json
system_on_extra_from_master_to_extra.json

pipe_on_master_from_extra_to_master.json
pipe_on_master_from_master_to_extra.json
system_on_master_from_extra_to_master.json
system_on_master_from_master_to_extra.json
``` 
`<what>-on_<where>_from_<where>_to_<where>.json`

You only need to define a template if you're actually going to need it.

#### Where do i put the templates?
Good question! You put them inside the folder which contains your node folder. Example:

```
root
    extra_nodes
        <extra_node_name>
            node-metadata.conf.json
            pipe_on_extra_from_extra_to_master.json
            pipe_on_extra_from_master_to_extra.json
            system_on_extra_from_extra_to_master.json
            system_on_extra_from_master_to_extra.json
            pipe_on_master_from_extra_to_master.json
            pipe_on_master_from_master_to_extra.json
            system_on_master_from_extra_to_master.json
            system_on_master_from_master_to_extra.json
    node
        pipes
            ….conf.json
        systems
            ….conf.json
        deployment
            whitelist-master.txt
            whitelist-test.txt
            whitelist-prod.txt
        variables
            variables-test.json
            variables-prod.json
```


#### Cool! Now wtf are the whitelist and variables folders in the node folder?
This script uses whitelists and variables to be able to upload different stuff for different environments and easily remove and add pipes.

#### Whitelist
Base folder is node/, therefore it would be written to as follows:
```
node-metadata.conf.json
pipes/some-pipe.conf.json
systems/some-system.conf.json
```
* whitelist-master.txt is used for CI (Testing)
* whitelist-prod & test used for prod & test deployments.

#### Variables:
Simply a json entity which contains variables for the different environments. Same structure as 'test-env.json' inside your node folder.

## Ok. I want to use the special extra node feature.
Well, then you just set up a git repo for this script to push the extra node config to!

This repo can then be used together with sesam-community/github-autodeployer to auto deploy to the extra node!

# Example environment config:
```
PYTHONUNBUFFERED=1
LOG_LEVEL=debug
NODE_FOLDER=template_node_root_folder
VERIFY_SECRETS=true
VAULT_GIT_TOKEN=***
VAULT_MOUNTING_POINT=sesam/kv2
VAULT_URL=https://vault.<org>.io
MASTER_NODE={
  "URL": "datahub-<url>.sesam.cloud",
  "JWT": "***",
  "UPLOAD_VARIABLES": "True",
  "UPLOAD_SECRETS": "True"
}
EXTRA_NODES={
  "<extra-node-name>": {
    "EXTRA_NODE_GIT_URL": "github.com/<user/org>/<repo>.git",
    "EXTRA_NODE_GIT_USERNAME": "<user>",
    "EXTRA_NODE_GIT_TOKEN": "***",
    "EXTRA_NODE_TEMPLATE_PATH": "extra_nodes/<extra-node-name>/"
  }
}
VERIFY_VARIABLES=true
ENVIRONMENT=test
```
## I want to improve this! What can I do?
 * Add support for pushing straight to the extra node, instead of to a git repo. (not my use case, though it might be later)
 * Add support for keyvault 1 (kv1)
 * Add support for different authorization options for vault.
 * Add support for different secret storage services.
 * Add support for customized node configurations (E.g no whitelist files, no variables file.)
 * Add support for checking for changes so it could be used as a microservice on an independent node.
 * Add support for cloning master repo instead of having it available locally (just use functions inside gitter.py) (or build it in azure devops and mount it to the container :J )
 * Improve the readme or give me feedback for improvements.
 * Add more templates to the `template_node_root_folder/extra_nodes/` folder.

