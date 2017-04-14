## Template Layout

> Nearly everybody is convinced that every style but their own is
> ugly and unreadable. Leave out the "but their own" and they're
> probably right... <br>
> -- Jerry Coffin (on indentation)

* <a name="consistent-template"></a>
  Use a consistent structure in your template definitions.
<sup>[[link](#consistent-template)]</sup>

  ```Ruby
  # start with requires
  require 'cfer'
  require_relative '../../helpers/my_helper'

  # next, included templates, alphabetized for easy comparison
  include_template(
    '../common/security_groups.rb',
    '../common/tags.rb'
  )
    
  # description
  description 'Keep this short'

  # Env parameter, if applicable.
  parameter :Env, default: 'test', description: 'Used for tagging and setting ' \
    'some env-specific attribute values'
 
  # All other parameters
  parameter :InstanceType, default: 'db.m4.xlarge'
  parameter :Iops, default: 3000
  parameter :Storage, default: 500

  # Next, all calls to lookup_output, so it's easy to see what external
  # stacks this template refers to.
  vpc_id = lookup_output(:VpcStack, :VpcId)
  private_subnet_ids = [
    lookup_output(:VpcStack, :PrivateSubnet1),
    lookup_output(:VpcStack, :PrivateSubnet2)
  ]
  
  ## in a top-level template (one that isn't included by others)
  
  # any local method definitions.
  def tags(role)
    [ { 'Key' => 'Name', 'Value' => "#{role}-#{parameters[:Env]}" } ]
  end

  # resource definitions, ordered by dependency
  resource :Parent, 'Aws::Type' do
    # ...
  end

  resource :Child, 'Aws::Type' do
    # ...
    self[:DependsOn] = [:Parent]
  end

  # in a template designed to be included by other templates, define public
  # interface methods first, then define methods intended to be used internally
  # to the template.
  ```
## File Names

* <a name="template-dependency"></a>
  When using smaller templates that export and lookup/import values from one
  another, use the filename to show the expected order in which they should
  be built. When templates have a common dependency, but don't depend on
  each other, they should have the same number.
<sup>[[link](#template-dependency)]</sup>

  * 00_vpc.rb
  * 10_common_security_groups.rb
  * 20_database.rb
  * 20_iam_instance_roles.rb
  * 20_redis_server.rb
  * 30_nfs.rb
  * 40_web_auto_scaling_group.rb

* <a name="purpose-not-implementation"></a>
  Choose template names that reflect the purpose of a particular element within the architecture, and not the resource type used to serve that purpose. This way if you change the underlying technology (e.g. from redis to memcached, mysql to postgres) the template name will stay the same and the change history will be easier to follow.
<sup>[[link](#purpose-not-implementation)]</sup>

  ```
  # bad
  mysql_rds.rb
  # good
  customer_database.rb
  
  # bad
  elasticache.rb
  redis_cluster.rb
  
  # good
  content_cache.rb
  ```
  
 ## Naming
 
 * <a name="consistent-stack-names"></a>
   Keep stack names short, but have a consistent schema for encoding important data in the name. For example, a schema could encode platform or application, role/template, and environment, with dash separators. Not only does this make it easier to find stacks in a list, it allows other developers to be able to predict a stack name without having to look it up.
 <sup>[[link](#consistent-stack-names)]</sup>
 
   * ecommerce-network-prod
   * ecommerce-db-prod
   * ecommerce-db-qa
   
   *Discussion topics* Ed suggested using env as the first element, rather than the last. To save characters, we could use mixed case instead of dashes. 
      
* <a name="prefer-symbols"></a>
  Use CamelCase symbols instead of quoted strings for resource names, and for arguments to Fn.ref, Fn.get_att, the first argument of output(), and the second argument of lookup_output.
  <sup>[[link](#prefer-symbols)]</sup>
  
  ```Ruby
  # bad
  resource 'my-resource', 'Aws::SomeType'
  Fn.ref("my-resource")
  Fn.get_att('my-resource', 'PrivateIp')
  output('my-resource', resource_id)
  lookup_output(stack_name, 'other-resource-id')
  
  # good
  resource :MyResource, 'Aws::SomeType'
  Fn.ref(:MyResource)
  Fn.get_att(:MyResource, :PrivateIp)
  output(:MyResource, resource_id)
  lookup_output(stack_name, :OtherResourceId
  ```

  However, if a name has to be calculated, don't jump through unnecessary hoops trying to symbolize it. Strings are OK too, when they are more convenient.
  ```Ruby
  # bad
  1.up_to(parameters[:InstanceCount]) do |index|
    resource "Instance#{index}".to_sym, 'Aws::SomeType' do
    end
    output(:"Instance#{index}", :PrivateIp)
  end
  
  # good
  1.up_to(parameters[:InstanceCount]) do |index|
    resource "Instance#{index}", 'Aws::SomeType' do
    end
    output("Instance#{index}", :PrivateIp)
  end
  ```
