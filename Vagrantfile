# -*- mode: ruby -*-
# vi: set ft=ruby :

NetworkAdapter = Struct.new(:name, :mac_address)

# If bridged, the adapter name will be returned. Otherwise, `nil` is returned.
def is_bridged
  id_path = '.vagrant/machines/default/virtualbox/id'
  if File.exist?(id_path)
    id = File.open(id_path) { |io| io.read }
  end

  output = %x[VBoxManage showvminfo "#{id}" --machinereadable 2>&1]
  adapter_name = nil
  output.lines.find do |line|
    if md = /^bridgeadapter2="(.+)"$/.match(line.chomp)
      adapter_name = md[1]
    end
  end
  adapter_name
end

def get_network_adapters
  output = %x[VBoxManage list bridgedifs]
  network_adapters = []
  network_adapter = NetworkAdapter.new
  output.each_line do |line|
    parts = line.split(':', 2)
    next if parts.size != 2
    case parts[0]
    when 'Name'
      if network_adapter.name
        network_adapters << network_adapter
        network_adapter = NetworkAdapter.new
      end
      network_adapter.name = parts[1].strip
    when 'HardwareAddress'
      network_adapter.mac_address = parts[1].strip
    end
  end

  network_adapters << network_adapter if network_adapter.name
  network_adapters
end

if ENV['RESET_BRIDGE'] == '1' || !(bridged_adapter_name = is_bridged)
  network_adapters = get_network_adapters
  if network_adapters.size > 1
    puts 'Available network interfaces to bridge to:'
    puts network_adapters.map.with_index(1) { |na, idx| "#{idx}. #{na.name}" }
    input = nil
    loop do
      print 'Please input a number: '
      input = STDIN.gets.chomp.to_i
      break if input > 0 && input <= network_adapters.size
    end
  elsif network_adapters.size == 1
    input = 1
  else
    puts 'No network adapter found'
    exit 1
  end
  selected_network_adapter = network_adapters[input - 1]
end

Vagrant.configure('2') do |config|
  config.vm.box = 'uzxmx/mybox'
  config.ssh.forward_agent = true

  if selected_network_adapter
    # Specify a host NIC that you want to bridge to by using `bridge` option. The
    # NIC should already connect to the Internet. The bridged IP address will be
    # the gateway (router) address for other devices that want to connect to the
    # Internet. You can specify a fixed ip address by using `ip` option.
    config.vm.network :public_network, bridge: selected_network_adapter.name

    config.vm.provider :virtualbox do |vb|
      # Change the promiscuous mode of the second NIC. We also need to do this again in the VM.
      vb.customize ['modifyvm', :id, '--nicpromisc2', 'allow-all']

      # When client device sends ARP request to get the mac address of the router
      # VM, it's the mac address of the bridged NIC that will be returned. So the
      # destination mac address of the incoming packet will be the mac address of
      # the bridged NIC. The router VM will drop such packet because it thinks
      # that packet is not for it (PACKET_OTHERHOST, also see net/ipv4/ip_input.c).
      #
      # To avoid this issue, We must change the mac address of the second NIC to the
      # mac address of the bridged NIC.
      vb.customize ['modifyvm', :id, '--macaddress2', selected_network_adapter.mac_address.gsub(':', '')]
    end
  else
    # We must specify the `public_network` every time, otherwise Vagrant will
    # disable the second NIC. And we also need to specify the bridged adapter
    # name, otherwise Vagrant will ask us to select a network interface again.
    config.vm.network :public_network, bridge: bridged_adapter_name
  end

  config.vm.provision :shell, inline: <<-EOF
    # Enable Linux kernel ip forwarding permanently. This is the key to
    # functioning as a router.
    echo net.ipv4.ip_forward=1 >>/etc/sysctl.conf

    # In order to enable ip forwarding after provision, we must do this.
    # Otherwise, we need to restart the VM.
    sysctl net.ipv4.ip_forward=1

    yum install -y tcpdump
  EOF

  config.vm.provision :shell, run: :always, inline: <<-EOF
    # Change the promiscuous mode of the second NIC.
    ip link set eth1 promisc on

    # SNAT
    iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE

    # DNS relay to the first nameserver entry in /etc/resolv.conf
    dns_server=$(cat /etc/resolv.conf | grep '^nameserver' | head -1 | awk '{print $2}')
    iptables -t nat -A PREROUTING -p tcp --dport 53 -j DNAT --to-destination $dns_server
    iptables -t nat -A POSTROUTING -p tcp -d $dns_server --dport 53 -j MASQUERADE
    iptables -t nat -A PREROUTING -p udp --dport 53 -j DNAT --to-destination $dns_server
    iptables -t nat -A POSTROUTING -p udp -d $dns_server --dport 53 -j MASQUERADE
  EOF
end
