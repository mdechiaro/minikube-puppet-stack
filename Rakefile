# frozen_string_literal: true

require 'rake'

def minikube_running?
  return true if system('docker ps | grep minikube > /dev/null')

  false
end

def puppetca_running?
  sleep 5 until system('kubectl get pods | grep puppetca | grep Running > /dev/null')

  puppetca = `kubectl get pods | grep puppetca | awk '{print $1}'`.chomp
  status = `kubectl exec -it #{puppetca} -- curl -s -k https://localhost:8140/status/v1/simple 2> /dev/null`

  return true if status == 'running'

  false
end

desc 'Adjust minikube configs'
task :minikube_config do
  sh 'minikube config set cpus 4'
  sh 'minikube config set memory 8192'
end

desc 'Load docker images into minikube'
task :minikube_load_images, [:file] do |_, args|
  images = File.read(args.file).split("\n")
  images.each do |image|
    sh "minikube image load #{image}" unless image.start_with?('#') || image.empty?
  end
end

desc 'Generate a pseudo-random key'
task :generate_urandom_key do
  print `tr -dc [:alpha:] < /dev/urandom | head -c 32`
end

desc 'Generate Puppet certificate for applications'
task :generate_puppet_cert, [:app_name] do |_, args|
  puppetca = `kubectl get pods | grep puppetca | awk '{print $1}'`.chomp
  puppetdir = '/etc/puppetlabs/puppet/ssl'
  exec_cmd = "kubectl exec -it #{puppetca}"

  puts "Generating puppet certificate for #{args.app_name}"

  gen_cert = <<~HEREDOC
    #{exec_cmd} -- puppetserver ca generate --certname #{args.app_name}
  HEREDOC

  sh gen_cert

  ca = "#{puppetdir}/ca/ca_crt.pem"
  key = "#{puppetdir}/private_keys/#{args.app_name}.pem"
  cert = "#{puppetdir}/certs/#{args.app_name}.pem"
  local = "secrets/#{args.app_name}"

  sh "kubectl cp default/#{puppetca}:#{ca} #{local}/ca.pem"
  sh "kubectl cp default/#{puppetca}:#{key} #{local}/key.pem"
  sh "kubectl cp default/#{puppetca}:#{cert} #{local}/cert.pem"
end

desc 'Install NATS.io server'
task :install_nats_server do
  service_name = 'nats-server'
  local = "secrets/#{service_name}"
  Rake::Task[:generate_puppet_cert].invoke(service_name)
  Rake::Task[:generate_puppet_cert].reenable

  sh "mv #{local}/cert.pem #{local}/tls.crt"
  sh "mv #{local}/key.pem #{local}/tls.key"
  sh "mv #{local}/ca.pem #{local}/ca.crt"

  sh "kubectl create secret generic nats-server-tls-certs --from-file #{local}"
  # ca setup needs its own secret with ca.pem
  sh "kubectl create secret generic nats-ca-tls-certs --from-file #{local}/ca.crt"
end

desc 'Install NATS.io client'
task :install_nats_client do
  service_name = 'nats-client'
  local = "secrets/#{service_name}"
  Rake::Task[:generate_puppet_cert].invoke(service_name)
  Rake::Task[:generate_puppet_cert].reenable

  sh "mv #{local}/cert.pem #{local}/tls.crt"
  sh "mv #{local}/key.pem #{local}/tls.key"
  sh "kubectl create secret generic nats-client-tls-certs --from-file #{local}"
end

desc 'Install NATS.io broker'
task :install_nats do
  Rake::Task[:install_nats_server].execute
  Rake::Task[:install_nats_client].execute

  sh 'helm install nats-server nats/nats -f charts/nats.yaml'
end

desc 'Install puppetboard'
task :install_puppetboard do
  service_name = 'puppetboard.default.svc.cluster.local'
  local = "secrets/#{service_name}"
  Rake::Task[:generate_puppet_cert].invoke(service_name)
  Rake::Task[:generate_puppet_cert].reenable

  <<~`HEREDOC`
    kubectl create secret generic puppetboard-ssl-cert \
      --from-literal=PUPPETBOARD_SSL_CERT="$(base64 #{local}/cert.pem)"
  HEREDOC
  <<~`HEREDOC`
    kubectl create secret generic puppetboard-ssl-ca-cert \
      --from-literal=PUPPETBOARD_CA_CERT="$(base64 #{local}/ca.pem)"
  HEREDOC
  <<~`HEREDOC`
    kubectl create secret generic puppetboard-ssl-key \
      --from-literal=PUPPETBOARD_SSL_KEY="$(base64 #{local}/key.pem)"
  HEREDOC
  <<~`HEREDOC`
    kubectl create secret generic puppetboard-secret-key \
      --from-literal=PUPPETBOARD_SECRET_KEY="$(rake generate_urandom_key)"
  HEREDOC

  sh 'kubectl apply -f apps/puppetboard.yaml'
end

desc 'Setup TLS logs'
task :setup_tls_logs do
  syslog_local = 'secrets/syslog-ng.default.svc.cluster.local'
  Rake::Task[:generate_puppet_cert].invoke('syslog-ng.default.svc.cluster.local')
  Rake::Task[:generate_puppet_cert].reenable
  sh "kubectl create secret generic syslog-tls-certs --from-file #{syslog_local}"

  fluentd_local = 'secrets/fluentd'
  Rake::Task[:generate_puppet_cert].invoke('fluentd')
  Rake::Task[:generate_puppet_cert].reenable
  sh "kubectl create secret generic fluentd-tls-certs -n kube-system --from-file #{fluentd_local}"

  sh 'kubectl apply -f logs/'
end

desc 'Run all steps to setup stack'
task :setup_stack do
  sh 'minikube start' unless minikube_running?

  `rm -rf secrets && mkdir secrets`

  Rake::Task[:minikube_load_images].invoke('images.list')

  psql_pass = `rake generate_urandom_key`.to_s
  sh "kubectl create secret generic postgres-password --from-literal=POSTGRES_PASSWORD=#{psql_pass}"

  sh 'kubectl apply -f puppet/'

  puts 'Waiting for PuppetCA to start...'

  sleep 5 until puppetca_running?

  Rake::Task[:setup_tls_logs].execute
  Rake::Task[:install_nats].execute
  Rake::Task[:install_puppetboard].execute

  `rm -rf secrets`
end
