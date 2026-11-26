# mytonprovider-contracts

Shared gRPC/protobuf definitions for mytonprovider agent and clients.

Module path: `github.com/mytonprovider/mytonprovider-agent/contracts`

## Generate

From repo root:

```bash
task proto:gen
```

## Release tag

After proto changes:

```bash
git tag contracts/v0.3.0
git push origin contracts/v0.3.0
```

In client modules (coordinator in mytonprovider-backend, mytonstorage-backend):

```bash
go get github.com/mytonprovider/mytonprovider-agent/contracts@v0.3.0
```

**Note:** use `@v0.3.0`, not `@contracts/v0.3.0`.

The agent builds against this directory via `replace` in its `go.mod` (`=> ./contracts`).
