## Step-by-Step: Integrating LDAP with OpenShift and Automating Group Sync

Managing users and groups centrally is critical in enterprise Kubernetes environments. This guide walks through integrating FreeIPA (LDAP) with OpenShift and automating the group synchronization process so we never have to manually import groups again.

### What We Are Building

* **Centralized Authentication:** Users log in using their FreeIPA credentials.
* **Automatic Onboarding:** New LDAP users can immediately access OpenShift.
* **Continuous Synchronization:** A CronJob runs in the background to automatically update OpenShift groups whenever changes happen in LDAP.

---

1. **Create a Bind User in FreeIPA:** Run this on LDAP server.
First, create a dedicated "bind" user that OpenShift will use to query LDAP directory. It is best practice to use a service account rather than a personal admin account.

N.B: A bind user is required because OpenShift must authenticate to the LDAP server before it can search for users and groups. It acts as a dedicated read-only service account that allows OpenShift to locate a user's Distinguished Name (DN) and retrieve group information. Using a bind user is more secure than using an administrator account and avoids enabling anonymous LDAP access.

```bash
ipa user-add ldapbind --first=ldap --last=bind --password
# Enter password when prompted

```


2. **Store the Password in OpenShift:** Run this on OpenShift cluster.
Take the password I just created and store it securely as a Secret in OpenShift. This allows the cluster to authenticate against FreeIPA without exposing the password in plain text.

```bash
oc create secret generic ldap-bind-password \
  --from-literal=bindPassword='password_here' \
  -n openshift-config

```


3. **Configure OpenShift OAuth:**
Next, tell OpenShift where FreeIPA server is and how to map the user attributes. Create a file named `ldap.yaml` with the following configuration (be sure to update the IP address and domain details to match Our environment):

```yaml
apiVersion: config.openshift.io/v1
kind: OAuth
metadata:
  name: cluster
spec:
  identityProviders:
  - name: freeipa
    mappingMethod: claim
    type: LDAP
    ldap:
      url: "ldap://192.168.122.246/dc=example,dc=com?uid"
      bindDN: "uid=ldapbind,cn=users,cn=accounts,dc=example,dc=com"
      bindPassword:
        name: ldap-bind-password
      insecure: true
      attributes:
        id: [dn]
        name: [cn]
        email: [mail]
        preferredUsername: [uid]

```

Apply the configuration to cluster:

```bash
oc apply -f ldap.yaml

```

*Note: Once applied, test logging in as an LDAP user. However, LDAP groups will not be imported yet.*


4. **Define Group Sync Rules:** Create a ConfigMap.
To pull in groups, OpenShift needs a synchronization mapping. Instead of keeping this mapping in a local file, we will store it natively in OpenShift as a `ConfigMap`.

Create a file named `ldap-group-sync-config.yaml` and apply it:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: ldap-group-sync-config
data:
  group-sync.yaml: |
    kind: LDAPSyncConfig
    apiVersion: v1
    url: ldap://192.168.122.246
    bindDN: "uid=ldapbind,cn=users,cn=accounts,dc=example,dc=com"
    bindPassword:
      file: "/etc/secrets/bindPassword"
    insecure: true
    rfc2307:
      groupsQuery:
        baseDN: "cn=groups,cn=accounts,dc=example,dc=com"
        scope: sub
        derefAliases: never
        filter: "(objectClass=groupofnames)"
      groupUIDAttribute: dn
      groupNameAttributes: [ cn ]
      groupMembershipAttributes: [ member ]
      usersQuery:
        baseDN: "cn=users,cn=accounts,dc=example,dc=com"
        scope: sub
        derefAliases: never
      userUIDAttribute: dn
      userNameAttributes: [ uid ]

```


5. **Automate with a CronJob:** The final piece of the puzzle.
Finally, create a CronJob that periodically triggers the `oc adm groups sync` command. This job spins up a temporary pod, mounts Our ConfigMap and Secret, runs the sync, and then terminates.

Create and apply this CronJob manifest:

```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: ldap-group-sync
spec:
  schedule: "* * * * *"  # Runs every minute. Adjust as needed!
  jobTemplate:
    spec:
      template:
        spec:
          serviceAccountName: ldap-group-sync
          restartPolicy: OnFailure
          containers:
          - name: ldap-group-sync
            image: registry.redhat.io/openshift4/ose-cli
            command:
            - /bin/bash
            - -c
            - |
              oc adm groups sync --sync-config=/config/group-sync.yaml --confirm
            volumeMounts:
            - name: sync-config
              mountPath: /config
            - name: bind-secret
              mountPath: /etc/secrets
          volumes:
          - name: sync-config
            configMap:
              name: ldap-group-sync-config
          - name: bind-secret
            secret:
              secretName: ldap-bind-password

```


### Verifying Setup

Once the CronJob runs, check the logs of the generated job (`oc logs job/<job-name>`). It will show a list of groups (like `group/admins` or `group/developers`) being successfully pulled into openshift cluster. Also verify this by listing the groups directly in OpenShift:

```bash
oc get groups

```

OpenShift environment now features a fully automated, production-grade Identity and Access Management integration!
