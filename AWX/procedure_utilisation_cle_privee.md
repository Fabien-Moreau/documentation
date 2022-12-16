# Solution pour utiliser une clé privée dans AWX

<!-- vscode-markdown-toc -->
* 1. [Objectif de la documentation](#Objectifdeladocumentation)
* 2. [Stockage de la clé dans Vault](#StockagedelacldansVault)
* 3. [Création du playbook](#Crationduplaybook)
* 4. [Création du credential type](#Crationducredentialtype)
* 5. [Récupération du Vault CA CERT](#RcuprationduVaultCACERT)
	* 5.1. [Astuce](#Astuce)
* 6. [Création du job_template](#Crationdujob_template)

<!-- vscode-markdown-toc-config
	numbering=true
	autoSave=true
	/vscode-markdown-toc-config -->
<!-- /vscode-markdown-toc -->
<br><br>

##  1. <a name='Objectifdeladocumentation'></a>Objectif de la documentation
Le but de cette documentation est de décrire le mode opératoire afin de mettre en place l'authentification par clé SSH unique par machine pour AWX. Ces clés seront stockées dans un Hashicorp Vault. Cet Hashicorp Vault sera interconnecté avec un AWX.
<br><br>

##  2. <a name='StockagedelacldansVault'></a>Stockage de la clé dans Vault
1. Sur un serveur, générer une paire de clé
2. Déclarer la clé publique sur le serveur à gérer dans le fichier .ssh/authorized_keys
3. Coller la clé privée dans Vault. Dans notre exemple, dans l'arborescence suivante:
automation_stack1/{{ inventory_hostname }}
dans la clé private_key
<br><br>

##  3. <a name='Crationduplaybook'></a>Création du playbook
Créer le playbook qui va réaliser les actions suivantes:
* Définition de la variable ansible_private_key_file
* Copie de la clé dans un fichier
* Ajout d'une nouvelle ligne à la fin du fichier sinon le format de la clé n'est pas reconnue

```yml
- name: test PPK connection
  hosts: all
  gather_facts: false
  become: false

  vars:
    ansible_private_key_file: "/root/{{ inventory_hostname }}"
  
  tasks:
    - name: Copy file with owner and permissions
      ansible.builtin.copy:
        content: "{{ lookup('hashi_vault', 'secret=automation_stack1/data/{{ inventory_hostname }}:private_key auth_method=approle validate_certs=False', role_id=vault_role_id, secret_id=vault_secret_id) }}"
        dest: "/root/{{ inventory_hostname }}"
        owner: root
        group: root
        mode: '0600'
      delegate_to: localhost

    - name: Add new line
      shell: "echo \n >> /root/{{ inventory_hostname }}"
      delegate_to: localhost

    - name: displaying key
      shell: "cat /root/{{ inventory_hostname }}"
      delegate_to: localhost

    - name: ping
      ping:
```

Il n'est pas nécessaire de supprimer la clé car nous sommes dans un environnement d'exécution qui sera supprimé à la fin de l'exécution.
<br><br>

##  4. <a name='Crationducredentialtype'></a>Création du credential type
Il n'est pas possible de lier un credential de type Hashicorp proposé par AWX. Vous devez donc créer votre propre credential type.

Dans AWX,
1. Aller dans credential types
2. cliquer sur Add
3. Indiquer un nom et dans Input configuration, spécifier les champs suivant:
```yaml
fields:
  - id: vault_server
    type: string
    label: URL for Vault Server
  - id: vault_role_id
    type: string
    label: Vault AppRole ID
  - id: vault_secret_id
    type: string
    label: Vault Secret ID
    secret: true
  - id: vault_cacert
    type: string
    label: Vault CA CERT
    secret: true
required:
  - vault_server
  - vault_role_id
  - vault_secret_id
  - vault_cacert
```

Dans Injector configuration, indiquer les variables suivantes:
```yml
env:
  VAULT_ADDR: '{{ vault_server }}'
  VAULT_CACERT: '{{ vault_cacert }}'
  VAULT_ROLE_ID: '{{ vault_role_id }}'
  VAULT_SECRET_ID: '{{ vault_secret_id }}'
  VAULT_AUTH_METHOD: approle
  VAULT_SKIP_VERIFY: 'true'
extra_vars:
  vault_addr: '{{ vault_server }}'
  vault_cacert: '{{ vault_cacert }}'
  vault_role_id: '{{ vault_role_id }}'
  vault_secret_id: '{{ vault_secret_id }}'
  vault_auth_method: approle
  vault_validate_certs: 'false'
```

4. Créer ensuite un credential qui va se baser sur le credential type que vous venez de créer.
Indiquer donc les champs suivant:
* l'URL de Vault
* l'Approle ID
* le vault secret ID
* le Vault CA CERT
<br><br>

##  5. <a name='RcuprationduVaultCACERT'></a>Récupération du Vault CA CERT
Dans un navigateur,
* aller sur votre Vault
* afficher le certificat et télécharger le PEM (cert)
* afficher le dans votre éditeur de code préféré pour copier son contenu
<br><br>

###  5.1. <a name='Astuce'></a>Astuce
Pour tester le bon fonctionnement, vous pouvez créer  un credential basé sur HashiCorp Vault Secret Lookup fourni par défaut (mais pas attachable à un playbook). Au moment de valider, vous pouvez cliquer sur Test.
<br><br>

##  6. <a name='Crationdujob_template'></a>Création du job_template
Une fois votre playbook poussé dans votre Git, dans AWX, créer un job_template.
Dans le champs Credentials, indiquer le credential Vault que vous venez de créer.
