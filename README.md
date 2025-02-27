# Ansible - Intalação/Configuração/Utilização.

O Ansible é uma ferramenta poderosa que permite às equipes de TI automatizar tarefas repetitivas e complexas, como configuração de servidores, implantação de aplicações, cópia de arquivos para diversos servidores e gerenciamento de redes. Ele funciona com varias playbooks que define o estado desejado da infraestrutura e as ações necessárias para atingi-lo.


Por exemplo, uma playbook de atualização de servidores seria esse arquivo yaml abaixo:

$ cat update.yml

```
---
- name: Update and upgrade all servers
 hosts: cluster_servers
 become: yes
 tasks:
  - name: Run apt update
   shell: apt update -y

  - name: Run apt upgrade with custom options
   shell: apt upgrade -o Dpkg::Options::="--force-confold" -y
```


Após essa configuração basta rodar ansible-playbook playbooks/update.yml

Porém é necessário configurar no minimo os hosts com essa playbook eu consigo atualizar quantos servidores eu quiser em apenas um comando. Da para utilizar o ansible de várias formas, porém vou mostrar a maneira que utilizo no servidores do NIC.


## 1 - Instalação Ansible

```
$ sudo apt update && sudo apt upgrade -y
$ sudo apt install -y ansible
```


## 2 - Configuração Ansible

Segue um tree do diretório do ansible para uma fácil compreensão dos arquivos utilizados e necessários para a automação.


```
$ tree ansible/
ansible/
├── ansible.cfg
├── hosts
└── playbooks
    ├── copiar_pasta_copy.yml
    ├── copiar_pasta_rsync.yml
    ├── files # A pasta files é utilizada para copiar arquivos para outros diretórios
    │   ├── ansible.tar
    │   └── scripts 
    │       ├── adduser.sh
    │       ├── limpvar.sh
    │       └── ufw.sh
    ├── mover_arquivo.yml
    └── update.yml


4 directories, 10 files
```


