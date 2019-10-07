# Proyecto DevOps Engineer

## Resumen 📃

Este proyecto se basa en el aprovisionamiento de infraestructura  a través de herramientas automatizadas, usando virtualización que permitan la aceleraciónintegración continuaentre distintos sistemas creando asi ambientes mas estables, tanto de desarrollo, Prueba, producción.

## Prerrequisitos :exclamation:

- Necesitarás un software de virtualización, utilizaremos Virtual Box versión 6.0 
- Necesitarás una herramienta de creación de entornos de desarrollo, utilizaremos Vagrant version 2.2.5
- Editor de código fuente donde almacenar las carpetas y el código, utilizaremos Visual Studio Code versión 1.36.31 (Recomendación) :nerd_face:

## Paso a Paso :dizzy:

- ### Crear Maquina Virtual con Vagrant y Provisionar Docker Engine con Ansible :heavy_check_mark:
- Abrir la consola de comandos y crear una carpeta en donde realizaremos el proyecto. **mkdir folder**
- Luego nos posicionamos en esa carpeta. **cd folder**
- Ejecutamos el comando **vagrant init** para que se cree un archivo Vagrantfile en la ubicación donde ejecutamos el comando, que cuenta con los datos mínimos e indispensables para empezar a crear nuestra maquina virtual. :muscle:
- En este archivo Vagrantfile, definiremos el sistema operativo que vamos a utilizar dentro de nuestra maquina virtual. Para este caso utilizaremos Linux Centos 7.
- El Vagrantfile que usaremos se encuentra a continuación. :computer:

                  Vagrant.configure(2) do |config|
                  config.vm.network "public_network"

                  config.vm.box = "centos/7"

                  config.vm.network "forwarded_port", guest: 8080, host: 8080

                  config.vm.provider "virtualbox" do |v|
                      v.name = "PFG_VM"
                      v.memory = 1024
                  end

                  config.vm.provision "ansible_local", run: "always" do  |ansible|
                    ansible.playbook = "initial-setup.yml"
                  end

                  config.vm.provision "ansible_local" do |ansible|
                    ansible.playbook = "app-provision.yml"
                  end :computer:

          end
          
          
En el cual definiremos el proveedor a utilizar en la maquina virtual (config.vm.provider "virtualbox" do |v|), definiremos el box del sistema operativo que utilizaremos **config.vm.box = "centos/7"**, se configuraran los puertos para la maquina virtual **config.vm.network "forwarded_port, guest: 8080, host: 8080**, además definiremos nuestra herramientra de provisionamiento de infraestructura que usaremos el cual será ansible **config.vm.provision "ansible_local", run: "always" do  |ansible|**, haremos uso de playbook o tareas para el aprovisionamiento de la infraestructura de nuestra maquina y lo invocaremos mediante la siguiente instrucción **ansible.playbook = "initial-setup.yml** en la cual hacemos referencia a un archivo llamado "initial-setup.yml" el cual contiene todas las tareas a ejecutarse para el aprovisionamiento de nuestro docker engine. :whale:

Además de esto encontramos el archivo **ansible.playbook = "app-provision.yml"** el cual utilizaremos para desplegar una aplicación dockerizada en nuestra máquina virtual haciendo uso de docker-compose. :whale:

