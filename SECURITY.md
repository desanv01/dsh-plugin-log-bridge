# Security

## Reporting a vulnerability

Open a private security advisory through this repository's **Security** tab, or contact the maintainer through
the GitHub profile `desanv01`. Please do not open a public issue for anything that could be exploited.

Include the version, the harness version, reproduction steps, and what an attacker gains.

## What this plugin does with data

| Property | Value |
| --- | --- |
| Network access | **none** |
| Reads credentials | **no** |
| Writes files | **no** |
| Writes to the process | **stdout** (one ready line per launch) and **stderr** (bridged records) |

## The one thing to be careful with

This plugin copies harness log output — error and warning records — to **stderr**. That output is whatever the
harness or another plugin chose to log: it can contain prompt text, file paths, tool arguments, error messages
and stack traces. **The plugin does not redact anything.**

Consequences worth stating plainly:

- Treat captured stderr as sensitive as the session logs themselves.
- Do not paste it into a public issue without reading it.
- A supervisor that rotates or protects its child's stderr is protecting this data too.

## Rate limiting

100 messages per 60-second window, then a single line reporting how many were dropped. This bounds the volume
a misbehaving plugin can inject into the log; it is not a security control, and it does not sanitise content.

## Supported versions

The current major version. Security fixes are released as patch or minor versions.