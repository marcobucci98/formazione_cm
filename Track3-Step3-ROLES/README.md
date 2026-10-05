# Step 3 - ROLES

Per questo step vengon creati i Role tramite `ansible-galaxy role init`. 

## ROLE

**tasks/main.yaml**

Viene verificato cosa è installato sul sistema, se Podman o Docker

```yaml
- name: Select Engine
  ansible.builtin.set_fact:
    my_engine: >-
      {{
        'podman' if (engine_auto == 'auto' and podman_check.rc | default(1) == 0)
        else (engine_auto if engine_auto != 'auto' else 'docker')
      }}
```

Viene selezionato Docker se Podman non è presente:

```yaml
- name: Task for {{ my_engine }}
  ansible.builtin.include_tasks: "{{ my_engine }}.yaml"
```

**docker.yaml**

Build con Docker: 

```yaml
- name: Build with Docker
  community.docker.docker_image:
    name: "{{ container_imm }}"
    build:
      path: "{{ playbook_dir }}/Dockerfiles-OS/ubuntu-OS"          
      dockerfile: "{{ dockerfile | default('Dockerfile') }}"
      pull: true  
    source: build
    state: present
```

**podman.yaml**

Build con Podman.

```yaml
- name: Build with Podman
  containers.podman.podman_image:
    name: "{{ container_imm }}"
    path: "./Dockerfiles-OS"         
    dockerfile: "{{ dockerfile | default('Dockerfile') }}" 
    build:
      cache: true
      extra_args: "--no-cache"     
    state: build
```