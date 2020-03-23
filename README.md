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

- ### Desplegar una aplicación dockerizada sobre la Máquina virtual con Ansible y docker-compose. ✔️

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

- ### Pasos para ejecutar ansible-playbook de manera remota mediante ssh :heavy_check_mark:

1. Lo primero que debemos hacer es dirigirnos a la primera máquina virtual **PFG_VM** y colocar en consola **ip addr** para conocer la ip con la cual está conectandose la aplicación. Copiamos esa ip con la cual nos conectaremos via ssh a través de la segunda máquina virtual **Jenkins_Vm**. Nos dirigimos a nuestra máquina virtual de jenkins, nos logueamos con el usuario **vagrant**, y utilizamos el comando **vi hosts**  para crear un archivo de **hosts** el cual nos ayudará en nuestra conexión. 

A continuación el código del archivo Hosts :computer:

    [all:vars]

    ansible_connection = ssh

    [test]

    apptest ansible_host=192.168.1.125 ansible_user=vagrant ansible_private_key_file=/var/jenkins_home/ansible/key
    
    
Primero declaramos todas las variables de ansible, luego determinamos de que manera ansible se va a conectar a nuestra maquina remota **(ansible_connection = ssh)** el cual es por medio de **SSH**. Definimos un grupo llamado **test** en el cual definiremos nuestro host de ansible. Le colocaremos como nombre a nuestro host **apptest** el cual usaremos luego en nuestro ansible-playbook para establecer la conexión. Definimos el host al cual nos vamos a conectar con **ansible_host=192.168.1.125** (En este caso coloque esta ip a modo de ejemplo la cual estaba operativa con la aplicación corriendo al momento de realizar la prueba). Luego definimos el usuario con el cual nos vamos a conectar **ansible_user=vagrant**, en este caso sería vagrant. Luego definiremos donde está la key que usaremos para conectarnos a nuestra Vm remota, **ansible_private_key_file=/var/jenkins_home/ansible/key**, en este caso la private_key la podemos encontrar en nuestro archivo .vagrant de nuestra máquina virtual a la cual queremos conectarnos. (/.vagrant/machines/default/virtualbox/private_key). Nótese que la ruta a la cual hacemos referencia esta key se encuentra en nuestra segunda máquina virtual de jenkins dentro de una carpeta llamada ansible. Esta carpeta debemos crearla dentro de nuestro jenkins_home y ubicar el archivo de nuestra private_key **"key"**.

Luego de esto nos dirigimos a crear nuestro archivo **vm-provision.yml**, el cual será el playbook de ansible que ejecutaremos en nuestra máquina virtual remota mediante ssh. 🚀

A continuación el código de nuestro playbook de ansible. :computer:

      ---
      - name: App Provision
        hosts: apptest
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


 Como podemos observar es el mismo código usado en nuestro playbook **"app-provision.yml"**, con la pequeña diferencia de definicón de hosts **(hosts: apptest)** el cual ya no será **all** sino que colocaremos el ya definido en nuestro archivo de **hosts** con el nombre de **apptest**.
 
Luego de la creación de estos 3 archivos, nos dirigimos a entrar en nuestro contenedor de jenkins con privilegios de usuario root. Necesitamos privilegios de usuario root para instalar ansible en nuestro contenedor de jenkins y poder ejecutar nuestro playbook. Para realizar esto, ejecutamos el comando **docker exec -u 0 -it jenkins bash** Procedemos a instalar el administrador de paquetes **pip** el cual nos ayudará con nuestra instalación de ansible. Ejecutamos el comando **(curl "https://bootstrap.pypa.io/get-pip.py" -o "get-pip.py" && python get-pip.py)** el cual descargará e instalara pip en nuestro contenedor. Una vez instalado pip, procederemos a instalar ansible mediante el siguiente comando **pip install -U ansible**. Una vez instalado ansible en nuestro contenedor, hacemos **logout** para salir del usuario root y entrar con nuestro usuario jenkins, para esto ejecutamos el siguiente comando **docker exec -ti jenkins bash**.

Una vez dentro de nuestro contenedor jenkins, nos dirigimos a jenkins_home con el comando **cd /var/jenkins_home** en el cual crearemos una carpeta llamado **ansible** con el comando **mkdir**. Esta carpeta de ansible tendrá nuestros 3 archivos definidos anteriormente (hosts, key, vm-provision.yml). Para mover estos archivos anteriormente creados, los cuales están ubicados en la ruta **/home/vagrant**, debemos hacer **exit** de nuestro usuario jenkins para regresar al usuario vagrant. Una vez allí ejecutamos el comando **docker cp hosts jenkins:/var/jenkins_home/ansible**, siendo hosts el nombre del archivo y jenkins: el nombre de nuestro contenedor al cual queremos copiar nuestro archivos conjuntamente con la ruta especifica a la cual queremos copiarlos **(/var/jenkins_home/ansible)**.

