
# AAP DRP (Controller + Hub)

This project is intended to aid you in the configuration of the replication of the DB servers used in an Ansible Automation Platfor deployment, and the execution of the DRP and its respective rollback actions.
## Features

- Configures streaming replication among DB servers (Master - Slave) ([Based on this guide](https://www.cherryservers.com/blog/how-to-set-up-postgresql-database-replication)).
- Performs DRP of the platform (Controller and Hub instances) in case of a complete or partial failure in the main site (where the Master database is running).
- Performs the rollback of the DRP, by syncing the data of the slave database to the master database, and restoring the replication.
## Usage/Examples

### Requirements/considerations
- For the nodes where the playbooks are going to be executed, SSH key-based authentication must be set against the database servers (this is required by the ansible.posix.synchronize module). Please, note that this playbooks where developed by using root user as the user to perform the tasks in the managed nodes. If you are using a different user, ensure it has the appropiate privileges to avoid permissions issues.
- SSH key-based authentication must be set among Slave and Master DB servers (i.e. Slave DB root user must be able to login to the Master DB server without specifying a password).
- Ensure to adjust the playbooks according to your platform requirements.

### Edit the inventory file
Specify your master and slave DB servers in the "db_servers" group. Add your controller and hub nodes to their respective groups. **Host aliases must not be changed**.

```ini
[db_servers]
db_primary ansible_host={master_db_server}
db_secondary ansible_host={slave_db_server}

[automation_controller]
control_node1 ansible_host={control_node_1}
control_node2 ansible_host={control_node_2}
control_nodeN ansible_host={control_node_N}
...

[automation_hub]
automation_hub1 ansible_host={automation_hub_node_1}
automation_hub2 ansible_host={automation_hub_node_1}
automation_hubN ansible_host={automation_hub_node_N}
...

```

### Configure the database replication
If you need to configure the database replication run the "db_replication.yml" playbook. **Don't run this playbook if the database replication is already running**.

```bash
ansible-playbook db_replication.yml
```

This playbook will:
1. Create a new user, called "replica_user", in the Master DB, which will be used for replication purpuses.
2. Set Trust authentication method in the "pg_hba.conf" file, in order to allow the Slave server to login to the Master postgres engine without specifying a password.
3. Set the following parameters in the "postgresql.conf" file:
    - listen_addresses = '*' **&rarr;** Allows connections from any network to the DB server.
    - wal_level = logical **&rarr;** Specifies the amount of information to be written to the Write Ahead Log (WAL) file.
    - wal_log_hints = on **&rarr;** Allows the PostgreSQL server to write the entire content of each disk page to the WAL file during the first modification of the page.
4. Delete all data from Slave server, and start to replicate the Master server data.

### Perform DRP (from Main to Secondary DB)
If you need to execute the DRP, e.g. your Master DB server or your entire Main site are down, the "drp_aap.yml" playbook must be executed.

```bash
ansible-playbook drp_aap.yml
```

This playbook will:
1. Stop the Automation Controller and Automation Hub services, in order to prevent from new connections or data injection to the DB.
2. Configure the DB parameters pointing to the secondary DB in the "/etc/tower/conf.d/postgres.py" and "/etc/pulp/settings.py" files, for Cotroller and Hub nodes, respectively.
3. Promote the secondary (Slave) DB to assume the Master role, while the primary DB is down.
4. Start the Automation Controller and Automation Hub services, using now the secondary database.

### Perform DRP rollback (from Secondary to Main DB)
Once the Main site, or the Master DB server are back online, you can execute the "rollback_drp_aap.yml" playbook in order to swap back to the Master server as Main DB server, and restore the replication to the Slave server.

```bash
ansible-playbook rollback_drp_aap.yml
```

This playbook will:
1. Stop Controller and Hub services, to prevent new connections or data injectio to the database.
2. Configure the DB parameters pointing to the primary DB in the "/etc/tower/conf.d/postgres.py" and "/etc/pulp/settings.py" files, for Cotroller and Hub nodes, respectively.
3. Delete outdated data from the primary server, and copy current data from the secondary server.
4. Restore replication from Main to Secondary.
5. Start the Automation Controller and Automation Hub services, using now the primary database.
