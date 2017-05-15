# Deployment playbooks for NGI-Pipeline and related software (Piper, TACA, Tarzan, etc.) 

The NGI pipeline is deployed on Irma using Ansible playbooks. Ansible playbooks are scripts written for easily adaptable automated deployment. The Ansible playbooks are stored here.

## Bootstrap the Ansible environment

Before any deployments can be done we need to setup the Ansible environment. If we've got a clean environment then this can be done by running the bootstrap script: 

```
newgrp ngi-sw
curl -L https://raw.githubusercontent.com/NationalGenomicsInfrastructure/irma-provision/master/bootstrap/bootstrap.sh -o /tmp/bootstrap.sh
bash /tmp/bootstrap.sh
```

It is recommended that the user adds the following two lines (or something similar) into `~/.bashrc`: 

```
alias irmaenv='source /lupus/ngi/irma3/bashrc'
alias ansibleenv='source /lupus/ngi/irma3/ansible-env/bin/activate'
```

## User prerequisites before developing or deploying 

Before a user starts developing new Ansible playbooks/roles or deploy them, the current umask and GID needs to be set, and the Ansible virtual environment needs to be loaded. 

This can be accomplished by *manually* running the two bash aliases defined above: `irmaenv` followed by `ansibleenv`.  

Note that the order is important, and that they should not be run automatically at login, because that will cause an infinite loop that will lock out the user from `irma3`. 

Also note that the `NouGAT` role sometimes has problems running, and may require you to load the `boost` module prior to usage.

## Deployment of the NGI pipeline

### Requirements
The following files need to be present on irma3 in order to successfully deploy a working version of NGI pipeline:

- A working connection to charon or charon-dev as well as a generated API token. 

- A valid GATK key placed under `/lupus/ngi/irma3/deploy/files`. The filename must be specified in the `gatk_key` variable in `host_vars/127.0.0.1/main.yml`. 

- A `charon_credentials.yml` file placed under `/lupus/ngi/irma3/deploy/host_vars/127.0.0.1/` listing the variables `charon_base_url_{stage,prod}`, `charon_api_token_upps_{stage,prod}` and `charon_api_token_sthlm_{stage,prod}`

- A valid `statusdb_creds_{stage,prod}.yml` access file placed under `/lupus/ngi/irma3/deploy/files`. Necessary layout is described at https://github.com/SciLifeLab/statusdb

- Valid SSL certificates for the web proxy under `/lupus/ngi/irma3/deploy/files` (see `roles/tarzan/README.md` for details) 

### Typical production deployments

A typical deployment of a production environment to Irma consists of two steps

- running through the Ansible playbook for a production release, which will install all the software under `/lupus/ngi/production/<version>` (and create a symlink `/lupus/ngi/production/latest` pointing to it)
- syncing everything under `/lupus/ngi/production` to the cluster 

To accomplish this run the following commands: 

```
   cd /lupus/ngi/irma3/deploy
   git fetch --tags 
   git checkout tags/vX.Y
   ansible-playbook install.yml -e deployment_environment=production
   python sync.py production 
```

This will install and sync over the Irma environment version `vX.Y`. 

To see all available production releases go to https://github.com/NationalGenomicsInfrastructure/irma-provision/releases

### Typical development and staging deployments

Typically roles are developed (or at least tested) locally on `irma3`. 

Start by forking the repository https://github.com/NationalGenomicsInfrastructure/irma-provision to your private Github repo. Then clone this private repository to `/lupus/ngi/irma3/devel`. Inside there you can develop your own Ansible roles in your private feature branch. 

If you want to test your roles/playbook run `ansible-playbook install.yml`. This will install your development run in `/lupus/ngi/irma3/devel-root/<username>_<branch_name>`. 

When you are satisfied with your changes you need to test it in staging. First create a pull request from your feature branch into upstream irma-provisioning's master branch. 

Do the following once the feature has been approved: 

```
    cd /lupus/ngi/irma3/deploy
    git checkout master 
    git pull 
    ansible-playbook install.yml -e deployment_environment=staging -e deployment_version=<YYMMDD>.<commit_short>
    python sync.py staging
```

where `FOO` is a unique staging version you pick. This will install your run under `/lupus/ngi/staging/FOO` and symlink `/lupus/ngi/staging/latest` to it, for easier access. 

