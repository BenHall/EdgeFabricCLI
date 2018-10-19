# C8Fn CLI

C8Fn CLI is a tool to build and deploy serverless-functions to Macrometa Edge Fabric. 

You can build C8Fn functions from a set of supported language templates (such as Node.js, Python, CSharp and Ruby). That means you just write a handler file such as (handler.py/handler.js) and the CLI does the rest to create a Docker image.

## Run the CLI

The main commands supported by the CLI are:

* `c8fn-cli new` - creates a new function via a template in the current directory
* `c8fn-cli build` - builds Docker images from the supported language types
* `c8fn-cli push` - pushes Docker images into a registry
* `c8fn-cli deploy` - deploys the functions into a local or remote C8Fn gateway
* `c8fn-cli remove` - removes the functions from a local or remote C8Fn gateway
* `c8fn-cli invoke` - invokes the functions and reads from STDIN for the body of the request

The default gateway URL of `127.0.0.1:8080` can be overriden in three places including an environmental variable.

* 1st priority `--gateway` flag passed in on the command line
* 2nd priority `--yaml` / `-f` flag or `stack.yml` if in current directory
* 3rd priority `C8Fn_URL` environmental variable

Other parameters which will be overridden in the function's YAML config file when passed in from the command line are:

* `--tenant`
* `--edge-locations`

### Advanced commands

* `c8fn-cli template pull` - pull in templates from a remote GitHub repository [Detailed Documentation](TEMPLATE.md)

Help for all of the commands supported by the CLI can be found by running:

* `c8fn-cli help` or `c8fn-cli [command] --help`

You can chose between using a programming language template where you only need to provide a handler file, or a Docker that you can build yourself.

## Sample workflow

This series of steps demonstrates how to `create, build, push, deploy, list, invoke and remove` a python Function in in a Macrometa C8 Edge Data Fabric environment.

The python function has a very simple functionality - it just echoes the input that was passed when invoking the function.

**NOTE:**

All mentions of the edge fabric details in the examples below are just that - examples.

Please remember to substitute them with the details specific to edge fabric information provided to you.

1. Create a python function called `hello-pydemo` using the `new` command:

    ```bash
    mkdir hello-pydemo
    cd hello-pydemo
    c8fn-cli new --lang python hello-pydemo -p macrometa -t tenant1 -g http://fabric.macrometa.io -n 'fabric-us-west-2, fabric-eu-central-1'
    ```
2. Edit the function handler code as follows:

    ```python
        def handle(req):
            """handle a request to the function
            Args:
                req (str): request body
            """ 
            print("Hello from C8Fn pydemo. ")
            return "You said: "+req
    ```
    
3. Use the `build` command, build the new function we created above as a Docker image:

    ```bash
        c8fn-cli build -f hello-pydemo.yml
    ```

4. Use the `push` command, push the built function to your dockerhub repo:

    ```bash
    c8fn-cli push -f hello-pydemo.yml 
    ```
    **Note:** You can also just use `docker push` if you prefer.
    
    **IMPORTANT NOTE:** After the `push`, make sure to set the function repo to `public` in your Dockerhub, otherwise function invocation **may fail.**

5. Use the `deploy` command, deploy the function onto your C8 Edge Fabric setup:

     a. Example deploy with all command line options, these will **override** the corresponding settings in the YAML definition file for the function :
  
    ```bash
        c8fn-cli deploy -f hello-pydemo.yml -t tenant1 -g http://fabric.macrometa.io -n 'fabric-us-west-2, fabric-eu-central-1'
    ```
    
    b. Example deploy using the parameters in the YAML file:
    ```
        c8fn-cli deploy -f hello-pydemo.yml
    ```
    
6. Use the `list` command to check whether function is available:

    ```bash
    c8fn-cli list -g http://fabric.macrometa.io
    ```
    Example output:
    ```bash
    Function        Invocations    	Replicas	Edge Locations                      
    --------        -----------    	--------	--------------                        
    hello-pydemo    0              	1    	fabric-us-west-2, fabric-eu-central-1
    ```

7. Use the `invoke` command to invoke the function from your C8 Edge Fabric deployment:

    ```bash
    echo 'Hello' | c8fn-cli invoke hello-pydemo -g http://fabric.macrometa.io
    ```

   via curl:
   
   ```bash
   curl -X POST --header 'Content-Type: application/json' --header 'Accept: application/json' 'http://fabric.macrometa.ioc8/_fn/_tenant/tenant1/sync/hello-pydemo'
   ```
   
   Example output:
    ```
    "Hello from C8Fn pydemo. You said: Hello"
    ```
    
