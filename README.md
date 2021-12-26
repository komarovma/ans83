<p class="has-line-data" data-line-start="0" data-line-end="3">Работаю на Windows 10 PRO<br>
Создал в <a href="http://cloud.yandex.ru">cloud.yandex.ru</a> 3 виртуальных машины CentOS 7.0<br>
Ansible уставлен в WSL 2.0 (Ubuntu)</p>
<p class="has-line-data" data-line-start="4" data-line-end="5">Изменил файл hosts.yml</p>
<pre><code>---
all:
  hosts:
    el-instance:
      ansible_host: 62.84.127.122
    kb-instance:
      ansible_host: 62.84.127.54
    fb-instance:
      ansible_host: 51.250.1.224
  vars:
    ansible_connection: ssh
    ansible_user: mike
elasticsearch:
  hosts:
    el-instance:
kibana:
  hosts:
    kb-instance:
filebeat:
  hosts:
    fb-instance:  
</code></pre>
<p class="has-line-data" data-line-start="28" data-line-end="29">Добавил фаилы переменных playbook/inventory/prod/group_vars/</p>
<pre><code>kibana.yml
 ---
kbn_stack_version: &quot;7.14.0&quot;
filebeat.yml 
---
flb_stack_version: &quot;7.14.0&quot;
</code></pre>
<p class="has-line-data" data-line-start="37" data-line-end="38">Добавил файл в Temlpate kibana.yml.j2</p>
<pre><code>path.data: /var/lib/kibana
path.logs: /var/log/kibana
network.host: 0.0.0.0
discovery.seed_hosts: [&quot;{{ ansible_facts['default_ipv4']['address'] }}&quot;]
node.name: node-a
cluster.initial_master_nodes: 
 - node-a
</code></pre>
<p class="has-line-data" data-line-start="47" data-line-end="48">Добавил файл в Temlpate filebeat.yml.j2</p>
<pre><code>path.data: /var/lib/filebeat
path.logs: /var/log/filebaet
network.host: 0.0.0.0
discovery.seed_hosts: [&quot;{{ ansible_facts['default_ipv4']['address'] }}&quot;]
node.name: node-a
cluster.initial_master_nodes: 
   - node-a
</code></pre>
<p class="has-line-data" data-line-start="57" data-line-end="58">Добавил 2 блока в site.yml</p>
<pre><code>- name: Install Kibana
  hosts: kibana
  handlers:
    - name: restart kibana
      become: true
      service:
        name: kibana
        state: restarted
  tasks:
    - name: &quot;Download Kibana's rpm&quot;
      get_url:
        url: &quot;https://artifacts.elastic.co/downloads/kibana/kibana-{{ kbn_stack_version }}-x86_64.rpm&quot;
        dest: &quot;/tmp/kibana-{{ kbn_stack_version }}-x86_64.rpm&quot;
      register: download_kibana
      until: download_kibana is succeeded
    - name: Install Kibana
      become: true
      yum:
        name: &quot;/tmp/kibana-{{ kbn_stack_version }}-x86_64.rpm&quot;
        state: present
    - name: Configure Kibana
      become: true
      template:
        src: kibana.yml.j2
        dest: /etc/kibana/kibana.yml
      notify: restart kibana
- name: Install filebeat
  hosts: filebeat
  handlers:
    - name: restart filebeat
      become: true
      service:
        name: filebeat
        state: restarted
  tasks:
    - name: &quot;Download Filebeat's rpm&quot;
      get_url:
        url: &quot;https://artifacts.elastic.co/downloads/beats/filebeat/filebeat-{{ flb_stack_version }}-x86_64.rpm&quot;
        dest: &quot;/tmp/filebeat-{{ flb_stack_version }}-x86_64.rpm&quot;
      register: download_filebeat
      until: download_filebeat is succeeded
    - name: Install filebeat
      become: true
      yum:
        name: &quot;/tmp/filebeat-{{ flb_stack_version }}-x86_64.rpm&quot;
        state: present
    - name: Configure Filebeat
      become: true
      template:
        src: filebeat.yml.j2
        dest: /etc/filebeat/filebeat.yml
      notify: restart filebeat
</code></pre>
<p class="has-line-data" data-line-start="112" data-line-end="113">Попробуйте запустить playbook на этом окружении с флагом --check.</p>
<pre><code>mike@HOMEDX79SR:~/ans83/playbook$ ansible-playbook -i inventory/prod/ site.yml --check
PLAY [Install Elasticsearch] ***************
TASK [Gathering Facts] **************************
ok: [el-instance]
TASK [Download Elasticsearch's rpm] *************************************
ok: [el-instance]
TASK [Install Elasticsearch] ***********************************
ok: [el-instance]
TASK [Configure Elasticsearch] ************************************
ok: [el-instance]
PLAY [Install Kibana] **************************************
TASK [Gathering Facts] *************************************
ok: [kb-instance]
TASK [Download Kibana's rpm] **************************************
ok: [kb-instance]
TASK [Install Kibana] **************************************
ok: [kb-instance]
TASK [Configure Kibana] ***************************************
ok: [kb-instance]
PLAY [Install filebeat] ***************************************
TASK [Gathering Facts] ****************************************
ok: [fb-instance]
TASK [Download Filebeat's rpm] ****************************************
ok: [fb-instance]
TASK [Install filebeat] *****************************************
ok: [fb-instance]
TASK [Configure Filebeat] *****************************************
ok: [fb-instance]
PLAY RECAP ******************************************
el-instance                : ok=4    changed=0    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0
fb-instance                : ok=4    changed=0    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0
kb-instance                : ok=4    changed=0    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0
</code></pre>
<p class="has-line-data" data-line-start="147" data-line-end="148">Запустите playbook на prod.yml окружении с флагом --diff. Убедитесь, что изменения на системе произведены.</p>
<pre><code>mike@HOMEDX79SR:~/ans83/playbook$ ansible-playbook -i inventory/prod/ site.yml --diff
PLAY [Install Elasticsearch] ******************************************
TASK [Gathering Facts] ********************************************
ok: [el-instance]
TASK [Download Elasticsearch's rpm] ***********************************************
ok: [el-instance]
TASK [Install Elasticsearch] **********************************************
ok: [el-instance]
TASK [Configure Elasticsearch] *************************************************
ok: [el-instance]
PLAY [Install Kibana] *******************************************************
TASK [Gathering Facts] *******************************************************
ok: [kb-instance]
TASK [Download Kibana's rpm] *******************************************************
ok: [kb-instance]
TASK [Install Kibana] ******************************************************
ok: [kb-instance]
TASK [Configure Kibana] ******************************************************
ok: [kb-instance]
PLAY [Install filebeat] *****************************************************
TASK [Gathering Facts] ******************************************************
ok: [fb-instance]
TASK [Download Filebeat's rpm] ****************************************************
ok: [fb-instance]
TASK [Install filebeat] ******************************************************
ok: [fb-instance]
TASK [Configure Filebeat] *****************************************************
ok: [fb-instance]
PLAY RECAP **************************************************
el-instance                : ok=4    changed=0    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0
fb-instance                : ok=4    changed=0    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0
kb-instance                : ok=4    changed=0    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0
</code></pre>
