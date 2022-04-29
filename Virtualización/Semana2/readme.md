# Examen Semana 2 

## Instrucción
El objeto de la práctica es crear una máquina virtual, y configura-la de forma automatizada , con la ayuda de Terraform y Ansible. 

En la configuración además de actualizar los paquetes, se instalara Docker, y qemu-agent y una imagen de Nginx exponiendo los puertos 80 y 443.

##  Pasos Previo 
Crear un par de llaves ssh (si es necesario )

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

Esto crea el archivo ~/.ssh/id_rsa.pub y para copiar el contenido se puede usar re-direccionamiento:
```
cat ~/.ssh/id_rsa_pub >> cloud_init.cfg
```
Que copia el final del archivo la llave pública. 
El archivo final debería ser:
```
#cloud-config
# vim: syntax=yaml
#
ssh_pwauth: True
chpasswd:
  list: |
     root:sesamo
  expire: False

users:
  - name: jorge
    ssh_authorized_keys:
      - ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABgQDE6KtvQDDhTVG1IpLzxnCck4Cjhq8E0QY1AyqazEZ0ZHpOhVGBUApga/fCmvnI5wgXOTWpLINXUHKItKNA1y10es112a9VpA+OH2rZ6FiUVtxQLGpMdlCtSlXNqr5AQ7djpUrT/167qeKkHkfq2tnolRbfFmGkG8L/Vn2wGHg/WjOAH95WH/vn5SKMN5HQwb3RuihoIF8ge2+8pi+APSnS7Npqjq6jirpzL+DEFfskJOyFkbsJX868g1LVCDqZUsfOBRlot+hTfeXIFUfZfptT1jUreJUp6ASmXaeOul45IOtntFZbsqxKJ3eceFqILGTJ+j4zCg08/clYPwMpQUBkXeKWrM9oWpSE+3AxAf1vZ0CdNAd6AP8nTGA3VWiGPdBvS1+lpwAVrkO/FMdueLpmSs3gqU3VVBsaEwSNfaw0Fd/0lJzUe1fFHkrsXt8k8rMm7wthd+WVGHO3qIQtD7UC4upiGqjwHXyTb+a+NYOnnzMCIa0+VOKtJ8JmuhUR6MU= jorge@c1
    sudo: ['ALL=(ALL) NOPASSWD:ALL']
    shell: /bin/bash
    groups: sudo
```
## Intalacion de ansible
Ansible es una herramienta de configuración programada (Configuration management), que se apoya en el concepto de infraestructura como código (Infrastructure as Code). La primera versión apareció en 2012 de la mano de Michael DeHaan como un proyecto pequeño en github, que en unos meses, tubo un ascenso meteorico, (17,000 estrellas y mas de 14100 contribuidores en github).

La instalación es regular, a las 

