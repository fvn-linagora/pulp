Upload RPM packages using Pulp REST endpoints
=================================

Setup
----------

A vagrant-managed VM is used for hosting pulp officially provided docker images. Host is running CentOS/7

- Starting host after syncing pulp dockerfiles with VM shared folders :

        vagrant rsync
        vagrant ssh

- Starting up Pulp containers

        cd /path/to/your/github.com/pulp/packaging/clone/docker/dockerfiles/centos
        setenforce 0 # disable SELinux
        sudo mkdir /var/pulp-data
        ./start.sh /var/pulp-data

- Manage repositories using a docker container as client CLI

        sudo docker run --rm -it -v /vagrant:/opt --link pulpapi:pulpapi pulp/adm

        pulp-admin login -u admin -p admin
        pulp-admin rpm repo uploads rpm --repo-id tested -f /opt/james-3.0_beta6-2.x86_64.rpm
        pulp-admin rpm repo publish  run --repo-id tested

- Perform HTTP queries to the REST endpoints using a linked docker container

        sudo docker run -ti --rm --link pulpapi:pulpapi -v /vagrant:/opt centos:7 /bin/bash

- Initialize a package upload session

    *POST /pulp/api/v2/content/uploads/*

        curl -k --user admin:admin -X POST https://pulpapi/pulp/api/v2/content/uploads/

    You are provided a *session* as response:

        {"upload_id": "b57de43d-9a20-4814-93bf-ee0c90eafd29", "_href": "/pulp/api/v2/content/uploads/b57de43d-9a20-4814-93bf-ee0c90eafd29/"}

- Upload parts of the binary packages (sharing same upload session)

    *PUT /pulp/api/v2/content/uploads/<upload_id>/<offset/*

        curl -k --user admin:admin -X PUT https://pulpapi/pulp/api/v2/content/uploads/b57de43d-9a20-4814-93bf-ee0c90eafd29 --upload-file /opt/james-3.0_beta6-2.x86_64.rpm

- Import uploaded item into existing repository (eg `tested`)

    *POST /pulp/api/v2/repositories/<repo_id>/actions/import_upload/*

        echo '{"override_config": {}, "unit_type_id": "srpm", "upload_id": "b57de43d-9a20-4814-93bf-ee0c90eafd29", "unit_key": {}, "unit_metadata": {"checksum_type": null}}' | curl -d @- -k --user admin:admin -X POST https://pulpapi/pulp/api/v2/repositories/tested/actions/import_upload/ 

    Or alternatively (for readability purpose) :

        echo '{"override_config": {}, "unit_type_id": "srpm",
                "upload_id": "b57de43d-9a20-4814-93bf-ee0c90eafd29", "unit_key": {},
                "unit_metadata": {"checksum_type": null}}' \
                | curl -d @- -k --user admin:admin -X POST https://pulpapi/pulp/api/v2/repositories/tested/actions/import_upload/ 

    This will spawn a task (identified by `task_id` for tracking progress).
    *202 - if the request for the import was accepted but postponed until later*
    Received response:

        {"spawned_tasks": [{"_href": "/pulp/api/v2/tasks/b2942634-ce52-4e48-9163-0f211b3ea4cf/", "task_id": "b2942634-ce52-4e48-9163-0f211b3ea4cf"}], "result": null, "error": null}

- Finally, once all items have been uploaded, repository have to be published. You can do that either on a schedule or manually like so  :

    *POST /pulp/api/v2/repositories/<repo_id>/actions/publish/*

    {
    "id": "distributor_1",
    "override_config": {},
    }

    First, list all available distributors

        curl -k --user admin:admin -X GET https://pulpapi/pulp/api/v2/repositories/tested/distributors/

        [{"repo_id": "tested", "_ns": "repo_distributors", "last_publish": "2016-12-14T13:20:48Z", "auto_publish": true, "scheduled_publishes": [], "distributor_type_id": "yum_distributor", "scratchpad": null, "_id": {"$oid": "585143311760e5000e3151d4"}, "config": {"http": false, "https": true, "relative_url": "tested"}, "id": "yum_distributor"}, {"repo_id": "tested", "_ns": "repo_distributors", "last_publish": null, "auto_publish": false, "scheduled_publishes": [], "distributor_type_id": "export_distributor", "scratchpad": null, "_id": {"$oid": "585143311760e5000e3151d5"}, "config": {"http": false, "https": true}, "id": "export_distributor"}]

    We notice, by default there are 2 distributors per repo (`yum_distributor` and `export_distributor`). We'll use `yum_distributor` below.

        echo '{ "id": "yum_distributor", "override_config": {} }' | curl -d @- -k --user admin:admin -X POST https://pulpapi/pulp/api/v2/repositories/tested/actions/publish/

        {"spawned_tasks": [{"_href": "/pulp/api/v2/tasks/3a0f5677-681e-4f08-b4fa-9324245246e3/", "task_id": "3a0f5677-681e-4f08-b4fa-9324245246e3"}], "result": null, "error": null}

    To check that repository is including recently uploaded item (ie RPM package), issue an authenticated request:  

        curl -k --user admin:admin https://pulpapi/pulp/repos/tested/ | grep '<a href="james-3.0_beta6-2.x86_64.rpm">james-3.0_beta6-2.x8..&gt;</a>'

    