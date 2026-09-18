# Into PyTorch and TensorFlow

A tree read with [`xrdroot`](https://github.com/rob-c/xrdroot) comes out a
basket at a time, as arrays of whatever the columns hold.
`xrdml.tensors` turns those batches into tensors. Neither framework is imported
until it is called, so nothing here costs anything on a machine with neither:

```python
import torch, xrdroot, xrdml.tensors

tree = xrdroot.open_root("root://eos.example.org//store/events.root").tree()
loader = torch.utils.data.DataLoader(
    xrdml.tensors.dataset(tree, ["Muon_pt", "Muon_eta"], step=8192),
    batch_size=None,  # each item is already a batch
    num_workers=4,  # each worker reads its own share of the entries
)

for batch in loader:
    model(batch["Muon_pt"])
```

`batch_size=None` is the point: one basket read serves thousands of rows, so
the batching belongs here and not in the loader. Several workers split the
entry range between them, so each reads a different part of the file rather
than all of them reading all of it.

A worker is a process of its own, and it opens the file again for itself
rather than reading down the handle it inherited — two processes seeking one
descriptor read over each other's shoulders, and two sharing one XRootD
session read each other's replies. Nothing has to be arranged for that: the
first read in a child notices whose process it is in, dials its own
connection and carries on. A tree opened from a file object cannot do this and
says so, because a handle somebody else opened cannot be reopened by name.

The sets in `xrdroot.datasets` keep one tree per class, which is the wrong
order to learn in: a loop over those trees in turn shows a model five thousand
cats and then five thousand dogs. `mixed` reads `step` entries from each tree,
shuffles that pool together and cuts `batch` rows off it at a time, so every
minibatch holds every class:

```python
with xrdroot.open_root("root://127.0.0.1:21094//cifar10.root") as handle:
    trees = [handle[name] for name in handle.trees() if name.startswith("train_")]
    loader = torch.utils.data.DataLoader(
        xrdml.tensors.mixed(trees, ["image", "label"], step=1024, batch=256, device="cuda"),
        batch_size=None,
    )
```

`step` is what the memory costs — a pool of that many rows from each tree, and
nothing else of the file. The shuffling is PyTorch's own, so `torch.manual_seed`
settles it; `shuffle=False` is for the pass that scores a model, where the
order makes no difference and the rows can be watched going by. Three worked
examples are in [Training playbooks](playbooks.md).

Fixed-size array columns arrive shaped `(entries, width)`. Variable ones are
padded to the widest row in the batch, or to a width you give:

```python
xrdml.tensors.to_tensor(jets, width=4, fill=0.0)  # (entries, 4)
xrdml.tensors.iter_tensors(tree, step=8192, device="cuda")
```

Unsigned columns wider than a byte are widened into the signed type that holds
them, because neither framework's unsigned types beyond `uint8` are supported
well enough to hand anybody; a value too large for `int64` is refused by name
rather than wrapped around.

TensorFlow takes the same tree through `tf_dataset`, which declares its shapes
and types up front — from what the file says the columns are, without a first
pass over the data — so the rest of `tf.data` works on it:

```python
data = xrdml.tensors.tf_dataset(tree, ["Muon_pt"], step=8192).prefetch(2)
for batch in data:
    model(batch["Muon_pt"])
```

With no column names, either dataset takes every numeric column and leaves the
strings and objects out, rather than leaving them in to fail later.


## Training off a `root://` URL

Converted, a dataset is a ROOT file like any other, so a training loop reads it
the way it would read events. If there is no storage element to hand, share the
directory over `root://` yourself — an unprivileged port, no login, no daemon:

```console
$ python -m xrdclient.testing datasets --port 21094 --pattern '*.root'
serving 5 files on root://127.0.0.1:21094/ with no login
```

Then the ten class trees are one stream, mixed a batch at a time. Nothing is
downloaded first; each pull is one basket off the wire:

```python
import torch
from xrdroot import open_root
from xrdml.tensors import mixed

URL = "root://127.0.0.1:21094//home/you/datasets/mnist.root"

with open_root(URL) as handle:
    trees = [handle[f"train_{cls}"] for cls in range(10)]
    loader = torch.utils.data.DataLoader(
        mixed(trees, ["image", "label"], step=512, batch=256, device="cuda"), batch_size=None
    )
    for batch in loader:  # 60,000 rows an epoch
        train(batch["image"].float().view(-1, 1, 28, 28) / 255, batch["label"].long())
```

Three programs that run as they stand — an MLP, an autoencoder and a small
convolutional net — are in [Training playbooks](playbooks.md).

A regression set is simpler still, being one tree of every row, with the number
to predict a column beside the rest:

```python
with open_root("root://127.0.0.1:21094//victorian_electricity.root") as handle:
    loader = torch.utils.data.DataLoader(
        dataset(handle["rows"], ["temperature", "holiday", "time", "demand"], step=4096),
        batch_size=None,
    )
```

