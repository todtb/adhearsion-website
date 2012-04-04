require 'bundler/setup'

desc "Generate dexy documentation and copy to views directory"
task :docs do
  Dir.mkdir 'views/docs' rescue nil
  puts `cd docs && dexy`
  Dir['docs/output/*.html'].each do |path|
    filename = /docs\/output\/(.*).html/.match(path)[1]
    new_path = "views/docs/#{filename}.html.erb"
    File.rename path, new_path
  end

  Dir['docs/output/_include/css/*.css'].each do |path|
    filename = /docs\/output\/_include\/css\/(.*).css/.match(path)[1]
    new_path = "public/stylesheets/#{filename}.css"
    File.rename path, new_path
  end
end