# -*- mode: ruby -*-
# vi: set ft=ruby :
require 'fileutils'
startSSHPort = 2250
# Generate a random port number
# fixes issue where two boxes try and map port 22, if you run multiple vagrant environments in one host
vagrant_root = File.dirname(__FILE__)
dev_domain = 'enjo.test'
mysql_password = ENV['MYSQL_ROOT_PASSWORD'] || "root"
persistent_storage = vagrant_root + '/persistent_storage'
mode = ENV['VAGRANT_MODE'] || 'dev'
ip_range = "172.23.1"

puts "========================================================"
puts "domain : #{dev_domain}"
puts "folder : #{vagrant_root}"
puts "mysql root password : #{mysql_password}"
puts "mode: #{mode}"
puts "persistent storage: #{persistent_storage}"
puts "ip range used: #{ip_range}"
puts "========================================================"

FileUtils.mkdir_p(persistent_storage)
FileUtils.mkdir_p(persistent_storage+"/mysql")
FileUtils.mkdir_p(persistent_storage+"/elasticsearchm2")
FileUtils.mkdir_p(persistent_storage+"/elasticsearch")


Vagrant.configure('2') do |config|
    config.vm.boot_timeout = 1800
    config.hostmanager.enabled = true
    config.hostmanager.manage_host = true
    config.hostmanager.manage_guest = true
    config.hostmanager.ignore_private_ip = false
    config.hostmanager.include_offline = false
    config.trigger.after :up do |trigger|
      trigger.run = {inline: "bash -c 'vagrant hostmanager --provider docker'"}
    end
    config.vm.define "magento", primary: true do |magento|
        magento.hostmanager.aliases = [ "magento."+dev_domain ]
        magento.vm.provision "file", source: "#{vagrant_root}/magento.nginx.conf", destination: "/tmp/magento.nginx.conf"
        magento.vm.provision "shell" do |s|
            s.path = "bootstrap.sh"
            s.args = "#{dev_domain} #{ip_range}.200"
        end
        if File.exist?("provision/bootstrap.sh")
            magento.vm.provision "shell" do |s|
                s.path = "provision/bootstrap.sh"
                s.args = "#{dev_domain} #{ip_range}.200"
            end
        end

        magento.ssh.username = "vagrant"
        magento.ssh.password = "vagrant"
        magento.ssh.keys_only = false
        magento.vm.network :private_network, ip: "#{ip_range}.200", subnet: "#{ip_range}.0/16"
        magento.vm.network "forwarded_port", guest: 22, host: 2230, id: 'ssh', auto_correct: true
        magento.vm.hostname = "magento"
        magento.vm.provider 'docker' do |d|
            d.image = "proxiblue/magento2:latest"
            #d.build_dir = "./Docker/magento"
            d.has_ssh = true
            d.name = "magento"
            d.create_args = ["--cap-add=NET_ADMIN"]
            d.remains_running = true
            d.volumes = ["/tmp/.X11-unix:/tmp/.X11-unix", ENV['HOME']+"/.ssh/:/home/vagrant/.ssh", ENV['HOME']+"/.composer:/home/vagrant/.composer"]
            d.env = { "DEV_DOMAIN" => "#{dev_domain}", "WEB_IP" => "#{dev_domain}" }
        end
    end

    config.vm.define "database", primary: false do |database|
        database.hostmanager.aliases = [ "database."+dev_domain ]
        database.vm.network :private_network, ip: "#{ip_range}.208", subnet: "#{ip_range}.0/16"
        database.vm.hostname = "database"
        database.vm.communicator = 'docker'
        database.vm.provider 'docker' do |d|
            d.image = "proxiblue/mysql:latest"
            d.has_ssh = false
            d.name = "database"
            d.remains_running = true
            d.volumes = ["#{persistent_storage}/mysql:/var/lib/mysql"]
            d.env = { "MYSQL_ROOT_PASSWORD" => "#{mysql_password}" }
        end
    end

    config.vm.define "redis", primary: false do |redis|
        redis.hostmanager.aliases = [ "redis."+dev_domain ]
        redis.vm.network :private_network, ip: "#{ip_range}.201", subnet: "#{ip_range}.0/16"
        redis.vm.hostname = "redispwa"
        redis.vm.provider 'docker' do |d|
            d.image = "redis:latest"
            d.has_ssh = false
            d.name = "redispwa"
            d.remains_running = true
        end
    end

    config.vm.define "elasticsearchm2", primary: false do |elasticsearchm2|
        elasticsearchm2.hostmanager.aliases = [ "elasticsearchm2."+dev_domain ]
        elasticsearchm2.vm.network :private_network, ip: "#{ip_range}.202", subnet: "#{ip_range}.0/16"
        elasticsearchm2.vm.hostname = "elasticsearchm2pwa"
        elasticsearchm2.vm.provider 'docker' do |d|
            d.image = "docker.elastic.co/elasticsearch/elasticsearch:6.8.3"
            d.has_ssh = false
            d.name = "elasticsearchm2pwa"
            d.remains_running = true
            d.volumes = [
                "#{persistent_storage}/elasticsearchm2:/usr/share/elasticsearch/data"
            ]
        end
    end

    config.vm.define "rabbitmq", primary: false do |rabbitmq|
        rabbitmq.hostmanager.aliases = [ "rabbitmq."+dev_domain ]
        rabbitmq.vm.network :private_network, ip: "#{ip_range}.203", subnet: "#{ip_range}.0/16"
        rabbitmq.vm.hostname = "rabbitmqpwa"
        rabbitmq.vm.provider 'docker' do |d|
            d.image = "rabbitmq:latest"
            d.has_ssh = false
            d.name = "rabbitmqpwa"
            d.remains_running = true
        end
    end

    config.vm.define "elasticsearch", primary: false do |elasticsearch|
        elasticsearch.hostmanager.aliases = [ "elasticsearch."+dev_domain ]
        vue_elastic_config="#{vagrant_root}/sites/vue-storefront-api/docker/elasticsearch/config/elasticsearch.yml"
        elasticsearch.trigger.before :all do |trigger|
            trigger.name = "overlay config"
            # Check if overlay config for elastic search exists.
            if File.exist?("#{vagrant_root}/vuestorefront-config-overlay/elasticsearch/config/elasticsearch.yml")
                vue_elastic_config="#{vagrant_root}/vuestorefront-config-overlay/elasticsearch/config/elasticsearch.yml"
                trigger.info = "Found overlay config: #{vue_elastic_config}"
            end
            trigger.ignore = [:destroy, :halt]
        end
        elasticsearch.vm.network :private_network, ip: "#{ip_range}.204", subnet: "#{ip_range}.0/16"
        elasticsearch.vm.hostname = "elasticsearchpwa"
        elasticsearch.vm.provider 'docker' do |d|
            d.build_dir = "#{vagrant_root}/sites/vue-storefront-api/docker/elasticsearch"
            d.dockerfile = "Dockerfile"
            d.has_ssh = false
            d.name = "elasticsearchpwa"
            d.remains_running = true
            d.volumes = [
                "#{vue_elastic_config}:/usr/share/elasticsearch/config/elasticsearch.yml:ro",
                "#{persistent_storage}/elasticsearch:/usr/share/elasticsearch/data"
                ]
            d.env =  { "ES_JAVA_OPTS" => "-Xmx512m -Xms512m" }
        end
    end

    config.vm.define "kibana", primary: false do |kibana|
        kibana.hostmanager.aliases =  [ "kibana."+dev_domain ]
        vue_kibana_config="#{vagrant_root}/sites/vue-storefront-api/docker/kibana/config/"
        kibana.trigger.before :all do |trigger|
            trigger.name = "overlay config"
            # Check if overlay config for kibana exists.
            if File.exist?("#{vagrant_root}/vuestorefront-config-overlay/kibana/config/kibana.yml")
                vue_kibana_config="#{vagrant_root}/vuestorefront-config-overlay/kibana/config/"
                trigger.info = "Found overlay config: #{vue_kibana_config}"
            end
            trigger.ignore = [:destroy, :halt]
        end
        kibana.vm.network "forwarded_port", guest: 22, host: Random.new.rand(1000...5000), id: 'ssh', auto_correct: true
        kibana.vm.network :private_network, ip: "#{ip_range}.205", subnet: "#{ip_range}.0/16"
        kibana.vm.hostname = "kibanapwa"
        kibana.vm.communicator = 'docker'
        kibana.vm.provider 'docker' do |d|
            d.build_dir = "#{vagrant_root}/sites/vue-storefront-api/docker/kibana"
            d.dockerfile = "Dockerfile"
            d.has_ssh = true
            d.name = "kibanapwa"
            d.remains_running = true
            d.volumes = [
                "#{vue_kibana_config}:/usr/share/kibana/config:ro"
                ]
        end
    end

    config.vm.define "vueapi", primary: false do |vueapi|
        vueapi.hostmanager.aliases = [ "vueapi."+dev_domain ]
        vueapi.trigger.before :all do |trigger|
            trigger.name = "overlay config"
            # Check if vue local.json config exists, and copy it to the vue config folder
            # any edits must be made in teh overlay file. Edits in teh destination file will be overwritten
            config_file="local.json"
            if File.exist?("#{vagrant_root}/vuestorefront-config-overlay/vue-storefront-api/config/#{config_file}")
                FileUtils.copy_file("#{vagrant_root}/vuestorefront-config-overlay/vue-storefront-api/config/#{config_file}",
                "#{vagrant_root}/sites/vue-storefront-api/config/local.json")
                trigger.info = "Found overlay #{config_file}. It was copied to the base vue config folder."
            end
            # check that the /tmp/vueapi folder exists (which is used to simulated the tmpfs setup as per vue composer files
            if File.directory?("/tmp/vueapi")
                FileUtils.rm_rf("/tmp/vueapi")
                FileUtils.mkdir_p("/tmp/vueapi")
                trigger.info = "Temp folder /tmp/vueapi created."
            end
            trigger.ignore = [:destroy, :halt]
        end

        if File.exist?("#{vagrant_root}/vuestorefront-config-overlay/vue-storefront-api/boot.sh")
                vueapi.vm.provision "shell", path: "#{vagrant_root}/vuestorefront-config-overlay/vue-storefront-api/boot.sh", privileged: true
        end

        vueapi.vm.network :private_network, ip: "#{ip_range}.206", subnet: "#{ip_range}.0/16"
        vueapi.vm.hostname = "vueapi"
        vueapi.vm.communicator = 'docker'
        vueapi.vm.provider 'docker' do |d|
            d.build_dir = "#{vagrant_root}/sites/vue-storefront-api/"
            d.dockerfile = "docker/vue-storefront-api/Dockerfile"
            d.has_ssh = true
            d.name = "vueapi"
            d.remains_running = true
            d.volumes = [
                "#{vagrant_root}/sites/vue-storefront-api/config:/var/www/config",
                "#{vagrant_root}/sites/vue-storefront-api/ecosystem.json:/var/www/ecosystem.json",
                "#{vagrant_root}/sites/vue-storefront-api/migrations:/var/www/migrations",
                "#{vagrant_root}/sites/vue-storefront-api/package.json:/var/www/package.json",
                "#{vagrant_root}/sites/vue-storefront-api/babel.config.js:/var/www/babel.config.js",
                "#{vagrant_root}/sites/vue-storefront-api/tsconfig.json:/var/www/tsconfig.json",
                "#{vagrant_root}/sites/vue-storefront-api/nodemon.json:/var/www/nodemon.json",
                "#{vagrant_root}/sites/vue-storefront-api/scripts:/var/www/scripts",
                "#{vagrant_root}/sites/vue-storefront-api/src:/var/www/src",
                "#{vagrant_root}/sites/vue-storefront-api/var:/var/www/var",
                "/tmp/vueapi:/var/www/dist"
                ]
            d.env = { "BIND_HOST" => "0.0.0.0",
                      "ELASTICSEARCH_HOST" => "elasticsearch",
                      "ELASTICSEARCH_PORT" => "9200",
                      "REDIS_HOST" => "redis",
                      "VS_ENV" => "#{mode}",
                      "PM2_ARGS" => "--no-daemon",
                      "NODE_TLS_REJECT_UNAUTHORIZED" => "0"
                    }
        end
    end

    config.vm.define "vuestorefront", primary: false do |vuestorefront|
        vuestorefront.hostmanager.enabled = true
        vuestorefront.hostmanager.aliases =  [ "vuestorefront."+dev_domain ]
        vuestorefront.trigger.before :all do |trigger|
            trigger.name = "overlay config"
            # Check if vue local.json config exists, and copy it to the vue config folder
            # any edits must be made in teh overlay file. Edits in teh destination file will be overwritten
            config_file="local.json"
            if File.exist?("#{vagrant_root}/vuestorefront-config-overlay/vue-storefront/config/#{config_file}")
                FileUtils.copy_file("#{vagrant_root}/vuestorefront-config-overlay/vue-storefront/config/#{config_file}",
                "#{vagrant_root}/sites/vue-storefront/config/local.json")
                trigger.info = "Found overlay #{config_file}. It was copied to the base vue config folder."
            end
            # check that the /tmp/vuestorefront folder exists (which is used to simulated teh tmpfs setup as per vue composer files
            if File.directory?("/tmp/vuestorefront")
                FileUtils.rm_rf("/tmp/vuestorefront")
                FileUtils.mkdir_p("/tmp/vuestorefront")
                trigger.info = "Temp folder /tmp/vuestorefront created."
            end
            trigger.ignore = [:destroy, :halt]
        end
        if File.exist?("#{vagrant_root}/vuestorefront-config-overlay/vue-storefront/boot.sh")
                vuestorefront.vm.provision "shell", path: "#{vagrant_root}/vuestorefront-config-overlay/vue-storefront/boot.sh", privileged: true
        end
        vuestorefront.vm.network :private_network, ip: "#{ip_range}.207", subnet: "#{ip_range}.0/16"
        vuestorefront.vm.network "forwarded_port", guest: 22, host: Random.new.rand(1000...5000), id: 'ssh', auto_correct: true
        vuestorefront.vm.hostname = "vuestorefront"
        vuestorefront.vm.communicator = 'docker'
        vuestorefront.vm.provider 'docker' do |d|
            d.build_dir = "#{vagrant_root}/sites/vue-storefront/"
            d.dockerfile = "docker/vue-storefront/Dockerfile"
            d.has_ssh = true
            d.name = "vuestorefront"
            d.remains_running = true
            d.volumes = [
                "#{vagrant_root}/sites/vue-storefront/babel.config.js:/var/www/babel.config.js",
                "#{vagrant_root}/sites/vue-storefront/config:/var/www/config",
                "#{vagrant_root}/sites/vue-storefront/core:/var/www/core",
                "#{vagrant_root}/sites/vue-storefront/ecosystem.json:/var/www/ecosystem.json",
                "#{vagrant_root}/sites/vue-storefront/.eslintignore:/var/www/.eslintignore",
                "#{vagrant_root}/sites/vue-storefront/.eslintrc.js:/var/www/.eslintrc.js",
                "#{vagrant_root}/sites/vue-storefront/lerna.json:/var/www/lerna.json",
                "#{vagrant_root}/sites/vue-storefront/tsconfig.json:/var/www/tsconfig.json",
                "#{vagrant_root}/sites/vue-storefront/tsconfig-build.json:/var/www/tsconfig-build.json",
                "#{vagrant_root}/sites/vue-storefront/shims.d.ts:/var/www/shims.d.ts",
                "#{vagrant_root}/sites/vue-storefront/package.json:/var/www/package.json",
                "#{vagrant_root}/sites/vue-storefront/src:/var/www/src",
                "/tmp/vuestorefront:/var/www/dist"
                ]
            d.env = { "BIND_HOST" => "0.0.0.0",
                      "NODE_CONFIG_ENV" => "docker",
                      "VS_ENV" => "#{mode}",
                      "PM2_ARGS" => "--no-daemon",
                      "NODE_TLS_REJECT_UNAUTHORIZED" => "0"
                    }
        end
    end

    if File.exist?("#{vagrant_root}/reverseproxy/nginx.conf")
        config.vm.define "reverseproxy", primary: false do |reverseproxy|
            reverseproxy.hostmanager.aliases = [ "reverseproxy."+dev_domain, "api."+dev_domain  ]
            reverseproxy.vm.network :private_network, ip: "#{ip_range}.210", subnet: "#{ip_range}.0/16"
            reverseproxy.vm.network "forwarded_port", guest: 22, host: Random.new.rand(1000...5000), id: 'ssh', auto_correct: true
            reverseproxy.vm.hostname = "reverseproxy"
            reverseproxy.vm.communicator = 'docker'
            reverseproxy.vm.provider 'docker' do |d|
                d.image = "nginx:latest"
                d.has_ssh = true
                d.name = "reverseproxy"
                d.remains_running = true
                d.volumes = [
                    "#{vagrant_root}/reverseproxy/nginx.conf:/etc/nginx/nginx.conf:ro",
                    "#{vagrant_root}/Docker/magento/common/nginx/ssl:/etc/nginx/ssl"
                ]
            end
        end
    end
end
