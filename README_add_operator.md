This process was worked out on a system that hosted the registry server and was connected to the Internet.  The steps needed for a disconnected registry are used (download to disk first), but no actual transfer of those file was used.

requires:
* oc
* opm
* sqlite
* correct credentials setup (registry.redhat.io and disconnected registry creds)
  * ~/.docker/config.json
  * ${XDG_RUNTIME_DIR}/containers/auth.json

Use `oc adm catalog mirror --manifests-only` to mirror the whole `redhat-operator-index` catalog.  This creates the database you need to query to find the operator image to add.  Make note of the path listed after `using database path mapping` for the next step.
```
$ oc adm catalog mirror registry.redhat.io/redhat/redhat-operator-index:v4.7 registry.ds.local.lab:8443/olm-mirror/redhat-operator-index --manifests-only
src image has index label for database path: /database/index.db
using database path mapping: /database/index.db:/tmp/107008609
W0908 17:47:10.127899  250694 manifest.go:442] Chose linux/amd64 manifest from the manifest list.
wrote database to /tmp/107008609
using database at: /tmp/107008609/index.db
no digest mapping available for registry.redhat.io/amq7/amq-online-1-controller-manager-rhel7-operator:1.4, skip writing to ImageContentSourcePolicy
no digest mapping available for registry.redhat.io/rhpam-7/rhpam-rhel8-operator:7.6.0, skip writing to ImageContentSourcePolicy
<snip>
.
.
.
</snip>
no digest mapping available for registry.redhat.io/3scale-amp26/3scale-operator:latest, skip writing to ImageContentSourcePolicy
no digest mapping available for registry.redhat.io/amq7/amq-online-1-topic-forwarder:1.4, skip writing to ImageContentSourcePolicy
wrote mirroring manifests to manifests-redhat-operator-index-1631123227
```
In the $PWD, there is a directory, remove the directory specified in the last line of the output from the previous step to elimiate potential confusion in later steps.
```
$ ls
manifests-redhat-operator-index-1631123227
$ rm -rf manifests-redhat-operator-index-1631123227/
```
Use `sqlite3` to inspect the database.  This dumps everything in the `related_image` table into a file.  Search the file of the operator you wish to add, and make note of the `operatorbundle_name` for the next step.
```
$ echo "select * from related_image;" | sqlite3 -line /tmp/107008609/index.db > related_image.txt
$ vim relate_image.txt 
```

Once you have the correct `operatorbundle_name`, you can grep the specific image name from the database.  Or simply find the appropriate image name while using `vim` in the previous step.
```
$ echo "select * from related_image where operatorbundle_name like 'mtc-operator.v1.5.1%';" | sqlite3 -line /tmp/107008609/index.db | grep operator-bundle
              image = registry.redhat.io/rhmtc/openshift-migration-operator-bundle@sha256:68ed55736af9e054b777315600f433f8ac304833f48a3765cc27d0355a3b4617
```

