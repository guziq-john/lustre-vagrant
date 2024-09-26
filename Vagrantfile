Vagrant.configure("2") do |config|
	config.vm.box = "generic/rhel8"
	config.vm.box_version = "4.3.2"

	config.vm.provider "qemu" do |qe, override|
	  qe.arch = "x86_64"
	  qe.machine = "q35"
	  qe.cpu = "max"
      qe.net_device = "virtio-net-pci"
	end

	config.vm.define "server" do |adm|
		adm.vm.provider "qemu" do |qe|
		  qe.ssh_port = "50023"
		  qe.extra_qemu_args = %w(-drive file=server-disk1.img,format=raw,if=virtio -netdev vmnet-shared,id=net1 -device virtio-net-pci,netdev=net1,mac=54:54:00:55:54:51)
		end
	end

	config.vm.define "client" do |adm|
		adm.vm.provider "qemu" do |qe|
		  qe.ssh_port = "50024"
		  qe.extra_qemu_args = %w(-netdev vmnet-shared,id=net1 -device virtio-net-pci,netdev=net1,mac=54:54:00:55:54:52)
		end
	end
end
