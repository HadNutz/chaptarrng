# Remote path mapping issue #199 — fork proof

Issue: https://github.com/snapetech-org/chaptarrng/issues/199

The reported published image (0.9.977.0) accepted POST with HTTP 201 but
returned mapping ID 0, GET always returned empty results, and POST /test
returned default fields. A fresh isolated container reproduced it. This is
distinct from qBittorrent connectivity and Docker volume permissions.

This branch first tests the **unmodified fork source** as a baseline. It
does not assume that edits to RemotePathMappingService will fix a possible
release binary mismatch.

The GitHub Actions workflow builds an ephemeral image using Dockerfile.build
and verifies create/test/read/restart/delete against a real HTTP listener,
along with a SQLite row check. It uses a synthetic Windows UNC path, its
own throwaway /config and /downloads, and no production API keys.

To run locally from repository root:

    docker build -f Dockerfile.build -t chaptarrng:fork-rpm199 .
    python3 integration/test_remote_path_mapping_image.py --image chaptarrng:fork-rpm199

If source-built image passes but upstream published image fails, compare
published DLL hashes, image digest and build provenance. If both fail,
inspect route discovery, runtime service resolution and middleware first.

The proof does **not** establish an actual qBittorrent import end-to-end.
That must follow after this HTTP/persistence defect is resolved.

## Diagnosed root cause (fork Docker run)

The image built from unchanged source reproduced the published-image failure.
Opt-in controller diagnostics showed that DryIoc was injecting
`NzbDrone.Core.Download.Clients.Direct.DirectDownloadClient+NoopRemotePathMappingService`
instead of `NzbDrone.Core.RemotePathMappings.RemotePathMappingService`.
The nested private Noop service in `DirectDownloadClientItems.cs` implements the
same interface and is inadvertently picked up by assembly-wide `RegisterMany`.
That implementation intentionally returns an empty list, ID zero and a default
mapping-test object. This exactly explains all three user-visible symptoms.

The targeted correction registers the real service after `AutoAddServices` in
both console-host and utility-host paths, replacing the accidental no-op
registration.

## Verified fork image regression

The source-built pre-fix image reproduced the no-op failure in GitHub Actions
run 42422076052. An opt-in diagnostic run (42422478434) revealed the actual
resolved service type. Patched source at commit 6c7bb175985e9e9a4b80de3aa2c828d0c2c24a83
then passed the full Docker HTTP and SQLite lifecycle in run 42422924087:
request mapping, created ID 1, GET list/by-ID, actual SQLite row, restart and
DELETE. Temporary controller logging was removed after diagnosis.

The automated workflow now publishes only after that same live regression
succeeds, into the fork-owned `ghcr.io/hadnutz/chaptarrng` namespace, under
`remote-path-199` and a commit-specific tag. Production import from a
real Windows qBittorrent download client is a separate acceptance gate.
