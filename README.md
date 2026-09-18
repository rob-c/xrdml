# xrdml

Training data, from a URL to a training loop, without a download.

```python
import xrdml

data = xrdml.load("root://eos.example.org//store/mnist.root")
print(data)
# mnist.root: 70,000 rows, 10 classes
#   inputs   image: 784 x uint8, scaled to 0-1
#   answer   label: int32
#   splits   train 60,000 rows, test 10,000 rows

for images, labels in data.train.batches(256):
    loss = criterion(model(images), labels)
```

That is the whole of it. Nothing is downloaded, nothing is opened twice, and
nothing about the file's layout has to be known first: the trees say which rows
are for training and which class each of them is, the columns say which is the
picture and which the answer, and every minibatch is a read of one basket out
of the file wherever it lives.

## Install

    pip install git+https://github.com/rob-c/xrdml

That brings [`xrdroot`](https://github.com/rob-c/xrdroot) — the ROOT file
format in pure Python — and
[`xrdclient`](https://github.com/rob-c/xrdclient) underneath it, which is
where `root://`, `https://`, HEP WebDAV and `s3://` come from.

PyTorch and TensorFlow are **not** dependencies. Neither is imported until a
batch is asked for, so a file can be opened, described and read on a machine
that has no framework installed at all.

## What a batch holds

Decided the way a person would decide it. Every input column becomes one
`(rows, features)` block of `float32`, side by side if there are several; a
column of bytes is a picture, so it is divided by 255 into the nought-to-one
that networks expect; and the answer comes back as whole numbers for a
classifier or as floats for a regression, because that is what the loss
functions of each take. Pass `scale=False` to be handed the numbers exactly as
the file holds them.

## A layer down

`xrdml.tensors` is where the tensors are actually made, and where to go for
control over any of it — a tree straight into a PyTorch `DataLoader` or a
`tf.data.Dataset`, a basket at a time:

```python
import torch, xrdroot, xrdml.tensors

tree = xrdroot.open_root("root://eos.example.org//store/events.root").tree()
loader = torch.utils.data.DataLoader(
    xrdml.tensors.dataset(tree, ["Muon_pt", "Muon_eta"], step=8192),
    batch_size=None,
    num_workers=4,
)
```

## Datasets by name

A bare name — no scheme, no slash — is looked up in a catalogue:

```python
data = xrdml.load("mnist")                  # the public ScotGrid AI catalogue
data = xrdml.load("mnist", cache=True)      # pull it once, read it locally after
```

The catalogue is the `index.json` that an
[`xrddatasets`](https://github.com/rob-c/xrddatasets) site serves. Point
`XRD_CATALOGUE` at another one, or pass `config=xrdml.Config(catalogue=...)`;
`xrdml.Config(catalogue=None)` turns bare-name lookup off entirely.

Those two settings — `catalogue` and `cache_dir` — live on `xrdml.Config`,
which is `xrdclient.Config` with them added, so everything about connecting, copying
and timing out is set in the same object and means what it means in the client.

## Where this sits

    xrdclient    the XRootD protocol, files, copies, authentication
      └─ xrdroot        the ROOT file format
           └─ xrdml     this package: trees to tensors, a URL to a training loop
                └─ xrddatasets   open data converted to ROOT, and the site that serves it

## Examples

`examples/` holds the programs the [training playbooks](docs/playbooks.md)
describe — an MLP on MNIST, an autoencoder on CIFAR-10, a CNN on
Fashion-MNIST, a JARVIS crystal projection — each of them reading over
`root://` and holding none of the file.

## Tests

    pip install -e ".[dev]"
    pytest -q

PyTorch is not installed there either: the batching tests stand a small module
in for it, and everything above them uses no framework at all, which is the
point of that half of the API.

## Licence

LGPL-3.0-or-later. See [COPYING](COPYING) and [LICENSE](LICENSE).
