require "pathname"
require "webrick"
require "bookdown/builder"

desc "Clean out wip"
task :clean do
  FileUtils.rm_rf "work"
  FileUtils.rm_rf "parsed_markdown"
end

desc "Build it all"
task :default do
  book = Bookdown::Book.new(
                src_dir: ".",
      static_images_dir: "images",
           markdown_dir: "markdown",
               work_dir: "work",
    parsed_markdown_dir: "parsed_markdown",
               site_dir: "site",
               site_dir: Pathname("../what-problem-does-it-solve.com/site").expand_path / "serverless",
                  title: "Serverless and Functions-as-a-Service",
               subtitle: "Don't believe the hype—understand it",
                 author: "David Bryant Copeland AKA @davetron5000"
  )

  logger = Logger.new(STDOUT)
  logger.level = ENV["DEBUG"] == "true" ? Logger::DEBUG : Logger::INFO
  builder = Bookdown::Builder.new(logger: logger)
  builder.build(book)
end

task :serve do
  WEBrick::HTTPServer.new(:Port => 8000, :DocumentRoot => Pathname(Dir.pwd) / "site").start
end
