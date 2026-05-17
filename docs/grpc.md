# gRPC integration

The dist ships a gRPC server (`snoextract-server`) and the contract file at `proto/snoextract/v1/snoextract.proto`. Use this for **high-throughput, multi-client, or cross-language** integrations where you want strongly-typed messages.

## Start the server

```bash
# Linux / macOS
./snoextract-server                      # binds 127.0.0.1:50051

# Windows (PowerShell or cmd)
.\snoextract-server.exe                  # binds 127.0.0.1:50051
```

Loopback-only by default — no auth, no TLS.

Server flags:

| Flag | Default | Purpose |
|---|---|---|
| `--listen ADDR`      | `127.0.0.1:50051` | gRPC bind address (loopback only) |
| `--http-listen ADDR` | _(disabled)_      | REST/HTTP+JSON bind address (loopback only) |
| `--data-dir DIR`     | `./data`          | Pipeline data directory (env: `SNOMED_DATA_DIR`) |
| `--config PATH`      | _(embedded)_      | Pipeline config TOML override |
| `--log-level LVL`    | `info`            | `trace` / `debug` / `info` / `warn` / `error` |
| `--log-format FMT`   | `pretty`          | `pretty` or `json` |

## Smoke test from the bundled client

```bash
./snoextract-client health
# => SERVING

./snoextract-client extract --text "Patient with chest pain. No fever."
./snoextract-client --json extract --text "Patient with hypertension."
```

## The service contract

Package `snoextract.v1`, service `Ner`:

| RPC | Purpose |
|---|---|
| `Extract`              | Single-note entity extraction (text → entities, sections, context flags, med info) |
| `IsDescendant`         | Is CUI A a descendant of CUI B in the SNOMED IS-A hierarchy? |
| `IsDescendantBatch`    | Same, batched |
| `Parents` / `Children` | Direct IS-A neighbours of a CUI |
| `Ancestors` / `Descendants` | Transitive closure of IS-A from a CUI |

Hierarchy methods are backed by the bundled `hierarchy.bin` — no DB, no network. CUIs are decimal strings (e.g. `"73211009"`).

Full message schema lives in `proto/snoextract/v1/snoextract.proto` shipped in the dist.

## Generating clients

### Python (`grpcio-tools`)

```bash
pip install grpcio grpcio-tools
python -m grpc_tools.protoc \
    -I proto \
    --python_out=. --grpc_python_out=. \
    proto/snoextract/v1/snoextract.proto
```

### Go (`protoc-gen-go` + `protoc-gen-go-grpc`)

```bash
protoc -I proto \
    --go_out=. --go_opt=paths=source_relative \
    --go-grpc_out=. --go-grpc_opt=paths=source_relative \
    proto/snoextract/v1/snoextract.proto
```

### Node.js (`@grpc/grpc-js` + dynamic loading)

```js
const protoLoader = require('@grpc/proto-loader');
const grpc = require('@grpc/grpc-js');
const pkg = grpc.loadPackageDefinition(
  protoLoader.loadSync('proto/snoextract/v1/snoextract.proto')
);
const client = new pkg.snoextract.v1.Ner(
  '127.0.0.1:50051',
  grpc.credentials.createInsecure()
);
```

### C# (`Grpc.Tools` NuGet)

Add the `.proto` to your `.csproj`:

```xml
<ItemGroup>
  <Protobuf Include="proto/snoextract/v1/snoextract.proto" GrpcServices="Client" />
</ItemGroup>
```
