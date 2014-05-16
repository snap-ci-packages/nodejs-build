require 'rubygems'
require 'bundler/setup'

require 'rake/clean'

distro = nil
fpm_opts = ""

if File.exist?('/etc/system-release') && File.read('/etc/redhat-release') =~ /centos|redhat|fedora|amazon/i
  distro = 'rpm'
  fpm_opts << " --rpm-user root --rpm-group root "
elsif File.exist?('/etc/os-release') && File.read('/etc/os-release') =~ /ubuntu|debian/i
  distro = 'deb'
  fpm_opts << " --deb-user root --deb-group root "
end

unless distro
  $stderr.puts "Don't know what distro I'm running on -- not sure if I can build!"
end

%w(0.8.26 0.10.28 0.11.13).each do |version|
  namespace version do
    release = ENV['GO_PIPELINE_COUNTER'] || ENV['RELEASE'] || 1

    description_string = %Q{Node.js is a server-side JavaScript environment that uses an asynchronous event-driven model. This allows Node.js to get excellent performance based on the architectures of many Internet applications.}

    jailed_root = File.expand_path('../jailed-root', __FILE__)
    prefix = File.join('/opt/local/nodejs', version)

    CLEAN.include("downloads")
    CLEAN.include("jailed-root")
    CLEAN.include("log")
    CLEAN.include("pkg")
    CLEAN.include("src")

    task :init do
      mkdir_p "log"
      mkdir_p "pkg"
      mkdir_p "src"
      mkdir_p "downloads"
      mkdir_p "jailed-root"
    end

    task :download do
      cd 'downloads' do
        sh("curl --fail http://nodejs.org/dist/v#{version}/node-v#{version}.tar.gz                         > node-v#{version}.tar.gz 2>/dev/null")
        sh("curl --fail http://nodejs.org/dist/v#{version}/SHASUMS256.txt | grep  node-v#{version}.tar.gz  > node-v#{version}.tar.gz.shasum 2>/dev/null")
        sh("sha256sum --status --check node-v#{version}.tar.gz.shasum")
      end
    end

    task :configure do
      cd "src" do
        sh "tar -zxf ../downloads/node-v#{version}.tar.gz"
        cd "node-v#{version}" do
          sh "./configure --prefix=#{prefix} > #{File.dirname(__FILE__)}/log/configure.#{version}.log 2>&1"
        end
      end
    end

    task :make do
      num_processors = %x[nproc].chomp.to_i
      num_jobs       = num_processors + 1

      cd "src/node-v#{version}" do
        sh("make -j#{num_jobs} > #{File.dirname(__FILE__)}/log/make.#{version}.log 2>&1")
      end
    end

    task :make_install do
      rm_rf  jailed_root
      mkdir_p jailed_root
      cd "src/node-v#{version}" do
        sh("make install DESTDIR=#{jailed_root} > #{File.dirname(__FILE__)}/log/make-install.#{version}.log 2>&1")
      end
    end

    task :fpm do
      cd 'pkg' do
        sh(%Q{
             bundle exec fpm -s dir -t #{distro} --name nodejs-#{version} -a x86_64 --version "#{version}" -C #{jailed_root} --directories #{prefix} --verbose #{fpm_opts} --maintainer snap-ci@thoughtworks.com --vendor snap-ci@thoughtworks.com --url http://snap-ci.com --description "#{description_string}" --iteration #{release} --license 'https://raw.github.com/joyent/node/v#{version}/LICENSE' .
        })
      end
    end

    desc "build and package nodejs-#{version}"
    task :all => [:clean, :init, :download, :configure, :make, :make_install, :fpm]
  end

  task :default => "#{version}:all"
end

desc "build all nodejs"
task :default
