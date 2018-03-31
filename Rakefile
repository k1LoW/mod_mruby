require 'erb'
require 'fileutils'

namespace :rpm do
  namespace :mod_mruby do
    desc 'Build mod_mruby'
    task all: ['rpm:mod_mruby:prep',
               'rpm:mod_mruby:generate_spec',
               'rpm:mod_mruby:build',
               'rpm:mod_mruby:check',
               'rpm:mod_mruby:release']

    task :prep do
      `rpmdev-setuptree`
      ENV['OS_MAJOR_VERSION'] = `cat /etc/redhat-release`.chomp.gsub(/^[^\d]+(\d).+/, '\1')
      ENV['HTTPD_VERSION'] = `yum info httpd | grep Version | awk '{print $3}'`.chomp
      ENV['HTTPD_RELEASE'] = `yum info httpd | grep Release | awk '{print $3}'`.chomp
      ENV['HTTPD_ARCH'] = `yum info nginx | grep Arch | awk '{print $3}'`.chomp
      ENV['MOD_MRUBY_VERSION'] = `curl -s https://api.github.com/repos/matsumotory/mod_mruby/releases | jq -r '.[0].tag_name' | sed 's/v//'`.chomp
    end

    task :generate_spec do
      spec = ERB.new(File.read('mod_mruby.spec.erb')).result(binding)
      File.write('mod_mruby.spec', spec)
    end

    task :build do
      `rpmdev-setuptree`
      `spectool -g -R mod_mruby.spec`
      `rpmbuild -ba mod_mruby.spec`
    end

    task :check do
      rpm = "mod_mruby-#{ENV['MOD_MRUBY_VERSION']}.httpd#{ENV['HTTPD_VERSION']}-#{ENV['HTTPD_RELEASE']}.#{ENV['HTTPD_ARCH']}.rpm"
      rpmfile = "/root/rpmbuild/RPMS/#{ENV['HTTPD_ARCH']}/#{rpm}"
      raise "#{rpmfile} does not exist" unless File.exist?(rpmfile)
      if ENV['OS_MAJOR_VERSION'] == '6'
        # @see https://discuss.circleci.com/t/httpd-yum-install-in-centos-docker-build-fails-with-cap-set-file/312
        puts `yum install -y #{rpmfile} --disablerepo=ius`
        FileUtils.copy(File.join(__dir__, 'test', 'mruby.conf'), '/etc/httpd/conf.d')
        puts `service httpd start`
        puts res = `curl localhost/mruby`.chomp
        raise "mod_mruby does no work" unless res == 'Hello mod_mruby world!'
      end
    end

    desc 'Release mod_mruby'
    task :release do
      if ENV['CIRCLE_BRANCH'] == 'master'
        rpm = "mod_mruby-#{ENV['MOD_MRUBY_VERSION']}.httpd#{ENV['HTTPD_VERSION']}-#{ENV['HTTPD_RELEASE']}.#{ENV['HTTPD_ARCH']}.rpm"
        rpmfile = "/root/rpmbuild/RPMS/#{ENV['HTTPD_ARCH']}/#{rpm}"
        status_code = `curl -LI https://packagecloud.io/k1low/mod_mruby/packages/el/#{ENV['OS_MAJOR_VERSION']}/#{rpm} -o /dev/null -w '%{http_code}\n' -s`.chomp
        if status_code == '200'
          puts "#{rpm} already exists."
        else
          cmd = "package_cloud push k1low/mod_mruby/el/#{ENV['OS_MAJOR_VERSION']} #{rpmfile}"
          puts cmd
          system(cmd)
        end
      end
    end
  end
end
