# Confidential Bash workspace configuration

This repository contains the configuration for one owner's confidential Bash
computer. Its public SSH key was prepared by the protected calendar operator
after verifying the existing account. The configuration has not yet been
measured, validated by the Tinfoil control plane, provisioned, or tested inside a
confidential VM. The private owner key and provider credentials stay on the
trusted calendar host. The calendar application and its private source are not included.

## Scope and pinned inputs

The configuration uses the official `confidential-ubuntu` v0.6.5 image approved
by the signed vendor release at commit
`24c3d5ea361b9111d5c860291e8184989bb1f0b6`, pinned by image digest:

`sha256:3c96130f0c586e5f4b301111624f7dbd4dbb942894611679518fb165496ed917`

The public build ran from its parent commit
`09dd691155427267fda932a8b31ef79d2f64b37f`; only the configuration image pin
changed afterward. The Dockerfile and bootstrap are identical across those
commits. Signed release metadata authenticates the vendor's approval of this
digest; signed OCI build provenance tying the image to those source files has
not been verified.

The measured-release action is pinned to
`tinfoilsh/measure-image-action@cb61fb5c942ae2d99a9316ada725122bd52faccd`.
It measures the configuration and referenced image, signs the deployment
manifest, and publishes a GitHub release. It creates no Tinfoil instance.

This candidate requests the documented minimum CPU shape: 2 CPUs and 8192 MB
RAM. It has no GPU, model pack, persistent volume, provider secret, direct
inbound port, or debug mode. `/workspace` and `/tmp` are bounded tmpfs mounts.
Ordinary filesystem state is ephemeral and must be expected to disappear on
container recreation or CVM reboot. A confirmed stop is a lifecycle assertion,
not a proof of physical memory erasure. The private executor closes task stdin;
this configuration disables interactive terminals and SSH forwarding.

The pinned CVM 0.14.8 also stores ordinary container writable layers and image
content in protected guest RAM under `/mnt/ramdisk/private`. A container restart
can retain its writable layer; recreation or CVM reboot loses it. The 8 GiB
guest's image ramdisk has a 4 GiB limit. All six pinned image layers were streamed
and their compressed sizes and SHA256 hashes verified: expanded archives total
331.8 MiB. An illustrative estimate including retained compressed blobs,
4 KiB-rounded file allocation, and 16 KiB metadata per archive entry is 689 MiB.
That supports image-size feasibility, not successful boot or account placement.
During live acceptance, verify actual filesystem backing and an empty
`/proc/swaps`; this draft does not claim swap support is compiled out.

The pinned image is an administrator workspace. `cvm_admin: true` grants root
authority over the entire confidential VM, including any colocated workload.
The candidate must be dedicated to one owner with no other workload or provider
credential. The named network has closed egress. That is configured policy,
not a proven isolation boundary against privileged code administering the VM.
There is no sandbox within this personal administrator machine.

Measured startup replaces the image's SSH server configuration. There is no
Include, alternate authorized-key file, authorized-key command, trusted CA,
password authentication, user RC, or forwarding path. Only root may log in,
using the sole public key in measured `SSH_KEYS`, installed by the original
bootstrap into `/run/authorized_keys`. That bootstrap generates host keys,
clears `SSH_KEYS`, and starts systemd. UsePAM retains the pinned image's public
key login behavior; password and keyboard-interactive authentication remain off.

## Publication and release boundary

Tinfoil requires a public configuration repository. This separate repository
contains only configuration, its measured-release workflow, and documentation.
The calendar repository remains private. Do not copy its application,
credentials, database, conversation data, account email, or private owner key.

Generate a dedicated owner Ed25519 key on the trusted operator/application host.
Keep its private half service-owned with mode 0600. Replace the placeholder with
exactly one `ssh-ed25519 BASE64` public key, without a comment or line break.
Review that public-key binding before publication; it identifies who can log in
to this workload and is not a secret. Never configure `SSH_KEYS` through saved
per-instance variables. Never send an SSH private key to the workload.

When the dedicated owner public key is available and its binding has been
reviewed, create the immutable release tag, then deliberately dispatch
`measure-release.yml` on that tag. The workflow refuses a branch ref,
the unreplaced placeholder, or a malformed public key. Its result is a measured
release, not a running machine.

This single-workflow dispatch on a tag matches the pinned publication action's
preconditions. CLI 0.19.0's pinned verifier accepts any single workflow filename
directly under the expected repository's `.github/workflows` on a tag, with a
GitHub-hosted runner. It does not require the template's two workflow names.
The control-plane validator's separate workflow checks have not been tested;
successful account validation remains required before creating an instance.

Download that release's `tinfoil-deployment.json` and `tinfoil.hash`. Verify the
manifest SHA256 equals the hash and verify the GitHub/Sigstore attestation
against the new repository and release workflow. Record a complete policy:

`OWNER/CONFIG@APPROVED_TAG@sha256:NEW_DEPLOYMENT_MANIFEST_DIGEST`

The new manifest digest is not the vendor image digest and is not the vendor
release's manifest digest. Refuse a repo-only or tag-only execution policy.

## Separate lifecycle and command credentials

Pin the local official CLI to version 0.19.0. Native sandbox and ordinary
Container transports in that source read an enclave bearer only from explicit
`--api-key` or `TINFOIL_TUNNEL_API_KEY`; there is no saved/admin fallback.
Nevertheless use separate environments for management and command execution.

Read status/list and request lifecycle changes only on the trusted host with
the separate organization admin credential. Keep it in the existing encrypted
systemd credential mechanism. Do not include it in measured configuration,
container secrets, saved variables, SSH arguments, model prompts, or output.

