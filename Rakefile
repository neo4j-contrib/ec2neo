require 'json'


def get_stack_state(stack_name)
   state_output = `aws cloudformation describe-stacks --stack-name #{stack_name} 2>&1`
   return 'ERRNOEXIST' if state_output.match(/does not exist/)
   state = JSON.parse(state_output)['Stacks'].first['StackStatus']
end

def wait_for_stack(stack_name)
   state = 'Unknown'
   while get_stack_state(stack_name) != 'CREATE_COMPLETE' && state != 'CREATE_FAILED'
     puts "Wating for stack #{stack_name} ..."
     sleep 10
   end
   return state
end

def delete_stack(stack_name)
  sh "aws cloudformation delete-stack --stack-name #{stack_name}"
  while get_stack_state(stack_name) != 'ERRNOEXIST'
    puts "Deleting stack #{stack_name} ..."
    sleep 10
  end
end

desc "Publish to S3"
task :publish => [:validate, :test] do 
  sh "s3cmd put --acl-public *.json s3://cf-templates.neo4j.org"
end

task :validate do
  Dir.glob('*.json').each do |template|
    sh "aws cloudformation validate-template --template-body file:///#{Dir.pwd}/#{template}"
  end
end

task :test do
  raise "Set the SSHKEYNAME var first" unless ENV['SSHKEYNAME']
  Dir.glob('*.json').each do |template|
    timestamp = Time.now.strftime('%FT%T').gsub(/:|-/,'')
    stack_name = template.split('.').first.gsub('_','') +  timestamp
    parameters = {DBUsername: 'test',
                  DBPassword: 'testtest',
                  SSHKeyName: ENV['SSHKEYNAME']}
    expanded_params = '--parameters '
    parameters.each_pair { |key, value| expanded_params << "ParameterKey=#{key},ParameterValue=#{value} " }

    sh "aws cloudformation create-stack --template-body  file:///#{Dir.pwd}/#{template} --stack-name #{stack_name} --parameters ParameterKey=SSHKeyName,ParameterValue=CI_FARM #{expanded_params}"
    state = wait_for_stack(stack_name)
    delete_stack(stack_name)
    raise 'Stack creation failed!' unless state != 'CREATE_COMPLETE'
  end
end
