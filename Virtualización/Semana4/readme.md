# Examen Semana 4 

## Instrucción
El objeto de la práctica es crear una máquina virtual, y configura-la de forma automatizada , con la ayuda de Terraform y Ansible. 

En este sentido se debe prepara una máquina virtual con sistema operativo ***Ubuntu Server*** 18.04 LTS <https://ubuntu.com/download/server> con las siguientes características: 

- Memoria RAM 4G - 8G
- 4 CPU
- Almacenamiento 16 G - 32 G
- Una Tarjeta de Red
- Acceso a Internet
 
# Detalles de la configuración

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
Debe salir un valor mayor que cero
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

##  Pasos Previo 
En la maquina virtual crear un usuario de nombre igual al que se le asigno en el modulo 2 del diplomado 

Luego crear un par de llaves ssh para el este usuario 
```
ssh-keygen
Generating public/private rsa key pair.
Enter file in which to save the key (/home/jorge/.ssh/id_rsa):
Your public key has been saved in /home/jorge/.ssh/id_rsa.pub
The key fingerprint is:
SHA256:dBhDli7sh/5MR3E+ue9Wp/zWvmTnzDxFKo5b2KjbL9k jorge@c1
The key's randomart image is:
+---[RSA 3072]----+
|       .=.       |
|       ..+       |
|     . .o o .    |
|      o... + .  .|
|     . oS . +  o |
|      o .. +.o. +|
|     . .. +=+o ==|
|      .o ++.E.**+|
|       .=.o+.oo=X|
+----[SHA256]-----+
```

Esto crea los archivos 
 - ~/.ssh/id_rsa.pub   
 - ~/.ssh/id_rsa 
  
 Que son la llave pública y la llave privada respectivamete

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
  uri = "qemu:///system"
}
```

El segundo archivo libvirt.tf

```
# Defining VM Volume
resource "libvirt_volume" "xenial-qcow2" {
  name = "bionic.qcow2"
  pool = "default" # List storage pools using virsh pool-list
  source = "./bionic-server-cloudimg-amd64.img
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
  user_data = "${data.template_file.user_data.rendered}"
}