Before execution, validate the returned container UUID, logical name, exact
repository/tag, non-debug confidential mode, expected CPU shape, and empty
provider-secret/variable bindings under the existing per-machine ownership
lease. Obtain its enclave hostname from that validated record. Do not accept a
hostname or repository policy supplied by the model.

Execute against the validated bare enclave hostname with the full policy and
dedicated SSH identity. Provide a credential-free CLI configuration file and
an allowlisted environment containing no `TINFOIL_ADMIN_KEY`, `TINFOIL_API_KEY`,
`TINFOIL_TUNNEL_API_KEY`, or SSH agent. Do not pass `--api-key` or `--allow-debug`.
The bare hostname avoids a second management lookup inside the execution CLI.

The command shape, with placeholders requiring trusted operator substitution,
is:

```text
tinfoil ssh VALIDATED_ENCLAVE_HOSTNAME --port 22 --user root \
  --repo OWNER/CONFIG@APPROVED_TAG@sha256:NEW_DEPLOYMENT_MANIFEST_DIGEST -- \
  -F /dev/null -T -i TRUSTED_OWNER_PRIVATE_KEY -o IdentitiesOnly=yes \
  -o BatchMode=yes -o IdentityAgent=none -o ForwardAgent=no \
  -o ClearAllForwardings=yes -o ForwardX11=no -o PermitLocalCommand=no \
  -o SendEnv=-* REMOTE_WRAPPER
```

`--port 22` refers to the enclave side of the attested TLS CONNECT tunnel;
do not substitute an externally allocated direct SSH port. The explicit owner
identity authenticates login. Fresh attestation and TLS-key pinning authenticate
the expected workload on every new tunnel connection. `shim.authenticated:false`
requires no inference/tunnel API key; SSH public-key authentication is mandatory.

## Private integration and pending acceptance

The private calendar application contains an ordinary-Container execution
backend and a protected operator setup path. Their source is not included here.
Activation still requires the measured owner-key release, successful account
validation and live acceptance. Ordinary Container lifecycle differs from native
`/api/sandboxes`: the protected operator creates and deploys the machine, while
the runtime executes commands and stops an already owned machine. Equivalent
trusted CLI operations are:

| Method | Trusted CLI operation |
| --- | --- |
| create | `container create NAME --repo OWNER/CONFIG --tag TAG --mark-latest=false --secret= --output json` |
| start | `container deploy UUID --tag TAG --mark-latest=false --secret= --output json` |
| stop | `container stop UUID --output json`, then poll actual state |
| status | `container get UUID --output json` |
| list | `container list --output json`, validating the owned record |
| execute | Credential-free `ssh VALIDATED_ENCLAVE_HOSTNAME` as described above |

Container JSON uses UUID `id`, logical `name`, and `status`; store the UUID and
verify identity on every response. Preserve logical names for callers. Treat
failed or unknown lifecycle states explicitly; never infer Running from an
accepted create/deploy response. A stop is confirmed only by an actual Stopped
record. Failed creation or uncertain transport must retain a recoverable record
and attempt whole-instance stop without deleting the instance.

The private executor uses NUL-framed script input, a separate script descriptor,
closed task stdin, sanitized remote environment, the 256 KiB public output budget,
timeout limits, completion framing, and serialized tenant ownership. The remote environment
uses `/root` as HOME and `/workspace` as working directory. The completion frame
is diagnostic, not an authentication boundary against this privileged owner.

Before resource creation, use Tinfoil's non-provisioning config-validation request for
the approved repo/tag and intended instance name. Confirm CPU entitlement and
allocation independently; an empty `container hosts` list only reports that no
hosts were surfaced and does not prove default CPU placement unavailable.

After approved source publication and successful account validation, use a fresh synthetic name and verify:
attestation pin success and wrong-pin rejection; owner-only SSH authentication;
strict effective SSH server settings; ordinary Bash quoting/stdin/nonzero exit;
output bounds; absent forwarded credentials; correct workspace tmpfs; and
timeout followed by confirmed whole-instance Stopped status. On interruption
or uncertain cleanup, require confirmed whole-instance stop. This verifies
reported lifecycle state; detached-child survival and physical memory erasure
are not independently tested.
Do not send user data before those checks pass. Local fixtures have tested the
private integration, but these acceptance checks have not run against a live
Tinfoil machine.

Bash commands and results still pass through the trusted calendar host, selected
inference provider, and existing conversation persistence. A confidential Bash
machine alone does not make that entire application end-to-end private.

## Primary sources

- [Pinned vendor configuration](https://github.com/tinfoilsh/confidential-ubuntu/blob/24c3d5ea361b9111d5c860291e8184989bb1f0b6/tinfoil-config.yml)
- [Pinned vendor bootstrap](https://github.com/tinfoilsh/confidential-ubuntu/blob/24c3d5ea361b9111d5c860291e8184989bb1f0b6/entrypoint.sh)
- [Pinned release action](https://github.com/tinfoilsh/measure-image-action/blob/cb61fb5c942ae2d99a9316ada725122bd52faccd/action.yaml)
- [Pinned CLI tunnel credentials](https://github.com/tinfoilsh/tinfoil-cli/blob/025a7c879ad806c9d7309940f1467f3db21cce24/tunnel.go)
- [Pinned CVM RAM storage policy](https://github.com/tinfoilsh/cvmimage/blob/v0.14.8/docs/runtime-policy.md#cvm-administrator-containers)
- [Pinned SDK workflow identity policy](https://github.com/tinfoilsh/tinfoil-go/blob/8583ac141f6454936fa611056edf1386dc01d178/verifier/provenance/provenance.go)
- [Configuration schema](https://docs.tinfoil.sh/containers/configuration)
- [Networking and administrator scope](https://docs.tinfoil.sh/containers/config-networking)
- [Managing instances](https://docs.tinfoil.sh/containers/cli)
