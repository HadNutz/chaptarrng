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
