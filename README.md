# aws-elasticsearch-opsworks

Chef cookbook for installing Elasticsearch v5.0, kibana, x-pack and nginx on AWS Opsworks



## Setting up opworks


Create a stack with CHEF 12 and Amazon linux.  Make sure it has IAM user of placefull-search-? Which ever one you are launching.

The custom JSON in OPsWorks Stack Settings needs to have
{ 
"java":{
"install_flavor":"oracle",
"jdk_version":"8",
"oracle": {
"accept_oracle_download_terms":"true"
}
}
}


Opsworks repo url should be set to smenchions/aws-elasticsearch-opsworks/placefull-elasticsearch-5.0.git
That repo is the actual Chef cookbook that is used to install everything

Set EC2 instances to be us-east-1b   if you want to change this, then you need to also change the elasticsearch configuration in the default.rb 
because it points the discovery to us-east-1b

Add a layer and call it search

Under Layer > Settings > Recipes add placefull-elasticsearch-5 as a recipe in the setup section

Under Layer > Settings>Security add security group which opens up port 8181 (kibana) and 8080 (ES).  These ports
allow you to access the underlying ES and Kibana from the web.  They are password protected with NGINX
which this cookbook also installs


Launch 3 instances in the stack together.  You need to launch a minimum of three, no max.

Once they come up, navigate to ipaddress:8080

user is placefull
password is ??? Set the user/password by reading the README.md of the included placefull-nginx


That is the endpoint for elasticsearch.
If when you go to 8080 you get the Ngnix gateway error, it means that either not all three servers are up and “online” or they are but ES failed to find the three and start a cluster.  It timed out.  Resolve this by rebooting all three of the online servers together.  When they restart an ES cluster should form.

navigate to ipaddress:8181

same user:pass

That is the endpoint for kibana

Nginx is used to reroute these ports 8080 8181 and provide password protection to the internal ES 9200 and kibana 5601.


The newest Elasticsearch 5.0 now requires a license.  You can get a basic license for free and download it from elasticsearch website. 

Then using curl do this from the directory you have the license file in

curl -XPUT http://placefull:yourpassword@youripaddress:8080/_xpack/license?acknowledge=true -d @es-license-file.json

Do it for all three of your nodes


### Detailed Instructions on how to recreate this cookbook when ElasticSearch updates to new versions



The following is a detailed account of how to create the the wrapper cookbook from scratch. This is not required however, 
because it has already been done and is what is currently placefull-elasticsearch-5 and is the one that AWS Opsworks points to. 
Only follow these instructions in case you wish to rebuild this repo from a newer elasticsearch cookbook supplied by
elastic.cs



Create a new wrapper cookbook to run the above elasticsearch cookbook with the setup you desire.  This wrapper cookbook will also install kibana for visualization, Nginx for http password security and xPack for cluster monitoring

Here are the instructions to do it:

Download the Chef development Kit. We need this to create and then “vendor” the wrapper cookbook. We need to vendor it because Chef 12 on AWS Opsworks does not have Berksfile installed. This means that the linux instance cannot automatically pull down all the cookbook dependencies it needs from berksfiles. Instead when we vendor it, it copies all of the dependencies to your PC and then you put those until source control

Start it up with Windows Admin privs

Go to the directory you wish to create the new wrapper cookbook and create the cookbook with ‘knife cookbook create placefull-es-wrapper’ or whatever name you want

Ensure this new cookbook is within a directory on your harddrive where you can add more directories.  When you vendor it, it will create many more directories at the same level as this cookbooks directory.  Put this directory is under git source control.

To install java 8 on the linux boxes, you need to add Java cookbook as a dependency in the berksfile. You also need to set a depends on the metadata.rb

To install elasticsearch you need to put a depends in the metadata.rb for the elasticsearch cookbook

You also need to install nginx and kibana using cookbooks. These are referenced as depends in the metadata.rb. I wrote very basic cookbooks to do this, using command line executions.  This is not the best way to do this because it is specific to the Amazon Linux that is on the server. Instead in the future we should use a generic nginx and kibana cookbook to install them both. It was faster this way though.
All of these cookbooks called out in the metadata.rb need to be included in the default.rb recipe as ‘include_recipe’
The x-pack plugin is installed into both elasticsearch and kibana in the default.rb. Using the elasticsearch.yml we then turn off the security aspect of x-pack. We use Nginx instead for the password.

Nginx needs a nginx.conf setup to correctly route the incoming 8080 and 8181 ports to 9200 and 5601 locally for ES and Kibana. You will find this in the /files/default/nginx.conf location of the nginx cookbook.  It simply copies it onto the server.  In the future this should be done with a real nginx cookbook, not copying, but it works.
Once the cookbooks are done and the includes are all in the main wrapper cookbook, you need to use the chef dev kit to update and vendor it.

Go the CDK and go to the directory of the wrapper cookbook.  Do ‘berks update’ then do ‘berks vendor ..’  This should create many cookbook directories one level above.

Now make any changed to the wrapper cookbook recipe in default.rb, setting the elasticsearch configs, installing kibana/nginx using the other cookbooks, etc. 

Put it under source control and upload everything to git, including all the vendored cookbooks