Nesse exemplo, vamos alterar os arquivos **ansible.cfg**, **hosts**, e apenas duas playbooks do diretório **playbooks/**.


```
$ ansible# ls
ansible.cfg  hosts  playbooks/
```


**ansible.cfg** - é onde fica a configuração do ansible, ela é feita para reduzir a linha de comando ao usar o ansible, pois quando ela é configurada não utilizamos comandos extensos.

$ cat ansible.cfg

```

[defaults]
inventory = ./hosts


# Caminho para as playbooks
playbook_dir = ./playbooks

# Diretório onde ficam os arquivos que serão copiados
library = ./files  # Onde os arquivos que você quer transferir ficam (se necessário)

remote_user = marcos
private_key_file = ~/.ssh/id_rsa 
host_key_checking = False

[privilege_escalation]
become = true              # Permite executar comandos como root (privilege escalation)
#become_method = sudo       # Método para elevação de privilégio (sudo por padrão)
become_user = root 
```

**Hosts** - O arquivo hosts no Ansible é utilizado para definir os grupos de hosts e as máquinas nas quais os playbooks serão executados. Ele pode ser estruturado de várias formas para atender a diferentes necessidades.

Da forma que eu coloco abaixo é da forma que utilizamos os hosts a partir da nossa configuração do ssh em ~/.ssh/config

$ cat hosts

```
[cluster_servers]
servidor1
servidor2
servidor3
```


## 3 - Explicando as **Playbooks**

**Playbooks** são arquivos YAML que contêm uma série de instruções (plays) para configurar e gerenciar sistemas. Cada play define um conjunto de tarefas a serem executadas em um grupo de hosts.


Em uma playbook, as tarefas são listadas sequencialmente e executadas em ordem, podendo incluir módulos como apt, yum, copy para gerenciar pacotes, arquivos e serviços em servidores.

Ex, uma playbook configurada para copiar uma pasta para mais de um servidor.

Script1
$ cat playbook/copiar_pasta_rsync.yml

```
# Playbook para copiar um diretório para um destino sem remover arquivos existentes
#
# COMO USAR:
# 1. Especifique o diretório de origem (source_dir) e o destino (destination_dir).
# 2. Execute o comando abaixo:
#    ansible-playbook playbook/mover_pasta_rsync.yml --extra-vars "source_dir=ORIGEM destination_dir=DESTINO"
# EXEMPLOS:
# - Copiar 'files/scripts' para '/opt/scripts' sem remover arquivos existentes:
#   ansible-playbook mover.yml --extra-vars "source_dir=files/scripts destination_dir=/opt/scripts"
#
# - Copiar 'files/configs' para '/etc/app_configs':
#   ansible-playbook mover.yml --extra-vars "source_dir=files/configs destination_dir=/etc/app_configs"


- name: Copiar pasta para o destino sem remover arquivos existentes
  hosts: cluster_servers
  vars:
    source_dir: "{{ source_dir }}"        # Diretório fonte (dentro de 'files/')
    destination_dir: "{{ destination_dir }}"  # Diretório de destino


  tasks:
    - name: Criar diretório de destino se não existir
      ansible.builtin.file:
        path: "{{ destination_dir }}"
        state: directory
        mode: '0755'
      become: yes


    - name: Copiar arquivos mantendo os existentes
      ansible.builtin.synchronize:
        src: "{{ source_dir }}/"  # Origem com barra no final para copiar apenas o conteúdo
        dest: "{{ destination_dir }}/"  # Destino com barra no final para evitar cópia duplicada
        recursive: yes  # Copia recursivamente os subdiretórios e arquivos
        archive: yes  # Mantém permissões, timestamps e links simbólicos
        perms: yes  # Mantém permissões dos arquivos
      become: yes
```

Resumidamente ao executar a playbook com o comando abaixo:


```
$ ansible-playbook playbooks/copiar_pasta_rsync.yml
```
 
O ansible vai executar o comando de mover a pasta no servidores selecionados em sua configuração do hosts, se tiver 30 hosts setados, ele vai executar nos 30 hosts de uma vez.


Outro exemplo, uma playbook simples é a playbook de update dos servidores.


```
$ cat playbooks/update.yml

---
- name: Update and upgrade all servers
 hosts: cluster_servers
 become: yes
 tasks:
  - name: Run apt update
   shell: apt update -y

  - name: Run apt upgrade with custom options
   shell: apt upgrade -o Dpkg::Options::="--force-confold" -y
```

Explicando o comando, ele vai executar um apt update e o próximo vai forçar a atualização de pacotes no Ubuntu/Debian, mantendo as configurações existentes de arquivos alterados durante a instalação.


## 4 - Exemplo de execução das playbooks.

O comando com o --check acima roda a playbook porém sem executar, apenas um check de verificação para ver se tudo vai ocorrer sem erros.
```
$ ansible-playbook playbooks/update.yml --check
```

O comando com o --limit limita a execução para apenas um host, lembrando que para isso o servidor/host tem que estar setado no arquivo de hosts do ansible.

```
$ ansible-playbook playbooks/update.yml --check --limit servidor1
```

Se caso deseja executar em todos os servidores/hosts do arquivo hosts do ansible, é só tirar o comando com o --limit.
```
$ ansible-playbook playbooks/update.yml
```


Segue abaixo o resultado do comando 

```
$ ansible-playbook playbooks/update.yml --check --limit servidor1

PLAY [Update and upgrade all servers] **************************************************************************************************************************************************

TASK [Gathering Facts] *****************************************************************************************************************************************************************
ok: [servidor1]

TASK [Run apt update] ******************************************************************************************************************************************************************
skipping: [servidor1]

TASK [Run apt upgrade with custom options] *********************************************************************************************************************************************
skipping: [servidor1]

PLAY RECAP *****************************************************************************************************************************************************************************
servidor1           : ok=1  changed=0  unreachable=0  failed=0  skipped=2  rescued=0  ignored=0  
```

Agora segue abaixo um comando sem o check


```
ansible-playbook playbooks/update.yml --limit servidor1

PLAY [Update and upgrade all servers] **************************************************************************************************************************************************

TASK [Gathering Facts] *****************************************************************************************************************************************************************
ok: [servidor1]

TASK [Run apt update] ******************************************************************************************************************************************************************
changed: [servidor1]

TASK [Run apt upgrade with custom options] *********************************************************************************************************************************************
changed: [servidor1]

PLAY RECAP *****************************************************************************************************************************************************************************
servidor1           : ok=3  changed=2  unreachable=0  failed=0  skipped=0  rescued=0  ignored=0 
```




### Lembrando que isso só foi possivel porque foi feita a configuração no ansible.cfg, hosts e no diretório da playbook que é onde a magica acontece, porém tem que estar configurado dessa forma para executar apenas a playbook.

