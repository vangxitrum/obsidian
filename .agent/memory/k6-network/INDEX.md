# k6-network — memory index

- [[scaffold-decisions]] — scaffold-only pass details: placeholders used, ufw chosen for firewall role, what's left before a real run.
- [[hls-load-test]] — HLS/m3u8 viewer-sim load test (hls-load-test.js + playlist.m3u8): parse manifest, pull segments, swap playlist file/URL when tickets expire; verified live via dockerized k6.
- [[vxlan-bidirectional-8472]] — TestRun stuck at STAGE=created / ContainerCreating was a one-way firewall: flannel VXLAN 8472/udp must be open BOTH directions (ufw + DO cloud firewall); :6565 is pod-net only.