Use the `opm` command to add the new image to your existing disconnected catalog image.
- `--bundles`: the image from the previous step
- `-f`: the disconnected registry catalog image
- `-t`: the new disconnected registry image (changing the tag is optional)
``` 
$ opm index add --bundles registry.redhat.io/rhmtc/openshift-migration-operator-bundle@sha256:68ed55736af9e054b777315600f433f8ac304833f48a3765cc27d0355a3b4617 -f registry.ds.local.lab:8443/olm-mirror/redhat-operator-index:v4.7 -t registry.ds.local.lab:8443/olm-mirror/redhat-operator-index:v4.7
INFO[0000] building the index                            bundles="[registry.redhat.io/rhmtc/openshift-migration-operator-bundle@sha256:68ed55736af9e054b777315600f433f8ac304833f48a3765cc27d0355a3b4617]"
INFO[0000] Pulling previous image registry.ds.local.lab:8443/olm-mirror/redhat-operator-index:v4.7 to get metadata  bundles="[registry.redhat.io/rhmtc/openshift-migration-operator-bundle@sha256:68ed55736af9e054b777315600f433f8ac304833f48a3765cc27d0355a3b4617]"
INFO[0000] resolved name: registry.ds.local.lab:8443/olm-mirror/redhat-operator-index:v4.7  bundles="[registry.redhat.io/rhmtc/openshift-migration-operator-bundle@sha256:68ed55736af9e054b777315600f433f8ac304833f48a3765cc27d0355a3b4617]"
INFO[0000] fetched                                       bundles="[registry.redhat.io/rhmtc/openshift-migration-operator-bundle@sha256:68ed55736af9e054b777315600f433f8ac304833f48a3765cc27d0355a3b4617]" digest="sha256:29ba1bacc6af0673fddb6b343cb90a82c069a9eda27fd45b68985646c87800e6"
INFO[0000] fetched                                       bundles="[registry.redhat.io/rhmtc/openshift-migration-operator-bundle@sha256:68ed55736af9e054b777315600f433f8ac304833f48a3765cc27d0355a3b4617]" digest="sha256:5140b3a86fae096a58fa267c6fd7ba95b58fdb211633cd2e0d513cfbb435ea35"
INFO[0000] fetched                                       bundles="[registry.redhat.io/rhmtc/openshift-migration-operator-bundle@sha256:68ed55736af9e054b777315600f433f8ac304833f48a3765cc27d0355a3b4617]" digest="sha256:49beb9a91f1d82431c359d4906479042f50f64f69e679e09ec99079ccc7a2cab"
INFO[0000] fetched                                       bundles="[registry.redhat.io/rhmtc/openshift-migration-operator-bundle@sha256:68ed55736af9e054b777315600f433f8ac304833f48a3765cc27d0355a3b4617]" digest="sha256:b1313fc3a38a29224521852ea1d0e68a10dccea54503e03076d9b26150a17b08"
INFO[0000] fetched                                       bundles="[registry.redhat.io/rhmtc/openshift-migration-operator-bundle@sha256:68ed55736af9e054b777315600f433f8ac304833f48a3765cc27d0355a3b4617]" digest="sha256:3f64a6388cf7c7a07bb47425fd3e0ca8c1fc03ae41996d41feef2d28347935d7"
INFO[0000] fetched                                       bundles="[registry.redhat.io/rhmtc/openshift-migration-operator-bundle@sha256:68ed55736af9e054b777315600f433f8ac304833f48a3765cc27d0355a3b4617]" digest="sha256:3a9b902fe547f8e2e4b88bff46bb897a1f82ebab1581345926c99e821938a68e"
INFO[0000] fetched                                       bundles="[registry.redhat.io/rhmtc/openshift-migration-operator-bundle@sha256:68ed55736af9e054b777315600f433f8ac304833f48a3765cc27d0355a3b4617]" digest="sha256:0ef1a5ebad177f11913100e60d8ddbcb46b2e9af0f0bba537246b227f6d1accd"
INFO[0000] fetched                                       bundles="[registry.redhat.io/rhmtc/openshift-migration-operator-bundle@sha256:68ed55736af9e054b777315600f433f8ac304833f48a3765cc27d0355a3b4617]" digest="sha256:06127b9e1ec2a9c5be34a82ce895431d256f635bc769b7f985e84952b9876e0a"
WARN[0001] {"created":"2021-09-08T17:22:56.824635163Z","container":"79d6dbff1e27e28ced12f676f74af941870fae242c489b6f37a0af70cbd4e172","container_config":{"Hostname":"6764f356d26f","Domainname":"","User":"","AttachStdin":false,"AttachStdout":false,"AttachStderr":false,"ExposedPorts":{"50051/tcp":{}},"Tty":false,"OpenStdin":false,"StdinOnce":false,"Env":["PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin"],"Cmd":["registry","serve","--database","/database/index.db"],"Image":"sha256:5d8c978f05f35e7cb404b8039665f754c175bae56ac54114c70369a5cc66f41a","Volumes":{},"WorkingDir":"","Entrypoint":["/bin/opm"],"OnBuild":[],"Labels":{"io.buildah.version":"1.21.3","operators.operatorframework.io.index.database.v1":"/database/index.db"}},"config":{"Hostname":"6764f356d26f","Domainname":"","User":"","AttachStdin":false,"AttachStdout":false,"AttachStderr":false,"ExposedPorts":{"50051/tcp":{}},"Tty":false,"OpenStdin":false,"StdinOnce":false,"Env":["PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin"],"Cmd":["registry","serve","--database","/database/index.db"],"Image":"sha256:5d8c978f05f35e7cb404b8039665f754c175bae56ac54114c70369a5cc66f41a","Volumes":{},"WorkingDir":"","Entrypoint":["/bin/opm"],"OnBuild":[],"Labels":{"io.buildah.version":"1.21.3","operators.operatorframework.io.index.database.v1":"/database/index.db"}},"architecture":"amd64","os":"linux","parent":"sha256:977cb1d12b6d7ebe1867d80c75cc901898be47fe72676c4b25808bb9f5e67e19","rootfs":{"type":"layers","diff_ids":["sha256:bc276c40b172b1c5467277d36db5308a203a48262d5f278766cf083947d05466","sha256:4e7f383eb531db898a0835c445abdd067b256d452302b49b6b41eb712ac3d5bc","sha256:a98a386b6ec24d77ee4af183e773e6824515ee904249c31b0131192335359efb","sha256:be8d7a0551d12f8e048d563ec3a1b0580bc096ac012b975b05265724585d3673","sha256:a783728d270b5f58d55f15730810e9fece568933ce2de4aa437b738531486ded","sha256:6ff6011bec2dd960784ef0737c65901567d518e66aa4022942c7029600eccbcd"]},"history":[{"created":"2021-08-06T17:19:45.007652184Z","created_by":"/bin/sh -c #(nop) ADD file:34eb5c40aa00028921a224d1764ae1b1f3ef710d191e4dfc7df55e0594aa7217 in / "},{"created":"2021-08-06T17:19:45.183996158Z","created_by":"/bin/sh -c #(nop)  CMD [\"/bin/sh\"]","empty_layer":true},{"created":"2021-08-19T17:03:18.381942077Z","created_by":"/bin/sh -c apk update \u0026\u0026 apk add ca-certificates"},{"created":"2021-08-19T17:03:18.54249373Z","created_by":"/bin/sh -c #(nop) COPY file:d059a61e5fe70800aab11fbc409c1e5e191b9f484f20f506ef2a6ffca1b10fc6 in /etc/nsswitch.conf "},{"created":"2021-09-02T19:36:08.429247751Z","created_by":"/bin/sh -c #(nop) COPY file:5776f2d1ba46c96e4672ff080621549d492a97840e48aaa5994ba27c5481640d in /bin/opm "},{"created":"2021-09-02T19:36:09.108842544Z","created_by":"/bin/sh -c #(nop) COPY file:d152b88c7d1e4f825edb1173337c2994c80fce349331e1378947a054943f0c10 in /bin/grpc_health_probe "},{"created":"2021-09-08T10:45:57.693986133Z","created_by":"/bin/sh -c #(nop) LABEL operators.operatorframework.io.index.database.v1=/database/index.db","comment":"FROM quay.io/operator-framework/upstream-opm-builder:latest","empty_layer":true},{"created":"2021-09-08T17:22:55.001151621Z","created_by":"/bin/sh -c #(nop) ADD file:20f4f0c0da8dff2f5e20ae8dd515445ea0c4b8d39397d9f7641ea22fd42d92a7 in /database/index.db "},{"created":"2021-09-08T17:22:56.767864985Z","created_by":"/bin/sh -c #(nop) EXPOSE 50051","empty_layer":true},{"created":"2021-09-08T17:22:56.796739209Z","created_by":"/bin/sh -c #(nop) ENTRYPOINT [\"/bin/opm\"]","empty_layer":true},{"created":"2021-09-08T17:22:56.824977937Z","created_by":"/bin/sh -c #(nop) CMD [\"registry\", \"serve\", \"--database\", \"/database/index.db\"]","empty_layer":true}]}  bundles="[registry.redhat.io/rhmtc/openshift-migration-operator-bundle@sha256:68ed55736af9e054b777315600f433f8ac304833f48a3765cc27d0355a3b4617]"
INFO[0001] unpacking layer: {application/vnd.docker.image.rootfs.diff.tar.gzip sha256:06127b9e1ec2a9c5be34a82ce895431d256f635bc769b7f985e84952b9876e0a 2890181 [] map[] <nil>}  bundles="[registry.redhat.io/rhmtc/openshift-migration-operator-bundle@sha256:68ed55736af9e054b777315600f433f8ac304833f48a3765cc27d0355a3b4617]"
INFO[0001] unpacking layer: {application/vnd.docker.image.rootfs.diff.tar.gzip sha256:5140b3a86fae096a58fa267c6fd7ba95b58fdb211633cd2e0d513cfbb435ea35 2441601 [] map[] <nil>}  bundles="[registry.redhat.io/rhmtc/openshift-migration-operator-bundle@sha256:68ed55736af9e054b777315600f433f8ac304833f48a3765cc27d0355a3b4617]"
INFO[0001] unpacking layer: {application/vnd.docker.image.rootfs.diff.tar.gzip sha256:3f64a6388cf7c7a07bb47425fd3e0ca8c1fc03ae41996d41feef2d28347935d7 169 [] map[] <nil>}  bundles="[registry.redhat.io/rhmtc/openshift-migration-operator-bundle@sha256:68ed55736af9e054b777315600f433f8ac304833f48a3765cc27d0355a3b4617]"
INFO[0001] unpacking layer: {application/vnd.docker.image.rootfs.diff.tar.gzip sha256:3a9b902fe547f8e2e4b88bff46bb897a1f82ebab1581345926c99e821938a68e 16760568 [] map[] <nil>}  bundles="[registry.redhat.io/rhmtc/openshift-migration-operator-bundle@sha256:68ed55736af9e054b777315600f433f8ac304833f48a3765cc27d0355a3b4617]"
INFO[0001] unpacking layer: {application/vnd.docker.image.rootfs.diff.tar.gzip sha256:49beb9a91f1d82431c359d4906479042f50f64f69e679e09ec99079ccc7a2cab 3898751 [] map[] <nil>}  bundles="[registry.redhat.io/rhmtc/openshift-migration-operator-bundle@sha256:68ed55736af9e054b777315600f433f8ac304833f48a3765cc27d0355a3b4617]"
INFO[0001] unpacking layer: {application/vnd.docker.image.rootfs.diff.tar.gzip sha256:b1313fc3a38a29224521852ea1d0e68a10dccea54503e03076d9b26150a17b08 12044809 [] map[] <nil>}  bundles="[registry.redhat.io/rhmtc/openshift-migration-operator-bundle@sha256:68ed55736af9e054b777315600f433f8ac304833f48a3765cc27d0355a3b4617]"
INFO[0003] resolved name: registry.redhat.io/rhmtc/openshift-migration-operator-bundle@sha256:68ed55736af9e054b777315600f433f8ac304833f48a3765cc27d0355a3b4617
INFO[0003] fetched                                       digest="sha256:68ed55736af9e054b777315600f433f8ac304833f48a3765cc27d0355a3b4617"
INFO[0003] fetched                                       digest="sha256:2efa1be31bf9a59257e3b6b2abe5063ceb2af9b4292007ba380af06464fee07b"
INFO[0003] fetched                                       digest="sha256:412007458a1b87a9937134e6dc7541276f9a894f17f203ad966e2cbc8e8cfe7f"
INFO[0003] unpacking layer: {application/vnd.docker.image.rootfs.diff.tar.gzip sha256:2efa1be31bf9a59257e3b6b2abe5063ceb2af9b4292007ba380af06464fee07b 44994 [] map[] <nil>}
INFO[0003] Could not find optional dependencies file     dir=bundle_tmp658082752 file=bundle_tmp658082752/metadata load=annotations
INFO[0003] found csv, loading bundle                     dir=bundle_tmp658082752 file=bundle_tmp658082752/manifests load=bundle
INFO[0003] loading bundle file                           dir=bundle_tmp658082752/manifests file=backup.crd.yaml load=bundle
INFO[0003] loading bundle file                           dir=bundle_tmp658082752/manifests file=backupstoragelocation.crd.yaml load=bundle
INFO[0003] loading bundle file                           dir=bundle_tmp658082752/manifests file=crane-operator.v1.5.1.clusterserviceversion.yaml load=bundle
INFO[0003] loading bundle file                           dir=bundle_tmp658082752/manifests file=deletebackuprequest.crd.yaml load=bundle
INFO[0003] loading bundle file                           dir=bundle_tmp658082752/manifests file=downloadrequest.crd.yaml load=bundle
INFO[0003] loading bundle file                           dir=bundle_tmp658082752/manifests file=migration.openshift.io_directimagemigrations.yaml load=bundle
INFO[0003] loading bundle file                           dir=bundle_tmp658082752/manifests file=migration.openshift.io_directimagestreammigrations.yaml load=bundle
INFO[0003] loading bundle file                           dir=bundle_tmp658082752/manifests file=migration.openshift.io_directvolumemigrationprogresses.yaml load=bundle
INFO[0003] loading bundle file                           dir=bundle_tmp658082752/manifests file=migration.openshift.io_directvolumemigrations.yaml load=bundle
INFO[0003] loading bundle file                           dir=bundle_tmp658082752/manifests file=migration.openshift.io_miganalytics.yaml load=bundle
INFO[0003] loading bundle file                           dir=bundle_tmp658082752/manifests file=migration.openshift.io_migclusters.yaml load=bundle
INFO[0003] loading bundle file                           dir=bundle_tmp658082752/manifests file=migration.openshift.io_mighooks.yaml load=bundle
INFO[0003] loading bundle file                           dir=bundle_tmp658082752/manifests file=migration.openshift.io_migmigrations.yaml load=bundle
INFO[0003] loading bundle file                           dir=bundle_tmp658082752/manifests file=migration.openshift.io_migplans.yaml load=bundle
INFO[0003] loading bundle file                           dir=bundle_tmp658082752/manifests file=migration.openshift.io_migstorages.yaml load=bundle
INFO[0003] loading bundle file                           dir=bundle_tmp658082752/manifests file=migrationcontroller.crd.yaml load=bundle
INFO[0003] loading bundle file                           dir=bundle_tmp658082752/manifests file=podvolumebackup.crd.yaml load=bundle
INFO[0003] loading bundle file                           dir=bundle_tmp658082752/manifests file=podvolumerestore.crd.yaml load=bundle
INFO[0003] loading bundle file                           dir=bundle_tmp658082752/manifests file=resticrepository.crd.yaml load=bundle
INFO[0003] loading bundle file                           dir=bundle_tmp658082752/manifests file=restore.crd.yaml load=bundle
INFO[0003] loading bundle file                           dir=bundle_tmp658082752/manifests file=schedule.crd.yaml load=bundle
INFO[0003] loading bundle file                           dir=bundle_tmp658082752/manifests file=serverstatusrequest.crd.yaml load=bundle
INFO[0003] loading bundle file                           dir=bundle_tmp658082752/manifests file=volumesnapshotlocation.crd.yaml load=bundle
INFO[0003] Generating dockerfile                         bundles="[registry.redhat.io/rhmtc/openshift-migration-operator-bundle@sha256:68ed55736af9e054b777315600f433f8ac304833f48a3765cc27d0355a3b4617]"
INFO[0003] writing dockerfile: index.Dockerfile206360286  bundles="[registry.redhat.io/rhmtc/openshift-migration-operator-bundle@sha256:68ed55736af9e054b777315600f433f8ac304833f48a3765cc27d0355a3b4617]"
INFO[0003] running podman build                          bundles="[registry.redhat.io/rhmtc/openshift-migration-operator-bundle@sha256:68ed55736af9e054b777315600f433f8ac304833f48a3765cc27d0355a3b4617]"
INFO[0003] [podman build --format docker -f index.Dockerfile206360286 -t registry.ds.local.lab:8443/olm-mirror/redhat-operator-index:v4.7 .]  bundles="[registry.redhat.io/rhmtc/openshift-migration-operator-bundle@sha256:68ed55736af9e054b777315600f433f8ac304833f48a3765cc27d0355a3b4617]"
```
Push the image to the disconected registry
```
$ podman push registry.ds.local.lab:8443/olm-mirror/redhat-operator-index:v4.7
Getting image source signatures
Copying blob 0dd07d6d2ce0 done
Copying blob bc276c40b172 skipped: already exists
Copying blob 4e7f383eb531 skipped: already exists
Copying blob a783728d270b skipped: already exists
Copying blob a98a386b6ec2 skipped: already exists
Copying blob be8d7a0551d1 skipped: already exists
Copying config b1ded41213 done
Writing manifest to image destination
Storing signatures
```
Optionally, use podman to run the registry catalog image, and use `grpcurl` to inspect the image to ensure the new operator has been successfully added
```
$ podman run -p50051:50051 -it registry.ds.local.lab:8443/olm-mirror/redhat-operator-index:v4.7
WARN[0000] DEPRECATION NOTICE:
Sqlite-based catalogs and their related subcommands are deprecated. Support for
them will be removed in a future release. Please migrate your catalog workflows
to the new file-based catalog format.
time="2021-09-08T18:37:57Z" level=info msg="Keeping server open for infinite seconds" database=/database/index.db port=50051
time="2021-09-08T18:37:57Z" level=info msg="serving registry" database=/database/index.db port=50051
^Ctime="2021-09-08T18:38:13Z" level=info msg="shutting down..." database=/database/index.db port=50051

$ grpcurl -plaintext localhost:50051 api.Registry/ListPackages
{
  "name": "cincinnati-operator"
}
{
  "name": "compliance-operator"
}
{
  "name": "file-integrity-operator"
}
{
  "name": "mtc-operator"
}
```
Create a second catalog image with just the new operator.  This makes it easier to download the required images for just this operator (as opposed to filtering through `mapping.txt` to remove all the existing operators), and as a result, it is not necessary to download all of the operators again.
```
$ opm index add --bundles registry.redhat.io/rhmtc/openshift-migration-operator-bundle@sha256:68ed55736af9e054b777315600f433f8ac304833f48a3765cc27d0355a3b4617 -t registry.ds.local.lab:8443/olm-mirror/redhat-operator-index:v4.7-tmp
INFO[0000] building the index                            bundles="[registry.redhat.io/rhmtc/openshift-migration-operator-bundle@sha256:68ed55736af9e054b777315600f433f8ac304833f48a3765cc27d0355a3b4617]"
INFO[0000] resolved name: registry.redhat.io/rhmtc/openshift-migration-operator-bundle@sha256:68ed55736af9e054b777315600f433f8ac304833f48a3765cc27d0355a3b4617
INFO[0000] fetched                                       digest="sha256:68ed55736af9e054b777315600f433f8ac304833f48a3765cc27d0355a3b4617"
INFO[0000] fetched                                       digest="sha256:2efa1be31bf9a59257e3b6b2abe5063ceb2af9b4292007ba380af06464fee07b"
INFO[0000] fetched                                       digest="sha256:412007458a1b87a9937134e6dc7541276f9a894f17f203ad966e2cbc8e8cfe7f"
INFO[0001] unpacking layer: {application/vnd.docker.image.rootfs.diff.tar.gzip sha256:2efa1be31bf9a59257e3b6b2abe5063ceb2af9b4292007ba380af06464fee07b 44994 [] map[] <nil>}
INFO[0001] Could not find optional dependencies file     dir=bundle_tmp418488251 file=bundle_tmp418488251/metadata load=annotations
INFO[0001] found csv, loading bundle                     dir=bundle_tmp418488251 file=bundle_tmp418488251/manifests load=bundle
INFO[0001] loading bundle file                           dir=bundle_tmp418488251/manifests file=backup.crd.yaml load=bundle
INFO[0001] loading bundle file                           dir=bundle_tmp418488251/manifests file=backupstoragelocation.crd.yaml load=bundle
INFO[0001] loading bundle file                           dir=bundle_tmp418488251/manifests file=crane-operator.v1.5.1.clusterserviceversion.yaml load=bundle
INFO[0001] loading bundle file                           dir=bundle_tmp418488251/manifests file=deletebackuprequest.crd.yaml load=bundle
INFO[0001] loading bundle file                           dir=bundle_tmp418488251/manifests file=downloadrequest.crd.yaml load=bundle
INFO[0001] loading bundle file                           dir=bundle_tmp418488251/manifests file=migration.openshift.io_directimagemigrations.yaml load=bundle
INFO[0001] loading bundle file                           dir=bundle_tmp418488251/manifests file=migration.openshift.io_directimagestreammigrations.yaml load=bundle
INFO[0001] loading bundle file                           dir=bundle_tmp418488251/manifests file=migration.openshift.io_directvolumemigrationprogresses.yaml load=bundle
INFO[0001] loading bundle file                           dir=bundle_tmp418488251/manifests file=migration.openshift.io_directvolumemigrations.yaml load=bundle
INFO[0001] loading bundle file                           dir=bundle_tmp418488251/manifests file=migration.openshift.io_miganalytics.yaml load=bundle
INFO[0001] loading bundle file                           dir=bundle_tmp418488251/manifests file=migration.openshift.io_migclusters.yaml load=bundle
INFO[0001] loading bundle file                           dir=bundle_tmp418488251/manifests file=migration.openshift.io_mighooks.yaml load=bundle
INFO[0001] loading bundle file                           dir=bundle_tmp418488251/manifests file=migration.openshift.io_migmigrations.yaml load=bundle
INFO[0001] loading bundle file                           dir=bundle_tmp418488251/manifests file=migration.openshift.io_migplans.yaml load=bundle
INFO[0001] loading bundle file                           dir=bundle_tmp418488251/manifests file=migration.openshift.io_migstorages.yaml load=bundle
INFO[0001] loading bundle file                           dir=bundle_tmp418488251/manifests file=migrationcontroller.crd.yaml load=bundle
INFO[0001] loading bundle file                           dir=bundle_tmp418488251/manifests file=podvolumebackup.crd.yaml load=bundle
INFO[0001] loading bundle file                           dir=bundle_tmp418488251/manifests file=podvolumerestore.crd.yaml load=bundle
INFO[0001] loading bundle file                           dir=bundle_tmp418488251/manifests file=resticrepository.crd.yaml load=bundle
INFO[0001] loading bundle file                           dir=bundle_tmp418488251/manifests file=restore.crd.yaml load=bundle
INFO[0001] loading bundle file                           dir=bundle_tmp418488251/manifests file=schedule.crd.yaml load=bundle
INFO[0001] loading bundle file                           dir=bundle_tmp418488251/manifests file=serverstatusrequest.crd.yaml load=bundle
INFO[0001] loading bundle file                           dir=bundle_tmp418488251/manifests file=volumesnapshotlocation.crd.yaml load=bundle
INFO[0001] Generating dockerfile                         bundles="[registry.redhat.io/rhmtc/openshift-migration-operator-bundle@sha256:68ed55736af9e054b777315600f433f8ac304833f48a3765cc27d0355a3b4617]"
INFO[0001] writing dockerfile: index.Dockerfile532542177  bundles="[registry.redhat.io/rhmtc/openshift-migration-operator-bundle@sha256:68ed55736af9e054b777315600f433f8ac304833f48a3765cc27d0355a3b4617]"
INFO[0001] running podman build                          bundles="[registry.redhat.io/rhmtc/openshift-migration-operator-bundle@sha256:68ed55736af9e054b777315600f433f8ac304833f48a3765cc27d0355a3b4617]"
INFO[0001] [podman build --format docker -f index.Dockerfile532542177 -t registry.ds.local.lab:8443/olm-mirror/redhat-operator-index:v4.7-tmp .]  bundles="[registry.redhat.io/rhmtc/openshift-migration-operator-bundle@sha256:68ed55736af9e054b777315600f433f8ac304833f48a3765cc27d0355a3b4617]"
```
Push the image to your disconnected registry
```
$ podman push registry.ds.local.lab:8443/olm-mirror/redhat-operator-index:v4.7-tmp
Getting image source signatures
Copying blob bc276c40b172 skipped: already exists
Copying blob 4e7f383eb531 skipped: already exists
Copying blob 867c5c535875 done
Copying blob a98a386b6ec2 skipped: already exists
Copying blob a783728d270b skipped: already exists
Copying blob be8d7a0551d1 [--------------------------------------] 0.0b / 0.0b
Copying config 0f2843f4d4 done
Writing manifest to image destination
Storing signatures
```
Mirror the operator to disk.
```
$ oc adm catalog mirror registry.ds.local.lab:8443/olm-mirror/redhat-operator-index:v4.7-tmp file:///mirror -a ${XDG_RUNTIME_DIR}/containers/auth.json
src image has index label for database path: /database/index.db
using database path mapping: /database/index.db:/tmp/902199960
wrote database to /tmp/902199960
using database at: /tmp/902199960/index.db
<dir>
  mirror/olm-mirror/redhat-operator-index
    blobs:
      registry.ds.local.lab:8443/olm-mirror/redhat-operator-index sha256:3f64a6388cf7c7a07bb47425fd3e0ca8c1fc03ae41996d41feef2d28347935d7 169B
<snip>
.
.
.
</snip>
info: Mirroring completed in 1m21.1s (17.16MB/s)
wrote mirroring manifests to manifests-redhat-operator-index-1631127304

To upload local images to a registry, run:

	oc adm catalog mirror file://mirror/olm-mirror/redhat-operator-index:v4.7-tmp REGISTRY/REPOSITORY      
```
Remove the leftover `manifests-<operator-image>-<random-num>` directory.
```
$ rm -rf manifests-redhat-operator-index-1631127304
```
At this point, bundle the `v2` directory, and transfer it to your disconnected network.  

