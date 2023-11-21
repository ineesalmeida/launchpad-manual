===============================
Add builders to new arch/region
===============================

Deploy to qastaging
===================

1. Create openstack server:
    * `ssh launchpad-bastion-ps5`
    * `sudo -iu stg-vbuilder-qastaging`
    * `source ~/.scalingstack/scalingstack-bos03-vbuilder-staging.novarc`
    * Create openstack server by running the following command after replacing 
    the <> parameters::
        
        openstack server create --image auto-sync/ubuntu-focal-<distro number>-<arch>-server-<date>-disk1.img --flavor vbuilder-<arch> --key-name imagebuilder --nic net-id=<net id> imagebuilder-<arch>-<distro>
    
    For example::

        openstack server create --image auto-sync/ubuntu-focal-20.04-arm64-server-20231011-disk1.img --flavor vbuilder-arm64 --key-name imagebuilder --nic net-id=f80c2f02-a048-408c-9095-332153fd0b66 imagebuilder-arm64-focal

    * Get its IP by running `openstack server list`, the IP will be in Networks
    column, example `vbuilder_staging_test_net=10.144.0.127`
    (IP is `10.144.0.127`)
2. Update code in `launchpad-mojo-specs/vbuilder` to add the new builder/region
to `qastaging` only. The IP gotten in the step before should be added to the
region modifiers. For example,
[this MP|https://code.launchpad.net/~ines-almeida/launchpad-mojo-specs/+git/private/+merge/455059]
adds arm64 to bos03. 
3. SSH into `stg-vbuilder-qastaging@launchpad-bastion-ps5`, and update
`.local/share/mojo/LOCAL/mojo-stg-vbuilder-qastaging/vbuilder/qastaging/secrets`
file and add the new arch/region. Every builder in a region has the same
password, so you will be able to copy the entry for a same region setup, and
update the name. 
4. Run `mojo run`
5. Run `juju config glance-simplestreams-sync-<region>-<arch> run=true` (for
example, `juju config glance-simplestreams-sync-bos03-arm64 run=true`)
6. Verify the image was created:
    * `ssh launchpad-bastion-ps5`
    * `sudo -iu stg-vbuilder-qastaging`
    * `source ~/.scalingstack/scalingstack-bos03-vbuilder-staging.novarc`
    * Run `openstack image list` and check that new images are there
7. (You need to be part of launchpad-buildd-admins team for this:) Add builders
to the builder list by running `create-launchpad-vbuilder.py` (found in the
`launchpad-vbuilder-manage` repo). To add builder number 001, you can run it as:: 
    
    create-launchpad-vbuilder --domain vbuilder.qastaging.<region>.scalingstack --environment qastaging --manager-host vbuilder-manage-<region>.qastaging.lp.internal --processor <arch processor> qastaging-<region>-<arch>-001

For example, to add `arm64` builders to `bos03`::
    
    create-launchpad-vbuilder --domain vbuilder.qastaging.bos03.scalingstack --environment qastaging --manager-host vbuilder-manage-bos03.qastaging.lp.internal --processor arm64 --processor armel --processor armhf qastaging-bos03-arm64-001

8. Check that the new builder shows up in the `/builders` domain.
9. (You need to be part of launchpad-buildd-admins team for this:) Set the new
builders as ‘automatic’ instead of manual (within the /builders page, open the
builder, and click on `switch to automatic mode` button).
10. To test: run a snap build with the new architecture. You can ensure that
the build will run in the right builder (region) by setting the other builders
to manual (open the builder, and click on `switch to manual mode` button).

Notes: There is a script called `manage-builders`` that comes from
`ubuntu-archive-tools` project repository. This script can be used for bulk
operations for builders (try `manage-builders -l qastaging -v`). This can be
used for verifying the builders statuses, checking their configuration and so
on. During the process we used it to check that the builder was added
successfully.


Deploy to production
====================

After a successful qastaging deployment, we are ready to do something similar
in production.

1. Ask IS to create the image builder, return the image IP, and update the
deploy secrets. See exmaple in https://portal.admin.canonical.com/C160747.
    * Run equivalent to `openstack server create --image auto-sync/ubuntu-focal-20.04-arm64-server-20231011-disk1.img --flavor builder-cpu2-ram4-disk100 --key-name g-s-s --network vbuilder_imagebuilder_net imagebuilder-arm64-focal`
    * Request `openstack server list` in the end to check the IP
    * Update deploy secrets file `/home/prod-launchpad-vbuilders/.local/share/mojo/LOCAL/prod-launchpad-vbuilders/vbuilder/production/secrets`
2. Within the `launchpad-mojo-specs` repository, open a MP to remove the
`if qastaging` bit so that the new region arch is added to the lists for
production. Add the image IP to the region modifiers.
3. In `prod-launchpad-vbuilders@is-bastion-ps5`, ask IS to:
    * Run `mojo run`
    * Run `juju config glance-simplestreams-sync-<region, eg bos03>-<arch, eg arm64> run=true`
    * Share contents of juju status and openstack image list. Check that the
    new image is in the list.
4. Open new MP to mojo specs to add the builders:
https://code.launchpad.net/~ines-almeida/launchpad-mojo-specs/+git/private/+merge/455512.
5. Ask IS to run `mojo run`.
5. (You need to be part of launchpad-buildd-admins team for this:) Create the
new builders by running::
        
        create-launchpad-vbuilder --domain vbuilder.bos03.scalingstack --environment production --manager-host vbuilder-manage-bos03.lp.internal --processor arm64 --processor armel --processor armhf bos03-arm64-001

6. Test deployment by running a build in the newly added builder.
