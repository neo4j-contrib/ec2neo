require 'json'

MAX_ATTEMPTS=30
TIMESTAMP = Time.now.strftime('%FT%T').gsub(/:|-/,'')
PARAMETERS = {DBUsername: 'test',
              DBPassword: 'testtest',
              SSHKeyName: ENV['SSHKEYNAME']}

def get_expanded_params
  expanded_params = '--parameters '
  PARAMETERS.each_pair { |key, value| expanded_params << "ParameterKey=#{key},ParameterValue=#{value} " }
  expanded_params
end

def get_stack_name(template_file)
  template_file.split('.').first.gsub('_','') +  TIMESTAMP
end

def get_stack_attributes(stack_name)
   state_output = `aws cloudformation describe-stacks --stack-name #{stack_name} 2>&1`
   if state_output.match /does not exist/
     {'StackStatus' => 'ENOENT'}
   else
     JSON.parse(state_output)['Stacks'].first
   end
end

def get_stack_state(stack_name)
   get_stack_attributes(stack_name)['StackStatus'].inspect
end

def wait_for_stack(stack_name)
   while get_stack_state(stack_name) == '"CREATE_IN_PROGRESS"'
     get_stack_state(stack_name)
     puts "Waiting for stack #{stack_name} ..."
     sleep 10
   end
end

def delete_stack(stack_name)
  sh "aws cloudformation delete-stack --stack-name #{stack_name}"
  while get_stack_state(stack_name) != '"ENOENT"'
    get_stack_state(stack_name)
    puts "Deleting stack #{stack_name} ..."
    sleep 10
  end
end

def connect_to_neo(url,username, pass)
  response = ''
  attempts = 0
  until response.match(/cypher/) || attempts == MAX_ATTEMPTS
    response = `curl -L -v -u #{username}:#{pass} #{url} 2>&1`
    puts "DEBUG: #{response}"
    attempts += 1
    sleep 10
  end
  response
end

def test_for_ssh_key
  raise "Set the SSHKEYNAME var first" unless ENV['SSHKEYNAME']
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

task :test => :validate do
  test_for_ssh_key
  expanded_params = get_expanded_params
  Dir.glob('*.json').each do |template|
    stack_name = get_stack_name(template)

    sh "aws cloudformation create-stack --template-body  file:///#{Dir.pwd}/#{template} --stack-name #{stack_name} --parameters #{expanded_params}"
    wait_for_stack(stack_name)

    begin
      stack_attrs = get_stack_attributes(stack_name)
      neo4j_endpoint = stack_attrs['Outputs'].select {|output| output['OutputKey'] == 'Neo4jEndPoint' }.first['OutputValue']
      db_output = connect_to_neo(neo4j_endpoint, PARAMETERS[:DBUsername], PARAMETERS[:DBPassword])
    rescue Exception => e
      puts "Error: #{e}"
      db_output = "Something went wrong: #{e}"
    end

    delete_stack(stack_name)
    raise "Couldn't connect to Neo4j!" unless db_output.match(/db\/data\/cypher/)
  end
end

task :spawn do
  test_for_ssh_key
  template_file = 'ubuntu.json'
  expanded_params = get_expanded_params
  stack_name = get_stack_name(template_file)

  sh "aws cloudformation create-stack --template-body file:///#{Dir.pwd}/#{template_file} --stack-name #{stack_name} --parameters #{expanded_params}"
end
