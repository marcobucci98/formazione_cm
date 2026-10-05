# Step 2 - OS

Creare dei playbooks, con l'utilizzo di Ansible che buildino due container con OS diversi, utilizzando il playbook `playbooks/build-containers.yaml` e `Dockerfiles-OS/ubuntu-os/Dockerfile.ubuntu22` e `Dockerfiles-OS/rocky-os/Dockerfile.rocky9`. 

## Dockerfiles

I sistemi operativi utlizzati sono `ubuntu:22` e `rocky:9` viene creato l'utente, con privilegi di amministratore e senza la richiesta password:

```Dockerfile
RUN useradd -m -s /bin/bash ansible && \
    echo "ansible ALL=(ALL) NOPASSWD:ALL" > /etc/sudoers.d/ansible && \
    chmod 0440 /etc/sudoers.d/ansible
```

Creazione utente, creazione home e set di bash:

`useradd -m -s /bin/bash ansible`

Root senza utilizzo password:

`NOPASSWD:ALL in /etc/sudoers.d/ansible`

File in sola lettura per il proprietario e per il gruppo root:

`chmod 0440`

Chiave SSH per autenticazione:

`SSH_PUBLIC_KEY`

```Dockerfile
ARG SSH_PUBLIC_KEY
RUN mkdir -p /home/ansible/.ssh && chmod 700 /home/ansible/.ssh && \
    echo "$SSH_PUBLIC_KEY" > /home/ansible/.ssh/authorized_keys && \
    chmod 600 /home/ansible/.ssh/authorized_keys && \
    chown -R ansible:ansible /home/ansible/.ssh
```

Permessi per `.ssh`:

`chmod 700`

Permessi per `authorized_keys`:

`chmod 600` 

Proprietà per l'utente: 

`chown -R ansible:ansible`

Disabilitato login per root, accesso per il solo utente creato:

```Dockerfile
RUN mkdir /var/run/sshd && \
    sed -i 's/#PermitRootLogin yes/PermitRootLogin no/' /etc/ssh/sshd_config && \
    sed -i 's/#PasswordAuthentication yes/PasswordAuthentication no/' /etc/ssh/sshd_config && \
    sed -i 's/PasswordAuthentication yes/PasswordAuthentication no/' /etc/ssh/sshd_config && \
    echo 'AllowUsers ansible' >> /etc/ssh/sshd_config
```

Disattiva l'accesso per root: 

`PermitRootLogin no` 

Disattiva autenticazione tramite password: 

`PasswordAuthentication no`

Avvio server SSH: 

```Dockerfile
EXPOSE 22
CMD ["/usr/sbin/sshd", "-D"]
```

## File build-containers.yaml

Viene creata la cartella che conterrà le chiavi SSH, vengono poi generate le chiavi, e poi vengono salvate:

```yaml
- name: Dockerfiles-OS
  become: true
  ansible.builtin.file:
    path: "{{ playbook_dir }}/Dockerfiles-OS"
    state: directory
    mode: '0755'

# chiave 
- name: SSH Keys
  become: true
  community.crypto.openssh_keypair:
    path: "{{ playbook_dir }}/Dockerfiles-OS/id_rsa"
    type: rsa
    size: 4096
    state: present
  register: ssh_key_result

- name: Read Key
  become: true
  ansible.builtin.slurp:
    src: "{{ playbook_dir }}/Dockerfiles-OS/id_rsa.pub"
  register: pub_key_encoded

- name: Set Key
  ansible.builtin.set_fact:
    pub_key_content: "{{ pub_key_encoded.content | b64decode }}"

```
Viene successivamente effettuato il build e lo start:

```yaml
    # Build immagine Ubuntu

    - name: Build Ubuntu Image
      community.docker.docker_image:
        name: "{{ image_ubuntu }}:{{ tag | default('latest') }}"
        build:
          path: "{{ playbook_dir }}/Dockerfiles-OS/ubuntu-OS"
          dockerfile: "{{ dockerfile_ubuntu }}"
          args:
            SSH_PUBLIC_KEY: "{{ pub_key }}"
          pull: true
        source: build

    # avvia container Ubuntu

    - name: Start Ubuntu Container
      community.docker.docker_container:
        name: "{{ container_ubuntu }}"
        image: "{{ image_ubuntu }}:{{ tag | default('latest') }}"
        state: started
        restart_policy: always
        ports:
          - "{{ port_ubuntu }}"

    # build immagine Rocky

    - name: Build Rocky Image
      community.docker.docker_image:
        name: "{{ image_rocky }}:{{ tag | default('latest') }}"
        build:
          path: "{{ playbook_dir }}/Dockerfiles-OS/rocky-OS"
          dockerfile: "{{ dockerfile_rocky }}"
          args:
            SSH_PUBLIC_KEY: "{{ pub_key }}"
          pull: true
        source: build

    # avvia il container rocky

    - name: Start Rocky Container
      community.docker.docker_container:
        name: "{{ container_rocky }}"
        image: "{{ image_rocky }}:{{ tag | default('latest') }}"
        state: started
        restart_policy: always
        ports:
          - "{{ port_rocky }}"
```