# Step 1 - DOCKER-REGISTRY

Lo step prevede la creazione e configurazione di un registry tramite playbook. 

## container-playbook.yaml 

Vengono creati i vari role, la variabili utilizzate sono dichiarate nel file `group_vars/all.yaml`. 

```yaml
roles:
    - install_docker
    - docker_daemon
    - docker_registry
    - pull_and_push
```
## ROLE

Il primo role `install_docker` si occuperà di:

- Effettuare la pulizia del sistema
- Aggiungere la repo per Docker
- Installare Docker e dipendenze 
- Avvio di Docker Socket
- Inserimento utente nel gruppo Docker

```yaml
- name: Remove Packets 
  ansible.builtin.dnf:
    name: "{{ rm_pack }}"
    state: absent
  when: rm_pack | length > 0

- name: Enable EPEL Repo
  ansible.builtin.dnf:
    name: epel-release
    state: present
  become: true

- name: Docker Repo 
  ansible.builtin.yum_repository:
    name: docker-ce-stable
    description: Docker CE Stable - $basearch
    baseurl: "{{ docker_repo }}"
    enabled: true
    gpgcheck: true
    gpgkey: "{{ docker_key }}"
    state: present

- name: Docker
  ansible.builtin.dnf:
    name: "{{ packages }}"
    state: present
    update_cache: true
  notify: Start Docker

- name: Install SDK Python per Docker
  ansible.builtin.dnf:
    name: python3-docker
    state: present

- name:  Docker Service
  ansible.builtin.systemd_service:
    name: docker
    enabled: true
    state: started

- name: Wait for Docker socket
  ansible.builtin.wait_for:
    path: /var/run/docker.sock
    state: present
    timeout: 30

- name: Docker Group 
  ansible.builtin.user:
    name: "{{ docker_user }}"
    groups: docker
    append: true
```

Tramite Handler `roles/install_docker/handler/main.yaml`, viene avviato il servizio Docker: 

```yaml
- name: Start Docker
  ansible.builtin.systemd_service:
    name: docker
    enabled: true
    state: restarted
```

Il secondo Role **docker_daemon**, si occupa :

- configurare il demone Docker per supportare il registro privato non sicuro 
- check che il servizio sia ripartito e operativo.

```yaml
- name: Ensure /etc/docker directory exists
  ansible.builtin.file:
    path: /etc/docker
    state: directory
    owner: root
    group: root
    mode: '0755'

- name: Configure Docker daemon with insecure registry
  ansible.builtin.copy:
    dest: /etc/docker/daemon.json
    owner: root
    group: root
    mode: '0644'
    content: |
      {
        "insecure-registries": ["{{ registry_host }}:{{ registry_port }}"]
      }
  notify: Restart Docker

- name: Flush handlers
  ansible.builtin.meta: flush_handlers

- name: Wait for Docker socket
  ansible.builtin.wait_for:
    path: /var/run/docker.sock
    timeout: 30
```

Il terzo Role **docker_registry** si occupa di: 

- Creare la cartella per la persistenza dei dati
- Avvio del container
- Verifica di raggiungibilità 

```yaml
- name: Volume
  ansible.builtin.file:
    path: "{{ storage_dir }}"
    state: directory
    mode: '0755'
  
- name: Start Docker Container
  community.docker.docker_container:
    name: "{{ container_name }}"
    image: registry:2
    state: started
    restart_policy: always
    published_ports:
      - "{{ registry_port }}:5000"
    env:
      REGISTRY_STORAGE_FILESYSTEM_ROOTDIRECTORY: /var/lib/registry # salvataggio dati container
    volumes:
      - "{{ storage_dir }}:/var/lib/registry:z"
    healthcheck:
      test: ["CMD", "wget", "--spider", "-q", "http://localhost:{{ registry_port }}/v2/"]
      interval: 10s
      timeout: 5s
      retries: 3
      start_period: 10s

- name: Wait for Registry
  ansible.builtin.uri:
    url: "http://localhost:{{ registry_port }}/v2/"
    method: GET
    status_code: 200
  register: registry_health
  until: registry_health.status == 200
  retries: 10
  delay: 3
```

Quarto ed ultimo Role, **pull_and_push**, si occupa di fare un test per il corretto funzionamento del registry:

- Scarica un'immagine pubblica
- Applicazione del tag per il registry privato
- Push dell'immagine 
- Ottiene l'elenco dei tag
- Confronto del tag inviato con quello restituito
- Rimozione delle immagini locali
- Pull dal registry 

```yaml
- name: Pull hello-world from Docker Hub
  community.docker.docker_image:
    name: "{{ registry_test_local_image }}" 
    source: pull
    force_source: true

# tag immagine
- name: Tag image for registry
  community.docker.docker_image:
    name: "{{ registry_test_local_image }}"
    repository: "{{ registry_test_remote_image }}" 
    source: local
    force_tag: true

# push su registry 
- name: Push image to registry
  community.docker.docker_image:
    name: "{{ registry_test_remote_image }}"
    source: local
    push: true

- name: Query registry tags
  ansible.builtin.uri:
    url: "http://{{ registry_host }}:{{ registry_port }}/v2/{{ registry_test_repo_name }}/tags/list"
    method: GET
    status_code: 200
    return_content: true
  register: registry_tags
  retries: 3
  delay: 2
  until: registry_tags.status == 200

- name: Check
  ansible.builtin.assert:
    that:
      - registry_test_tag in (registry_tags.json | default({})).tags | default([])
    fail_msg: "Tag '{{ registry_test_tag }}' per l'immagine '{{ registry_test_repo_name }}' non trovato!"
    success_msg: "L'immagine '{{ registry_test_repo_name }}:{{ registry_test_tag }}' è salvata."

- name: Remove Local Images
  community.docker.docker_image:
    name: "{{ item }}"
    state: absent
    force_absent: true
  loop:
    - "{{ registry_test_local_image }}"
    - "{{ registry_test_remote_image }}"

- name: Pull Image from local registry
  community.docker.docker_image:
    name: "{{ registry_test_remote_image }}"
    source: pull
    force_source: true
```