# Define KVM domain to create
resource "libvirt_domain" "xenial" {
  name   = "bionic"
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
ssh_pwauth: True
chpasswd:
  list: |
     root:sesamo
  expire: False

users:
  - name: jorge # Change me (nombre del dominio sin .com)
    ssh_authorized_keys:
      - ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABgQDE6KtvQDDhTVG1IpLzxnCck4Cjhq8E0QY1AyqazEZ0ZHpOhVGBUApga/fCmvnI5wgXOTWpLINXUHKItKNA1y10es112a9VpA+OH2rZ6FiUVtxQLGpMdlCtSlXNqr5AQ7djpUrT/167qeKkHkfq2tnolRbfFmGkG8L/Vn2wGHg/WjOAH95WH/vn5SKMN5HQwb3RuihoIF8ge2+8pi+APSnS7Npqjq6jirpzL+DEFfskJOyFkbsJX868g1LVCDqZUsfOBRlot+hTfeXIFUfZfptT1jUreJUp6ASmXaeOul45IOtntFZbsqxKJ3eceFqILGTJ+j4zCg08/clYPwMpQUBkXeKWrM9oWpSE+3AxAf1vZ0CdNAd6AP8nTGA3VWiGPdBvS1+lpwAVrkO/FMdueLpmSs3gqU3VVBsaEwSNfaw0Fd/0lJzUe1fFHkrsXt8k8rMm7wthd+WVGHO3qIQtD7UC4upiGqjwHXyTb+a+NYOnnzMCIa0+VOKtJ8JmuhUR6MU= jorge@c1
    sudo: ['ALL=(ALL) NOPASSWD:ALL']
    shell: /bin/bash
    groups: sudo
    lock_passwd: false
```
Por último para conseguir la imagen de cloud de Ubuntu Xenial :
```
wget https://cloud-images.ubuntu.com/bionic/current/bionic-server-cloudimg-amd64.img
```

4. Aumentar el tamaño del disco de la maquina virtual.
   
La maquina virtual por defecto de ubuntu bionic es de aproximadamente de 3G, y necesitaremos más espacio una vez creada por lo que debemos cambiar su tamaño de despliegue (no se cambiara el tamaño del archivo)

```
qemu-img resize bionic-server-cloudimg-amd64.img 32G 
```

5.  Ejecutar Terraform

Con todas las piezas listas solo queda ejecutar Terraform
son cuatro comandos los mas utilizados:
``terraform init`` (que prepara todo el ambiente), ``terraform plan`` que se encarga de crear un plan de ejecución, luego ``terraform apply`` que aplica la configuración y por último ``terraform destroy`` que destruye todo lo echo

## ansible
1. Instalación
   
Ansible es una herramienta de configuración programada (Configuration management), que se apoya en el concepto de infraestructura como código (Infrastructure as Code). La primera versión apareció en 2012 de la mano de Michael DeHaan como un proyecto pequeño en github, que en unos meses, tubo un ascenso meteorico, (17,000 estrellas y mas de 14100 contribuidores en github).

La instalación es regular, con los administradores de paquetes de cada distribución en caso de ubuntu:

```
sudo apt-add-repository ppa:ansible/ansible
sudo apt-get update
sudo apt-get install ansible
```

2. Creación del playBook

Al igual que terraform Asible trabaja con varios archivos de configuración, en este caso usaremos dos:
El primero llamado simplemente inventory (en su vesrión antigua) 

inventory

```
[bionic]
192.168.122.173 ansible_user=jorge ansible_ssh_private_key_file=./bionic.key

```
En este archivo podemos ver dos secciones, la primera es una agrupación de servidores llamada **bionic** en donde se debe poner el **IP** de la maquina virtual que devuelve Terraform.

Las variables ***ansible_user***, debe tener el usuario de la máquina virtual; y la variable ***ansible_ssh_private_key_file*** debe contener el path al archivo que contiene la llave privada de ssh; por conveniencia esta llave creada en pasos anteriores debe ser copiada al archivo ***bionic.key***

El segundo archivo es el playboock propiamente dicho:

playbock.yml


```
---
- hosts: bionic
  become: yes
  become_method: sudo
  tasks:
    - name: Make sure that we can connect to the vm 
      ping:
    - name: Update an upgrade apt packages
      apt:
        upgrade: yes
        update_cache: yes
        cache_valid_time: 8600 # One Day  
    - name: Install docker and some dependencies
      apt:
        name: python3-pip, docker.io, docker-compose
        state: present
    - name: Start docker service
      service:
          name:  docker  
          state: started    
    - name: Install docker python module
      pip:
        name: 
          - docker
          - docker-compose
    - name: Create a docker volume
      docker_volume:
        name: portainer_data
    - name: Install  portainer 
      docker_container:
        name: portainer
        image: portainer/portainer-ce:2.11.1
        state: started
        restart_policy: always
        ports:
          - "8000:8000"
          - "9443:9443"
        volumes: 
          - /var/run/docker.sock:/var/run/docker.sock
          - portainer_data:/data_portainer/
    - name: copy Docker compose yml 
      copy: 
        src: ../authelia
        dest: /tmp/
    - name: Install Authelia 
      docker_compose:
        project_src: /tmp/authelia  
        state: present



```
En este archivo tenemos la referencia a ../authelia, que es el directorio donde esta el docker-compose, de la aplicación <https://www.authelia.com/> que es un servidor de autenticación de código abierto.

Este directorio debe tener el siguiente contenido , (también se incluye en el repositorio)
```
authelia
|   authelia
|   |-- configuration.yml
|   |-- users_database.yml
|-- docker-compose.yml
```
En el archivo configuration.yml se debe cambiar los parámetros de correo electrolito acorde con una cuenta gratis de <https://mailtrap.io/> 
```
  smtp:                                                                         
       username: 1125fa2a234dr                                                    
       # This secret can also be set using the env variables AUTHELIA_NOTIFIER_SMTP_PASSWORD_FILE
       # From mailtrap.io                                                          
       password: 774c90831ssef
       host: smtp.mailtrap.io                                                      
       port: 2525                                                                  
       sender: admin@example.com 
```

3. Por último ejecutamos el playboock con ansible-playboock

```
ansible-playbook -i inventory playbook.yml
```
La opción -i especifica el archivo de inventario que se usara.

Para probar la aplicación authelia se debe crear los siguientes registros en el archivo /etc/hosts del anfitrión linux 

```
192.168.122.xxx authelia.example.com                                                     
192.168.122.xxx public.example.com                                                       
192.168.122.xxx traefik.example.com                                                      
192.168.122.xxx secure.example.com
```
Donde el ip es el mismo que se utilizo en el archivo "inventory" 

4. Este ejercicio se debe hacer en otro repositorio (directorio) de su cuenta de github, y debe llenarse acorde el archivo ***alumno.conf***  

```
Alumno: Jorge Antonio Verastegui
Email: diplomado@verastegui.net
github: https://github.com/SoeLinux/Examene3.git
```
 
