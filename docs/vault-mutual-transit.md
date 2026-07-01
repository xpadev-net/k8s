# Vault mutual transit runbook

This repository prepares the osk and tky Vault installs for HA Raft plus mutual transit auto-unseal.

## Target topology

- osk Vault:
  - HA Raft replicas: 3
  - active API endpoint: `http://10.154.11.20:8200`
  - unsealed by the transit engine on tky
  - transit key expected on tky: `osk-auto-unseal`
- tky Vault:
  - HA Raft replicas: 3
  - active API endpoint: `http://10.153.11.20:8200`
  - unsealed by the transit engine on osk
  - transit key expected on osk: `tky-auto-unseal`

## Secret contract

Each cluster must have a local Secret in the `vault` namespace before the Vault Application syncs the transit seal configuration:

```bash
kubectl -n vault create secret generic vault-transit-seal \
  --from-literal=token='<peer-transit-token>'
```

The token must be issued by the peer Vault and must allow:

```hcl
path "transit/encrypt/<key-name>" {
  capabilities = ["update"]
}

path "transit/decrypt/<key-name>" {
  capabilities = ["update"]
}
```

Use orphan, renewable/periodic tokens where possible. Do not commit these tokens to Git.

## Bootstrap order

1. Ensure osk and tky have routable access to each other's active Vault endpoint.
2. Create or confirm DNS/routes for `10.154.11.20` and `10.153.11.20` if clients should use names instead of IPs.
3. On the current manually unsealed Vault, enable transit and create the peer key.
4. Create the peer token with encrypt/decrypt access to that key.
5. Store the peer token as `vault/vault-transit-seal` in the opposite cluster.
6. Repeat steps 3-5 in the other direction.
7. Let Argo CD auto-sync one Vault Application at a time when possible.
8. Initialize and unseal any Raft cluster that comes up as a fresh Vault.
9. Confirm `vault status`, Raft peers, and Argo CD health before relying on the second site.

## Current bootstrap state

- osk Vault has been initialized and manually unsealed.
- osk Vault has `transit/` enabled.
- osk Vault has transit key `tky-auto-unseal`.
- osk Vault has policy `tky-auto-unseal`.
- The generated tky-side transit token from osk is stored locally at `/private/tmp/tky-vault-transit-token-from-osk-20260701170701.txt`.

The token file contains secret material and must be moved to the tky cluster as `vault/vault-transit-seal` before tky Vault can use osk for auto-unseal.

## Break-glass recovery

Mutual transit is allowed to depend on manual recovery during a full dual-site outage.

1. Pick the break-glass first site. Prefer osk unless the incident dictates otherwise.
2. Manually unseal that site's Vault with Shamir keys.
3. Confirm its transit engine and peer auto-unseal key are available.
4. Start or restart the other site's Vault so it can auto-unseal from the recovered transit endpoint.
5. Confirm both sites are unsealed and serving.

The Shamir unseal keys for both sites remain mandatory operational material. Store them outside the clusters.

## Migration note

The initial osk Vault was installed as standalone file storage and currently has no required data. Switching it to HA Raft plus transit seal can replace that initial state. Keep the existing init output available until the replacement cluster is validated.