Once on the disconnected network, extract the bundle for the next step.  

Use `oc adm catalog mirror --manifests-only` to generate the `mapping.txt` file.  If no changes are needed to the structure/paths/etc. of the images, do not use `--manifests-only`, and directly mirror right to the registry.
```
$ oc adm catalog mirror file://mirror/olm-mirror/update-operator:latest registry.ds.local.lab:8443/olm-mirror -a ${XDG_RUNTIME_DIR}/containers/auth.json --manifests-only
src image has index label for database path: /database/index.db
using database path mapping: /database/index.db:/tmp/365898461
wrote database to /tmp/365898461
using database at: /tmp/365898461/index.db
no digest mapping available for file://mirror/olm-mirror/redhat-operator-index:v4.7-tmp, skip writing to ImageContentSourcePolicy
wrote mirroring manifests to manifests-olm-mirror/redhat-operator-index-1631127969
```
Modify the `mapping.txt` file as necessary.  The line that would mirror the temporary catalog index image can be removed.  The destination paths can be modified as desired as well.  

Upload the images using the modified file.
```
$  oc image mirror --filter-by-os='.*' -a ${XDG_RUNTIME_DIR} -f manifests-olm-mirror/redhat-operator-index-1631127969/mapping.txt
error: unable to load --registry-config: read /run/user/0: is a directory
[root@registry add-new-operator]# oc image mirror --filter-by-os='.*' -a ${XDG_RUNTIME_DIR}/containers/auths.json -f manifests-olm-mirror/redhat-operator-index-1631127969/mapping.txt
error: unable to load --registry-config: open /run/user/0/containers/auths.json: no such file or directory
[root@registry add-new-operator]# oc image mirror --filter-by-os='.*' -a ${XDG_RUNTIME_DIR}/containers/auth.json -f manifests-olm-mirror/redhat-operator-index-1631127969/mapping.txt
registry.ds.local.lab:8443/
  olm-mirror/rhmtc-openshift-migration-controller-rhel8
    blobs:
      file://mirror/olm-mirror/redhat-operator-index/rhmtc/openshift-migration-controller-rhel8 sha256:46cdcde062b27f6c24775848f60eeb6a7216a98d4d6d7617927f5024fc8edfc3 1.699KiB
<snip>
.
.
.
</snip>
info: Mirroring completed in 15.65s (87.21MB/s)
```
Grab the content of `repositoryDigestMirrors` from `manifests-<namespace>/<operator-image>-<random-num>/imageContentSourcePolicy.yaml`.  Update the appropriate `ImageContentSourcePolicy` on the disconnected cluster.  

If the operator was mirrored to disk first, remember to update the entries in the `imageContentSourcePolicy.yaml` file to change the disk based source to `registry.redhat.io`.