If you want to stage test many feature branches at the same time then an alternative is to create a pre-release of the master branch at Github (https://github.com/NationalGenomicsInfrastructure/irma-provision/releases/new), and then running the `ansible-playbook` command after having checkout the corresponding pre-release tag (similar to how it is done for production deployments). Make sure to write a good release note so it is clear what significant things are included. 

When everything is synced over to Irma properly then login to e.g. `irma1` as your personal user and run `source /lupus/ngi/staging/latest/conf/sourceme_<site>.sh && source activate NGI` (where `site` is `upps` or `sthlm` depending on location). For convenience add this to your personal file bash init file `~/.bashrc`. This will load the staging environment for your user with the appropriate staging variables set. 

When the staged environment has been verified to work OK (TODO: add test protocol, manual or automated sanity checks) proceed with making a production release at https://github.com/NationalGenomicsInfrastructure/irma-provision/releases/new. Make sure to write a good release note that summarizes all the significant changes that are included since the last production release. 

##### Arteria staging 

The Arteria roles will pick up on whether we're running in `deployment_environment` equal to `production` or `staging` and then use different ports, runfolders, log files, etc.

So in essence it should work almost as usual. You run something similar to: 

```
ansible-playbook install.yml -e deployment_environment=staging -e deployment_version=arteria-staging-FOO -e arteria_checksum_version=660a8ff
python sync.py staging
```

if you want to stage test a specific commit hash of `arteria-checksum`, and the default version of `arteria-siswrap`. When launching the staging services it is recommended to do this inside `screen`, although there are some bundled `supervisord` configs as well. So login to irma1, start a screen session with `screen -S arteria-staging`, and then launch the following commands in separate screen windows: 

```
/lupus/ngi/staging/arteria-staging-FOO/sw/arteria/siswrap_venv/bin/siswrap-ws --configroot=/lupus/ngi/staging/arteria-staging-FOO/conf/arteria/siswrap/ --port=10431 --debug
/lupus/ngi/staging/arteria-staging-FOO/sw/arteria/checksum_venv/bin/checksum-ws --configroot=/lupus/ngi/staging/arteria-staging-FOO/conf/arteria/checksum/ --port=10421 --debug
/lupus/ngi/staging/arteria-staging-FOO/sw/anaconda/envs/arteria-delivery/bin/pytohn /lupus/ngi/staging/arteria-staging-FOO/sw/anaconda/envs/arteria-delivery/bin/delivery-ws --configroot=/lupus/ngi/staging/arteria-staging-FOO/conf/arteria/delivery/ --port=10421 --debug
```

#### Nota bene

Remember that you will probably have to restart services manually after a new production release have been rolled out. First re-load the crontab as the func user on `irma1` with a `crontab /lupus/ngi/production/latest/conf/crontab_SITE`. Then, depending on what software your func user is running, continue with manually shutting down the old versions and re-start the new versions of the software. 

### Manual initiations on irma1

Run `crontab /lupus/ngi/conf/crontab_<site>` once per user to initialize the first instance of cron for the user. As mentioned above, this has to be done for each new production release. 

Run `/lupus/ngi/resources/create_ngi_pipeline_dirs.sh <project_name>` once per project (i.e. ngi2016003) to create the log, db and softlink directories for NGI pipeline (and generate the softlinks).

Add `source /lupus/ngi/production/latest/conf/sourcme_<site>.sh && source activate NGI`, where `site` can be `upps` or `sthlm`, to each functional account's bash init file `~/.bashrc`. 

### Quick integrity verification

Run `source /lupus/ngi/conf/sourceme_<SITE>.sh` where `<SITE>` is `upps` to initialize `funk_004` variables, and `sthlm` to initialize `funk_006` variables.

Run `source activate NGI` to start the environment.

`python NGI_pipeline_test.py create --fastq1 <R1.fastq.gz> --fastq2 <R2.fastq.gz> --FC 1` creates a simulated flowcell.

Run `ngi_pipeline_start.py` with the commands `organize flowcell`, `analyze project` and `qc project` to organize, analyze and qc the generated data respectively.

### Other worthwhile information

Deploying requires the deployer to be in both the `ngi-sw` and the `ngi` groups. Everything under `/lupus/ngi/` is owned by `ngi-sw`.

Only deployers can write new programs and configs, but all NGI functional accounts (`funk_004`, `funk_006` etc) can write log files, to the SQL databases, etc.

Global configuration values are set in the repository under `host_vars/127.0.0.1/main.yml`, and each respective role's in the `<role>/defaults/main.yml` file. 

The log `/lupus/ngi/irma3/log/ansible.log` logs the files installed by Ansible. 

The log `/lupus/ngi/irma3/log/rsync.log` logs the rsync history. 

When developing roles deployed directories must have the setgid flag `g+s`, and be created while the deployer had his gid set to `ngi-sw`. This ensures the files in the dirs recieve the correct group owner when they are created. This will be taken care of the user always sources the file `/lupus/ngi/irma3/bashrc` before doing any work.
