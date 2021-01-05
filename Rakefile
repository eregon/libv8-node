require 'bundler/setup'

Bundler::GemHelper.install_tasks

module Helpers
  module_function

  def binary_gemspec(platform: Gem::Platform.local, str: RUBY_PLATFORM)
    platform.instance_eval { @version = 'musl' } if str =~ /-musl/ && platform.version.nil?

    gemspec = eval(File.read('libv8-node.gemspec'))
    gemspec.platform = platform
    gemspec
  end

  def binary_gem_name(platform = Gem::Platform.local)
    File.basename(binary_gemspec(platform).cache_file)
  end
end

task :compile do
  #sh 'ruby ext/libv8-node/extconf.rb'
end

task :binary, [:platform] => [:compile] do |_, args|
  gemspec = Helpers.binary_gemspec(**args.to_h)
  gemspec.extensions.clear

  # We don't need most things for the binary
  gemspec.files = []
  gemspec.files += ['lib/libv8-node.rb', 'lib/libv8/node.rb', 'lib/libv8/node/version.rb']
  gemspec.files += ['ext/libv8-node/location.rb', 'ext/libv8-node/paths.rb']
  gemspec.files += ['ext/libv8-node/.location.yml']

  # V8
  gemspec.files += Dir['vendor/v8/include/**/*.h']
  gemspec.files += Dir['vendor/v8/out.gn/**/*.a']

  FileUtils.chmod(0o0644, gemspec.files)
  FileUtils.mkdir_p('pkg')

  package = if Gem::VERSION < '2.0.0'
              Gem::Builder.new(gemspec).build
            else
              require 'rubygems/package'
              Gem::Package.build(gemspec)
            end

  FileUtils.mv(package, 'pkg')
end

namespace :binary do
  task :all => :binary do
    return unless RUBY_PLATFORM =~ /darwin-?(\d+)/

    Helpers.binary_gemspec # loads NODE_VERSION

    current = Integer($1)
    major, minor = File.read(Dir["src/node-#{Libv8::Node::NODE_VERSION}/common.gypi"].last).lines.find { |l| l =~ /-mmacosx-version-min=(\d+).(\d+)/ } && [Integer($1), Integer($2)]
    first = minor + 4
    max = 20

    (first..max).each do |version|
      next if version == current

      platform = Gem::Platform.local.dup
      platform.instance_eval { @version = version }
      puts "> building #{platform}"

      Rake::Task["binary"].execute(Rake::TaskArguments.new([:platform], [platform]))
    end
  end
end