- A continuación tenemos nuestro archivo initial-setup.yml :computer:


      ---
      - name: Initial Provision
        hosts: all
        become: true

        vars:

          docker_repo: docker-ce-stable
          docker_version: latest
          docker_repo_url: "https://download.docker.com/linux/centos/docker-ce.repo"
          docker_users:
            - vagrant
          docker_conf: >
            ExecStart=/usr/bin/dockerd
            --storage-driver=devicemapper
            --selinux-enabled=false

        tasks:

          - name: Ensures yum-utils exists
            yum:
              pkg: yum-utils
              state: present

          - name: Adds docker repo
            shell: yum-config-manager --add-repo {{ docker_repo_url }}

          - name: Ensure latest docker version is installed
            yum:
              pkg: ['docker-ce', 'python-docker-py']
              state: installed
              enablerepo: "{{ docker_repo }}"
            notify:
              - restart docker
            when: docker_version == 'latest'

          - name: "Ensure that the specified docker version is installed"
            yum:
              pkg: ['docker-ce-{{docker_version }}.ce-1.el7.centos', 'python-docker-py']
              state: installed
              enablerepo: "{{ docker_repo }}"
            when: docker_version != 'latest'
            notify:
              - restart docker

          - name: Ensure docker is enabled to start at boot.
            service:
              name: docker
              enabled: yes

          - name: Add users to docker group
            shell: "usermod -a -G docker {{ item }}"
            with_items:
              - "{{ docker_users }}"

          - name: Apply custom configuration
            lineinfile:
              dest: /usr/lib/systemd/system/docker.service
              regexp: ^ExecStart
              line: "{{ docker_conf }}"
              backup: yes
            when: docker_conf is defined
            register: conf
            notify:
              - restart docker

          - name: Reload conf systemd
            shell: systemctl daemon-reload
            when: docker_conf is defined and conf.changed

          - name: "Check if docker compose is already installed"
            shell: "/usr/local/bin/docker-compose -v"
            register: compose
            ignore_errors: yes
            no_log: True
            failed_when: "'command not found' in compose.stderr"

          - name: "Installs docker compose"
            shell: >
              curl -L https://github.com/docker/compose/releases/download/1.18.0/docker-compose-`uname -s`-`uname -m` > /usr/local/bin/docker-compose && \
              chmod +x /usr/local/bin/docker-compose
            when: "'docker-compose: No such file or directory' in compose.stderr"

        handlers:

          - name: restart docker
            service:
              name: docker
              state: restarted
              
              
 En este archivo definiremos las tareas a ejecutarse para el aprovisionamiento de nuestro docker engine :whale:
 
 - Comenzaremos por la definición de nuestro inventario de hosts el cual le indicaremos que pueda ser accedido desde cualquier maquina definida sin restricciones con **hosts: all**, luego le decimos que somos usuario root para obtener todos los permisos para realizar las ejecuciones **become:true**. Definiremos algunas variables entre las cuales se encuentra la Url donde se encuentra almacenado nuestro repositorio de docker :whale:, la version que utilizaremos, definiremos un usuario para asignar al grupo de docker, y algunos comandos de configuración. :electric_plug:
 
 - Luego empezaremos con las tareas de aprovisionamiento de nuestra infraestructura, definiendo los paquetes que debemos usar en nuestro sistema operativo Centos7 **"Ensures yum-utils exists"**, añadiremos el repositorio donde se encuentra alojado docker para poder descargarlo **"Adds docker repo"**, nos aseguraremos que la ultima version de docker :whale: está instalada **"Ensure latest docker version is installed"**, nos aseguraremos que nuestro docker está habilitado para iniciar al arranque de nuestra máquina **"Ensure docker is enabled to start at boot"**, añadiremos el usuario que definimos en las variables **"vagrant"** a nuestro grupo de docker **"Add users to docker group"**, aplicaremos diversas configuraciones para luego proceder a chequear si efectivamente tenemos instalado el docker-compose **"Check if docker compose is already installed"**, para luego proceder a la instalación en caso de que no lo encuentre **"Installs docker compose"**, luego de esto reiniciamos nuestro docker engine y estará operativo para su uso :whale: :sunglasses:
 
Para provisionar nuestro docker engine haciendo uso de ansible verificamos que estemos en la carpeta de nuestro proyecto a traves del comando **"pwd"** y una vez estando dentro, procederemos a inicializar nuestra máquina virtual con el comando **vagrant up** :zap:, luego de inicializar se nos pedirá que definamos una interfaz de puente para una red pública definida en el Vagrantfile (Esto se necesitará para la segunda parte del proyecto, en la cual debemos conectar dos máquinas virtuales por medio de ssh). Al inicializarse la máquina virtual nos aparecerá el mensaje **"Which interface should the network bridge to?"** en el cual indicaremos el número "1" :rocket:


PS: Además de usar **vagrant up** para inicializar la máquina, podemos usar tambien **vagrant halt** para apagarla, **vagrant provision** para provisionar la infraestructura con ansible, **vagrant reload** para realizar un restart de la máquina virtual, entre otros. :neckbeard:

### Desplegar una aplicación dockerizada sobre la Máquina virtual con Ansible y docker-compose

Haremos uso de nuestro Vagrantfile para realizar esta ejecución de manera automática a través del playbook que definimos anteriormente **"app-provision.yml"** el cual lo veremos a continuación :computer:

    ---
    - name: App Provision
      hosts: all
      become: true

      tasks:

        - name: Install the latest version of Git
          yum:
            name: git
            state: latest

        - git: 
            repo: 'https://github.com/Diegonavia/application'
            dest: /app

        - name: Build application
          shell: /usr/local/bin/docker-compose up -d --build
          args:
            chdir: /app

Este playbook es mas corto que el anterior, ya tenemos instalado docker-compose el cual usaremos para desplegar la aplicación junto con ansible :boom:.

