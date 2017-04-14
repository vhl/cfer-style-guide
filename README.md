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

* <a name="template-dependency"></a>
  Name templates in a way that expresses order of dependency.
<sup>[[link](#template-dependency)]</sup>

  When using smaller templates that export and lookup/import values from one
  another, use the filename to show the expected order in which they should
  be built. When templates have a common dependency, but don't depend on
  each other, they should have the same number. e.g.
  * 00_vpc.rb
  * 10_common_security_groups.rb
  * 20_database.rb
  * 20_iam_instance_roles.rb
  * 20_redis_server.rb
  * 30_nfs.rb
  * 40_web_auto_scaling_group.rb


