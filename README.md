Yoke
====

Harness your Lambdas to API Gateway quickly and easily.

```
usage: yoke [-h] [-v] [--debug] {deploy,build,decrypt,encrypt} ...

AWS Lambda + API Gateway Deployment Tool

positional arguments:
  {deploy,build,decrypt,encrypt}
    deploy              Deploy lambda and (optionally) API Gateway.
    build               Only template config files and build lambda package.

optional arguments:
  -h, --help            show this help message and exit
  -v, --version         show program's version number and exit
  --debug               Debug logging
```


# Getting Started
1. Add a [`yoke.yml`](#yokeyml) file to the root of the project.
2. Set the appropriate config values in `yoke.yml` for you Lambda, API Gateway (optional), and stages.
3. Test your configuration by running `yoke build --stage <stagename>`. Given the following example structure there will be some new files generated:

        myproject/
        |-- yoke.yml
        |-- swagger.yml (Templated Swagger file generated by Yoke {build,deploy})
        |-- swagger_template.yml
        |-- myapplication/ (Lambda path defined in yoke.yml)
        |   |-- handler.py (Lambda handler defined in yoke.yml)
        |   |-- lambda.json (Lambda function configuration generated by Yoke {build,deploy})
        |   |-- config.json (Config values for your application generated by Yoke {build,deploy})
This will let you verify that the `swagger.yml` and `config.json` are templated as desired.
4. Run `yoke deploy --stage <stagename>` to deploy your Lambda and (optionally) API Gateway.


# yoke.yml

The heart of Yoke is the `yoke.yml` file. This file tells Yoke all the necessary information it needs to template and upload your Lambda and API Gateway. The structure of the file is as follows:

* `Lambda`: Information about the Lambda function to deploy and how to build it.
  * `config`: Specific information about the Lambda function's configuration:
    * `name`: The name of the function.
    * `description`: Description of the function.
    * `handler`: The module to handle the Lambda event.
    * `timeout`: The number of seconds before timing out the function.
    * `memory`: The amount (in Megabytes) of memory to configure for the function.
    * `ignore`: A list of regex patterns of files to exclude when building and uploading the function.
    * `role`: The IAM role to assume when this function is run.
  * `path`: The path to the root of your Lambda module.
  * `extraFiles`: A list of additional files or directories to include in the Lambda package when uploading. This is useful for including requirements using `pip install -t <requirements_directory> <somepackage>`.
* `apiGateway`: Optional - information about API Gateway configuration.
  * `name`: The name of the API Gateway.
  * `swaggerTemplate`: Path to the Swagger template file.
  * `role`: The IAM role to assume when API Gateway runs.
  * `customAuthorizer`: Optional - Custom Authorizer configuration.
    * `name`: The name of the Custom Authorizer
    * `role`: The role to assume when running the Custom Authorizer
    * `expression`: The matching expression used to validate the authorization header before it is passed to the Custom Authorizer.
    * `header`: The (case-sensitive) name of the header passed to the Custom Authorizer.
    * `ttl`: The time (in seconds) to cache authorization responses.
* `stages`: Application configuration per deployment stage.
  * `default`: The default stage - the configuration for this stage is applied to any stages not explicitly defined in this section. For example, if you have stages `default, dev, and prod`, but run `yoke deploy --stage awesome`, Yoke will use the configuration for the `default` stage.
    * `region`: What region to deploy to for this stage.
    * `keyName`: Optional - KMS key alias used to encrypt and decrypt the `secretConfig` section for this stage.
    * `keyRegion`: Optional - The region where `keyName` exists.
    * `secretConfig`: Optional - encrypted configuration for the stage - this section is decrypted and combined with `config` when running `yoke build` or `yoke deploy` and written to `config.json` in the `Lambda` path.
    * `config`: Optional - These values are combined with `secretConfig` and written to `config.json` in the `Lambda` path.

You can also template `yoke.yml` using Jinja-style templating. When you run `yoke {build,deploy}`, these template variables will be sourced from any variables you privide via `--environment/-e`. By default, `{{ stage }}` is automatically provided as it is a required argument for all operations.

For more information, see the [examples in this repo](examples/).


# Handling Secrets
So you have some secrets, it's ok, we all do - but let's not share them with everyone. Here's how you can encrypt your Lambda's configuration secrets.

### Encrypting `secretConfig`
In the `stages` section of `yoke.yml`, add a `secretConfig` section with plaintext secrets (don't worry, we won't keep it this way):

```
stages:
  default: &DEFAULT
    region: "us-east-1"
    keyName: my_kms_key
    keyRegion: us-east-1
    secretConfig:
      somesecret: "secret value"
      anothersecret: "ssshhhhhhhhh"
    config: &DEFAULT_CONFIG
      log_level: DEBUG
```

Once you have your secrets, run `yoke encrypt --stage <stagename>` to generate the encrypted secret. Replace `secretConfig` with the encrypted string:

```
stages:
  default: &DEFAULT
    region: "us-east-1"
    keyName: my_kms_key
    keyRegion: us-east-1
    secretConfig: "CiCCM/++OaG61j7Ld8RPAE32b1LZBas6Or1EVAwZxdThARLnAQEBAgB4gjP/vjmhutY+y3fETwBN9m9S2QWrOjq9RFQMGcXU4QEAAAC+MIG7BgkqhkiG9w0BBwagga0wgaoCAQAwgaQGCSqGSIb3DQEHATAeBglghkgBZQMEAS4wEQQMJesOjJpuXDNgIfyKAgEQgHd7N0goSpc013D8CuSwaVqVWawgp9PJ6F/TkKKGFQVmL7PT+x72EknOFXNUriorHLBX/FBvopkRZLqxRTxRyW9T/Lwm1lXZjHuNF/JGNLr9+F9W8uNTREHf7ZjhfxHMmBG06oBtIWnwa0E8hPZ+XMy7TpzUu/DMXA=="
    config: &DEFAULT_CONFIG
      log_level: DEBUG
```

When you run `yoke build` or `yoke deploy`, `secretConfig` is decrypted and added to the generated `config.json`.


### Decrypting `secretConfig`

`yoke decrypt --stage <stagename>` will decrypt the `secretConfig` section for the specified stage and display it in your terminal.

### Updating secrets
1. Run `yoke decrypt --stage <stagename>` to get the plaintext secrets.
2. In `yoke.yml`, replace `secretConfig`'s value with the plaintext from `decrypt`.
3. Add values, change values, etc. to the `secretConfig` section.
4. Run `yoke encrypt --stage <stagename>` and copy the ciphertext output.
5. Replace the `secretConfig` value with the ciphertext from step 4.
