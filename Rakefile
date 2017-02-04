desc "Create a new post."

task :post do
  unless FileTest.directory?('./_posts')
    abort("rake aborted: '_post' directory not found")
  end

  title = ENV["title"] || "new-post"
  slug = title.downcase.strip.gsub(' ', '-').gsub(/[^\w-]/, '')
  puts slug
  begin
    datetime = (ENV['date'] ? Time.parse([ENV['date']]) : Time.now).strftime('%Y-%m-%d %H:%M:%S %z')
    date = datetime.split.first
  rescue Exception => e
    puts "Error: date format must be YYYY-MM-DD!"
    exit -1
  end

  filename = File.join('.', '_posts', "#{date}-#{slug}.md")
  if File.exist?(filename)
    abort("rake abourted: #{filename} already exists.")
  end

  puts "Creating new post: #{filename}"
  open(filename, 'w') do |post|
    post.puts "---"
    post.puts "layout: post"
    post.puts "title: \"#{title.gsub(/-/,' ')}\""
    post.puts "date: #{datetime}"
    post.puts "categories:"
    post.puts "tags: []]"
    post.puts "---"
  end
end
