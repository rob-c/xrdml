# ROOT files used by the tests

Two small ROOT files, taken unchanged from the [go-hep](https://github.com/go-hep/hep)
project's `groot/testdata`, under the licence kept beside them. They are here
rather than generated because the only honest test of a reader is bytes
somebody else's writer produced - and what this package is checked on is that
a tree of real numbers batches into the tensors a training loop expects, and
that a column of strings is refused rather than guessed at.

The other twenty-nine files of that set are in
[xrdroot](https://github.com/rob-c/xrdroot), which is where the format itself
is read and where they earn their keep.

| File | What it is there for |
| --- | --- |
| `small-flat-tree.root` | every numeric width, fixed-size arrays and variable-length ones |
| `string-example.root` | a `std::string` standing on its own in a key |
go-hep is BSD-3-Clause; the licence is in `LICENSE.go-hep` next to these
files, and it is the whole of what is required to redistribute them.
