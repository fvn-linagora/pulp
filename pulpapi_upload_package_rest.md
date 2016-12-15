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

- Perform HTTP queries to the REST endpoints

        sudo docker run -ti --rm --link pulpapi:pulpapi -v /vagrant:/opt centos:7 /bin/bash

    *POST /pulp/api/v2/content/uploads/*

        curl -k --user admin:admin -X POST https://pulpapi/pulp/api/v2/content/uploads/

    response:

        {"upload_id": "b57de43d-9a20-4814-93bf-ee0c90eafd29", "_href": "/pulp/api/v2/content/uploads/b57de43d-9a20-4814-93bf-ee0c90eafd29/"}

    *PUT /pulp/api/v2/content/uploads/<upload_id>/<offset/*

        curl -k --user admin:admin -X PUT https://pulpapi/pulp/api/v2/content/uploads/b57de43d-9a20-4814-93bf-ee0c90eafd29 --upload-file /opt/james-3.0_beta6-2.x86_64.rpm

    *POST /pulp/api/v2/repositories/<repo_id>/actions/import_upload/*

        echo '{"override_config": {}, "unit_type_id": "srpm", "upload_id": "b57de43d-9a20-4814-93bf-ee0c90eafd29", "unit_key": {}, "unit_metadata": {"checksum_type": null}}' | curl -d @- -k --user admin:admin -X POST https://pulpapi/pulp/api/v2/repositories/tested/actions/import_upload/

    Or alternatively :

        echo '{"override_config": {}, "unit_type_id": "srpm",
                "upload_id": "b57de43d-9a20-4814-93bf-ee0c90eafd29", "unit_key": {},
                "unit_metadata": {"checksum_type": null}}' \
                | curl -d @- -k --user admin:admin -X POST https://pulpapi/pulp/api/v2/repositories/tested/actions/import_upload/

    Received response:

        {"spawned_tasks": [{"_href": "/pulp/api/v2/tasks/b2942634-ce52-4e48-9163-0f211b3ea4cf/", "task_id": "b2942634-ce52-4e48-9163-0f211b3ea4cf"}], "result": null, "error": null}

202 - if the request for the import was accepted but postponed until later
