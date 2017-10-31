# TerraformDevKit

[![Build Status](https://travis-ci.org/vistaprint/TerraformDevKit.svg?branch=master)](https://travis-ci.org/vistaprint/TerraformDevKit) [![Build status](https://ci.appveyor.com/api/projects/status/4vkyr196li83vju6/branch/master?svg=true)](https://ci.appveyor.com/project/vistaprint/terraformdevkit/branch/master)

Set of scripts to ease development and testing with [Terraform](https://www.terraform.io/).

The script collection includes support for:

* Managing AWS credentials
* Backing up the state from a failed Terraform execution
* Executing external commands
* Simple configuration management
* Simple reading and writing to AWS DynamoDB
* Multiplatform tools
* Making simple HTTP requests
* Retrying a block of code
* Terraform environment management
* Locally installing Terraform and [Terragrunt](https://github.com/gruntwork-io/terragrunt)
* Filtering Terraform logging messages

Most of these scripts exist to provide support to a module development and testing environment for Terraform: [TerraformModules](https://github.com/vistaprint/TerraformModules). But, they might be useful for other purposes too.

## Installation

Add this line to your application's Gemfile:

```ruby
gem 'TerraformDevKit'
```

And then execute:

    $ bundle

Or install it yourself as:

    $ gem install TerraformDevKit

## Usage

To use the library simply import it as:

```ruby
require 'TerraformDevKit'
```    

## Managing Terraform Environments

TerraformDevKit provides a set of Rake tasks and Ruby scripts to ease the management of multiple Terraform environments. Three major environment types are supported: `dev`, `test` and `prod`.

There might be many development environments (`dev`), each one with its own name. Development environments use a local Terraform backend. They are intented to be used by developers while adding features to the infrastructure.

Testing (`test`) and production (`prod`) environment use a remote backend. Thus, the Terraform state file is not kept in the local disk, but on S3. This allows multiple developers to easily work on the same infrastructure instance. For safety reasons, operations that affect testing and production environments require manual user input. This is not the case for development environments.

TerraformDevKit expects templated versions (using [Mustache](https://mustache.github.io/)) of the Terraform files. Such files might contain placeholders for several fields such as `Environment`, (AWS) `Region` or (AWS) `Profile`, among others. TerraformDevKit uses the template files to generate the final files that will be consumed by Terraform and Terragrunt. As an example, for the production environment, the resulting files are placed in a directory named `envs/prod`.

### A Minimal Rakefile

```ruby
ROOT_PATH = File.dirname(File.expand_path(__FILE__))

spec = Gem::Specification.find_by_name 'TerraformDevKit'
load "#{spec.gem_dir}/tasks/devkit.rake"

task :custom_test, [:env] do |_, args|
  # Add some tests here
end
```

### Sample Terraform/Terragrunt Templates

The following file (`main.tf.mustache`) contains the infrastructure configuration (a single S3 bucket) as well as information related to the AWS provider.

```hcl
locals {
  env    = "{{Environment}}"
}

terraform {
    backend {{#LocalBackend}}"local"{{/LocalBackend}}{{^LocalBackend}}"s3"{{/LocalBackend}} {}
}

provider "aws" {
  profile = "{{Profile}}"
  region  = "{{Region}}"
}

resource "aws_s3_bucket" "raw" {
  bucket = "foo-${local.env}"
  acl    = "private"

{{#LocalBackend}}
  force_destroy = true
{{/LocalBackend}}
{{^LocalBackend}}
  lifecycle {
    prevent_destroy = true
  }
{{/LocalBackend}}
}
```

An additional file (`terraform.tfvars.mustache`) contains the Terragrunt configuration. Terragrunt is used as it makes it easier to manage a remote backend. While Terraform is able to keep the state file in S3, it does not transparently create the DynamoDB lock table that prevents multiple developers from concurrently modifying the state file. Terragrunt, however, is able to do this without user intervention.

```hcl
terragrunt = {
  remote_state {
  {{#LocalBackend}}
    backend = "local"
    config {}
  {{/LocalBackend}}
  {{^LocalBackend}}
    backend = "s3"
    config {
      bucket     = "foo-remote-state"
      key        = "foo-{{Environment}}.tfstate"
      lock_table = "foo-{{Environment}}-lock-table"
      encrypt    = true
      profile    = "{{Profile}}"
      region     = "{{Region}}"
    }
  {{/LocalBackend}}
  }
}
```

## Development

After checking out the repo, run `bin/setup` to install dependencies. Then, run `rake spec` to run the tests. You can also run `bin/console` for an interactive prompt that will allow you to experiment.

To install this gem onto your local machine, run `bundle exec rake install`. To release a new version, update the version number in `version.rb`, and then run `bundle exec rake release`, which will create a git tag for the version, push git commits and tags, and push the `.gem` file to [rubygems.org](https://rubygems.org).

## Contributing

Bug reports and pull requests are welcome on GitHub at https://github.com/vistaprint/TerraformDevKit.

## License

The gem is available as open source under the terms of the [Apache License, Version 2.0](https://opensource.org/licenses/Apache-2.0).
