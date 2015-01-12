---
layout: default
---

This is a `format converter` + `useful tool`.
It supports the following format:

* JSON
* Ruby
* YAML
* JavaScript
* CoffeeScript (experimental)
* JSON5 (experimental)

[![Gem Version](https://badge.fury.io/rb/kumogata.svg?201406152020)](http://badge.fury.io/rb/kumogata)
[![Build Status](https://travis-ci.org/winebarrel/kumogata.svg?branch=master)](https://travis-ci.org/winebarrel/kumogata)

It can define a template in Ruby DSL, such as:

{% highlight ruby %}
AWSTemplateFormatVersion "2010-09-09"

Description (<<-EOS).undent
  Kumogata Sample Template
  You can use Here document!
EOS

Parameters do
  InstanceType do
    Default "t1.micro"
    Description "Instance Type"
    Type "String"
  end
end

Resources do
  myEC2Instance do
    Type "AWS::EC2::Instance"
    Properties do
      ImageId "ami-XXXXXXXX"
      InstanceType { Ref "InstanceType" }
      KeyName "your_key_name"

      UserData do
        Fn__Base64 (<<-EOS).undent
          #!/bin/bash
          yum install -y httpd
          service httpd start
        EOS
      end
    end
  end
end

Outputs do
  AZ do
    Value do
      Fn__GetAtt "myEC2Instance", "AvailabilityZone"
    end
  end
end
{% endhighlight %}

Ruby template structure is almost the same as [JSON template](http://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/template-structure.html).

(**You can also use JSON templates**)

## Installation

    $ gem install kumogata

## Usage

{% highlight sh %}
Usage: kumogata <command> [args] [options]

Commands:
  create         PATH_OR_URL [STACK_NAME]   Create resources as specified in the template
  validate       PATH_OR_URL                Validate a specified template
  convert        PATH_OR_URL                Convert a template format
  update         PATH_OR_URL STACK_NAME     Update a stack as specified in the template
  delete         STACK_NAME                 Delete a specified stack
  list           [STACK_NAME]               List summary information for stacks
  export         STACK_NAME                 Export a template from a specified stack
  show-events    STACK_NAME                 Show events for a specified stack
  show-outputs   STACK_NAME                 Show outputs for a specified stack
  show-resources STACK_NAME                 Show resources for a specified stack
  diff           PATH_OR_URL1 PATH_OR_URL2  Compare templates logically (file, http://..., stack://...)

Options:
    -k, --access-key ACCESS_KEY
    -s, --secret-key SECRET_KEY
    -r, --region REGION
        --profile CONFIG_PROFILE
        --credentials-path PATH
        --format TMPLATE_FORMAT
        --output-format FORMAT
        --skip-replace-underscore
        --deletion-policy-retain
    -p, --parameters KEY_VALUES
    -j, --json-parameters JSON
    -e, --encrypt-parameters KEYS
        --encryption-password PASS
        --skip-send-password
        --capabilities CAPABILITIES
        --disable-rollback
        --notify SNS_TOPICS
        --timeout MINUTES
        --result-log PATH
        --command-result-log PATH
        --detach
        --force
    -w, --ignore-all-space
        --color
        --no-color
        --debug
    -v, --verbose
{% endhighlight %}

### KUMOGATA_OPTIONS

`KUMOGATA_OPTIONS` variable specifies default options.

e.g. `KUMOGATA_OPTIONS='-e Password'`

### Create resources

    $ kumogata create template.rb

If you want to save the stack, please specify the stack name:

    $ kumogata create template.rb any_stack_name

If you want to pass parameters, please use `-p` option:

    $ kumogata create template.rb -p "InstanceType=m1.large,KeyName=any_other_key"


**Notice**

**The stack will be delete if you do not specify the stack name explicitly.**
(And only the resources will remain)

### Convert JSON to Ruby

JSON template can be converted to Ruby template.

    $ kumogata convert https://s3.amazonaws.com/cloudformation-templates-us-east-1/Drupal_Single_Instance.template

* Data that cannot be converted will be converted to Array and Hash
* `::` is converted to `__`
  * `Fn::GetAtt` => `Fn__GetAtt`
* `_{ ... }` is convered to Hash
  * `SecurityGroups [_{Ref "WebServerSecurityGroup"}]` => `{"SecurityGroups": [{"Ref": "WebServerSecurityGroup"}]}`
* `_path()` creates Hash that has a key of path
  * `_path("/etc/passwd-s3fs") { content "..." }` => `{"/etc/passwd-s3fs": {"content": "..."}}`
* ~~_user_data() creates Base64-encoded UserData~~
  * `_user_data()` has been removed
* `_join()` has been removed

### String#fn_join()

Ruby templates will be converted as follows by `String#fn_join()`:

{% highlight ruby %}
UserData do
  Fn__Base64 (<<-EOS).fn_join
    #!/bin/bash
    /opt/aws/bin/cfn-init -s <%= Ref "AWS::StackName" %> -r myEC2Instance --region <%= Ref "AWS::Region" %>
  EOS
end
{% endhighlight %}

{% highlight javascript %}
"UserData": {
  "Fn::Base64": {
    "Fn::Join": [
      "",
      [
        "#!/bin/bash\n",
        "/opt/aws/bin/cfn-init -s ",
        {
          "Ref": "AWS::StackName"
        },
        " -r myEC2Instance --region ",
        {
          "Ref": "AWS::Region"
        },
        "\n"
      ]
    ]
  }
}
{% endhighlight %}

### Split a template file

* template.rb

{% highlight ruby %}
Resources do
  _include 'template2.rb'
end
{% endhighlight %}

* template2.rb

{% highlight ruby %}
myEC2Instance do
  Type "AWS::EC2::Instance"
  Properties do
    ImageId "ami-XXXXXXXX"
    InstanceType { Ref "InstanceType" }
    KeyName "your_key_name"
  end
end
{% endhighlight %}

* Converted JSON template

{% highlight javascript %}
{
  "Resources": {
    "myEC2Instance": {
      "Type": "AWS::EC2::Instance",
      "Properties": {
        "ImageId": "ami-XXXXXXXX",
        "InstanceType": {
          "Ref": "InstanceType"
        },
        "KeyName": "your_key_name"
      }
    }
  }
}
{% endhighlight %}

### Encrypt parameters

* Command line

{% highlight sh %}
$ kumogata create template.rb -e 'Password1,Password2' -p 'Param1=xxx,Param2=xxx,Password1=xxx,Password2=xxx'
{% endhighlight %}

* Template

{% highlight ruby %}
Parameters do
  Param1 { Type "String" }
  Param2 { Type "String" }
  Password1 { Type "String"; NoEcho true }
  Password2 { Type "String"; NoEcho true }
end # Parameters

Resources do
  myEC2Instance do
    Type "AWS::EC2::Instance"

    Properties do
      ImageId "ami-XXXXXXXX"

      UserData do
        Fn__Base64 (<<-EOS).fn_join
          #!/bin/bash
          /opt/aws/bin/cfn-init -s <%= Ref "AWS::StackName" %> -r myEC2Instance --region <%= Ref "AWS::Region" %>
        EOS
      end
    end

    Metadata do
      AWS__CloudFormation__Init do
        config do
          commands do
            any_command do
              command (<<-EOS).fn_join
                ENCRYPTION_PASSWORD="`echo '<%= Ref Kumogata::ENCRYPTION_PASSWORD %>' | base64 -d`"

                # Decrypt Password1
                echo '<%= Ref "Password1" %>' | base64 -d | openssl enc -d -aes256 -pass pass:"$ENCRYPTION_PASSWORD" > password1

                # Decrypt Password2
                echo '<%= Ref "Password2" %>' | base64 -d | openssl enc -d -aes256 -pass pass:"$ENCRYPTION_PASSWORD" > password2
              EOS
            end
          end
        end
      end
    end
  end # myEC2Instance
end # Resources
{% endhighlight %}

## Iteration

You can use the Iteration in the template using `_(...)` method.

{% highlight ruby %}
Resources do
  ['instance1', 'instance2', 'instance3'].echo {|instance_name|
    _(instance_name) do
      Type "AWS::EC2::Instance"
      Properties do
        ImageId "ami-XXXXXXXX"
        InstanceType { Ref "InstanceType" }
        KeyName "your_key_name"

        UserData (<<-EOS).undent.encode64
          #!/bin/bash
          yum install -y httpd
          service httpd start
          hostname #{instance_name}
        EOS
      end
    end
  }
end
{% endhighlight %}

## Post command

You can run shell/ssh commands after building servers using `_post()`.

* Template
{% highlight ruby %}
Parameters do
  ...
end

Resources do
  ...
end

Outputs do
  MyPublicIp do
    Value { Fn__GetAtt name, "PublicIp" }
  end
end

_post do
  my_shell_command do
    command <<-EOS
      echo <%= Key "MyPublicIp" %>
    EOS
  end
  my_ssh_command do
    ssh do
      host { Key "MyPublicIp" } # or '<%= Key "MyPublicIp" %>'
      user "ec2-user"
      # see http://net-ssh.github.io/net-ssh/classes/Net/SSH.html#method-c-start
      #options :timeout => 300
      #connect_tries 36
      #retry_interval 5
      #request_pty true
    end
    command <<-EOS
      hostname
    EOS
  end
end
{% endhighlight %}

* Execution result
{% highlight sh %}
...
Command: my_shell_command
Status: 0
1> 54.199.251.30

Command: my_ssh_command
Status: 0
1> ip-10-0-129-20

(Save to `/foo/bar/command_result.json`)
{% endhighlight %}

## JavaScript template

You can also use the JavaScript template instead of JSON and Ruby.

{% highlight javascript %}
function fetch_ami() {
  return "ami-XXXXXXXX";
}

/* For JS Object is evaluated last, it must be enclosed in parentheses */
({
  Resources: { /* comment */
    myEC2Instance: {
      Type: "AWS::EC2::Instance",
      Properties: {
        ImageId: fetch_ami(),
        InstanceType: "t1.micro"
      }
    }
  },
  Outputs: {
    AZ: { /* comment */
      Value: {
        "Fn::GetAtt": [
          "myEC2Instance",
          "AvailabilityZone"
        ]
      }
    }
  }
})

/*
 {
   "Resources": {
     "myEC2Instance": {
       "Type": "AWS::EC2::Instance",
       "Properties": {
         "ImageId": "ami-XXXXXXXX",
         "InstanceType": "t1.micro"
       }
     }
   },
   "Outputs": {
     "AZ": {
       "Value": {
         "Fn::GetAtt": [
           "myEC2Instance",
           "AvailabilityZone"
         ]
       }
     }
   }
 }
 */
{% endhighlight %}

### Convert JSON template to JavaScript

    $ kumogata convert Drupal_Single_Instance.template --output-format=js

## CoffeeScript template

You can also use the CoffeeScript template instead of JSON and Ruby.

{% highlight coffeescript %}
fetch_ami = () -> "ami-XXXXXXXX"

/* For JS Object is evaluated last, it must use `return` */
return {
  Resources:
    myEC2Instance:
      Type: "AWS::EC2::Instance",
      Properties:
        ImageId: fetch_ami(),
        InstanceType: "t1.micro"
  Outputs:
    AZ: # comment
      Value:
        "Fn::GetAtt": [
          "myEC2Instance",
          "AvailabilityZone"
        ]
}

###
 {
   "Resources": {
     "myEC2Instance": {
       "Type": "AWS::EC2::Instance",
       "Properties": {
         "ImageId": "ami-XXXXXXXX",
         "InstanceType": "t1.micro"
       }
     }
   },
   "Outputs": {
     "AZ": {
       "Value": {
         "Fn::GetAtt": [
           "myEC2Instance",
           "AvailabilityZone"
         ]
       }
     }
   }
 }
###
{% endhighlight %}

## YAML template

You can also use the YAML template instead of JSON and Ruby.

{% highlight yaml %}
---
Resources:
  myEC2Instance:
    Type: AWS::EC2::Instance
    Properties:
      ImageId: ami-XXXXXXXX
      InstanceType: t1.micro
Outputs:
  AZ:
    Value:
      Fn::GetAtt:
      - myEC2Instance
      - AvailabilityZone

# {
#   "Resources": {
#     "myEC2Instance": {
#       "Type": "AWS::EC2::Instance",
#       "Properties": {
#         "ImageId": "ami-XXXXXXXX",
#         "InstanceType": "t1.micro"
#       }
#     }
#   },
#   "Outputs": {
#     "AZ": {
#       "Value": {
#         "Fn::GetAtt": [
#           "myEC2Instance",
#           "AvailabilityZone"
#         ]
#       }
#     }
#   }
# }
{% endhighlight %}

### Convert JSON template to YAML

    $ kumogata convert Drupal_Single_Instance.template --output-format=yaml

## [JSON5](http://json5.org/) template

You can also use the [JSON5](http://json5.org/) template instead of JSON and Ruby.

{% highlight javascript %}
{
  Resources: { /* comment */
    myEC2Instance: {
      Type: "AWS::EC2::Instance",
      Properties: {
        ImageId: "ami-XXXXXXXX",
        InstanceType: "t1.micro"
      }
    }
  },
  Outputs: {
    AZ: { /* comment */
      Value: {
        "Fn::GetAtt": [
          "myEC2Instance",
          "AvailabilityZone"
        ]
      }
    }
  }
}

/*
 {
   "Resources": {
     "myEC2Instance": {
       "Type": "AWS::EC2::Instance",
       "Properties": {
         "ImageId": "ami-XXXXXXXX",
         "InstanceType": "t1.micro"
       }
     }
   },
   "Outputs": {
     "AZ": {
       "Value": {
         "Fn::GetAtt": [
           "myEC2Instance",
           "AvailabilityZone"
         ]
       }
     }
   }
 }
 */
{% endhighlight %}

## Outputs Filter

{% highlight ruby %}
Outputs do
  MyPublicIp do
    Value { Fn__GetAtt "MyInstance", "PublicIp" }
  end
end

_outputs_filter do |output|
  outputs["MyPublicIp"].gsub!('.', '_')
  # MyPublicIp: XXX.XXX.XXX.XXX => XXX-XXX-XXX-XXX
end

_post do
  ...
end
{% endhighlight %}

## Configuration File

Kumogata supports [aws-sdk configuration file](http://docs.aws.amazon.com/AWSSdkDocsRuby/latest/DeveloperGuide/ruby-dg-setup.html#set-up-creds).

{% highlight ini %}
[default]
aws_access_key_id=AKIAIOSFODNN7EXAMPLE
aws_secret_access_key=wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY
aws_session_token=texample123324
{% endhighlight %}

## Demo

* [Create resources](https://asciinema.org/a/7979)
* [Convert a template](https://asciinema.org/a/7980)
* [Create a stack while outputting the event log](https://asciinema.org/a/8075)
* [Create a stack and run post commands](https://asciinema.org/a/8088)

## Similar tools
* [Codenize.tools](http://codenize.tools/)