Primero, asi como en el playbook anterior, definimos el host y nos otorgamos permisos de usuario root (become: true) para ejecutar las tareas. Instalamos el software de control de versiones GIT ya que será necesario para poder descargar nuestra aplicación de un repositorio remoto, en este caso "Github" y luego procedemos a la creación de nuestra imagen de la aplicación, el cual en este caso es un login, pero puede ser cualquier aplicación que ustedes deseen.

Procedemos a levantar nuestra máquina virtual con nuestro comando **vagrant up** y esperamos que se ejecuten todas las validaciones y tareas... :recycle:

PS: al igual que en el paso anterior, debemos especificar con cual interfaz la conexión hará el puente, por lo que le volvemos a indicar el número "1" :rocket:

Una vez nuestra máquina virtual arriba, ubicamos nuesta Vm en Virtual Box con el nombre que seleccionamos en nuestro Vagrantfile **PFG_VM** la cual debería estar en estado **"running"** :white_check_mark:. Procedemos a loguearnos con el (usuario: vagrant, password: vagrant), realizamos un chequeo de nuestra aplicación creada con el comando **docker images/docker ps**. Verificamos la ip de nuestra Vm con el comando **ip addr**, copiamos esa ip, exponemos el puerto 3000 (XXX.XXX.X.XXX:3000) y obtendremos nuestra aplicación funcionando, en este caso un Login.

- ### Crear otra máquina virtual con Jenkins incorporado :heavy_check_mark:

Para este caso haremos también uso del Vagrantfile pero con una pequeña diferencia que hará que podamos incorporar jenkins en nuestra máquina virtual.

A continuación el código de nuestro Vagrantfile para levantar nuestra Jenkins Vm. :computer: 


    Vagrant.configure(2) do |config|
      config.vm.network "public_network"

      config.vm.box = "centos/7"

      config.vm.network "forwarded_port", guest: 80, host: 80

      config.vm.provider "virtualbox" do |v|
          v.name = "Jenkins_Vm"
          v.memory = 1024
      end

      config.vm.provision "ansible_local", run: "always" do |ansible|
        ansible.playbook = "initial-setup.yml"
      end

      config.vm.provision "shell",
      inline: "ls /jenkins || mkdir /jenkins && chown 1000:1000 /jenkins -R &&
      docker run -d -v /jenkins:/var/jenkins_home --name jenkins -p 8080:8080 jenkins/jenkins"

      end

El Vagrantfile está practicamente igual al anterior, con la diferencia que tiene puertos diferentes para poder ejecutar esta máquina virtual simultaneamente con la anterior y los puertos no entren en conflicto uno con otro. Además tiene la particularidad de tener la configuración inline de un comando shell, el cual verifica si ya existe jenkis en la máquina, y sino es así crea una carpeta llamada jenkins en la cual se va almacenar una imagen docker de jenkins que se ejecutará en el puerto 8080. Para conocer la ip de nuestra 2da Vm, introducimos de igual manera el comando **ip addr** para obtener la dirección ip desde la cual podremos acceder a nuestra aplicación de Jenkins mediante el puerto ya expuesto (XXX.XXX.X.XXX:8080). Básicamente eso es lo que hace el script shell, levantar la imagen de jenkins en nuestra máquina virtual, para que luego se pueda comunicar por ssh hacia nuestra primera máquina. :checkered_flag:

Para este caso también deberemos indicar la interfaz de puente de conexión que queremos para nuestra máquina, colocaremos el número "1" de la misma  forma. 🚀


## Diagrama de flujo 📊

<<<<<<< HEAD

![Diagrama_de_flujo](https://user-images.githubusercontent.com/45079819/66224616-b790e300-e6ac-11e9-8049-8fdf27640ad0.png)
















































=======
![Diagrama_de_flujo](https://user-images.githubusercontent.com/45079819/66224616-b790e300-e6ac-11e9-8049-8fdf27640ad0.png)
>>>>>>> Dev

En el diagrama podemos observar el flujo que sigue nuestro proyecto. A través de un Vagrantfile inicializamos nuestra máquina virtual por medio de **vagrant up** el cual realiza la provision de la infraestructura a través de **Ansible**. Mediante un playbook local logramos instalar **Docker** el cual se encarga de generar una imagen de nuestra aplicación Web para que sea desplegada a través de Ansible en nuestra máquina virtual. Mediante la ejecución de un job en **Jenkins** gatillado por medio de un **git push** a nuestro repositorio, se crea un archivo **Zip** del playbook de ansible el cual es enviado a nuestra máquina virtual a través de SSH para que luego sea ejecutada la playbook de **Ansible** en la máquina remota.


