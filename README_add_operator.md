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
<snip>
...
</snip>
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
```
Push the image to the disconected registry
```
$ podman push registry.ds.local.lab:8443/olm-mirror/redhat-operator-index:v4.7
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
```
Push the image to your disconnected registry
```
$ podman push registry.ds.local.lab:8443/olm-mirror/redhat-operator-index:v4.7-tmp
```
Mirror the operator to disk.
```
$ oc adm catalog mirror registry.ds.local.lab:8443/olm-mirror/redhat-operator-index:v4.7-tmp file:///mirror -a ${XDG_RUNTIME_DIR}/containers/auth.json
<snip>
...
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
<snip>
...
</snip>
wrote mirroring manifests to manifests-olm-mirror/redhat-operator-index-1631127969
```
Modify the `mapping.txt` file as necessary.  The line that would mirror the temporary catalog index image can be removed.  The destination paths can be modified as desired as well.  

Upload the images using the modified file.
```
$ oc image mirror --filter-by-os='.*' -a ${XDG_RUNTIME_DIR}/containers/auth.json -f manifests-olm-mirror/redhat-operator-index-1631127969/mapping.txt
```
Grab the content of `repositoryDigestMirrors` from `manifests-<namespace>/<operator-image>-<random-num>/imageContentSourcePolicy.yaml`.  Update the appropriate `ImageContentSourcePolicy` on the disconnected cluster.  

If the operator was mirrored to disk first, remember to update the entries in the `imageContentSourcePolicy.yaml` file to change the disk based source to `registry.redhat.io`.