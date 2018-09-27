We can also run some chaos engineering if your director has been deployed with turbulence.

* [Turbulence API](https://github.com/cppforlife/turbulence-release/blob/master/docs/api.md)
* [Turbulence API Docs](https://github.com/cppforlife/turbulence-release/blob/master/docs/)

!!! note
    By default BUCC does not build with turbulence

    ```
    cp src/bosh-deployment/turbulence.yml ops/9-turbulence.yml
    ```

```
export TURBULENCE_PASSWORD=$(bosh int state/creds.yml --path /turbulence_api_password)
cat << "EOF" > kill.sh
set -e

body='
{
	"Tasks": [{
		"Type": "Kill"
	}],

	"Selector": {
		"Deployment": {
			"Name": "static-web"
		},
		"Group": {
			"Name": "webserver"
		},
		"ID": {
			"Limit": "10%-60%"
		}
	}
}
'

echo $body | curl -vvv -k -X POST https://turbulence:${TURBULENCE_PASSWORD}@192.168.209.7:8080/api/v1/incidents -H 'Accept: application/json' -d @-
echo
EOF
chmod +x kill.sh
```
Run it
```
./kill.sh
```
Checking in the Openstack console, you will see that one or more vms will be gone.

After about a minute or so, BOSH will see this machine missing and start the process of rebuilding it. You can also schedule instances to be destroyed, and a bunch of other chaos engineering tasks, like full disk, high cpu, high memory etc..
