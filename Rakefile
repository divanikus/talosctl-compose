require 'erb'
require 'yaml'
require 'json'
require 'pp'

Dir.chdir(Rake.original_dir)

TALOS_VERSION      = "1.7.4"

module Plan
  @@_data = File.exist?("plan.yaml") ? YAML.load_file("plan.yaml") : {}

  class << self
    def data
      @@_data
    end

    def nodes
      @@_data['nodes'].map do |node|
        node['machines'].map do |m|
          m['ip']
        end
      end.flatten
    end

    def endpoints
      @@_data['nodes'].map do |node|
        if node['type'] == 'controlplane'
          node['machines'].map do |m|
            m['ip']
          end
        end
      end.flatten
    end

    def assetsUrl
      data['assetsBaseUrl'] + data['name'] + '/'
    end

    def talosManifest(index)
      return nil if @@_data['nodes'].nil? or @@_data['nodes'][index].nil?
      "%02d-%s" % [index, @@_data['nodes'][index]['type']]
    end

    def talosProfile(index)
      return nil if @@_data['nodes'].nil? or @@_data['nodes'][index].nil?
      "%s-%02d-%s" % [@@_data['name'], index, @@_data['nodes'][index]['type']]
    end

    def find_file(type, name)
      file = "#{type}/#{name}"
      ["#{file}.yaml", "#{file}.yaml.erb", "../../#{file}.yaml", "../../#{file}.yaml.erb"].select { |t| File.exist?(t)  }.first
    end

    def get_type(type, index)
      items  = (@@_data.key?(type) and @@_data[type].is_a?(Array)) ? @@_data[type] : []
      items += @@_data['nodes'][index][type] if (@@_data['nodes'][index].key?(type) and @@_data['nodes'][index][type].is_a?(Array))
      items
    end

    def each(type, index, &block)
      items = get_type(type, index)

      items.each do |item|
        file = find_file(type, item)
        puts "WARN! Item '#{type}/#{item}' not found" if file.nil?
        next if file.nil?

        if block_given?
          yield(item, file)
        end
      end
    end

    def render(template)
      ERB.new(template).result(binding)
    end
  end
end

def generate(item, src, dst)
  puts "  Genereting #{dst}..."
  file = File.read(src)
  file = Plan.render(file) if src.end_with?(".erb")

  opts = YAML.load(file)
  case opts['type']
  when "helm"
    generate_helm(item, opts, dst)
  else
    raise "Unknown generator type #{opts['type']}"
  end
end

def generate_helm(item, opts, dst)
  if opts.key?("repo")
    sh "helm repo add #{opts['repo']['name']} #{opts['repo']['url']}"
    sh "helm repo update"
  end

  if opts.key?("values")
    File.write("out/tmp/#{item}.values.yaml", opts['values'].to_yaml)
  end

  cmd  = "helm template #{item} #{opts['chart']} --include-crds "
  cmd += "--namespace #{opts['namespace']} " if opts.key?("namespace")
  cmd += "--version #{opts['version']} " if opts.key?("version")
  cmd += "-f out/tmp/#{item}.values.yaml " if opts.key?("values")
  cmd += "> #{dst}"
  sh cmd
end

file "/usr/local/bin/talosctl" do |t|
  sh "curl -L --fail -o talosctl https://github.com/siderolabs/talos/releases/download/v#{TALOS_VERSION}/talosctl-linux-amd64"
  chmod "+x", "talosctl"

  if Process.uid == 0
    mv "talosctl", "/usr/local/bin/"
  else
    puts "WARN! Re-run task as root to install"
  end
end

file "talosctl.exe" do |t|
  sh "curl -L --fail -o talosctl.exe https://github.com/siderolabs/talos/releases/download/v#{TALOS_VERSION}/talosctl-windows-amd64.exe"
end

case RUBY_PLATFORM 
when /x86_64-linux/
  task talosctl: ["/usr/local/bin/talosctl"]
when /x64-mingw/
  task talosctl: ["talosctl.exe"]
end

task "clean" do |t|
  Dir.glob('**/out').each do |x|
    next unless File.directory?(x)
    puts "Removing #{x}..."
    FileUtils.rm_rf x
  end
end

unless Plan.data.empty?
  directory "out"
  
  file "plan.yaml" do |t|
    puts "Create plan.yaml first!"
    exit -1
  end
  
  file "secrets.yaml": ["plan.yaml"] do |t|
    # Forcibly ignore existing secrets.yaml
    if File.exist?("secrets.yaml")
      puts "File 'secrets.yaml' already exists"
      #FileUtils.touch("secrets.yaml")
    else
      sh "talosctl gen secrets"
    end
  end
  
  file "out/talosconfig": ["secrets.yaml", "out"] do |t|
    sh "talosctl gen config -t talosconfig -o out/talosconfig --with-secrets secrets.yaml #{Plan.data['name']} #{Plan.data['controlPlane']['endpoint']}"
    sh "talosctl --talosconfig out/talosconfig config endpoint #{Plan.endpoints.join(' ')}"
    sh "talosctl --talosconfig out/talosconfig config node #{Plan.nodes.join(' ')}"
  end
  
  task talosconfig: ["out/talosconfig"]
  
  task install_talosconfig: :talosconfig do |t|
    sh "talosctl config merge out/talosconfig"
  end
end

assets   = []
profiles = []
groups   = []

