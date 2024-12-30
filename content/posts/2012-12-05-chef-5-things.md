---
title: "5 Things You Always Wanted to Know About Chef But Were Afraid to Ask"
description: You may not want finally
date: 2012-12-05T22:23:44
tags: ['devops', 'chef', 'webinar']
---

At the last *Chef Conf 2012* event, *Nathen Harvey* in all honesty revealed when he first approched Chef, he stay away from some advanced concepts. He comes back today to help all of us better understand the arcanes of it. By the way he is currently working for [*CustomInk*](http://www.customink.com/), as we can consider them sponsors, don't hesitate to buy their great T-Shirts.

<!-- more -->

### Attributes precedences

Attributes can be set on the node from the following objects:

* Cookbooks
* Environments (>=0.10.0)
* Roles
* Nodes, but mind the ohai

Many types of attributes, here is the preference order from low to high :

1. **default**: attribute file < environment < role < applied on a node directly in a recipe
2. **normal** or set: attribute file < applied on a node directly in a recipe
3. **overrides**: attribute file < environment < roles < applied on a node directly in a recipe
4. **automatic** (come out of Ohai, like platform, cannot be overrided)

So in Summary, default get overwritten by normal which again gets overwritten by overrides.
Automatic have the highest precedence and may not be modified

		default < normal < overrides

		attribute < environment < roles < recipes

Be carefull you will likely have too much power here.

### Encrypted databags

To create an encrypted Databag for database credentials with knife

	% knife databag create db creds --secret-file ~/.chef/secret

Opens up your local editor to edit JSON

```json
{
	"id": "creds",
	"production" : {
		"username": prod_user
		"password": "xxxxxxx"
	},
	"staging" : {
		"username": user
		"password": "xxxxxx"
	}
}
```

To inspect encrypted Databags

	% knife databag show db creds --secret-file ~/.chef/secret

To edit your encrypted Databags

	% knife databag edit db creds --secret-file ~/.chef/secret

To use the encrypted Databags

```ruby
creds = Chef::EncryptedDataBagItem.load("db", "creds")
env_db_creds = db_creds[node["rails_env"]]

template "#{app_dir}/shared/config/database.yml" do
  source "database.yml.erb"
  variables(
    :rails_env => node["rails_env"],
    :username => env_db_creds["username"],
    :password => env_db_creds["password"]
  )
end
```

Encrypted databags can be checked in source code repository without issues, users will only see the encrypted version (don't checkin your secret file). To upload them to your git repository in an encrypted format

	% knife databag show db creds -Fj > data_bags/db/creds.json

That file can then be checked in your repository.

### Light-Weight Resources and Providers

The resources is the interface and the provider is the implementation. 

To build a new LWRP to easily add new aliases on an Unix machine, we first have to declare a resource file

##### File resources/alias.rb

	#!ruby
	actions :add, :remove
	
	default_action :add
	
	attribute :alias_name, :kind_of => String, :name_attribute => true
	attribute :command, :kind_of => String, :default => :add

Our LWRP is a wrapper around `file` Chef resource

##### File providers/alias.rb

	#!ruby
	action :add do
	  command_name = new_resource.alias_name.gsub(/ /,"_")
	  unless new_resource.command.nil?
	    Chef::Log.info("Adding #{command_name}.sh to /etc/profile.d/")
	    file_contents = "# This alias was generated by Chef for #{node["fqdn"]}\n"
	    file_contents += "alias #{command_name}='#{new_resource.command}'"
	    resource = file "/etc/profile.d/#{command_name}.sh" do
	      owner "root"
	      group "root"
	      mode "0755"
	      content file_contents
	      action :nothing
	    end
	    resource.run_action(:create)
	    new_resource.updated_by_last_action(true) if resource.updated_by_last_action?
	  end
	end

	action :remove do
	  command_name = new_resource.alias_name.gsub(/ /,"_")
	  resource =  file "/etc/profile.d/#{command_name}.sh" do
	    action :nothing
	  end

	  resource.run_action(:delete)
	  new_resource.updated_by_last_action(true) if resource.updated_by_last_action?
	end

To use that within a recipe, previous LWRP is bundled in [magic_shell](https://github.com/customink-webops/magic_shell)

	#!ruby
	include_recipe "magic_shell"
	
	magic_shell_alias "current" do
		command "cd /opt/myrailsapp/current"
		action :add
	end

### Exception and Report Handlers

* Allow you to run arbitrary code when Chef run, starts, ends, fails or succeed.
* Most obvious use is to notify when a Chef run fails
* Can be used to gather rich data about your Chef runs

You have access to

* node
* all_resource
* updated_resource
* elapse_time
* start/end time
* run_context

A lot of [handlers](http://wiki.opscode.com/display/chef/Exception+and+Report+Handlers) already exists in the community site, for example a [Mail report handler](https://github.com/kisoku/chef-handler-mail) to receive an email if a Chef runs fails.

Another example is [DATADOG handler](https://github.com/DataDog/chef-handler-datadog) which will graph anything for you by connecting your data to the [DATADOG](http://www.datadoghq.com/) SaaS solution.

### Capistrano vs Chef

Nathen thought about Deploying using Chef Resource or Capistrano ? You can use Deploy resources to deploy your application using Chef, however *CustomInk* doesn't use Chef for that task for the following reasons :

1. They have hundreds of web servers running their application, Chef runs at regular interval it varies within the range of an hour. So they don't want their code to diverge from machine to machine.
2. A lot of applications not yet baked with Chef, these uses Capistrano, so they don't want to have two deployment logic.
3. Some Rails plugins and gems already comes with Capistrano hooks, so they don't want to reinvent the wheel.

Capistrano is a way to deploy your code, you have to specifies a list of web servers. They are using Chef search to find which servers have a particular role and then use Capistrano to deploy to them by looping over FQDN.

	#!ruby
	webservers = []
	web_query = Chef::Search::Query.new
	web_query.search(:node, 'role:chefconf_web') do |h|
		webservers << h["fdqn"]
	end
	role :web, *webservers

### Links

* <http://nathenharvey.com>
* [The Food Fight Show](http://foodfightshow.org) - The Chef Community Podcast
* [*CustomInk*](https://github.com/customink) GitHub repository
* [Slide decks](https://speakerdeck.com/nathenharvey) from *Nathen* 

### Conclusion

*CustomInk* relies heavily on *Opscode* configuration management technology to maintain their infrastructure and they share a lot of content in their [GitHub](https://github.com/customink-webops) repository. Enjoy !!! Thanks Nathen.