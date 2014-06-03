require 'json'

MAX_ATTEMPTS=30

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

    sh "aws cloudformation create-stack --template-body  file:///#{Dir.pwd}/#{template} --stack-name #{stack_name} --parameters ParameterKey=SSHKeyName,ParameterValue=#{parameters[:SSHKeyName]} #{expanded_params}"
    wait_for_stack(stack_name)

    begin
      stack_attrs = get_stack_attributes(stack_name)
      neo4j_endpoint = stack_attrs['Outputs'].select {|output| output['OutputKey'] == 'Neo4jEndPoint' }.first['OutputValue']
      db_output = connect_to_neo(neo4j_endpoint, parameters[:DBUsername], parameters[:DBPassword])
    rescue Exception => e
      puts "Error: #{e}"
      db_output = "Something went wrong: #{e}"
    end

    delete_stack(stack_name)
    raise "Couldn't connect to Neo4j!" unless db_output.match(/db\/data\/cypher/)
  end
end
