# Orka transit speed test

Measures download throughput from an Orka cluster to GitHub Actions artifact
storage, using an artifact we own so no coordination with anyone else is needed.

GitHub serves repo files and release assets from a CDN that terminates locally,
but Actions artifacts live in Azure Blob storage in whichever region GitHub
assigned. Those are different paths with very different performance, so a file
committed to a repo is useless as a test object. The artifact has to be a real
Actions artifact.

## Setup

Run the workflow once from the Actions tab, then note the artifact id from the
run. The artifact is durable for its retention window; only the signed download
URL is short-lived, so there is no need to run the workflow again.

## Mint a fresh signed URL

    curl -sI -H "Authorization: Bearer $(gh auth token)" \
      https://api.github.com/repos/OWNER/REPO/actions/artifacts/ARTIFACT_ID/zip \
      | grep -i '^location:'

The Location header is a signed Azure Blob URL, valid for a few minutes. Treat it
as a bearer credential while it lives.

## Check which Azure region the artifact landed in

Take the hostname out of the signed URL, resolve it, and look the address up in
Microsoft's published service tags file (ServiceTags_Public, refreshed weekly):

    python3 - <<'PY'
    import json, ipaddress, sys
    d = json.load(open("ServiceTags_Public.json"))
    ip = ipaddress.ip_address(sys.argv[1] if len(sys.argv) > 1 else input("ip: "))
    for v in d["values"]:
        for p in v["properties"].get("addressPrefixes") or []:
            try: net = ipaddress.ip_network(p)
            except ValueError: continue
            if ip.version == net.version and ip in net:
                print(v["name"], v["properties"].get("region") or "-", p)
    PY

Region assignment may vary by account, which is worth knowing before concluding
that distance is unavoidable for any given org.

## Probe from a cluster

Time-boxed so each call fits inside host_exec / vm_exec's 30s limit. Nothing is
written to disk; the bytes are received and discarded.

    curl -s --max-time 10 -o /dev/null \
      -w 'single %{speed_download} B/s connect=%{time_connect}\n' "$URL"

Eight parallel streams, to separate a per-connection limit from a path limit:

    ( for i in $(seq 8); do
        curl -s --max-time 10 -o /dev/null -w '%{speed_download}\n' "$URL" &
      done; wait ) | awk '{s+=$1} END {printf "multi8 %.0f B/s\n", s}'

Take five samples and report the median; single samples vary by more than the
effect being measured. Run the sources concurrently so they see the same network
conditions. Pair it with a European endpoint as a control, and with the firewall
interface and discard counters over the same window.
