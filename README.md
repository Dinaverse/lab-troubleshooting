# Lab Troubleshooting

Postmortems from running a multi-node homelab: distributed systems, security, and networking. Each entry is a real incident — symptom, what was ruled out, confirmed root cause, and the fix.

## Index

### Kernel / Storage
- [ ] TSC firmware bug causing deterministic JIT crashes (SIGILL at a fixed instruction pointer, masked by `tsc=reliable`)
- [ ] Diagnosing a wedged/hung-task storage issue (D-state kernel threads, NFS client wedge, USB re-enumeration)
- [ ] Self-healing USB drive remount via udev (UUID-keyed auto-remount + dependent-container restart)

### Networking
- [ ] MTU/PMTUD blackhole on direct-peer overlay network connections
- [ ] Proxy-DHCP + TFTP PXE boot behind a consumer router
- [ ] Stale route shadowing a working interface by metric
- [ ] False-positive route verification across multiple overlay networks

### Virtualization / Containers / Security
- [ ] TUN device passthrough for VPN daemons in unprivileged LXC containers
- [ ] Dropping a systemd service from root to least-privilege (capabilities vs. AmbientCapabilities)
- [ ] Per-process VPN isolation via network namespaces (P2P traffic only, zero host-wide tunnel)

### Desktop / Devices
- [ ] NVIDIA/i3 desktop stability — ruling out false leads before finding the real fix
- [ ] Remote-controlling a broken-touchscreen iOS 17+ device

Each entry gets its own file once written up. Status: scaffolding in progress.
