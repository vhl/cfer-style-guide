## Template Structure

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
  resource :Parent, 'AWS::SomeType' do
    # ...
  end

  resource :Child, 'AWS::SomeType' do
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
  resource 'my-resource', 'AWS::SomeType'
  Fn.ref("my-resource")
  Fn.get_att('my-resource', 'PrivateIp')
  output('my-resource', resource_id)
  lookup_output(stack_name, 'other-resource-id')
  
  # good
  resource :MyResource, 'AWS::SomeType'
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

## Resources
* <a name="assign-meta-attributes-within-block"></a>
 Don't pass CreationPolicy, DependsOn, MetaData, and UpdatePolicy as optional hash arguments to the resource method. Instead, specify them within the resource block, after all other attributes.
  <sup>[[link](#assign-meta-attributes-within-block)]</sup>

  ```Ruby
  # bad
  resource :MyResourceName, 'AWS::AutoScaling::AutoScalingGroup', 
    :DependsOn => [:Dependency1, :Dependency2] do
    resource_attr :one
    resource_attr :two
  end

  # worse
  resource :MyResourceName, 'AWS::AutoScaling::AutoScalingGroup', :DependsOn => [:Dependency1, :Dependency2] do
    resource_attr :one
    resource_attr :two
  end

  # good
  resource :MyResourceName, 'AWS::AutoScaling::AutoScalingGroup' do
    resource_attr :one
    resource_attr :two

    self[:DependsOn] = [:Dependency1, :Dependency2]
  end
  ```
  
## Parameters
* <a name="dont-fn-ref-params"></a>
  Even though this works when evaluated within CloudFormation, don't use Fn.ref with parameters within your templates.
  <sup>[[link](#dont-fn-ref-params)]</sup>
  
  ```Ruby
  # given
  parameter :MyParam, default: 'some_value'

  # bad
  resource :MyResourceName, 'AWS::SomeThing' do
    some_attr Fn.ref(:MyParam)
  end

  # good
  resource :MyResourceName, 'AWS::SomeThing' do
    some_attr parameters[:MyParam]
  end
  ```
* <a name="prefer-description-to-comments"></a>
  Use the optional :description key instead of ruby comments when annotating parameters. The advantage to using description is that it is displayed within CloudFormation.
  <sup>[[link]](#prefer-description-to-comments)]</sup>
  
  ```Ruby
  # bad
  # Specify a Ubuntu 16.04 AMI in the desired region.
  parameter :BaseAmi, default: 'ami-123456'
  parameter :MaxSize, default: 2 # Should be higher than DesiredSize

  # good
  parameter :BaseAmi, default: 'ami-123456',
                      description: 'Specify a Ubuntu 16.04 AMI in the desired region.'
  parameter :MaxSize, default: 2, description: 'Should be higher than DesiredSize'
  ```

* <a name="alphabetize-params"></a>
  Declare parameters in alphabetical order.
  <sup>[[link]](#alphabetize-params)</sup>
  
  ```Ruby
  # bad
  parameter :InstanceType, default: 't2.medium'
  parameter :LiveAmiImage, default: 'ami-12456'
  parameter :MinSize, default: '2'
  parameter :MaxSize, default: '6'

  # good
  parameter :InstanceType, default: 't2.medium'
  parameter :LiveAmiImage, default: 'ami-12456'
  parameter :MaxSize, default: '6'
  parameter :MinSize, default: '2'
  ```

  Exception: if the default value for one parameter references another, the referenced parameter must be declared first.

  ```Ruby
  parameter :BatchSize, default: '2'
  parameter :MinSize, default: '2'
  parameter :DesiredSize, default: parameters[:MinSize]
  parameter :MaxSize, default: (parameters[:DesiredSize].to_i + parameters[:BatchSize].to_i).to_s
  ```  
## Argument Lists

* <a name="alphabetize-everything"></a>
  Alphabetize arguments to include_template(), attributes within a resource definition, or hash keys within a hash argument to a method, especially when the hash spans multiple lines. Basically, if you've got a list of things, alphabetize it. It makes it much easier to detect the presence or absence of an element within the list, and to compare different lists.
  <sup>[[link]](#alphabetize-everything)</sup>
  ```Ruby
  # hash arguments
  # bad
  create_auto_scale_group(
    resource_name,
    load_balancer_names: [Fn.ref(lb_resource_name)],
    vpc_zone_ids: [subnet_1, subnet_2]
    creation_policy: creation_policy(parameters[:MinSize], 'PT10M'),
    desired_capacity: parameters[:DesiredSize],
    launch_config: Fn.ref(launch_config_name),
    min_size: parameters[:MinSize],
    max_size: parameters[:MaxSize],
    tags: build_tags(resource_name, 'web'),
  )

  # good
  create_auto_scale_group(
    resource_name,
    creation_policy: creation_policy(parameters[:MinSize], 'PT10M'),
    desired_capacity: parameters[:DesiredSize],
    launch_config: Fn.ref(launch_config_name),
    load_balancer_names: [Fn.ref(lb_resource_name)],
    max_size: parameters[:MaxSize],
    min_size: parameters[:MinSize],
    tags: build_tags(resource_name, 'web'),
    vpc_zone_ids: [subnet_1, subnet_2]
  )
  
  # resource definitions
  # bad
  def create_auto_scale_group(resource_name, opts = {})
    resource resource_name, 'AWS::AutoScaling::AutoScalingGroup' do
      max_size opts[:max_size]
      min_size opts[:min_size]
      desired_capacity opts[:desired_capacity]

      v_p_c_zone_identifier opts[:vpc_zone_ids]

      load_balancer_names opts[:load_balancer_names] if opts.key?(:load_balancer_names)
      launch_configuration_name opts[:launch_config]

      self[:CreationPolicy] = opts[:creation_policy] if opts.key?(:creation_policy)

      tags opts[:tags] || []
    end
  end

  # good
  def create_auto_scale_group(resource_name, opts = {})
    resource resource_name, 'AWS::AutoScaling::AutoScalingGroup' do
      desired_capacity opts[:desired_capacity]
      launch_configuration_name opts[:launch_config]
      load_balancer_names opts[:load_balancer_names] if opts.key?(:load_balancer_names)
      max_size opts[:max_size]
      min_size opts[:min_size]
      tags opts[:tags] || []
      v_p_c_zone_identifier opts[:vpc_zone_ids]

      self[:CreationPolicy] = opts[:creation_policy] if opts.key?(:creation_policy)
    end
  end

  # Lists specified with %w() or %i()
  # bad
  require_opts %i(vpc_zone_ids max_size min_size desired_capacity launch_config)

  # good
  require_opts %i(desired_capacity min_size max_size launch_config vpc_zone_ids)
  ``` 

## Exports

* <a name="limit-exports"></a>
  Values should not be exported using the output method unless it is absolutely necessary for some other stack to import them. Since declaring an export means that some stack may have already started depending on it, outputs should rarely be renamed or removed. If there is a need to output a value for debug purposes (e.g. the private ip of an instance so you can shell into it), prefix the output name with Debug.
  <sup>[[link]](#limit-exports)</sup>
  
  ```Ruby
  output(:DebugInstanceIp, Fn.get_att(:Instance, 'PrivateIp')
  ```
  
* <a name="immutable-output-names"></a>
  When calling the `output` method, the first argument should be a symbol or a literal string with no interpolation, and not a variable or a string using variable interpolation. Basically, it should be as immutable as possible, as there are probably other stacks that expect to be able to look up or import values using this identifier.
  <sup>[[link]](#immutable-output-names)</sup>
  
  ```Ruby
  # bad
  output("Prefix#{some_var}Id", some_value)
  
  # good
  output(:SolidUnchangingName, some_value)
  ```
  
