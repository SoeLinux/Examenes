# Examen Semana 1 

## Instrucción 
El propósito del primer Examen es introducir al alumno en la teología de Infraestructura como código (IAC).

En este sentido se debe prepara una máquina virtual con sistema operativo ***Ubuntu Server*** 18.04 LTS <https://ubuntu.com/download/server> con las siguientes características: 

- Memoria RAM 4G - 8G
- 4 CPU
- Almacenamiento 16 G - 32 G
- Una Tarjeta de Red
- Acceso a Internet

 # Detalle de configuración

 ## KVM 
 Para los estudiantes que tienen como sistema operativo anfitrión una distribución de Linux "moderna" kernel >= 4.4 pueden saltar a el punto **2**

 1. Se debe instalar KVM (dentro de la maquina) virtual, por lo que se debe habilitar la virtualización anidada en el anfitrión (host) Se ha podido comprobar con hyper-V que se puede habilitar y utilizar correctamente, en VirtualBox no funciona bien (es una función Beta) y en Mac es posible que la última versión de vmware Fusion funcione correctamente (Apple cambio algunas partes de la base de su soporte de virtualización por el cambio de procesadores ).
 
 En hyper-V se debe habilitar la virtualización anidada (de acuerdo a [https://docs.microsoft.com/en-us/virtualization/hyper-v-on-windows/user-guide/nested-virtualization](https://docs.microsoft.com/en-us/virtualization/hyper-v-on-windows/user-guide/nested-virtualization))

 En una consola de PowerShell (en modo administrador se debe correr) con la maquina virtual apagada:

 ```
 Set-VMProcessor -VMName UbuntuServer -ExposeVirtualizationExtensions $true 
 ```
Donde UbuntuServer es el nombre de la maquina virtual


2. Verificar que la maquina virtual tiene las extensiones de virtualización asistida por Hardware.
```
egrep -c '(vmx|svm)' /proc/cpuinfo
4
```   
Debe salir mayor que cero
Luego como siempre se deben actualizar toda la distro.
```
sudo apt update
sudo apt upgrade
```
Luego se puede instalar KVM

```
sudo apt -y install qemu-kvm libvirt-daemon bridge-utils virtinst libvirt-daemon-system
```   
Y luego las herramientas de administración 
```
sudo apt -y install virt-top libguestfs-tools libosinfo-bin  qemu-system virt-manager

```
Por último debemos estar seguros que el modulo de red este activo
```
sudo modprobe vhost_net
lsmod | grep vhost
vhost_net              32768  0
vhost                  49152  1 vhost_net
tap                    24576  1 vhost_net

echo vhost_net | sudo tee -a /etc/modules
```
# Terraform

1. Instalar Terraform:  
Terraform es una herramienta para infraestructura, como código (la más utilizada), y se distribuye directamente como, binario.   

```
sudo apt install wget curl unzip
TER_VER=`curl -s https://api.github.com/repos/hashicorp/terraform/releases/latest | grep tag_name | cut -d: -f2 | tr -d \"\,\v | awk '{$1=$1};1'`
wget https://releases.hashicorp.com/terraform/${TER_VER}/terraform_${TER_VER}_linux_amd64.zip

```   
Una vez que, el archivo este en la maquina virtual, es necesario copiar el binario en /usr/local/bin
```
unzip terraform_${TER_VER}_linux_amd64.zip
Archive:  terraform_1.1.8_linux_amd64.zip
  inflating: terraform  
sudo mv terraform /usr/local/bin/
``` 
Por último verificar que Terraform este listo para utilizar
```
terraform --version
Terraform v1.1.8
on linux_amd64
```
2. Instalar, el proveedor de KVM de Terraform.  
El proveedor de KVM, nos proporciona KVM a través de libvirt. Este esta mantenido por   [ Duncan Mac-Vicar P](https://github.com/dmacvicar/terraform-provider-libvirt) junto con otres contribuidores
El proveedor se autoinstala asi solo es necesario declararlo.
```
terraform {
  required_providers {
    libvirt = {
      source = "dmacvicar/libvirt"
    }
  }
}

provider "libvirt" {
  # Configuration options
}

```
3. Uso de Terraform para crear una máquina virtual KVM con una imagen de cloud, y la ayuda de cloud init.

```
mkdir -p ~/projects/terraform
cd ~/projects/terraform
```

se deben crear tres archivos : 
Primer archivo
main.tf
```
terraform {
  required_providers {
    libvirt = {
      source = "dmacvicar/libvirt"
    }
  }
}

provider "libvirt" {
  ## Configuration options
  uri = "qemu:///system"
  #alias = "server2"
  #uri   = "qemu+ssh://root@192.168.100.10/system"
}
```
El segundo archivo libvirt.tf

```
# Defining VM Volume
resource "libvirt_volume" "xenial-qcow2" {
  name = "xenial.qcow2"
  pool = "default" # List storage pools using virsh pool-list
  #source = "https://cloud.centos.org/centos/7/images/CentOS-7-x86_64-GenericCloud.qcow2"
  source = "./xenial-server-cloudimg-amd64-disk1.img"
  format = "qcow2"
}
# get user data info
data "template_file" "user_data" {
  template = "${file("${path.module}/cloud_init.cfg")}"
}

# Use CloudInit to add the instance
resource "libvirt_cloudinit_disk" "commoninit" {
  name = "commoninit.iso"
  pool = "default" # List storage pools using virsh pool-list
  user_data      = "${data.template_file.user_data.rendered}"
}


# Define KVM domain to create
resource "libvirt_domain" "xenial" {
  name   = "xenial"
  memory = "2048"
  vcpu   = 2

  network_interface {
    network_name = "default" # List networks with virsh net-list
    wait_for_lease = true
  }

  disk {
    volume_id = "${libvirt_volume.xenial-qcow2.id}"
  }

  cloudinit = "${libvirt_cloudinit_disk.commoninit.id}"

  console {
    type = "pty"
    target_type = "serial"
    target_port = "0"
  }

  graphics {
    type = "spice"
    listen_type = "address"
    autoport = true
  }
}

# Output Server IP
  output "ip" {
    value = "${libvirt_domain.xenial.network_interface.0.addresses.0}"
 }
```

Y el último archivo es cloud_init.cfg
```
#cloud-config
# vim: syntax=yaml
#
# ***********************
# 	---- for more examples look at: ------
# ---> https://cloudinit.readthedocs.io/en/latest/topics/examples.html
# ******************************
#
# This is the configuration syntax that the write_files module
# will know how to understand. encoding can be given b64 or gzip or (gz+b64).
# The content will be decoded accordingly and then written to the path that is
# provided.
#
# Note: Content strings here are truncated for example purposes.
# se debe reamplazar jorge por su nombre de dominio del anterior modulo sin el .com
ssh_pwauth: True
chpasswd:
  list: |
     root:sesamo
     jorge:sesamo  
  expire: False

users:
  - name: jorge # Change me (nombre del dominio sin .com)
    sudo: ['ALL=(ALL) NOPASSWD:ALL']
    shell: /bin/bash
    groups: sudo
```
Por último para conseguir la imagen de cloud de Ubuntu Xenial :
```
wget https://cloud-images.ubuntu.com/xenial/current/xenial-server-cloudimg-amd64-disk1.img
```
4.  Ejecutar Terraform
Con todas las piezas listas solo queda ejecutar Terraform
son cuatro comandos los mas utilizados:
``terraform init`` (que prepara todo el ambiente), ``terraform plan`` que se encarga de crear un plan de ejecución, luego ``terraform apply`` que aplica la configuración y por último ``terraform destroy`` que destruye todo lo echo

5. Se debe Crear un repositorio git llamado examen1 dentro de su cuenta de github, en este repositorio debe estar toda la configuración nesaria para que se pueda reproducir este ejercicio en cualquier ambiente. 
6. Para estandarizar la entrega de los repositorios se debe crear un último archivo de configuración ***"alumno.conf"*** con la siguiente contenido

```
Alumno: Jorge Antonio Verastegui
Email: diplomado@verastegui.net
github:https://github.com/SoeLinux/Examenes.git
```