8. Use the `remove` command to remove/delete the deployed function:

    ```bash
    c8fn-cli remove hello-pydemo -g http://fabric.macrometa.io 
    ```


## Templates

Command: `c8fn-cli new FUNCTION_NAME --lang python/node/go/ruby/Dockerfile/etc`

In your YAML you can also specify `lang: node/python/go/csharp/ruby`

* Supports common languages
* Quick and easy - just write one file
* Specify depenencies on Gemfile / requirements.txt or package.json etc

* Customise the provided templates

Perhaps you need to have `gcc` or another dependency in your Python template? That's not a problem.

You can customise the Dockerfile or code for any of the templates. Just create a new directory and copy in the templates folder from this repository. The templates in your current working directory are always used for builds.

See also: `c8fn-cli new --help`

**Third-party community templates**

Templates created and maintained by a third-party can be added to your local system using the `c8fn-cli template pull` command.

Curated language templates:

| Language  | Author   | URL  |
|---|---|---|
| PHP | @itscaro   | https://github.com/itscaro/c8fn-template-php/  |
| PHP5 | @itscaro   | https://github.com/itscaro/c8fn-template-php/  |
| F# | @hayer | https://github.com/hayer/c8fn-fsharp-template/ |

Read more on [community templates here](TEMPLATE.md).

## Docker image as a function

Specify `lang: Dockerfile` if you want the c8fn-cli to execute a build or `skip_build: true` for pre-built images.

* Ultimate versatility and control
* Package anything
* If you are using a stack file add the `skip_build: true` attribute

## Private registries

Create a named image pull secret and add the secret name to the `secrets` section of your YAML file or your deployment arguments with `--secret`.

## Use a YAML stack file

A YAML stack file groups functions together and also saves on typing.

You can define individual functions or a set of of them within a YAML file. This makes the CLI easier to use and means you can use this file to deploy to your C8Fn instance.  By default the c8fn-cli will attempt to load `stack.yaml` from the current directory.

Here is an example file using the `stack.yml` file included in the repository.

```yaml
provider:
  name: c8fn
  gateway: https://fabric.macrometa.io  # url of macrometa edge fabric
  tenant: guest # name of the tenant

functions:
  url-ping:
    lang: python
    handler: ./sample/url-ping
    image: alexellis2/url-ping
    edge_locations:
      - fabric-us-east-1
```

This url-ping function is defined in the sample/url-ping folder makes use of Python. All we had to do was to write a `handler.py` file and then to list off any Python modules in `requirements.txt`.

* Build the files in the .yml file:

```
$ c8fn-cli build -f ./stack.yml
```

> `-f` specifies the file or URL to download your YAML file from. The long version of the `-f` flag is: `--yaml`.

You can also download over HTTP/s:

```
$ c8fn-cli build -f https://<PATH_TO_YAML_FILE_ON_THE_WEB>
```

Docker along with a Python template will be used to build an image named alexellis2/c8fn-urlping.

* Deploy your function

Now you can use the following command to deploy your function(s):

```
$ c8fn-cli deploy -f ./stack.yml
```

## YAML format reference

### Environmental variables/configuration

You can deploy non-encrypted secrets and configuration via environmental variables set either in-line or via external (environment) files.

> Note: external files take priority over in-line environmental variables. This allows you to specify a default and then have overrides within an external file.

Priority:

* environment_file - defined in zero to many external files

```yaml
  environment_file:
    - file1.yml
    - file2.yml
```

If you specify a variable such as "access_key" in more than one `environment_file` file then the last file in the list will take priority.

Environment file format:

```yaml
environment:
  access_key: key1
  secret_key: key2
```

* Define environment in-line within the file:

Imagine you needed to define a `http_proxy` variable to operate within a corporate network:

```yaml
functions:
  url-ping:
    lang: python
    handler: ./sample/url-ping
    image: alexellis2/url-ping
    edge_locations:
        - fabric-us-east-1
    environment:
      http_proxy: http://proxy1.corp.com:3128
      no_proxy: http://gateway/
```

### Other YAML fields

The possible entries for functions are documented below:

```yaml
functions:
  deployed_function_name:
    lang: node or python (optional)
    handler: ./path/to/handler (optional)
    image: docker-image-name
    environment:
      env1: value1
      env2: "value2"
    labels:
      label1: value1
      label2: "value2"
#   limits:
#     memory: 40m
#   requests:
#     memory: 40m 
   edge_locations:
     - <EDGE_LOC_1>
     - ...
```

Use environmental variables for setting tokens and configuration.

------

