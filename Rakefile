desc "Publish to S3"
task :publish => :validate do 
  sh "s3cmd put --acl-public *.json s3://cf-templates.neo4j.org"
end

task :validate do
  Dir.glob('*.json').each do |template|
    sh "aws cloudformation validate-template --template-body file:///#{Dir.pwd}/#{template}"
  end
end
