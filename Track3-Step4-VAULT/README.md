# Step 4 - Ansible Vault

## Ansible Vault

**Ansible Vault** è una funzionalità Ansible, permette di crittografare dati sensibili all'interno del progetto, per fare in modo che siano protetti se il codice viene pubblicato su repository pubblici.

---

## Comandi Essenziali

| Comando | Descrizione |
|---------|-------------|
| `ansible-vault create secrets.yaml` | Per creare un file crittografato |
| `ansible-vault edit secrets.yaml` | Per modificare un file crittografato |
| `ansible-vault encrypt secrets.yaml` | Per crittografa un file |
| `ansible-vault decrypt secrets.yaml` | Per decrittografare un file|
| `ansible-vault view secrets.yaml` | Per visualizzare il contenuto del file crittografato |

---

## Uso delle variabili nei playbook

Per fare in modo di utilizzare dei dati racchiusi in un Vault:

```yaml
vars_files:
  - password.yaml
```
Sarà poi necessario inseirire una password per fare in modo di criptare i dati: 
```
ansible-playbook playbook.yml --ask-vault-pass
```
```
ansible-playbook playbook.yml --vault-password-file .password
```

## Come utilizzarli

Inserire le password in un file, in questo caso ```password.yaml```, 

```yaml
password1: 4Nsi8l3
password2: V4vl7
```

Criptare il file con il comando: 

```
ansible-vault encrypt password.yaml
```

Definire il file di configurazione utenti dove inseriremo le variabili: 

```yaml
---
users:
  user_1:
    state: present 
    groups: sudo
    append: true
    home: /home/user_1 
    shell: /bin/bash
    password: "{{ password1 }}" 
    update_password: always
  user_1:
    state: present 
    groups: sudo
    append: true
    home: /home/user_1
    shell: /bin/bash
    password: "{{ password2 }}" 
    update_password: always
...
```

Per poter fare in modo che si usi un password criptata, verrà utilizzato ```jinja2```.
