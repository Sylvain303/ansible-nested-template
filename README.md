# ansible nested template test

Goal generate an SQL GRANT file, without exposing password.
Passwords will be resolved in another step, `apply_grants.yml`

Can be acheived by changing the `variable_start_string` and `variable_end_string` while rendering the final template.
See: `apply_grants.yml`

https://docs.ansible.com/ansible/latest/collections/ansible/builtin/template_lookup.html

require ansible >= 2.8

## design

```
.
├── LICENSE
├── ansible.cfg
├── generate_grants.yml
├── apply_grants.yml
└── templates
    ├── database_scoped_credential.sql.j2
    └── sql_users_grants.sql.j2
```


`templates/database_scoped_credential.sql.j2` is a partial template that will generate some SQL.
it will be rendered in the variable `special_sql_output` by a playbook task


The final GRANT template is `templates/sql_users_grants.sql.j2` 
if `special_sql_output` is define we would like to embed it **verbatim**. 

As it can contains password that will be resolve during deploy process.

## tested

`!unsafe` fact attribute
`| e` `| safe` jinja filter
```yaml
- name:
  copy:
    content: "{{special_sql_output}}"
    dest: /tmp/partial_include.txt
```
`{% include '/tmp/partial_include.txt' %}


## run

```
ansible-playbook  generate_grants.yml 
```
result is stored in: `/tmp/output.sql` that can be commited in a git repos

```
# replace password value
ansible-playbook  apply_grants.yml
```

result is stored in: `/tmp/output_with_password.sql` not to be commited, just play the SQL on the database.

## tested

```
ansible 2.10.8
  config file = /root/ansible-nested-template/ansible.cfg
  configured module search path = ['/root/.ansible/plugins/modules', '/usr/share/ansible/plugins/modules']
  ansible python module location = /root/ansible_azure/lib/python3.6/site-packages/ansible
  executable location = /root/ansible_azure/bin/ansible
  python version = 3.6.10 (default, Jun  9 2020, 18:36:16) [GCC 8.3.0]
```

## output with recursive template 

on version initial with

```yaml
password: "{{ '{{' }}vault_secure.passwords[\"{{username}}\"]{{ '}}' }}"
```

```
~/ansible-nested-template@[main ?]$ ansible-playbook  generate_grants.yml 

[WARNING]: No inventory was parsed, only implicit localhost is available
[WARNING]: provided hosts list is empty, only localhost is available. Note that the implicit localhost does not match 'all'

PLAY [localhost] ****************************************************************************************************************************************************************************************************************************

TASK [play extra template if special_sql is requested] **************************************************************************************************************************************************************************************
ok: [localhost]

TASK [debug] ********************************************************************************************************************************************************************************************************************************
ok: [localhost] => {
    "special_sql_output": "VARIABLE IS NOT DEFINED!"
}

TASK [generate template] ********************************************************************************************************************************************************************************************************************
fatal: [localhost]: FAILED! => {
    "changed": false
}

MSG:

AnsibleUndefinedVariable: -- vim: set ft=sql:
IF NOT EXISTS(
    SELECT * FROM sys.database_scoped_credentials
    WHERE name = 'OurElasticQueryCred'
  )
  BEGIN 
    CREATE DATABASE SCOPED CREDENTIAL [OurElasticQueryCred]
    WITH IDENTITY = 'service_account',
        SECRET = '{{vault_secure.passwords["service_account"]}}';
  END
ELSE
  BEGIN
    -- change password or username
    ALTER DATABASE SCOPED CREDENTIAL [OurElasticQueryCred]
    WITH IDENTITY = 'service_account'  ,
    SECRET = '{{vault_secure.passwords["service_account"]}}';
  END

: 'vault_secure' is undefined
```
