require 'rake/clean'
require 'rake/packagetask'
require 'rake/gempackagetask'

task :default => :test

DLEXT = Config::CONFIG['DLEXT']
VERS = '0.1.0'

# spec =
#   Gem::Specification.new do |s|
#     s.name              = "rpeg-multimarkdown"
#     s.version           = VERS
#     s.summary           = "Fast MultiMarkdown implementation"
#     s.files             = FileList[
#                             'README.markdown','LICENSE','Rakefile',
#                             '{lib,ext,test}/**.rb','ext/*.{c,h}',
#                             'test/MarkdownTest*/**/*',
#                             'bin/rpeg-multimarkdown'
#                           ]
#     s.bindir            = 'bin'
#     s.executables       << 'rpeg-multimarkdown'
#     s.require_path      = 'lib'
#     s.has_rdoc          = true
#     s.extra_rdoc_files  = ['LICENSE']
#     s.test_files        = FileList['test/multimarkdown_test.rb']
#     s.extensions        = ['ext/extconf.rb']
# 
#     s.author            = 'Oliver Whyte'
#     s.email             = 'oawhyte@gmail.com'
#     s.homepage          = 'http://github.com/djungelvral/rpeg-multimarkdown'
#     s.rubyforge_project = 'wink'
#   end
# 
#   Rake::GemPackageTask.new(spec) do |p|
#     p.gem_spec = spec
#     p.need_tar_gz = true
#     p.need_tar = false
#     p.need_zip = false
#   end

namespace :submodule do
  desc 'Init the peg-multimarkdown submodule'
  task :init do |t|
    unless File.exist? 'peg-multimarkdown/markdown.c'
      rm_rf 'peg-multimarkdown'
      sh 'git submodule init peg-multimarkdown'
      sh 'git submodule update peg-multimarkdown'
    end
  end

  desc 'Update the peg-multimarkdown submodule'
  task :update => :init do
    sh 'git submodule update peg-multimarkdown' unless File.symlink?('peg-multimarkdown')
  end

  file 'peg-multimarkdown/markdown.c' do
    Rake::Task['submodule:init'].invoke
  end
  task :exist => 'peg-multimarkdown/markdown.c'
end

desc 'Gather required peg-multimarkdown sources into extension directory'
task :gather => 'submodule:exist' do |t|
  sh 'cd peg-multimarkdown && make markdown_parser.c'
  files =
    FileList[
      'peg-multimarkdown/markdown_{peg.h,parser.c,output.c,lib.c,lib.h}',
      'peg-multimarkdown/{utility,parsing}_functions.c',
      'peg-multimarkdown/odf.{c,h}'
    ]
  cp files, 'ext/',
    :preserve => true,
    :verbose => true
end

file 'ext/Makefile' => FileList['ext/{extconf.rb,*.c,*.h,*.rb}'] do
  chdir('ext') { ruby 'extconf.rb' }
end
CLEAN.include 'ext/Makefile'

file "ext/peg_multimarkdown.#{DLEXT}" => FileList['ext/Makefile', 'ext/*.{c,h,rb}'] do |f|
  sh 'cd ext && make'
end
CLEAN.include 'ext/*.{o,bundle,so}'

file "lib/peg_multimarkdown.#{DLEXT}" => "ext/peg_multimarkdown.#{DLEXT}" do |f|
  cp f.prerequisites, "lib/", :preserve => true
end
CLEAN.include "lib/*.{so,bundle}"

desc 'Build the peg_multimarkdown extension'
task :build => "lib/peg_multimarkdown.#{DLEXT}"

desc 'Run unit and conformance tests'
task :test => [ 'test:unit', 'test:conformance' ]

desc 'Run unit tests'
task 'test:unit' => [:build] do |t|
  ruby 'test/multimarkdown_test.rb'
end

desc "Run conformance tests (MARKDOWN_TEST_VER=#{ENV['MARKDOWN_TEST_VER'] ||= '1.0.3'})"
task 'test:conformance' => [:build] do |t|
  script = "#{pwd}/bin/rpeg-multimarkdown"
  test_version = ENV['MARKDOWN_TEST_VER']
  chdir("test/MarkdownTest_#{test_version}") do
    sh "./MarkdownTest.pl --script='#{script}' --tidy"
  end
end

desc 'Run version 1.0 conformance suite'
task 'test:conformance:1.0' => [:build] do
  ENV['MARKDOWN_TEST_VER'] = '1.0'
  Rake::Task['test:conformance'].invoke
end

desc 'Run 1.0.3 conformance suite'
task 'test:conformance:1.0.3' => [:build] do |t|
  ENV['MARKDOWN_TEST_VER'] = '1.0.3'
  Rake::Task['test:conformance'].invoke
end

desc 'Run unit and conformance tests'
task :test => %w[test:unit test:conformance]

desc 'Run benchmarks'
task :benchmark => :build do |t|
  $:.unshift 'lib'
  load 'test/benchmark.rb'
end

desc "See how much memory we're losing"
task 'test:mem' => %w[submodule:exist build] do |t|
  $: << File.join(File.dirname(__FILE__), "lib")
  require 'multimarkdown'
  FileList['test/mem.txt', 'peg-multimarkdown/MarkdownTest_1.0.3/Tests/*.text'].each do |file|
    printf "%s: \n", file
    multimarkdown = MultiMarkdown.new(File.read(file))
    iterations = (ENV['N'] || 100).to_i
    total, growth = [], []
    iterations.times do |i|
      start = Time.now
      GC.start
      multimarkdown.to_html
      duration = Time.now - start
      GC.start
      total << `ps -o rss= -p #{Process.pid}`.to_i
      next if i == 0
      growth << (total.last - (total[-2] || 0))
      # puts "%03d: %06.02f ms / %dK used / %dK growth" % [ i, duration, total.last, growth.last ]
    end
    average = growth.inject(0) { |sum,x| sum + x } / growth.length
    printf "  %dK avg growth (per run) / %dK used (after %d runs)\n", average, total.last, iterations
  end
end

# ==========================================================
# Rubyforge
# ==========================================================

# PKGNAME = "pkg/rpeg-multimarkdown-#{VERS}"
# 
# desc 'Publish new release to rubyforge'
# task :release => [ "#{PKGNAME}.gem", "#{PKGNAME}.tar.gz" ] do |t|
#   sh <<-end
#     rubyforge add_release wink rpeg-multimarkdown #{VERS} #{PKGNAME}.gem &&
#     rubyforge add_file    wink rpeg-multimarkdown #{VERS} #{PKGNAME}.tar.gz
#   end
# end
