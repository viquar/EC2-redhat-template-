# json template for hosting EC2 instances with user data

{
  "AWSTemplateFormatVersion" : "2010-09-09",
  "Description" : "EC2 RHEL instance",
  "Resources" : {
    "CfnUser" : {
      "Type" : "AWS::IAM::User",
      "Properties" : {
        "Path" : "/",
        "Policies" : [ {
          "PolicyName" : "root",
          "PolicyDocument" : {
            "Statement" : [ {
              "Effect" : "Allow",
              "Action" : "cloudformation:DescribeStackResource",
              "Resource" : "*"
            } ]
          }
        }, {
          "PolicyName" : "s3",
          "PolicyDocument" : {
            "Statement" : [ {
              "Effect" : "Allow",
              "Action" : [ "s3:Get*", "s3:List*" ],
              "Resource" : [ "arn:aws:s3:::*" ]
            } ]
          }
        } ]
      }
    },
    "HostKeys" : {
      "Type" : "AWS::IAM::AccessKey",
      "Properties" : {
        "UserName" : {
          "Ref" : "CfnUser"
        }
      }
    },
    "InstanceProfile" : {
      "Type" : "AWS::IAM::InstanceProfile",
      "Properties" : {
        "Path" : "/",
        "Roles" : [ "role-two" ]
      }
    },
    "AppLaunchConfiguration" : {
      "Type" : "AWS::AutoScaling::LaunchConfiguration",
      "Metadata" : {
        "AWS::CloudFormation::Init" : {
          "config" : {
            "packages" : {
            },
            "files" : {
              "/etc/chef/solo.rb" : {
                "content" : {
                  "Fn::Join" : [ "", [ "log_level :info\n", "log_location STDOUT\n", "file_cache_path \"/var/chef-solo\"\n", "json_attribs \"/etc/chef/node.json\"\n", "cookbook_path \"/etc/chef/cookbooks/\"\n", "data_bag_path \"/etc/chef/cookbooks/data_bags\"\n" ] ]
                },
                "mode" : "000644",
                "owner" : "root",
                "group" : "wheel"
              },
              "/etc/chef/node.json" : {
                "content" : {
                  "run_list" : [ "recipe[swap]", "recipe[users]", "recipe[timezone-ii]", "recipe[iptables]", "tomcat_base::setup", "recipe[cron]" ],
                  "iptables" : {
                    "app" : "tomcat"
                  },
                  "users" : {
                    "env" : "int"
                  },
                  "tomcat" : {
                    "java_options" : "-Xmx2048M -Xms1024M -Djava.awt.headless=true -DAPP_ENV=int"
                  },
                  "cron" : {
                    "bucket" : "chef-cookbook",
                    "app" : "tomcat"
                  }
                },
                "mode" : "000644",
                "owner" : "root",
                "group" : "wheel"
              }
            },
            "sources" : {
              "/etc/chef/cookbooks" : "https://s3.amazonaws.com/chef-template/chef_cookbooksv1.tar"
            }
          }
        },
        "AWS::CloudFormation::Authentication" : {
          "S3AccessCreds" : {
            "type" : "S3",
            "accessKeyId" : {
              "Ref" : "HostKeys"
            },
            "secretKey" : {
              "Fn::GetAtt" : [ "HostKeys", "SecretAccessKey" ]
            },
            "buckets" : "chef-cookbook"
          }
        }
      },
      "Properties" : {
        "ImageId" : "ami-12663b7a",
        "InstanceMonitoring" : "True",
        "InstanceType" : "t2.micro",
        "IamInstanceProfile" : {
          "Fn::GetAtt" : [ "InstanceProfile", "Arn" ]
        },
        "KeyName" : "ec2-one",
        "SecurityGroups" :[ {
          "Ref" : "AppSG"
        }
        ],
        "UserData" : {
          "Fn::Base64" : {
            "Fn::Join" : [ "", [ "#!/bin/bash\n", "curl -o /tmp/aws-cfn-bootstrap-latest.tar.gz  https://s3.amazonaws.com/cloudformation-examples/aws-cfn-bootstrap-latest.tar.gz\n", "tar -zxf /tmp/aws-cfn-bootstrap-latest.tar.gz\n", "cd aws-cfn-bootstrap-*\n", "python setup.py install\n", "/usr/bin/cfn-init -s ", {
              "Ref" : "AWS::StackName"
            }, " -r AppLaunchConfiguration ", " --access-key ", {
              "Ref" : "HostKeys"
            }, " --secret-key ", {
              "Fn::GetAtt" : [ "HostKeys", "SecretAccessKey" ]
            }, " --region ", {
              "Ref" : "AWS::Region"
            }, "\n", "curl -L https://www.opscode.com/chef/install.sh | bash\n", "chef-solo > /var/log/chef-solo.log\n" ] ]
          }
        }
       
      }
    },
    "AppAutoScalingGroup" : {
      "Type" : "AWS::AutoScaling::AutoScalingGroup",
      "Properties" : {
        "AvailabilityZones" : [ "us-east-1a", "us-east-1b" ],
        "Cooldown" : "600",
        "DesiredCapacity" : "1",
        "HealthCheckGracePeriod" : 300,
        "HealthCheckType" : "EC2",
        "LaunchConfigurationName" : {
          "Ref" : "AppLaunchConfiguration"
        },
        "LoadBalancerNames" : [ {
          "Ref" : "AppTierELB"
        } ],
        "MaxSize" : "1",
        "MinSize" : "1",
        "VPCZoneIdentifier" : [ "subnet-7c40ce57", "subnet-8b0142fc" ]
      }
    },
    "AppSG" : {
      "Type" : "AWS::EC2::SecurityGroup",
      "Properties" : {
        "GroupDescription" : "Instance Security Group",
        "VpcId" : "vpc-0ab3826f",
        "SecurityGroupIngress" : [ {
          "IpProtocol" : "tcp",
          "FromPort" : "8080",
          "ToPort" : "8080",
          "SourceSecurityGroupId" : {
            "Ref" : "ASELBSG"
          }
        } ]
      }
    },
    "AppTierELB" : {
      "Type" : "AWS::ElasticLoadBalancing::LoadBalancer",
      "Properties" : {
        "LoadBalancerName" : "ELBone-5-ELBONE-P9ZDYQASZ97W",
        "CrossZone" : "TRUE",
        "HealthCheck" : {
          "HealthyThreshold" : "10",
          "Interval" : "30",
          "Target" : "TCP:8080",
          "Timeout" : "5",
          "UnhealthyThreshold" : "2"
        },
        "Listeners" : [ {
          "LoadBalancerPort" : "80",
          "InstancePort" : "8080",
          "Protocol" : "HTTP"
        } ],
        "Scheme" : "internal",
        "Subnets" : [ "subnet-a416858f", "subnet-76c19501" ],
        "SecurityGroups" : [ {
          "Ref" : "ASELBSG"
        } ]
      }
    },
    "ASELBSG" : {
      "Type" : "AWS::EC2::SecurityGroup",
      "Properties" : {
        "VpcId" : "vpc-0ab3826f",
        "GroupDescription" : "Security Group for ELB",
        "SecurityGroupIngress" : [ {
          "IpProtocol" : "tcp",
          "FromPort" : "80",
          "ToPort" : "80",
          "CidrIp" : "0.0.0.0/0"
        } ]
      }
    }
  }
}
