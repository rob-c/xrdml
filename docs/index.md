# xrdml

Training data, from a URL to a training loop, without a download.

```python
import xrdml

data = xrdml.load("root://eos.example.org//store/mnist.root")
for images, labels in data.train.batches(256):
    loss = criterion(model(images), labels)
```

Nothing is downloaded and nothing about the file's layout has to be known
first. Start with [Machine learning](ml.md) for the whole of that surface,
[Into PyTorch and TensorFlow](tensors.md) for the layer below it, and the
[Training playbooks](playbooks.md) for programs that run.

## The stack

| Package | What it is |
| --- | --- |
| [`xrdclient`](https://github.com/rob-c/xrdclient) | the XRootD protocol, files, copies, authentication |
| [`xrdroot`](https://github.com/rob-c/xrdroot) | the ROOT file format |
| `xrdml` | this package: trees to tensors, a URL to a training loop |
| [`xrddatasets`](https://github.com/rob-c/xrddatasets) | open data converted to ROOT, and the site that serves it |