if Plan.data.key?('nodes') and !Plan.data['nodes'].empty?
  version = Plan.data.fetch("version", TALOS_VERSION)

  tmp_dir  = "out/tmp"
  ptch_dir = "out/tmp/patches"
  asst_base_dir = "../../out/assets"
  asst_dir  = "#{asst_base_dir}/#{Plan.data['name']}"
  talos_dir = "#{asst_base_dir}/talos/v#{version}"
  defined  = []

  directory tmp_dir
  directory ptch_dir
  directory asst_base_dir
  directory talos_dir
  directory asst_dir

  file talos_dir + "/vmlinuz" => talos_dir do |t|
    sh "curl -L --fail -o #{talos_dir}/vmlinuz https://github.com/siderolabs/talos/releases/download/v#{version}/vmlinuz-amd64"
  end
  assets.push(talos_dir + "/vmlinuz")
  
  file talos_dir + "/initramfs.xz" => talos_dir do |t|
    sh "curl -L --fail -o #{talos_dir}/initramfs.xz https://github.com/siderolabs/talos/releases/download/v#{version}/initramfs-amd64.xz"
  end
  assets.push(talos_dir + "/initramfs.xz")

  Plan.data['nodes'].each.with_index do |node, index|
    manifest = Plan.talosManifest(index) + ".yaml"
    path     = "#{asst_dir}/#{manifest}"
    deps     = []
    patches  = []

    Plan.each("patches", index) do |item, src|
      dst = "#{ptch_dir}/#{item}.yaml"
      deps.push(dst)
      patches.push(dst)
      next if defined.include?(dst)
      defined.push(dst)

      file dst => ptch_dir do |t|
        if src.end_with?('.erb')
          File.write(dst, Plan.render(File.read(src))) 
          puts "gen #{src} #{dst}"
        else
          cp src, dst
        end
      end
    end

    Plan.each("generate", index) do |item, src|
      dst = "#{asst_dir}/#{item}.yaml"
      deps.push(dst)
      next if defined.include?(dst)
      defined.push(dst)
      
      file dst => [asst_dir, tmp_dir] do |t|
        puts "gen #{src} #{dst}"
        generate(item, src, dst)
      end
    end

    Plan.each("manifests", index) do |item, src|
      dst = "#{asst_dir}/#{item}.yaml"
      deps.push(dst)
      next if defined.include?(dst)
      defined.push(dst)

      file dst => asst_dir do |t|
        if src.end_with?('.erb')
          File.write(dst, Plan.render(File.read(src)))
          puts "gen #{src} #{dst}"
        else
          cp src, dst
        end
      end
    end

    manifests = Plan.get_type("manifests", index).map { |x| "#{Plan.assetsUrl}#{x}.yaml" }
    unless manifests.empty?
      dst = "%s/%02d-manifests.yaml" % [ptch_dir, index]
      deps.push(dst)
      patches.push(dst)
      next if defined.include?(dst)
      defined.push(dst)

      file dst => ptch_dir do |t|
        File.write(dst, {
          "cluster" => {
            "extraManifests" => manifests
          }
        }.to_yaml)
        puts "gen #{dst}"
      end
    end

    file path => deps + ["secrets.yaml"] do |t|
      puts "Building #{path}..."

      type = node['type'] == 'controlplane' ? 'controlplane' : 'worker' # stupid sanity check
      cmd  = "talosctl gen config -t #{type} -o #{path} --with-secrets secrets.yaml "
      cmd += "#{Plan.data['name']} #{Plan.data['controlPlane']['endpoint']} "
      cmd += "--install-disk #{node['installDisk']} " if node.key?('installDisk')
      cmd += "--talos-version v#{Plan.data['version']} " if Plan.data.key?('version')
      cmd += "--kubernetes-version #{Plan.data['kubeVersion']} " if Plan.data.key?('kubeVersion')
      patches.each { |x| cmd += "--config-patch @#{x} " }

      sh cmd
    end

    assets.push(path)

    prof_dir  = "../../out/profiles"
    group_dir = "../../out/groups"
    directory prof_dir
    directory group_dir

    profile      = Plan.talosProfile(index)
    profile_file = "#{prof_dir}/#{profile}.json"
    profiles.push(profile_file)

    file profile_file => prof_dir do |t|
      puts "Building #{profile_file}..."
      File.write(profile_file, JSON.pretty_generate({
        "id": profile,
        "name": profile,
        "boot": {
          "kernel": "/assets/talos/v#{version}/vmlinuz",
          "initrd": ["/assets/talos/v#{version}/initramfs.xz"],
          "args": [
            "initrd=initramfs.xz",
            "init_on_alloc=1",
            "slab_nomerge",
            "pti=on",
            "console=tty0",
            "console=ttyS0",
            "printk.devkmsg=on",
            "talos.platform=metal",
            "talos.config=#{Plan.assetsUrl}#{manifest}"
          ]
        }
      }))
    end

    node['machines'].each.with_index do |machine, i|
      group      = "%s-%02d" % [profile, i]
      group_file = "#{group_dir}/#{group}.json" 
      groups.push(group_file)

      file group_file => group_dir do |t|
        puts "Building #{group_file}..."
        File.write(group_file, JSON.pretty_generate({
          "id": group,
          "name": group,
          "profile": profile,
          "selector": {
            "mac": machine['mac']
          }
        }))
      end
    end if node.key?("machines")
  end
end

if __dir__ == Rake.original_dir
  task :default do |t|
    Dir.glob("clusters/**").each do |dir|
      Dir.chdir(dir) do
        sh "rake"
      end
    end
  end
else
  task "assets" => assets
  task "profiles" => profiles
  task "groups" => groups
  task "matchbox" => ["profiles", "groups"]
  task :default => ["assets", "matchbox"]
end
