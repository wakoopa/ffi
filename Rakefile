require 'rubygems/tasks'
require 'rbconfig'
require 'rake/clean'
require_relative "lib/ffi/version"

require 'date'
require 'fileutils'
require 'rbconfig'
require 'rspec/core/rake_task'
require 'rubygems/package_task'

def java?
  /java/ === RUBY_PLATFORM
end

BUILD_DIR = "build"
BUILD_EXT_DIR = File.join(BUILD_DIR, "#{RbConfig::CONFIG['arch']}", 'ffi_c', RUBY_VERSION)

def gem_spec
  @gem_spec ||= Gem::Specification.load('ffi.gemspec')
end

RSpec::Core::RakeTask.new(:spec => :compile) do |config|
  config.rspec_opts = YAML.load_file 'spec/spec.opts'
end

desc "Build all packages"
task :package => %w[ gem:java gem:windows ]

CLOBBER.include 'lib/ffi/types.conf'
CLOBBER.include 'pkg'
CLOBBER.include 'log'

CLEAN.include 'build'
CLEAN.include 'conftest.dSYM'
CLEAN.include 'spec/ffi/fixtures/libtest.{dylib,so,dll}'
CLEAN.include 'spec/ffi/fixtures/*.o'
CLEAN.include 'spec/ffi/embed-test/ext/*.{o,def}'
CLEAN.include 'spec/ffi/embed-test/ext/Makefile'
CLEAN.include "pkg/ffi-*-{mingw32,java}"
CLEAN.include 'lib/1.*'
CLEAN.include 'lib/2.*'

task :distclean => :clobber

desc "Test the extension"
task :test => [ :spec ]


namespace :bench do
  ITER = ENV['ITER'] ? ENV['ITER'].to_i : 100000
  bench_files = Dir["bench/bench_*.rb"].sort.reject { |f| f == "bench/bench_helper.rb" }
  bench_files.each do |bench|
    task File.basename(bench, ".rb")[6..-1] => :compile do
      h %{#{Gem.ruby} #{bench} #{ITER}}
    end
  end
  task :all => :compile do
    bench_files.each do |bench|
      sh %{#{Gem.ruby} #{bench}}
    end
  end
end

task 'spec:run' => :compile
task 'spec:specdoc' => :compile

task :default => :spec

namespace 'java' do

  java_gem_spec = gem_spec.dup.tap do |s|
    s.files.reject! { |f| File.fnmatch?("ext/*", f) }
    s.extensions = []
    s.platform = 'java'
  end

  Gem::PackageTask.new(java_gem_spec) do |pkg|
    pkg.need_zip = true
    pkg.need_tar = true
    pkg.package_dir = 'pkg'
  end
end

task 'gem:java' => 'java:gem'

if RUBY_ENGINE == 'ruby' || RUBY_ENGINE == 'rbx'
  require 'rake/extensiontask'
  Rake::ExtensionTask.new('ffi_c', gem_spec) do |ext|
    ext.name = 'ffi_c'                                        # indicate the name of the extension.
    # ext.lib_dir = BUILD_DIR                                 # put binaries into this folder.
    ext.tmp_dir = BUILD_DIR                                   # temporary folder used during compilation.
    ext.cross_compile = true                                  # enable cross compilation (requires cross compile toolchain)
    ext.cross_platform = %w[i386-mingw32 x64-mingw32]                     # forces the Windows platform instead of the default one
    ext.cross_compiling do |spec|
      spec.files.reject! { |path| File.fnmatch?('ext/*', path) }
    end
  end

  # To reduce the gem file size strip mingw32 dlls before packaging
  ENV['RUBY_CC_VERSION'].to_s.split(':').each do |ruby_version|
    task "build/i386-mingw32/stage/lib/#{ruby_version[/^\d+\.\d+/]}/ffi_c.so" do |t|
      sh "i686-w64-mingw32-strip -S build/i386-mingw32/stage/lib/#{ruby_version[/^\d+\.\d+/]}/ffi_c.so"
    end

    task "build/x64-mingw32/stage/lib/#{ruby_version[/^\d+\.\d+/]}/ffi_c.so" do |t|
      sh "x86_64-w64-mingw32-strip -S build/x64-mingw32/stage/lib/#{ruby_version[/^\d+\.\d+/]}/ffi_c.so"
    end
  end
else
  task :compile do
    STDERR.puts "Nothing to compile on #{RUBY_ENGINE}"
  end
end

desc "build a windows gem without all the ceremony"
task "gem:windows" do
  require "rake_compiler_dock"
  sh "bundle package"
  RakeCompilerDock.sh "sudo apt-get update && sudo apt-get install -y libltdl-dev && bundle --local && rake cross native gem MAKE='nice make -j`nproc`' RUBY_CC_VERSION=${RUBY_CC_VERSION/:2.2.2/}"
end

directory "ext/ffi_c/libffi"
file "ext/ffi_c/libffi/autogen.sh" => "ext/ffi_c/libffi" do
  warn "Downloading libffi ..."
  sh "git submodule update --init --recursive"
end
task :libffi => "ext/ffi_c/libffi/autogen.sh"

LIBFFI_GIT_FILES = `git --git-dir ext/ffi_c/libffi/.git ls-files -z`.split("\x0")

# Generate files in gemspec but not in libffi's git repo by running autogen.sh
gem_spec.files.select do |f|
  f =~ /ext\/ffi_c\/libffi\/(.*)/ && !LIBFFI_GIT_FILES.include?($1)
end.each do |f|
  file f => "ext/ffi_c/libffi/autogen.sh" do
    chdir "ext/ffi_c/libffi" do
      sh "sh ./autogen.sh"
    end
    touch f
    if gem_spec.files != Gem::Specification.load('./ffi.gemspec').files
      warn "gemspec files have changed -> Please restart rake!"
      exit 1
    end
  end
end

require_relative "lib/ffi/platform"

types_conf = File.expand_path(File.join(FFI::Platform::CONF_DIR, 'types.conf'))
logfile = File.join(File.dirname(__FILE__), 'types_log')

task types_conf do |task|
  require 'fileutils'
  require_relative "lib/ffi/tools/types_generator"
  options = {}
  FileUtils.mkdir_p(File.dirname(task.name), mode: 0755 )
  File.open(task.name, File::CREAT|File::TRUNC|File::RDWR, 0644) do |f|
    f.puts FFI::TypesGenerator.generate(options)
  end
  File.open(logfile, 'w') do |log|
    log.puts(types_conf)
  end
end

desc "Create or update type information for platform #{FFI::Platform::NAME}"
task :types_conf => types_conf

Gem::Tasks.new do |t|
  t.scm.tag.format = '%s'
end

begin
  require 'yard'

  namespace :doc do
    YARD::Rake::YardocTask.new do |yard|
    end
  end
rescue LoadError
  warn "[warn] YARD unavailable"
end