Una vez ubicados nuestros archivos en la ruta correcta, procederemos a darle permisos a nuestro archivo **key** para no tener problemas al momento de validar la private_key para nuestra conexión remota. Para hacer esto ejecutamos el comando **chmod 400**. Luego de modificar los permisos a nuestro archivo **key**, procedemos a probar nuestra conexión remota por ssh desde nuestra máquina virtual actual **(Jenkins_Vm)** hacia nuestra primera máquina virtual **(PFG_VM)** con el siguiente comando **ansible -m ping -i hosts apptest**, mediante nuestro archivo **hosts** con **apptest**.

El resultado esperado debería ser algo como esto... 

![Captura](https://user-images.githubusercontent.com/45079819/66453152-bd365200-ea39-11e9-871e-7338e93a3110.PNG)


Una vez configurados nuestros archivos, procederemos a configurar nuestra aplicación de Jenkins. Para esto saldremos de nuestro usuario jenkins con el comando **exit** y una vez situados en nuestro usuario vagrant ejecutamos el comando **ip addr** para conocer la ip con la cual nos conectaremos a nuestra aplicación de jenkins a través del puerto 8080. 

![Sin título](https://user-images.githubusercontent.com/45079819/66453650-75b0c580-ea3b-11e9-80de-875fd38585a1.png)
      

En este caso para conectarnos a nuestra aplicación de Jenkins utilizaremos la siguiente dirección **http://192.168.1.124:8080**

Al ejecutar Jenkins por primera vez, debemos configurarlo e instalarlo. 

PS: "Se usarán imagenes de referencia de la instalación en Windows para que tengan una idea del proceso de instalación y los pasos que estos conllevan".

1. Se debe colocar la clave de administrador generada por defecto para asegurar que jenkins está siendo instalado por un administrador, la contraseña la obtienen siguiendo la ruta que les señala el proceso de instalación. 

![1](https://user-images.githubusercontent.com/45079819/66453848-261ec980-ea3c-11e9-904a-4d7b7aa391ef.PNG)


2. Accedemos a dicha ubicación, copiamos la clave indicada y la pegamos en la cajita donde pone “Administrator password”. Pulsamos sobre “Continue”. Ahora nos aparece la pantalla de bienvenida de Jenkins.

![2](https://user-images.githubusercontent.com/45079819/66454093-063bd580-ea3d-11e9-84d0-aed70fd81897.PNG)


3. Vamos instalar los plugins de Jenkins más populares. Para ello, pulsamos sobre la caja de la izquierda y esperamos que los plugins se vayan instalando.

![3](https://user-images.githubusercontent.com/45079819/66454143-400cdc00-ea3d-11e9-86bc-7eb5dbe72355.PNG)


4. Una vez instalados, nos aparece la ventana de creación del usuario administrador. Rellenamos los datos y pulsamos sobre “Save and Finish”.

![4](https://user-images.githubusercontent.com/45079819/66454230-9ed25580-ea3d-11e9-965d-555d40e8e54f.PNG)


5. Una vez realicemos estos pasos, ya tendríamos Jenkins instalado y listo para empezar a configurar nuestro Job. 

![5](https://user-images.githubusercontent.com/45079819/66454323-ea84ff00-ea3d-11e9-83c6-19fc6240e638.PNG)


- Una vez en la aplicación de Jenkins, nos dirijimos a la pestaña **Manage Jenkins** ubicada al medio a la izquierda de la pantalla, una vez dentro seleccionamos la pestaña **Manage Plugins**, una vez dentro nos ubicamos en la pestaña **Available** y filtramos a la derecha con la palabra **Ansible**. Aparecerá un plugin llamado **Ansible Plugin**, el cual necesitaremos para invocar nuestro playbook. Una vez seleccionado presionamos **"Install without restart"** y luego seleccionamos la casilla **"Restart Jenkins when installation is complete and no jobs are running"**. Esperamos que se reinicie la aplicación y ahora estamos listos para crear nuestro proyecto. :fire:


- Una vez dentro de Jenkins, nos dirigimos a la pestaña superior izquierda denomidada **"New Item"** en el cual definiremos nuestro proyecto. En este caso le colocaremos DevOps_Test y seleccionamos la opción **Freestyle project** y presionamos OK.


![6](https://user-images.githubusercontent.com/45079819/66454621-ba8a2b80-ea3e-11e9-9c92-bddad22ad658.PNG)


- Una vez dentro de la configuración de nuestro Job, nos dirigimos hacia abajo hasta la pestaña **Build**, ahí seleccionamos **Add build step** y seleccionamos la opción **Invoke Ansible Playbook**

![7](https://user-images.githubusercontent.com/45079819/66455427-22417600-ea41-11e9-9138-f8880e719501.PNG)


- Una vez dentro de la opción de **Invoke Ansible Playbook**, procederemos a colocar nuestro Playbook Path, el cual será nuestro archivo  **vm-provision** ubicado en (/var/jenkins_home/ansible) el cual se ejecutará en nuestra primera máquina virtual en dónde está nuestra aplicación dockerizada. En la opción de **Inventory** seleccionamos **"file or host list"** y seleccionamos nuestro archivo **"hosts"** ubicado en la ruta (/var/jenkins_home/ansible).

![8](https://user-images.githubusercontent.com/45079819/66455711-e6f37700-ea41-11e9-8bef-2b3660d7ba8a.png)


- Seleccionamos la opción de **SAVE** y procederemos a ejecutar nuestro job con la opción de **Build Now** ubicado en la parte izquierda de la pantalla. Una vez seleccionado el build, comenzará a ejecutarse nuestro playbook de ansible en nuestra máquina virtual remota. Luego se empezará a ejecutar nuestro Job, el cual lo veremos bajo la opción **Build History** con un número por cada iteración que realicemos del job. 

![9](https://user-images.githubusercontent.com/45079819/66456238-777e8700-ea43-11e9-8c9e-8f4000495bec.png)


- Luego apretamos el número de iteración de nuestro Build, en este caso para el ejemplo fué el número **4**. Vemos la opción **Console Output** en el cual observamos todos los pasos realizados por nuestro job. Se ejecutó de forma satisfactoria la provisión de nuestra aplicación en la máquina virtual remota **PFG_VM**.

![10](https://user-images.githubusercontent.com/45079819/66456442-068b9f00-ea44-11e9-9ebd-d600885dd6d6.png)


NOTA: Para corroborar el correcto funcionamiento de la ejecución del playbook en nuestra máquina remota **PFG_VM**, nos dirigiremos a esta y removeremos la imagen de la aplicación para detenerla y así poder provisionarla nuevamente de manera remota desde Jenkins. :fire:


- Para esto, nos dirigimos hacia la máquina virtual **PFG_VM** y escribiremos el comando **docker container ps** para asegurarnos que la aplicación está arriba. Nos aseguramos de que el status de la misma sea "UP". Luego de esto procederemos a eliminarla y corroborar que la aplicación dejó de funcionar. Esto lo corroboramos yendo al navegador con la dirección ip de la Vm y el puerto expuesto para la aplicación, en este caso el puerto 3000. **http://192.168.1.125:3000**. 

![11](https://user-images.githubusercontent.com/45079819/66457000-a138ad80-ea45-11e9-8719-a182b6e8e0f3.png)




![12](https://user-images.githubusercontent.com/45079819/66457147-042a4480-ea46-11e9-8377-e2f0e5c402ea.png)




- Luego de corroborar que nuestra aplicación está arriba procederemos a detenerla, para esto eliminamos la imagen de nuestra aplicación que está corriendo con el siguiente comando **docker rm -f app**. Luego nos dirigimos a nuestro navegador para comprobar que nuestra app ya no está funcionando. 

![13](https://user-images.githubusercontent.com/45079819/66457397-b95cfc80-ea46-11e9-8069-c8cd5cbc330d.png)




![14](https://user-images.githubusercontent.com/45079819/66457552-093bc380-ea47-11e9-8336-ee65dfce72ab.png)


- Ahora que hemos corroborado que nuestra aplicación está abajo, procederemos a ejecutar nuestro playbook de ansible de manera remota mediante ssh desde **Jenkins_Vm** hacia **PFG_VM**. 


![16](https://user-images.githubusercontent.com/45079819/66457855-b0205f80-ea47-11e9-9bea-6cc2659169ed.png)




![15](https://user-images.githubusercontent.com/45079819/66457945-ec53c000-ea47-11e9-8c03-f23ce8e33fdf.png)




- De esta manera queda demostrado que nuestro proyecto en Jenkins funciona, y que de manera remota mediante SSH realizó el aprovisionamiento y levantamiento de nuestá aplicación :fire:

PS: La fecha y hora se colocaron a modo ilustrativo para demostrar que el proceso se realizó uno trás otro comprobando el correcto funcionamiento de nuestro job. ⚙️




## Diagrama de flujo 📊

![Diagrama_de_flujo](https://user-images.githubusercontent.com/45079819/66224616-b790e300-e6ac-11e9-8049-8fdf27640ad0.png)


En el diagrama podemos observar el flujo que sigue nuestro proyecto. A través de un Vagrantfile inicializamos nuestra máquina virtual por medio de **vagrant up** el cual realiza la provision de la infraestructura a través de **Ansible**. Mediante un playbook local logramos instalar **Docker** el cual se encarga de generar una imagen de nuestra aplicación Web para que sea desplegada a través de Ansible en nuestra máquina virtual. Mediante la ejecución de un job en **Jenkins** gatillado por medio de un **git push** a nuestro repositorio, se crea un archivo **Zip** del playbook de ansible el cual es enviado a nuestra máquina virtual a través de SSH para que luego sea ejecutada la playbook de **Ansible** en la máquina remota.


