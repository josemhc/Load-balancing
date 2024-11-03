Vagrant.configure("2") do |config|
    # Define the first VM (Node 1)
    config.vm.define :node1 do |node1|
      node1.vm.box = "bento/ubuntu-22.04"
      node1.vm.network :private_network, ip: "192.168.60.11"
      node1.vm.hostname = "node1"
    end
  
    # Define the second VM (Node 2)
    config.vm.define :node2 do |node2|
      node2.vm.box = "bento/ubuntu-22.04"
      node2.vm.network :private_network, ip: "192.168.60.12"
      node2.vm.hostname = "node2"
    end
  
    # Define the third VM (Node 3)
    config.vm.define :node3 do |node3|
      node3.vm.box = "bento/ubuntu-22.04"
      node3.vm.network :private_network, ip: "192.168.60.13"
      node3.vm.hostname = "node3"
    end

    # Define the fourth VM (Sysbench)
    config.vm.define :sysbench do |sysbench|
      sysbench.vm.box = "bento/ubuntu-22.04"
      sysbench.vm.network :private_network, ip: "192.168.60.14"
      sysbench.vm.hostname = "sysbench"
    end
  end
  
  
