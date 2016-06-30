# Higher level customers pillar for saltstack

This customers formula for salt is designed to produce derivated pillar configuration files. See [diagram](https://github.com/opensource-expert/customers-formula/blob/master/doc/pillar_diagram.pdf)

It will generate some more pillar files on the saltmaster.

It is a **Proof of Concept** and could porbably be achieved with `ext_pillar`. The main advantages are:

* the resulting pillar can be put under version control
* you can stop at pillar generation
* you can achieve some change detection too, as you have acces to both previously generated pillar and new values.

**Context:** `customers` for a web agency are defined at higher level in `pillar/customers.sls`

See: `pillar.example`

The `init.sls` state is producing pillar for other formulas:

* apache-formula
* mysql-formula
* users-formula
* custom salt states for managing DNS zones with PowerDNS
* more

**Lecteurs francophones:** n'hésitez pas à demander une traduction si nécessaire, je la produirai.

## Source customers pillar

It looks like:

~~~yaml
customers_top: wsf
wsf:
  global:
    webmaster: someone@webmaster.com
    dbserver: datbase.domain.com
    webserver: web.domain.com
  customers:
    client1:
      domain_name: client1-domain.fr
      webmaster: client1@webmaster.com
      enabled: true
      delete: false
      # service to configure for this customer
      services:
        - webhost
        - dns
        - db
        - sftp
    # client2 as more default values
    client2:
      domain_name: more-domain.com
      enabled: true
      delete: false
      services:
        - webhost
        - dns
    client3:
      domain_name: somedomain.fr
      enabled: true
      # default, delete: false
      services:
        - webhost
        - dns
        - db
        - sftp
~~~

## Produced mysql-formula pillar

For this pillar above, we expect to generate users for having access to mariaDB.

Here is a *representation* of the merged output of `pillar/auto/mysql_db.sls` + `pillar/auto/mysql_users.sls`
Password management dosen't work that way, but it is just for description here. This pillar example
is on the format expected by [mysql-formula](https://github.com/saltstack-formulas/mysql-formula).

~~~yaml
## generated jinja include here, omitted
mysql:
  # Managed databases for customers
  # those databases are generated by the state/customers/init.sls
  database:
    - client1
    - client2
    - client3
  # [ merged… ]
  # Managed mariaDB users for customers
  # those mariaDB users are generated by the state/customers/init.sls
  # passwords are handled externally by a custom python script.
  user:
    client1:
      password: "{{ pass['client1']['mysql'] }}"
      hosts:
        - localhost
      databases:
        - database: client1
          grants: ['all privileges']
    client2:
      password: "{{ pass['client2']['mysql'] }}"
      hosts:
        - localhost
      databases:
        - database: client2
          grants: ['all privileges']
    client3:
      password: "{{ pass['client3']['mysql'] }}"
      hosts:
        - localhost
      databases:
        - database: client3
          grants: ['all privileges']
~~~

## install in the pillar

### `pillar/top.sls`

~~~yaml
# vim: set ft=yaml:
#
# Pillar top inclusions
base:
  '*':
    # top level definition for customers avail to all rules
    - customers
  'db*':
    # pillar are merged
    # mysql.defaults it local site common usage of mysql-formula config…
    - mysql.defaults
    - auto.mysql_db
    - auto.mysql_users
~~~

### `state`

Put the formula code in your `file_roots:`, in `/etc/salt/master`

* `pillar_dir` [line 23](/customers/init.sls#L23) (TODO: define it in pillar)
* `target_dir` [line 24](/customers/init.sls#L24) is computed with `customers_top:customers_dir`

## generate
run on the master, with care ;) it creates file in your pillar or where you put the path above

~~~bash
salt-call state.apply customers
~~~

This should create the `auto/` folder and files inside, which need be propagated that way:

~~~bash
salt '*' saltutil.refresh_pillar
~~~

### More usages

Usage: (on the saltmaster)

Only generate pillar

~~~bash
salt-call state.apply customers
~~~


Verify with git
~~~bash
salt-call state.apply customers
cd target_dir
git diff # double check
~~~

Propagate to mininons
~~~bash
salt '*' saltutil.refresh_pillar
~~~

Check some values

~~~bash
salt 'db*' config.get mysql:user
salt 'web*' config.get apache:sites
salt 'web*' config.get users
~~~

run rules based on the generated pillar 

~~~bash
salt 'db*' state.apply mysql.user
~~~

commit changes:
~~~bash
cd target_dir
git commit -a
~~~

~~~bash
salt-call state.apply customers pillar='{"customers_top" : "another_value" }'
~~~

## orchestration

~~~bash
salt-run state.orch orch.all
~~~

Only the pillar:

~~~bash
salt-run state.orch orch.pillar
~~~

(TODO: add files in this repos)

`orch/pillar.sls`:

~~~yaml
genrate-customers-pillar:
  salt.function:
    - tgt: 'salt*'
    - name: state.apply
    - arg:
      - customers
    - kwarg:
       pillar:
          customers_top: wsf
    #- failhard: True

synchronize-pillar:
  salt.function:
    - name: saltutil.refresh_pillar
    - tgt: '*'
~~~


`orch/all.sls`
~~~yaml
include:
  - orch.pillar

make-world:
  salt.state:
    - tgt: '*'
    - highstate: True
~~~

## TODO

manage password with https://clinta.github.io/random-local-passwords/ passwordstore
