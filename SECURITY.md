# Security

## Reporting a vulnerability

Mail <robert.andrew.currie@gmail.com> with a description and, if you have one,
a reproducer. Please do not open a public issue for anything that lets one
party read or write another party's data. Expect an acknowledgement within a
few working days.

## What this package is trusted with

It turns a file somebody else published into the tensors a training loop is
fed. Two things arrive from outside, and both are treated as untrusted: the
ROOT file, whose parsing belongs to
[xrdroot](https://github.com/rob-c/xrdroot), and the catalogue, which is a
JSON index on a web server. Credentials, TLS and the transport are
[PyXRootDClient](https://github.com/rob-c/xrd)'s, and its
[SECURITY.md](https://github.com/rob-c/xrd/blob/main/SECURITY.md) is the
document for those.

## What the implementation guarantees

**A catalogue names files; it never names code.** An index entry supplies a
file name, a length and a checksum. The name is joined onto the catalogue's own
base URL, so an entry cannot redirect a lookup somewhere else, and nothing in
an entry is imported, evaluated or executed.

**A bare name never shadows a file that exists.** Anything that could plausibly
be a place — a scheme, a slash, a `.root` suffix, or a path that is actually
there — is treated as one, so a catalogue cannot take over a URL or a local
file you meant.

**A pulled file is verified before it is usable.** `download` writes to a
`.part` beside the target and renames it into place only once the length and
the catalogue's checksum agree, so an interrupted or corrupted transfer leaves
nothing a later run would mistake for the dataset. A copy that disagrees is
deleted rather than kept.

**A framework is imported only when a batch is asked for.** Nothing here
imports PyTorch or TensorFlow to describe, open or read a file, so a machine
with neither runs the whole of the non-batching API.
