# SSH Access via One-Time Authorization Codes

The use of a user's access token as the password for SSH access to Diego containers is problematic for two reasons:

1. A coincidental one: These access tokens are frequently longer than 1KB in length, but the password buffer on the OpenSSH client is hard-coded to 1KB. The token is then truncated (or not even sent), and authentication fails.
2. A security one: These access tokens are issued for the 'cf' UAA client, but are being used by a service (the proxy) that is not that client.

We therefore propose an entirely different authentication flow for CF app instances, similar to that presented in [this proposal](https://docs.google.com/document/d/1DTLNW-0twYnIHs9z7OE0v5BDNQhgNIO9kZ0fyqWiMo0/edit):

- The Diego SSH Proxy is registered as a UAA client with a specific name (say, 'ssh-proxy').
- The end user obtains from UAA a one-time authorization code issued for that SSH Proxy client and sends that as the SSH password.
- The Diego SSH Proxy then sends a request to UAA as that client to exchange the one-time code for an access token, and uses that token to authorize the user's access to the CF app instance.

This flow should be the only supported one for the CF authenticator, and we should remove the access-token-as-SSH-password option that is currently implemented in the SSH proxy. Fortunately, the UAA can now issue such tokens as of [story #102931196](https://www.pivotaltracker.com/story/show/102931196), but this work must be included in the next UAA release, which then must be updated in cf-release.

Implementing this flow and removing the old one requires the following stories:

---

The Diego SSH Proxy can receive an authorization code as the SSH password to access a CF app instance


This requires the proxy to be configured with a client name and secret to present to UAA along with the token. Spiff-based generation of the Diego BOSH manifest should be updated to retrieve this secret from the UAA client configuration in the CF manifest.


Acceptance:

- I can follow documentation in the diego-ssh README to obtain a one-time authorization code from UAA for the SSH proxy's client.
- I can then use the code as the password to establish an SSH connection to a CF app instance for which I am authorized access.
- The existing access-token-as-password behavior should continue to work for now (until we update the SSH plugin to use the new flow).

L: ssh


---

CC presents the Diego SSH Proxy client name in the /v2/info endpoint

- The endpoint response should present this name in its `app_ssh_oauth_client` field.
- This name should be BOSH configurable in the CC configuration.

L: ssh


---

The SSH plugin establishes SSH connections to CF app instances by sending an authorization code as the SSH password

- For each connection attempt, the plugin retrieves a new one-time authorization code for the SSH proxy client and uses it as the password.
- The plugin should not have the SSH proxy UAA client name hardcoded, but should instead read it from the CC `/v2/info` endpoint.
- Release version 0.2.0 of the plugin and submit to the [Community Plugin Repo](https://github.com/cloudfoundry-incubator/cli-plugin-repo).

L: ssh


---

The SSH plugin provides a command to print a one-time authorization code issued for the SSH proxy client

Acceptance:

- I can run `cf get-ssh-code` and use the output as my password for connection with the OpenSSH `ssh` and `scp` clients.
- This command should also look up client info from CC's `/v2/info` endpoint.

L: ssh


---

The Diego SSH Proxy no longer accepts a user's access token as an SSH password for CF app instances

Acceptance:

- Version 0.1.x of the SSH plugin no longer allows connections.


L: ssh

---

