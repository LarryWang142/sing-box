# Larry Customizations

This document records the local custom changes added on top of `sing-box v1.13.13`.

## 1. TrustTunnel support

### Goal

Add `trusttunnel` inbound and outbound support to `sing-box v1.13.13`, based on the implementation from `qr243vbi/sing-box`.

### Main changes

- Added proxy type constant:
  - `constant/proxy.go`
- Added option definitions:
  - `option/trusttunnel.go`
- Added protocol implementations:
  - `protocol/trusttunnel/inbound.go`
  - `protocol/trusttunnel/outbound.go`
- Registered protocol:
  - `include/registry.go`
- Added documentation:
  - `docs/configuration/inbound/trusttunnel.md`
  - `docs/configuration/outbound/trusttunnel.md`

### Dependency changes

- Added:
  - `github.com/xchacha20-poly1305/sing-trusttunnel v0.2.3-larry.1`
- Replaced:
  - `github.com/sagernet/sing => github.com/LarryWang142/sing v0.8.11-larry.2`
- Replaced:
  - `github.com/xchacha20-poly1305/sing-trusttunnel => github.com/LarryWang142/sing-trusttunnel v0.2.3-larry.1`
- Updated:
  - `github.com/sagernet/quic-go v0.59.0-sing-box-mod.5`

### Notes

- `trusttunnel` was verified with a TCP-based inbound configuration.

### Build notes

For server builds, keep the same feature tags as the current deployment:

```text
with_gvisor,with_quic,with_dhcp,with_wireguard,with_utls,with_acme,with_clash_api,with_tailscale,with_ccm,with_ocm,with_naive_outbound,with_purego,with_cronet,badlinkname,tfogo_checklinkname0
```

The binary used in this session was built with:

- `CGO_ENABLED=0`
- `go1.25.10`

### OpenWrt/LEDE (musl) compatibility

The binary compiled with `go1.25.10` and `CGO_ENABLED=0` is still dynamically linked
against glibc (`libdl.so.2`, `libpthread.so.0`, `libc.so.6`) and requests the glibc
dynamic linker `/lib/ld-linux-aarch64.so.1`. This is a behavior of the custom `go1.25.10`
toolchain.

OpenWrt/LEDE uses **musl libc** instead of glibc. musl provides a unified `libc.so` that
contains all symbols from libdl and libpthread, and uses `/lib/ld-musl-aarch64.so.1`
as its dynamic linker.

Since the binary is built with `CGO_ENABLED=0` (no actual CGO calls into libc),
the NEEDED entries are linker artifacts — musl can satisfy them trivially.

**Fix** — create compatibility symlinks on the router:

```sh
ln -sf /lib/ld-musl-aarch64.so.1 /lib/ld-linux-aarch64.so.1
ln -sf libc.so /lib/libdl.so.2
ln -sf libc.so /lib/libpthread.so.0
ln -sf libc.so /lib/libc.so.6
```

- `ld-musl-aarch64.so.1` → `ld-linux-aarch64.so.1`: lets the kernel find a valid ELF interpreter
- `libc.so` → `libdl.so.2` / `libpthread.so.0` / `libc.so.6`: musl's unified libc satisfies all glibc NEEDED entries

> **Windows cronet note**: When the `with_cronet` tag is enabled, the Windows binary requires `libcronet.dll` at runtime. Copy it from the `cronet-go` module to the executable directory:
> ```
> copy %GOPATH%\pkg\mod\github.com\sagernet\cronet-go\lib\windows_amd64@*\libcronet.dll .
> ```
> The DLL is prebuilt and included in `github.com/sagernet/cronet-go`, no separate compilation needed.

## 2. Restore `sniff override destination` in rule action

### Background

In older configurations such as `1.12.25`, the following inbound fields were commonly used:

```json
{
  "sniff": true,
  "sniff_override_destination": true,
  "sniff_timeout": "1s"
}
```

In `1.13.13`, legacy inbound fields are removed from normal configuration use, but the runtime logic for overriding destination after sniffing still exists internally.

The missing part was the JSON configuration entry for `route.rules[].action = "sniff"`.

### Goal

Expose `override_destination` in route sniff action, so a sniffed domain can replace the original IP destination.

This is useful for cases such as Netflix on TV devices, where the client may connect directly to an IPv6 address first, preventing DNS-based routing rules from taking effect unless the sniffed domain is written back into destination metadata.

### Main changes

- Added JSON field to sniff rule action:
  - `option/rule_action.go`
- Wired the field into runtime rule action creation:
  - `route/rule/rule_action.go`
- Updated documentation:
  - `docs/configuration/route/rule_action.md`
  - `docs/configuration/route/rule_action.zh.md`

### New configuration format

Use route action instead of legacy inbound fields:

```json
{
  "route": {
    "rules": [
      {
        "inbound": "tun-in",
        "action": "sniff",
        "timeout": "1s",
        "override_destination": true
      }
    ]
  }
}
```

### Migration advice

For `tun` inbound or other inbounds that previously used:

```json
{
  "sniff": true,
  "sniff_override_destination": true,
  "sniff_timeout": "1s"
}
```

move the behavior to route rules and place the `sniff` rule before DNS hijack / resolve / route rules that depend on domain matching.
